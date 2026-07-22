# legacy/dylan-testing-images

Dylan's original testing image set, preserved **verbatim** from his repo
(`dylanphillips27/pillbox-data-with-training/testing-images`). Kept here as a
frozen legacy artifact so it isn't lost.

## What's in it

- **42 whole-box photos** named `empty_<N>.jpg` / `full_<N>.jpg` — full pillbox
  shots (an "empty box" vs "box with pills" naming), predating the
  `photo_<YYYYMMDD>_<HHMMSS>.jpg` capture naming the app now uses.
- **168 per-cell crops** named `empty_<N>_<DAY>_<SLOT>.jpg` /
  `full_<N>_<DAY>_<SLOT>.jpg` — individual compartment crops.

## Why it's under legacy/ and not in the normal pipeline

This set is **not** wired into `labels/labels.json` or `splits/` — its ad-hoc
names don't map to a `photo_*` stem, so the `raw/` + `labels/` + `splits/`
tooling (`make_splits.py`, `export_dataset.py`) can't trace or regenerate it.
The repo's rule is "only sources of truth, crops are regenerated" — so committed
crops are normally avoided. This folder is the documented exception: it's a
pre-existing artifact that **can't be regenerated** from raw photos, so it's
frozen as-is rather than dropped.

Dylan's labelled captures that *do* fit the pipeline (the July 12–18 `photo_*`
photos and their labels) already live in `raw/` and `labels/labels.json` and
came in via the Pi's sync — this folder is only the older, separately-named
material.

## Using it

Point a training or eval job straight at this directory; it's already in the
Ultralytics `<class>/…` spirit if you group by the `empty_`/`full_` prefix.
Treat it as **held-out** unless you've checked it doesn't overlap in content
with the `raw/` photos, to avoid train/test leakage.
