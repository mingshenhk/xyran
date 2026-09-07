# Xyran

**Local-first image and animation content moderation for Python.**

[![PyPI](https://img.shields.io/pypi/v/xyran?label=PyPI)](https://pypi.org/project/xyran/)
[![Python](https://img.shields.io/pypi/pyversions/xyran)](https://pypi.org/project/xyran/)
[![License](https://img.shields.io/pypi/l/xyran)](https://github.com/mingshenhk/xyran/blob/main/LICENSE)
[![Hugging Face](https://img.shields.io/badge/Hugging%20Face-xyran--image--safety-yellow)](https://huggingface.co/MingSafeR/xyran-image-safety)

Xyran is a free, offline-capable content-safety SDK for Python. It scans static
images, animated GIF/WebP files, directories, and large datasets locally with a
bundled ONNX classifier.

- **No Xyran account or API key**
- **No cloud moderation request**
- **No model download at inference time**
- **Python API + CLI**
- **CPU or NVIDIA CUDA ONNX Runtime**
- **True ONNX batching**
- **Animated GIF/WebP frame-aware moderation**
- **Folder Scan for one-shot directory jobs**
- **Dataset Mode for resumable large-scale cleaning**

> Xyran performs probabilistic classification. It can produce false positives
> and false negatives. Calibrate thresholds on representative data before using
> automated results for high-impact decisions.

## Links

- **GitHub:** https://github.com/mingshenhk/xyran
- **PyPI:** https://pypi.org/project/xyran/
- **Hugging Face model page:** https://huggingface.co/MingSafeR/xyran-image-safety
- **Issues:** https://github.com/mingshenhk/xyran/issues

## Install

```bash
pip install xyran
```

Then scan an image:

```bash
xyran scan image.jpg
```

Or use Python:

```python
from xyran import Moderator

mod = Moderator()
result = mod.scan("image.jpg")

print(result.decision)        # ALLOW / REVIEW / BLOCK
print(result.scores.sexual)
print(result.scores.graphic)
print(result.scores.safe)
```

The bundled model is included in the published Xyran wheel/sdist. A fresh
installation with no ONNX Runtime can let Xyran choose CPU/GPU Runtime on first
inference, or you can prepare it explicitly with `xyran setup-runtime`.

---

# Xyran 1.3

Xyran 1.3 adds two major free features:

1. **Dataset Mode** — a dedicated workflow for large, resumable dataset-cleaning
   jobs with SQLite state/cache, JSONL/CSV manifests, filtering, sharding,
   slicing, checkpoints, and configurable fingerprints.
2. **Hardware-aware runtime setup** — Xyran detects supported NVIDIA hardware
   and selects GPU ONNX Runtime only when appropriate; CPU-only systems select
   the CPU runtime.

Folder Scan remains a separate one-shot workflow. Dataset Mode is not an alias
for Folder Scan.

---

## Automatic CPU/GPU runtime setup

Standard Python package dependency metadata cannot execute `nvidia-smi` while
pip resolves dependencies, so Xyran performs hardware selection at the Xyran
runtime layer instead of using an architecture-only dependency guess.

On a fresh environment with no ONNX Runtime installed:

```text
first Xyran inference / xyran setup-runtime
                |
                +-- supported NVIDIA GPU detected
                |       -> onnxruntime-gpu[cuda,cudnn]
                |
                +-- no supported NVIDIA GPU detected
                        -> onnxruntime
```

Preview what Xyran would choose without changing the environment:

```bash
xyran setup-runtime --dry-run
```

Install the selected runtime now:

```bash
xyran setup-runtime
```

Force a runtime when you want deterministic CI/container setup:

```bash
pip install "xyran[cpu]"
pip install "xyran[gpu]"
```

or:

```bash
xyran setup-runtime --runtime cpu
xyran setup-runtime --runtime gpu
```

Xyran does **not** silently replace an already installed ONNX Runtime variant.
If an environment contains overlapping or damaged ORT distributions, use the
explicit repair path:

```bash
xyran repair-runtime --dry-run
xyran repair-runtime
```

To disable first-use runtime installation:

```text
XYRAN_AUTO_INSTALL_RUNTIME=0
```

Then install `xyran[cpu]` / `xyran[gpu]` yourself or run `xyran setup-runtime`.

> ONNX Runtime CPU/GPU Python distributions share the same import namespace.
> A clean virtual environment remains the most predictable installation.

---

## Static images

```python
from xyran import Moderator

mod = Moderator()  # model/session resident by default
result = mod.scan("image.jpg")

print(result.decision)
print(result.scores)
print(result.provider)
```

CLI:

```bash
xyran scan image.jpg
xyran scan image.jpg --json
```

### Device selection

```python
Moderator(device="auto")  # CUDA when available/usable, otherwise CPU
Moderator(device="cpu")   # strict CPU
Moderator(device="cuda")  # strict CUDA
```

`device="auto"` can fall back to CPU if CUDA is enumerated but a real inference
call fails.

---

## Animated GIF / WebP

`scan()` automatically detects multi-frame GIF/WebP files and uses frame-aware
moderation:

```python
from xyran import Moderator, AnimationModerationResult

mod = Moderator()
result = mod.scan("animation.gif")

if isinstance(result, AnimationModerationResult):
    print(result.decision)
    print(result.processing.sampled_frames, result.processing.total_frames)
    print(result.processing.exhaustive)
    print(result.worst_frame.frame_index)
    print(result.worst_frame.scores)
```

Explicit animation API:

```python
result = mod.scan_animation(
    "animation.webp",
    sampling="smart",       # smart | uniform | all
    max_samples=32,
    batch_size=16,
)
```

CLI:

```bash
xyran scan-animation animation.gif --sampling smart
xyran scan-animation animation.gif --sampling all
```

### Smart sampling

The deterministic `smart` strategy combines endpoints, duration-aware temporal
coverage, visual-change peaks, long-dwell frames, and coverage fill.

Animations with 32 frames or fewer are exhaustive by default. Longer animations
are sampled to at most 32 model-scanned frames by default.

Smart sampling reduces inference cost but **does not guarantee every unsafe
frame is inspected**. For exhaustive frame moderation:

```bash
xyran scan-animation animation.gif --sampling all
```

---

## Folder Scan

Folder Scan is the simple one-shot directory workflow. It keeps the complete
structured result in memory and can immediately write human/machine-readable
reports.

```python
from xyran import Moderator

mod = Moderator()
report = mod.scan_folder(
    "./uploads",
    recursive=True,
    batch_size=16,
)

report.write_reports(
    "./reports/xyran-report",
    formats=("json", "md", "txt", "csv"),
)
```

CLI:

```bash
xyran scan-folder ./uploads \
  --output ./reports/xyran-report \
  --report-formats json md txt csv
```

Use Folder Scan when you want to inspect a directory once and keep the full
result object in memory.

---

# Dataset Mode

Dataset Mode is a **separate** workflow designed for long-running dataset
cleaning where jobs may contain tens of thousands or millions of files, may be
interrupted, or may be split across workers.

Basic CLI:

```bash
xyran scan-dataset ./dataset --output ./runs/train-clean
```

Outputs:

```text
train-clean.sqlite3       resumable state/cache
train-clean.jsonl         streaming-friendly manifest
train-clean.summary.json  run configuration + summary
```

Optional CSV:

```bash
xyran scan-dataset ./dataset \
  --output ./runs/train-clean \
  --manifest-formats jsonl csv
```

### Resume and cache

Re-running the same job reuses unchanged entries:

```bash
xyran scan-dataset ./dataset --output ./runs/train-clean
```

The SQLite state stores file fingerprints and a moderation configuration
signature. Cache reuse is invalidated when relevant semantics such as dataset
root, model, preprocessing, policy thresholds, or animation sampling change.

Useful controls:

```bash
# Disable cache reuse for this run
xyran scan-dataset ./dataset --no-resume

# Retry files previously cached as errors
xyran scan-dataset ./dataset --retry-errors

# Remove old state before starting
xyran scan-dataset ./dataset --reset-state

# Commit state every 250 completed entries
xyran scan-dataset ./dataset --checkpoint-every 250
```

If the process is interrupted, committed entries remain reusable on the next
run. Dataset Mode does not keep every per-file result in one Python list.

### Fingerprints

Fast default:

```bash
xyran scan-dataset ./dataset --fingerprint stat
```

`stat` uses file size + high-resolution modification time.

Stronger content fingerprint:

```bash
xyran scan-dataset ./dataset --fingerprint sha256
```

`sha256` reads every selected file and is therefore more expensive.

### Include / exclude

```bash
xyran scan-dataset ./dataset \
  --include "**/*.jpg" \
  --include "**/*.png" \
  --exclude "**/thumbnails/**" \
  --exclude "**/tmp/**"
```

Restrict extensions directly:

```bash
xyran scan-dataset ./dataset --extensions jpg jpeg png webp gif
```

Unknown extensions are skipped by default. To probe regular files whose
extensions cannot be trusted:

```bash
xyran scan-dataset ./dataset --probe-unknown
```

If Dataset Mode output files are placed under the scanned dataset root, Xyran
1.3 excludes its own SQLite/manifest/summary artifacts from the scan.

### Offset / limit

```bash
xyran scan-dataset ./dataset --offset 10000 --limit 50000
```

### Deterministic sharding

```bash
xyran scan-dataset ./dataset \
  --num-shards 4 --shard-index 0 \
  --output ./runs/shard-0

xyran scan-dataset ./dataset \
  --num-shards 4 --shard-index 1 \
  --output ./runs/shard-1
```

`shard_index` is zero-based and must be smaller than `num_shards`.

**Use a different output stem for each concurrently running shard.** Dataset
state files are not intended to be shared by multiple writers at the same time.

### Batch + animation controls

```bash
xyran scan-dataset ./dataset \
  --batch-size 64 \
  --animation-sampling smart \
  --animation-max-samples 48 \
  --animation-batch-size 16 \
  --animation-max-source-frames 5000 \
  --animation-max-duration-ms 600000
```

### Manifest detail

Compact default:

```bash
xyran scan-dataset ./dataset --detail summary
```

Full result object per item:

```bash
xyran scan-dataset ./dataset --detail full
```

### Python Dataset Mode API

```python
from xyran import Moderator

mod = Moderator()
run = mod.scan_dataset(
    "./dataset",
    output="./runs/train-clean",
    recursive=True,
    batch_size=64,
    include=("**/*.jpg", "**/*.png"),
    exclude=("**/thumbs/**",),
    fingerprint="stat",
    resume=True,
    checkpoint_every=250,
    num_shards=1,
    shard_index=0,
    manifest_formats=("jsonl", "csv"),
    detail="summary",
)

print(run.summary)
print(run.state_db)
print(run.manifests)
```

### Dataset Mode parameters

| Parameter | Purpose | Default |
|---|---|---|
| `recursive` | recurse through subdirectories | `True` |
| `batch_size` | static ONNX batch size | `32` |
| `include` / `exclude` | repeatable relative-path globs | none |
| `extensions` | explicit extension allow-list | supported image extensions |
| `probe_unknown` | inspect files with unknown extensions | `False` |
| `offset` / `limit` | slice selected items in a shard | `0` / unlimited |
| `num_shards` / `shard_index` | deterministic partitioning | `1` / `0` |
| `fingerprint` | cache fingerprint: `stat` or `sha256` | `stat` |
| `resume` | reuse matching cached entries | `True` |
| `retry_errors` | retry cached errors | `False` |
| `checkpoint_every` | SQLite commit interval | `100` |
| `manifest_formats` | `jsonl`, optional `csv` | `jsonl` |
| `detail` | compact `summary` or complete `full` | `summary` |
| `progress_every` | callback/CLI progress interval | `100` |
| `fail_fast` | stop on the first isolated item error | `False` |

---

## Real ONNX batching

Static inputs use dynamic `[batch, 3, 224, 224]` ONNX tensors rather than a
Python loop disguised as batching:

```python
results = mod.scan_batch(paths, batch_size=16)
```

Dataset Mode uses real batching internally for static samples and isolates bad
files if one corrupt sample causes a grouped batch to fail.

---

## Policy

The bundled classifier class order is:

```text
NSFL -> graphic
NSFW -> sexual
SFW  -> safe
```

Current default thresholds:

```text
sexual REVIEW  >= 0.35
sexual BLOCK   >= 0.85
graphic REVIEW >= 0.35
graphic BLOCK  >= 0.85
```

These thresholds are not universal safety policy.

```python
from xyran import Moderator, ModerationPolicy

policy = ModerationPolicy(
    sexual_review=0.40,
    sexual_block=0.90,
    graphic_review=0.35,
    graphic_block=0.85,
)

mod = Moderator(policy=policy)
```

---

## Image formats

`scan()` accepts local paths, encoded image bytes, and `PIL.Image.Image`.
Xyran identifies common image families from file content where possible rather
than trusting only filename extensions. EXIF orientation is applied and alpha is
flattened onto white before preprocessing.

| Format | Extensions | Policy |
|---|---|---|
| JPEG | `.jpg`, `.jpeg`, `.jpe` | static supported |
| PNG | `.png` | static supported; multi-frame/APNG fail-closed |
| WebP | `.webp` | static + animated supported |
| BMP | `.bmp`, `.dib` | static supported |
| TIFF | `.tif`, `.tiff` | single-page only |
| GIF | `.gif` | static + animated supported |

Extended raster formats can work when the installed decoder exposes the codec,
including HEIC/HEIF, AVIF, JPEG 2000, and ICO. Multi-page families other than
GIF/WebP remain fail-closed. SVG, PDF, and PSD are outside the normal raster
moderation contract.

Inspect local support:

```bash
xyran formats
xyran formats --json
```

---

## Defaults

```text
preprocess                    BlurPad + Lanczos3
input                         224 x 224
model resident                true
device                        auto
animation sampling            smart
animation max samples         32
animation batch size          16
folder static batch           16
dataset static batch          32
dataset fingerprint           stat
dataset resume                true
dataset checkpoint every      100
dataset manifest              JSONL
runtime                       ONNX Runtime / hardware-aware setup
telemetry                     disabled
cloud API                     none
```

Optional alternate preprocessing:

```python
mod = Moderator(preprocess="warp")
```

---

## Diagnostics

```bash
xyran doctor
xyran doctor --json
xyran formats
xyran setup-runtime --dry-run
xyran repair-runtime --dry-run
```

`xyran doctor` differentiates between a missing runtime (`setup-runtime`) and an
installed/conflicting/broken runtime (`repair-runtime`).

---

## Bundled model and provenance

The current Xyran free SDK bundles a pinned ONNX file from:

```text
Repository: OwenElliott/image-safety-classifier-s
Source commit: eb8b0b203952b70db191e990217174af4af39767
File: onnx/image-safety-classifier-s.onnx
Size: 23,701,765 bytes
SHA256: fef443ed68ae25ed693b6fef9e456071692ed3963cff4168acb39c3de6f017e7
Upstream license metadata: MIT
```

The model is not authored or trained by the Xyran project. Xyran pins and
verifies the redistributed model binary and provides the local SDK, runtime,
preprocessing, animation, batching, Folder Scan, Dataset Mode, reports, and
policy layer around it.

See `THIRD_PARTY_NOTICES.md` for attribution and license details.

Hugging Face integration/model page:

https://huggingface.co/MingSafeR/xyran-image-safety

---

## Offline behavior

The model is bundled inside published release artifacts and is not downloaded at
inference time.

A fresh `pip install xyran` intentionally leaves the CPU/GPU ONNX Runtime choice
to Xyran's hardware-aware setup. If no runtime is installed, first inference may
use pip and therefore may require package-index/network access.

For a machine that must be fully offline during inference, prepare the runtime
before disconnecting:

```bash
pip install "xyran[cpu]"
# or
pip install "xyran[gpu]"
```

After Xyran + the selected runtime are installed:

```text
model download during inference: NO
Xyran API key:                  NO
cloud moderation request:       NO
telemetry:                       NO
network required for inference: NO
```

---

## CLI overview

```bash
xyran --version
xyran doctor
xyran formats
xyran setup-runtime --dry-run
xyran scan image.jpg
xyran scan animation.gif
xyran scan-animation animation.gif --sampling all
xyran scan-folder ./uploads --output xyran-report --report-formats json md txt csv
xyran scan-dataset ./dataset --output xyran-dataset --manifest-formats jsonl csv
```

---

## Limitations

Xyran helps classify content-safety risk; it is not a guarantee that every
unsafe image/frame will be detected or that every flagged item is unsafe.

Important limitations include:

- classification quality depends on the bundled model and domain distribution;
- photography, anime/manga, illustrations, AI-generated imagery, unusual crops,
  and ambiguous content can behave differently;
- smart animation sampling is non-exhaustive for long animations;
- `stat` Dataset Mode fingerprints prioritize speed over cryptographic change
  detection;
- thresholds must be calibrated for the application;
- automated moderation should not be treated as a legal or factual authority.

Use `sampling="all"` when exhaustive GIF/WebP frame inspection is required, and
consider `--fingerprint sha256` when stronger dataset cache validation matters.

---

## License

The Xyran SDK source is licensed under **Apache-2.0**.

The bundled third-party image-safety model is redistributed under its upstream
MIT license metadata and attribution. See `THIRD_PARTY_NOTICES.md` and the
included third-party license file.

---

## Project status

Xyran is under active development. Bug reports and reproducible cases are
welcome:

https://github.com/mingshenhk/xyran/issues

Please do not upload private or sensitive media publicly just to demonstrate a
classification problem.
