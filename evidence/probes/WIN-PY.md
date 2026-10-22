# WIN-PY branch

## Pre-registered kill bar (written before any run)
Success = >= 1 Windows-specific defect in a documented villa Python entrypoint, reproducible on real public data, with no issue/PR in any state (gh issue/pr list -R ScrollPrize/villa --state all, incl. closed/bot-closed, plus open PRs touching same file). 0 after ~90 min = KILL. #2240 (zarr_tasks merge create_dataset zarr 3) skipped.

(log below)

## Log (wt-pyprobe @ e2254ebe0, uv sync --frozen --extra tests --extra label-transfer, + pip tqdm/scikit-image/opencv/pyyaml; ~35 min)
| entrypoint | result |
|---|---|
| surface_preflight (real grown segment + PHercParis4 vol.zarr, with/without --volume) | works, atomic json write OK, exit 2 on real gate failure (support fraction) |
| zarr_tasks threshold/transpose/recompress/scale/zero-range --inplace (spawn, -n 2) | work |
| zarr_tasks export-tifs | works (prints "Saved 2/? chunks") |
| zarr_tasks edt-dilate/remap (group input) | create_dataset AttributeError on zarr 3 = #2240 family, platform-neutral, skip |
| zarr_tasks threshold --level 3 on group | TypeError in worker (zarr group getitem), platform-neutral |
| clean_labels x2, validate_segments | work |
| **merge_predictions (ink_detection.preprocessing)** | **DEFECT: leaks every temp .memmap file in %TEMP% on Windows** |
| torch-dependent (predict, compute_st, find_patches) and trimesh (voxelize_obj) not run (extras not installed) |

## Defect 1: merge_predictions temp memmap leak (Windows)
python -m vesuvius.ink_detection.preprocessing.merge_predictions ROOT --workers 1 (ROOT/seg/preds/*_forward_*.tif|png from real rendered tifs). Exit 0, "merged_outputs=1", but unlink of every temp memmap raises PermissionError [WinError 32], swallowed by `except Exception: pass` (cleanup_prepared_predictions, and merge_prediction_files finally). Leaves merge_predictions_*.memmap and merge_predictions_out_*.memmap in %TEMP% (3 per merged direction for 2 inputs, each full raster size). np.memmap objects are still referenced when unlink runs; never del/_mmap.close()d.
Code: merge_predictions.py write_array_to_temp_memmap / prepare_prediction (tif asarray(out="memmap")) / cleanup_prepared_predictions / merge_prediction_files finally block.
Prior art: none for this file (gh issue/pr --state all; open PRs touching merge_predictions|staged_write: none). Same class but other module: #1671 (open), #2006/#1724 (closed) tifxyz_label_transfer/io.py; #1705 closed (staged_write publish retry only).
Repro dir: C:\Users\<user>\vesuvius\pyprobe-out (dbg_merge.py spies Path.unlink).

## Probe 2 (pre-registered before any run)
Success = >= 1 NEW Windows-specific defect in a documented entrypoint (voxelize_obj, other vesuvius README/docs commands not run in probe 1, ink-detection CLIs, lasagna/spiral-fitting without C++, scripts/ README entrypoints), reproducible on real public data, with no issue/PR in any state (gh issue/pr list --state all, incl. closed + open PRs touching file). 0 after ~90 min = KILL. Non-Windows bugs listed separately, not counted. Probe-1 findings (D5 memmap leak, zarr_tasks zarr3, threshold-on-group #1431) not re-reported.

### Probe 2 results (~40 min)
- voxelize_objs (trimesh+numba+dask installed): real segA_merged.obj -> 6-level label zarr OK; on Windows it takes the "Fork start method unavailable" single-process branch (perf only, handled, not a defect).
- **Defect D6: cv2.imread/imwrite fail on non-ASCII paths on Windows (opencv 5.0.0 wheel).** Repro: `python -m vesuvius.ink_detection.preprocessing.create_label_zarrs "<dir with u-umlaut>"` containing real-data-derived `20250919184428_inklabels.png` -> "Processed 1 image(s): 0 written, 1 failed ... Failed to read image data" exit 1; same file in ASCII dir works. Code: create_label_zarrs.py:182 load_image (PNG branch). Same class, not run: tifxyz/reader.py:553 and tifxyz/types.py:303 (label images), scripts/labels_to_zarr.py:391, ink-detection/optimized_inference/processing.py:192 + entrypoint.py:414. Workaround verified: cv2.imdecode(np.fromfile(p, np.uint8), IMREAD_UNCHANGED). Prior art: none (gh issue/pr --state all for create_label_zarrs, cv2.imread, imdecode, non-ASCII, Unicode path); open PRs touching file: #2101 (singleton axes), unrelated. Repro dir: C:\Users\<user>\vesuvius\pyprobe-out2 (mk.py, lab_Scan u-umlaut).
VERDICT: SUCCESS (1 new defect).
