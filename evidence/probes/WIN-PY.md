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

## Probe 3 (pre-registered before any run, 2026-10-22)
Success = >= 1 NEW Windows-specific defect in a documented entrypoint (ink_detection composite_from_zarr, prepare_9um_isotropic_input, download_required_zarr_chunks, prepare_canvas_offset_evidence; lasagna/ and spiral-fitting/ Python CLIs needing no C++; scripts/ README entrypoints; ink-detection/ subproject CLIs if installable without C++), reproducible on real public data, no issue/PR in any state (gh search incl. closed + open PRs on the file). 0 after ~75 min = KILL. D5/D6/zarr3/#1431 not re-reported. Ends with COVERAGE table.

### Probe 3 results (~65 min, wt-pyprobe detached @ e2254ebe0; repro dir C:\Users\<user>\vesuvius\pyprobe-out3; added pip: connected-components-3d)
VERDICT: SUCCESS, 2 new Windows defects (D7 solid, D8 weaker: it is a portable latent bug that Windows makes deterministic).

**D7: lasagna/labels_to_cc_zarr.py --video hangs forever on Windows.** `_encode_video` does `Popen(ffmpeg, stdin=PIPE, stderr=PIPE)`, writes all frames to stdin, and only reads stderr after `proc.wait()`. ffmpeg's banner+config alone exceeds the 4 KB Windows pipe buffer, ffmpeg blocks on stderr, never reads stdin, python blocks on the stdin write: deadlock. Repro: real PHercParis4 crop (vol.zarr level 1, z448-512,y/x640-768 thresholded -> labels.tif) `python labels_to_cc_zarr.py --input labels.tif --output cc.zarr --binary-zarr bin.zarr --video cc.mp4`: cc.zarr and bin.zarr written, then hangs at 0% CPU (killed after >2 min; ffmpeg 8.1 gyan build). Isolated test ffhang.py: _encode_video with 64 frames AND with 2 frames of 128x128 both hang >30 s; same call with stderr=DEVNULL finishes in 0.1 s. Linux has a 64 KB pipe so it works there (portable latent bug, deterministic on Windows). Fix: stderr=DEVNULL, or communicate(), or drain thread. Prior art: none (gh issue/pr --state all: labels_to_cc_zarr, labels_to_winding_volume, _encode_video, ffmpeg stderr/deadlock/hang; open PRs touching file: none). Same pattern elsewhere not checked.

**D8: vesuvius/scripts/collect_reviewed_fibers.py writes SHA256SUMS with CRLF on Windows** (`Path.write_text` text mode, no newline=''). `sha256sum -c SHA256SUMS` then fails for every row ("<path>\r: No such file"); passes after `tr -d '\r'`. manifest.json also CRLF (harmless to JSON). Repro on real public data: `--audit-only --originals <empty> --split <empty> --output rev_audit` against https://dl.ash2txt.org/other/annotators/ (1834 JSONs, 88 reviewed, 1m43s): od -c shows `.json\r\n`. Same write in bake_test_set. Prior art: none (file added by merged #2216; searches collect_reviewed_fibers, SHA256SUMS, CRLF, line endings, write_text, newline= found nothing on it; open PRs touching file: none). Impact is moderate (downstream integrity check, not data corruption).

COVERAGE (every command run)
| # | command | data | result |
|---|---|---|---|
| 1 | ink_detection.preprocessing.composite_from_zarr --input-root R --method max --start_z 896 --end_z 1152 --parallelism threads --workers 2 --overwrite --no-progress | copy of winvc3d\p3\vol.zarr (PHercParis4, sparse) | fine (wrote 1 TIFF) |
| 2 | same, --parallelism processes | same | fine |
| 3 | ink_detection.preprocessing.prepare_9um_isotropic_input vol.zarr iso.zarr --level 2 --workers 2 | same | fine (21x566x566) |
| 4 | create_label_zarrs on dsroot (segment auto_grown_20261022113620655 from winvc3d\run1 + png masks I generated) | real tifxyz geometry, synthetic label masks | fine (2 written) |
| 5 | download_required_zarr_chunks --dry-run (needed pip cc3d) | same dataset + vol.zarr via volumes.json | fine (zarr "not recognized as component" warnings for level_0/1/3 dirs, an artifact of the copy) |
| 6 | download_required_zarr_chunks --download-workers 2 --overwrite (spawn pool) | same | fine (150/27/8/2/1/1 chunks) |
| 7 | same, --download-workers 1 --overwrite | same | fine |
| 8 | same again without --overwrite (resume) | same | fine (all reused) |
| 9 | tifxyz_label_transfer.prepare_canvas_offset_evidence | none | not runnable: needs rclone (not installed) and a private ink rclone mirror (--ink-rclone-root required); source read only, no Windows hazard spotted statically |
| 10 | vesuvius/scripts/collect_reviewed_fibers.py --audit-only | live dl.ash2txt.org annotators (1834 JSONs) | DEFECT D8 (CRLF SHA256SUMS); download/rename/tempdir publish itself works |
| 11 | collect_reviewed_fibers.py with --bake-config / --test-source | none | not run: needs native vc_fiber_bake_annotations (C++) and manifests |
| 12 | lasagna/labels_to_cc_zarr.py --video | labels.tif from real vol.zarr crop | DEFECT D7 (hang); zarr outputs fine |
| 13 | lasagna/labels_to_winding_volume.py | labels.tif | not runnable: imports torch (dtypes.py), not installed |
| 14 | lasagna/analyze_zarr_channels.py --help | none | not runnable usefully (fails at import/arg stage, torch-family deps not installed; not investigated further) |
| 15 | other lasagna/*.py (fit2tifxyz, review_tiffs, preprocess_cos_omezarr, convert_fit_zarr..., omezarr_pyramid) | none | not run: torch/cupy/numba-cuda/lasagna C++ ext needed |
| 16 | spiral-fitting/*.py (get_ink_metrics, make_ink_report, convert_track_store, tracks_to_ome_zarr, extract_surface_tracks, etc.) | none | not run: torch/rustworkx/matplotlib/pandas not installed, time budget; no static review |
| 17 | scripts/fiber_5class (train, inspect_labels) | none | not runnable: cupy/cucim/cuws GPU watershed + DDP launcher is bash/tmux |
| 18 | scripts/convert_nnunet_to_vesuvius.py | none | not run (no nnU-Net dataset, no README entry) |
| 19 | ink-detection/ subproject CLIs | none | not run: time budget |
Cleanup: killed only my own hung PIDs (ffmpeg 29280, python 10272/24072; ffmpeg 33408, 8964 from ffhang.py). No outward actions.
