# VisionQC · Part 1: image anomaly detection

VisionQC is a local-first visual inspection starter project. It includes a FastAPI service, a React dashboard, SQLite inspection logs, optional server-side OpenRouter image interpretation, and an Anomalib PatchCore training path. This project covers image inspection only. Conveyor controls, Arduino, and LiDAR are out of scope.

## What the score means

PatchCore returns an anomaly score based on distances between image patches and the memory bank learned from approved, normal training photos. **It is not a calibrated confidence, probability, or percent likelihood of a defect.** The initial PASS/FAIL values (0.35 and 0.55) are placeholders. Use held-out validation examples to choose useful triage bands, then evaluate once on the untouched test set before relying on the labels. The middle interval returns REVIEW, and an unavailable model always returns REVIEW.

The optional OpenRouter vision model provides a short visual description only. It does not affect the PatchCore score or decision. Its API key stays on the backend.

## Requirements

- Python 3.10–3.12 recommended. Anomalib and its PyTorch dependencies are large; a CUDA-capable NVIDIA setup can speed up training, while CPU is supported for a small starter set.
- Node.js 20 or later.
- 20–30 approved, in-focus photos of normal products for training, plus distinct normal validation and test photos. Include representative angles, backgrounds, and lighting. Defect examples can be added later for operational threshold assessment; this starter trains one-class PatchCore on good product photos.

## Run locally

Open two terminals from the project root.

### Backend

```powershell
py -3.11 -m venv .venv
.venv\Scripts\Activate.ps1
python -m pip install -r backend\requirements.txt
Copy-Item .env.example .env
uvicorn app.main:app --app-dir backend --reload
```

The API is at `http://localhost:8000`; interactive API docs are at `http://localhost:8000/docs`.

### Dashboard

```powershell
cd frontend
npm install
npm run dev
```

Open the local URL printed by Vite (normally `http://localhost:5173`).

## USB camera test mode

Open **Test mode**, connect a USB camera, and allow browser camera access. Choose an input device from the selector if more than one camera is available. The live preview includes a large, physical-style Capture button; pressing it captures one frame and sends it through the same inspection, heatmap, verdict, and SQLite logging flow as an uploaded image. Camera access is provided by the browser, so use the Vite localhost URL (or HTTPS when hosted). This is a dashboard button and browser camera capture; no GPIO, Arduino, or conveyor hardware trigger is connected in this image-only part.

In **Reference library**, choose a target split and select **Capture with camera** to capture a USB camera frame directly into that split as an approved reference photo.

## Train a model

1. Open **Reference library** and upload normal product images, choosing Train, Validation, or Test before each upload.
2. Keep the sets distinct. Assign 20–30 approved normal images to Train. Validation and Test each need at least one separate image; more is strongly recommended.
3. Review the assignment and approval state. Only approved Train images enter the PatchCore memory bank. Validation and Test images are stored separately and excluded from training.
4. Select **Train PatchCore**. The backend reports a clear error if the split requirements are not met.
5. Inspect images and review the heatmap and raw score. Set thresholds only after reviewing results on validation images. Keep the test set untouched until final evaluation.

PatchCore training and inference use Anomalib 2.x. The generated dataset lives under `models/dataset`, checkpoints under `models/training`, and active model metadata under `models/active.json`. Reference images and the SQLite database stay under `data/`. These local assets are excluded from Git.

## Optional OpenRouter interpretation

Put `OPENROUTER_API_KEY` in the project root `.env` (never in the React app), optionally choose `OPENROUTER_MODEL` (defaults to `typesafe/jev-router`), restart the backend, and select **Optional visual interpretation** when inspecting. The image is sent to the configured OpenRouter model when this option is enabled. Leave the key empty to disable the integration.

## API overview

- `GET /api/health` — API, model, and threshold state
- `POST /api/references` — upload images with a split and approval flag
- `GET /api/references` — list reference metadata
- `PATCH /api/references/{id}` / `DELETE /api/references/{id}` — update or remove a reference
- `POST /api/train` — validate splits and fit PatchCore
- `POST /api/inspect` — inspect a single image; optional form field `interpret=true`
- `PUT /api/thresholds` — set `pass_below` and `fail_at_or_above`
- `GET /api/inspections`, `GET /api/stats` — local SQLite history and counts

Rejection rate is FAIL count divided by all inspections. REVIEW remains a separate count, not a rejection.

## Basic tests

```powershell
python -m pytest backend\tests
```

The tests cover threshold boundaries, health/stats endpoints, and invalid image upload handling. They do not prove model accuracy; that requires representative held-out images from the actual product line.
