# DeepFakeSol — Deepfake Video & Image Detection

A research-oriented, face-based deepfake detection project based on **DFSpot (DeepFake-Spot)**. It uses BlazeFace to locate faces and PyTorch image-classification models to produce fake-class signals from sampled video frames or still images.

> **Status (repository audit: October 2026):** The code currently committed to this repository is the upstream-style DFSpot implementation. It supports model inference, CSV predictions, and optional annotated videos, but **does not yet include** the enhanced `src/dfspot_report.py` detailed-JSON reporter or the configurable `--threshold` CLI option discussed for the local Windows POC. The enhanced JSON format and commands are documented below as a **planned/local variant**, not as features verified in this repository. Commit those implementation files before using the enhanced command.

## Contents

- [Overview](#overview)
- [How it works](#how-it-works)
- [Models and training datasets](#models-and-training-datasets)
- [Repository layout](#repository-layout)
- [Prerequisites and compatibility](#prerequisites-and-compatibility)
- [Installation on Windows](#installation-on-windows)
- [Installation on Linux](#installation-on-linux)
- [Model checkpoints](#model-checkpoints)
- [Run inference](#run-inference)
- [CLI options](#cli-options)
- [Current outputs](#current-outputs)
- [Detailed JSON reporting — local enhancement](#detailed-json-reporting--local-enhancement)
- [Understanding scores and thresholds](#understanding-scores-and-thresholds)
- [POC validation methodology](#poc-validation-methodology)
- [Troubleshooting](#troubleshooting)
- [Security and data handling](#security-and-data-handling)
- [Credits and license](#credits-and-license)

## Overview

DeepFakeSol is intended for prototyping deepfake screening workflows, including evaluation of short selfie or video-verification clips. It is **not** a production-ready fraud decision engine, a proof-of-authenticity service, or a detector of specific forgery mechanisms.

### Capabilities in the currently committed code

- Infer from **videos** or **images** containing faces.
- Use one or more of four DFSpot model variants.
- Select model weights trained for DFDC, FaceForensics++, or Celeb-DF.
- Sample a configurable number of frames per video (non-annotated mode).
- Write a basic video prediction CSV.
- Optionally annotate processed videos with model/frame signals (legacy path).
- Run on CPU, or CUDA GPU if the installed PyTorch build and hardware support it.

### Limitations

- A **low fake score is not proof that a video is genuine**.
- Scores are **not calibrated probabilities**.
- Models inspect facial image patterns; they do not establish precisely *why* a frame was flagged.
- Audio, speech/lip synchrony, identity-document matching, injection attacks, and end-to-end liveness are **not validated** by this repository.
- The legacy dependency set predates Python 3.12.

## How it works

```text
Input MP4 / image
       |
       v
Video decoding & frame sampling (video)
       |
       v
BlazeFace detection & face extraction
       |
       v
Face crop preprocessing and normalization
       |
       v
PyTorch model(s): TimmV2 / TimmV2ST / ViT / ViTST
       |
       v
Per-model frame logits and fake-class scores
       |
       v
Video-level score + decision threshold
       |
       +--> Basic prediction CSV (current non-annotated video path)
       +--> Annotated video (current optional path)
       +--> Detailed JSON (requires uncommitted local enhancement)
```

The model detects learned statistical patterns associated with the **training data**. Its output is not an evidence-backed diagnosis of face swapping, blinking, skin texture, lighting, or a particular editing tool.

## Models and training datasets

Select any subset of the four supported architectures:

| CLI identifier | Checkpoint suffix (example for DFDC) |
|---|---|
| `TimmV2` | `dfdc_v2.pth` |
| `TimmV2ST` | `dfdc_v2st.pth` |
| `ViT` | `dfdc_vit.pth` |
| `ViTST` | `dfdc_vitst.pth` |

Supported training-dataset choices:

| CLI value | Dataset |
|---|---|
| `dfdc` | DeepFake Detection Challenge |
| `ffpp` | FaceForensics++ |
| `celeb` | Celeb-DF |

`--dataset` **selects the checkpoint family**. It does not identify or verify the source of the uploaded video. For example, you can evaluate an independently captured video with `--dataset dfdc` to load DFDC-trained checkpoints.

More models may provide useful comparison, but models trained on related data are **not statistically independent validators**.

## Repository layout

```text
DeepFakeSol/
├── README.md
├── LICENSE
├── requirements.txt       # Original/legacy pinned dependencies
├── setup.sh               # Original setup/download helper
├── examples.sh
├── assets/
└── src/
    ├── spot_deepfakes.py  # CLI entry point (currently upstream-style)
    ├── architectures/     # Network definitions
    ├── blazeface/         # Face detector, anchor and weight files
    ├── utils/             # Preprocessing, extraction, scoring, ensemble
    └── ensemble_model.ipynb
```

For local inference, you will also need directories such as `models/` and `sample_videos/`, which are created by you or by setup. Model checkpoints and private videos should generally **not** be tracked in Git.

**Not present in the published tree at the time of this audit:** `src/dfspot_report.py`, `requirements-py312.txt`.

## Prerequisites and compatibility

- Python and pip.
- Git.
- PyTorch and torchvision, matching your OS/CPU/CUDA setup.
- OpenCV, Albumentations, timm, SciPy, NumPy and the additional packages listed in `requirements.txt`.
- Downloaded DFSpot `.pth` checkpoints.

### Version warning

The repository's `requirements.txt` is the **original legacy dependency list** (including `numpy==1.21`, `pandas==1.1.5`, and `matplotlib==3.2.2`). These pins are **not a validated Python 3.12 environment**. The current `spot_deepfakes.py` also imports `distutils`, which was removed from Python 3.12.

For a reproducible run of the committed version, use an environment compatible with the original project or port and test its dependencies. The local Windows Python 3.12 compatibility modifications discussed for this project still need to be committed.

Do **not** assume the commands below constitute a validated Python 3.12 lockfile.

## Installation on Windows

Open **PowerShell**:

```powershell
git clone https://github.com/shailschauhan/DeepFakeSol.git
cd DeepFakeSol

# Create an isolated environment with a Python version compatible
# with the checked-out source and installed dependencies:
py -m venv dfspot_env
.\dfspot_env\Scripts\Activate.ps1

python -m pip install --upgrade pip
```

For the **currently committed legacy source**, install a Python/dependency combination compatible with the upstream project; do not blindly apply `requirements.txt` under Python 3.12.

```powershell
# Legacy dependency specification; installation may fail on modern Python:
python -m pip install -r requirements.txt
```

If you have already ported and tested this code with Python 3.12 locally, commit the ported `src/spot_deepfakes.py`, updated helper files, `src/dfspot_report.py`, and your tested requirements file together. This ensures GitHub clones reproduce your working installation.

### Recommended working directory

Examples below assume commands are executed from `src/`, so paths such as `../models/` resolve correctly:

```powershell
cd src
```

## Installation on Linux

```bash
git clone https://github.com/shailschauhan/DeepFakeSol.git
cd DeepFakeSol
python3 -m venv dfspot_env
source dfspot_env/bin/activate
python -m pip install --upgrade pip

# For an environment compatible with upstream's legacy requirements:
python -m pip install -r requirements.txt
```

The included `setup.sh` is an original Linux-oriented helper; review it before executing. It may download large model/data archives and install dependencies.

## Model checkpoints

The original setup script provides download instructions for the trained checkpoint archives. The DFDC checkpoint set previously used for this POC was distributed through Google Drive:

- [Original DFSpot source repository and setup instructions](https://github.com/chinmaynehate/DFSpot-Deepfake-Recognition)
- [DFDC checkpoint archive (Google Drive)](https://drive.google.com/file/d/1_Ofv_TBOeTDSaXhjnUi1BgNT3HWUmMPI/view)

Download model archives from sources you trust and extract the intended `.pth` files into:

```text
DeepFakeSol/
└── models/
    ├── dfdc_v2.pth
    ├── dfdc_v2st.pth
    ├── dfdc_vit.pth
    └── dfdc_vitst.pth
```

Use only those checkpoints you actually downloaded. Filenames must match the model/dataset combination selected in your command.

**Security:** PyTorch checkpoint loading can execute untrusted pickle content. Only load checkpoint files from a trusted, verified source.

## Run inference

> All commands in this section describe the **currently committed upstream-style CLI**. It uses a hardcoded classification threshold of `0.30` and does **not** accept `--threshold` until you commit the local modification.

### 1. Analyze a single video on Windows (CPU)

Create a directory (for example `sample_videos/test/`) and put your MP4 in it.

From the `src` directory:

```powershell
python spot_deepfakes.py `
  --media_type video `
  --data_dir ../sample_videos/test/ `
  --dataset dfdc `
  --model TimmV2 `
  --model_dir ../models/ `
  --video_id 0 `
  --n 1 `
  --fpv 32 `
  --annotate False `
  --device 0
```

`--device 0` selects CUDA device 0 **when CUDA is available**; otherwise this code falls back to CPU. It is **not** a CPU-forcing flag.

### 2. Use all four models

```powershell
python spot_deepfakes.py `
  --media_type video `
  --data_dir ../sample_videos/test/ `
  --dataset dfdc `
  --model TimmV2 TimmV2ST ViT ViTST `
  --model_dir ../models/ `
  --video_id 0 `
  --n 1 `
  --fpv 32 `
  --annotate False `
  --device 0
```

All four matching `.pth` checkpoints must be present.

### 3. Annotated video mode

```powershell
python spot_deepfakes.py `
  --media_type video `
  --data_dir ../sample_videos/test/ `
  --dataset dfdc `
  --model TimmV2 `
  --model_dir ../models/ `
  --video_id 0 `
  --annotate True `
  --output_dir output/ `
  --device 0
```

This legacy code path analyzes more frames and may use considerable CPU/RAM. It is not the same as the sampled-frame JSON reporting workflow.

### 4. Image inference

```powershell
python spot_deepfakes.py `
  --media_type image `
  --data_dir ../sample_images/ `
  --dataset dfdc `
  --model TimmV2 `
  --model_dir ../models/ `
  --device 0 `
  --output_dir output/
```

The legacy image path writes `img_predictions.json` relative to its configured output directory.

## CLI options

| Argument | Meaning | Upstream default |
|---|---|---|
| `--media_type` | `video` or `image` | none |
| `--data_dir` | Input directory | required |
| `--dataset` | Checkpoint family: `dfdc`, `ffpp`, `celeb` | required |
| `--model` | One or more model names | required |
| `--model_dir` | Checkpoint directory | required |
| `--video_id` | Zero-based indices into discovered video list | unset |
| `--n` | Number of video entries the legacy path processes | `2` |
| `--fpv` | Sampled frames per video, non-annotated mode | `32` |
| `--face_policy` | Face transform policy, e.g. `scale` | `scale` |
| `--face_size` | Model face-crop size | `224` |
| `--blazeface_dir` | Face detector assets | `blazeface/` |
| `--device` | CUDA device index if available | `0` |
| `--annotate` | Annotated video mode `True`/`False` | `False` |
| `--output_dir` | Annotation/image output directory | `output/` |

The proposed `--threshold` parameter is **not in the code currently committed**.

## Current outputs

### Non-annotated video

The current `utils.predict(..., save_csv=True)` writes a `predictions.csv` **in the current working directory**. It includes a video path and a basic predicted class (`real`/`fake`), not the enhanced evidence report.

### Annotated video

The existing `--annotate True` path creates annotated AVI output in the specified directory, subject to the original code's size, face-detection, and compatibility constraints.

### Image mode

The existing image path writes JSON predictions, using its separate legacy function.

## Detailed JSON reporting — local enhancement

A local Windows POC iteration introduced the design for:

- Configurable `--threshold`, defaulting to `0.30`.
- Per-video `ensemble_score` and per-model `model_scores`.
- `frames_requested`, `frames_sampled`, `frames_analyzed`, and `frames_without_face`.
- `frame_results` with original frame index, timestamp, per-model score, and classification.
- `suspicious_frames`, `model_agreement`, explanation, and limitations.
- A separate detailed report at `src/output/predictions_detailed.json`.

**Important:** `src/dfspot_report.py` and the modified CLI are **not currently tracked in GitHub**, so running the enhanced command against this checkout will fail. Publish the tested local files before following this example:

```powershell
# LOCAL ENHANCEMENT ONLY — requires the matching source changes
python spot_deepfakes.py `
  --media_type video `
  --data_dir ../sample_videos/test/ `
  --dataset dfdc `
  --model TimmV2 `
  --model_dir ../models/ `
  --video_id 0 `
  --n 1 `
  --fpv 32 `
  --threshold 0.30 `
  --annotate False `
  --device 0 `
  --output_dir output/
```

### Example enhanced report (illustrative)

```json
[
  {
    "file": "example.mp4",
    "dataset": "dfdc",
    "device": "cpu",
    "models": ["TimmV2"],
    "threshold": 0.3,
    "frames_requested": 32,
    "frames_sampled": 32,
    "frames_analyzed": 32,
    "frames_without_face": 0,
    "decision": "FAKE_MODEL_SIGNAL",
    "ensemble_score": 0.85581,
    "model_scores": {"TimmV2": 0.85581},
    "model_agreement": {
      "fake_votes": 1,
      "real_votes": 0,
      "total_models": 1
    },
    "suspicious_frames": 32,
    "frame_results": [
      {
        "frame": 0,
        "seconds": 0.0,
        "face_detected": true,
        "ensemble_frame_score": 0.751568,
        "model_scores": {"TimmV2": 0.751568},
        "signal": "FAKE"
      }
    ],
    "explanation": {
      "summary": "Video score exceeds the configured threshold.",
      "confidence_calibrated": false
    },
    "limitations": [
      "Scores are not calibrated probabilities.",
      "A classification does not prove authenticity or manipulation."
    ]
  }
]
```

Here, the single `frame_results` entry is **abridged for illustration**; a real report can contain all sampled frames.

## Understanding scores and thresholds

### What does `0.30` mean?

In the original prediction code, the rule is:

```python
predicted_class = "real" if ensemble_score < 0.30 else "fake"
```

This is an implementation choice. The current repository does not contain validation evidence showing that `0.30` is optimal for your specific camera conditions, attack types, or population.

A score of `0.90` does **not** mean there is a scientifically validated 90% probability that a video is fake. Likewise, a score of `0.02` cannot verify that a video is genuine.

### How are video and frame scores aggregated?

The existing `src/utils/ensemble.py` calculates a per-model video score by applying sigmoid to the mean raw model output (logit) across frames:

```text
model_video_score = sigmoid(mean(frame_logits))
```

The outer prediction code then averages the **model-level video scores** across the selected model set:

```text
ensemble_video_score = mean(model_video_scores)
```

This means the final video score is **not necessarily equal to the simple mean of the displayed frame-level sigmoid scores**.

### Interpreting example patterns

| Pattern | Interpretation | Suggested action |
|---|---|---|
| High video score; nearly all frames flagged | Consistent model signal | Review and obtain independent corroboration |
| Low video score; a few frames flagged | Local spikes despite low overall signal | Review flagged frames and neighbors |
| Low video score; nearly no frames flagged | Low signal on sampled faces | Do not treat as verified genuine |
| No face found | Insufficient evidence for this face-based detector | Retry capture or route for review |
| Models disagree | Model-dependent result | Investigate domain shift and false-positive risk |

A single-model `fake_votes: 1` is **one classifier vote**, not corroboration by multiple systems. A threshold-based spike can also arise from blur, motion, lighting, face crops, occlusion, or compression.

## POC validation methodology

For a meaningful evaluation, use a labeled dataset containing:

1. Genuine and manipulated clips with known ground truth.
2. Realistic capture conditions: devices, lighting, bandwidth, pose, skin tones, glasses, motion, and compression.
3. Different manipulation families, including examples **not represented in training**.
4. Separate tuning, validation, and held-out test sets.

Measure at least:

| Metric | Why it matters |
|---|---|
| True positive rate / fake recall | Proportion of known manipulations detected |
| False-positive rate | Genuine videos incorrectly flagged |
| Precision | Fraction of flagged videos actually manipulated |
| ROC-AUC and PR-AUC | Model discrimination across thresholds |
| Confusion matrix | Absolute false positive/negative counts |
| Face-detection coverage | Whether the detector could inspect the clip |
| Runtime per video | CPU/GPU sizing and operational cost |

Select thresholds against labeled validation data, then lock the threshold before evaluating the held-out test set. When false positives can block legitimate users, include a **human-review state** rather than automatically rejecting a user.

For insurance PIVC/PIBC-style workflows, use this detector as **one signal** alongside identity verification, liveness/presentation-attack checks, device/session integrity, and documented escalation policies. These complementary checks are not implemented here.

## Troubleshooting

### `IndexError: list index out of range` near `models_loaded[0]`

Usually means **no model checkpoints were discovered or loaded**. Check that:

- Your `--dataset` matches the checkpoint filename prefix.
- The selected model has a corresponding `.pth` file.
- The `--model_dir` path is correct from `src/`.
- The legacy glob/path handling works on your OS.

### `np.int` / NumPy attribute error

Older BlazeFace video/frame utilities may contain `np.int`, removed from modern NumPy. Replace deprecated aliases with appropriate built-in integer types in your tested compatibility branch.

### `ModuleNotFoundError: distutils` on Python 3.12

The currently committed entry point uses `distutils.util.strtobool`. This is legacy code; port it to a supported alternative before claiming Python 3.12 compatibility.

### `--threshold: unrecognized arguments`

The committed CLI does not yet include configurable-threshold support. Commit the updated CLI and matching utility changes.

### `ModuleNotFoundError: dfspot_report`

The enhanced JSON reporting module is not in the published repository. Commit the tested `src/dfspot_report.py` file.

### `torch.load` checkpoint errors

PyTorch versions differ in default checkpoint-deserialization behavior. Only consider compatibility overrides **for checkpoints you trust**; never disable safe loading for untrusted files.

### No face detected

The detector cannot classify frames lacking a usable face crop. Check video decoding, image quality, face size, head pose, and cropping; never silently interpret missing faces as genuine.

### Running from the wrong directory

Several legacy paths are relative. Run the script from `src/` as shown above, or consistently adjust relative paths.

## Security and data handling

- Do not commit private user videos, KYC documents, credentials, model archives, or production inference records.
- Treat extracted face images and verification videos as sensitive information.
- Encrypt stored evidence, restrict access, and define deletion/retention rules.
- Record model version, checkpoint provenance, preprocessing version, threshold, and timestamps for reproducible assessments.
- Do not use fake-class scores alone to make consequential identity or fraud decisions.

## Roadmap

- [ ] Publish the tested Python 3.12 compatibility changes and reproducible dependency specification.
- [ ] Publish `src/dfspot_report.py` and the updated `--threshold` CLI.
- [ ] Add flagged-frame image export with timestamps and configurable evidence retention.
- [ ] Add tests for missing faces, multiple videos, model ordering, output schemas, and threshold boundaries.
- [ ] Benchmark individual models and the four-model ensemble on labeled data.
- [ ] Add threshold calibration and drift monitoring for the intended deployment domain.
- [ ] Document model download integrity checks and approved checkpoint versions.

## Credits and license

This project builds on the open-source **DFSpot / DeepFake-Spot** research implementation. The original authors retain credit for the underlying model designs, code, and research.

- Original project: [chinmaynehate/DFSpot-Deepfake-Recognition](https://github.com/chinmaynehate/DFSpot-Deepfake-Recognition)
- Paper: C. Nehate, P. Dalia, S. Naik and A. Bhan, **“Exposing DeepFakes using Siamese Training,”** IEEE INDISCON (2022), DOI: [10.1109/INDISCON54605.2022.9862825](https://doi.org/10.1109/INDISCON54605.2022.9862825)
- License: [MIT LICENSE](LICENSE)

**Disclaimer:** DeepFakeSol is for research and proof-of-concept screening. Outputs are probabilistic model signals, not forensic proof or calibrated probabilities.
