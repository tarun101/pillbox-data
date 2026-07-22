# pillbox-data

Training, validation and test data for the [pillbox](https://github.com/tarun101/pillbox)
pill-presence detectors, plus the model registry. The Raspberry Pi pushes new
photos and labels here automatically (hourly); people and training jobs pull.

## Layout

```
raw/YYYY-MM-DD/photo_*.jpg      every capture, filed by date — the source of truth
labels/labels.json              per-cell ground truth: "<photo stem>/<DAY>_<SLOT>": "pill" | "empty"
splits/{train,valid,test}.txt   photo stems per split, assigned BY CAPTURE SCENE
references/<set-id>/            empty-box reference photos (for the 6-channel CNN)
models/<detector>/<version>/    trained models + card.json (see "Contributing a model")
legacy/<set-name>/              pre-existing datasets that can't be regenerated
                                from raw/ (e.g. ad-hoc-named crops); frozen as-is
export/                         generated train folders — gitignored, never committed
```

Only sources of truth are stored. **Cropped cell images are never committed** —
they are derived from the raw photos and go stale whenever the crop calibration
changes, so they get regenerated on demand (next section).

## Quick start (Dylan): get a ready-to-train dataset

You need both repos side by side, plus `opencv-python` and `numpy`:

```bash
git clone https://github.com/tarun101/pillbox
git clone https://github.com/tarun101/pillbox-data
pip install opencv-python-headless numpy

python3 pillbox/detect/export_dataset.py --data pillbox-data --clean
```

That writes the familiar Ultralytics-classify layout (same shape as the
Roboflow export you trained on) to `pillbox-data/export/`:

```
export/
  Train/Full/photo_20260713_144724_SAT_NIGHT.jpg   # <photo stem>_<DAY>_<SLOT>
  Train/Empty/…
  Valid/Full/…   Valid/Empty/…
  Test/Full/…    Test/Empty/…
```

Filenames keep provenance, so any crop traces back to its raw photo. Re-run
the export any time — it always reflects the latest photos and label
corrections. Train on it e.g. with:

```bash
yolo classify train data=pillbox-data/export model=yolov8n-cls.pt imgsz=640
yolo export model=runs/classify/train/weights/best.pt format=onnx imgsz=640
```

## Ground rules

- **Never train on `Test/`.** The test split is frozen — photos are never moved
  between splits — so everyone's accuracy numbers stay comparable over time.
  Report test accuracy only for a final, chosen model.
- **Don't hand-edit `splits/*.txt` or file images into Train/Valid/Test
  yourself.** Splits are assigned by *capture scene* (shots taken seconds
  apart are near-duplicates; letting them straddle splits inflates accuracy).
  `pillbox/detect/make_splits.py` handles it, and the Pi runs it automatically.
- **Don't commit `export/`** (it's gitignored). If you want to pin exactly what
  a model trained on, note this repo's commit hash in the model's `card.json`
  instead — the export is reproducible from any commit.
- **Labels come from the app.** Corrections are made in the pillbox web app's
  Analyze modal ("Your labels" grid) and sync here hourly. If you spot a wrong
  label while training, say so / fix it in the app rather than editing
  `labels.json` by hand, so the two never diverge.

## Contributing a model

Drop trained models into the registry with a small metadata card:

```
models/
  yolo/
    dylan-2026-07-16/
      best.onnx          # what the Pi runs (onnxruntime)
      best.pt            # the source checkpoint (kept for re-export)
      card.json
  cnn/
    2026-07-13-v1/
      pill_classifier.onnx
      card.json
```

`card.json` — a few lines so "which model is live and why" stays answerable:

```json
{
  "trained_by": "Dylan P",
  "date": "2026-07-16",
  "data_commit": "<git rev-parse HEAD of this repo when exported>",
  "base_model": "yolov8n-cls",
  "imgsz": 640,
  "val_accuracy": 0.97,
  "test_accuracy": 0.95,
  "notes": "trained on Roboflow export + July 13 set"
}
```

**Promotion** (making a model live on the Pi): copy the winner into the app
repo — `detect/yolo/best.onnx` for YOLO, `detect/pill_classifier.onnx` for the
CNN — and open a PR there. Merging auto-deploys to the Pi within ~2 minutes;
the PR diff is the audit log and rollback is `git revert`. Models must be
ONNX to run on the Pi (onnxruntime only — no PyTorch there); classify models
should name their classes so the "pill present" one contains `full` or `pill`.

## Importing pre-existing datasets

If you have data from before this repo (e.g. your original Roboflow export):
if the filenames still identify the source photo + cell, it can be absorbed
into `labels.json`/`splits` losslessly; if not (Roboflow-renamed or augmented
crops), it gets preserved as a frozen snapshot under `legacy/<set-name>/` and
concatenated at training time. Post a file listing (`find <dataset> -type f |
head -20`) in the group chat and we'll wire it in.

## How data flows

```
Pi camera  →  web app (capture + Analyze labeling)  →  ~/photos + labels.json
    →  hourly sync (deploy/sync-data.sh)  →  this repo (raw/, labels/, splits/)
    →  export_dataset.py  →  export/ (Train|Valid|Test / Full|Empty)
    →  training  →  models/<detector>/<version>/  →  PR to pillbox  →  Pi
```
