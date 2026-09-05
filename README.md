# Xyran

**Local-first content moderation for Python.**

Run image safety checks entirely on your own machine — **no API key, no cloud upload, no external inference service required.**

```bash
pip install xyran
```

```bash
xyran scan ./images
```

Xyran is designed for developers who need fast, private, local content moderation in Python applications, backend services, dataset pipelines, desktop software, and self-hosted environments.

## Why Xyran?

Most content moderation services require sending images to a cloud API.

Xyran takes a different approach:

* 🔒 **100% local inference**
* 🔑 **No API key**
* ☁️ **No image uploads**
* 🐍 **Python SDK + CLI**
* ⚡ **CPU and NVIDIA GPU acceleration**
* 📁 **Folder and batch scanning**
* 🎞️ **Animated GIF / WebP support**
* 📄 **JSON / Markdown / text reports**
* 🧠 **ONNX Runtime inference**
* 📴 **Works offline after installation**

Your content stays on your machine.

## Quick Start

Install Xyran:

```bash
pip install xyran
```

Check the runtime:

```bash
xyran doctor
```

Scan an image:

```bash
xyran scan image.jpg
```

Scan a directory:

```bash
xyran scan ./images
```

Xyran can classify content into safety categories such as:

```text
safe
sexual
graphic
```

and provides probability scores that applications can use to implement their own moderation policy.

## What is Xyran for?

Xyran is useful for:

* image upload moderation
* AI image platforms
* dataset cleaning
* Discord / Telegram / community bots
* FastAPI / Flask / Django backends
* desktop applications
* self-hosted services
* NAS and private environments
* offline moderation pipelines
* batch media analysis

## Xyran vs Cloud Moderation APIs

|                              | Xyran | Cloud moderation APIs |
| ---------------------------- | ----- | --------------------- |
| Runs locally                 | ✅     | ❌                     |
| API key required             | ❌     | Usually               |
| Upload images to third party | ❌     | Usually               |
| Offline operation            | ✅     | ❌                     |
| Python SDK                   | ✅     | ✅                     |
| CLI                          | ✅     | Varies                |
| Folder scanning              | ✅     | Usually custom code   |
| Local GPU acceleration       | ✅     | N/A                   |
| Infrastructure control       | ✅     | Limited               |

Cloud moderation services remain a good choice when you want a fully managed service.

Xyran is designed for cases where **privacy, local execution, offline operation, or infrastructure control matter more.**

## Xyran vs NSFWJS

NSFWJS is a strong option for JavaScript and browser-side NSFW classification.

Xyran targets a different environment:

**NSFWJS**

```text
JavaScript
Browser
TensorFlow.js
Client-side moderation
```

**Xyran**

```text
Python
Backend / desktop / local server
ONNX Runtime
Batch and folder workflows
Offline moderation
```

If your application is built around Python or requires server-side/local processing without cloud uploads, Xyran is designed for that workflow.

## Privacy

Xyran performs inference locally.

Images do not need to be uploaded to Xyran, Microsoft Azure, Google Cloud, or another external moderation service for inference.

This makes Xyran suitable for environments where media privacy or data residency matters.

## Performance

Xyran supports ONNX Runtime and can use available hardware acceleration where supported.

Actual throughput depends on:

* CPU/GPU
* image size
* batch size
* runtime provider
* preprocessing configuration

A reproducible public benchmark suite is planned so Xyran can be compared transparently with other moderation approaches.

## Limitations

Content moderation is probabilistic.

Xyran can produce false positives and false negatives and should not be treated as a perfect authority.

Performance may vary across:

* photography
* anime
* manga
* illustrations
* unusual crops
* heavily compressed media
* abstract imagery
* adversarial or ambiguous content

Applications should choose thresholds appropriate for their own risk profile.

For high-impact decisions, consider combining automated classification with additional safeguards or human review.

## Philosophy

Xyran is built around a simple idea:

> Content moderation should not always require sending user content to someone else's server.

The goal is to make private, local content safety infrastructure easy to add to Python applications.

## Project Status

Xyran is under active development.

Areas of ongoing work include:

* improved illustration / anime detection
* broader safety categories
* harder edge-case datasets
* improved benchmarking
* additional platforms and runtimes

Feedback, difficult samples, bug reports, and reproducible test cases are welcome.

## Contributing

Issues and pull requests are welcome.

If you find a false positive, false negative, performance issue, runtime problem, or integration problem, please open an issue with enough information to reproduce it.

Please avoid uploading sensitive private media publicly when reporting moderation issues.

## Install

```bash
pip install xyran
```

**Local inference. No API key. No cloud upload.**
