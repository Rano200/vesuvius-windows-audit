# Windows audit of the Vesuvius open-source tools (ScrollPrize/villa)

Date of runs: 2026-10-22. Machine: Windows 11 Home (10.0.26200) laptop.

## 1. Summary

I ran the official VC3D Windows release (build 5416c67, 51 `vc_*.exe` tools) and the documented Python CLIs on real public scroll data, the way a user on Windows would. Five probes, each with a kill bar written before any run. I found eight Windows-specific defects. Four are in the C++ tools: every tool that uses the process-wide chunk cache never exits after finishing (D1), `vc_merge_tifxyz` always fails at its last step (D2), `vc_zarr_recompress` on a local zarr silently writes an empty output with exit 0 (D3), and VC3D refuses to start any of its built-in `vc_*` tools (D4). Four are in Python: `merge_predictions` leaks temp memmaps (D5), `cv2.imread` fails on non-ASCII folders (D6), lasagna's `labels_to_cc_zarr.py --video` deadlocks on the ffmpeg pipe (D7), and `collect_reviewed_fibers.py` writes a `SHA256SUMS` that `sha256sum -c` rejects (D8). All eight have a fix with a before/after run on the same data. The four C++ fixes were built on the fork's CI and pass it; the four Python fixes pass new tests that fail on main. Nothing has been sent upstream. The fix for D1 also un-hangs 9 of the 10 tests in the open issue #1686 (fork CI, section 6).

## 2. Method

- **Official binaries, documented commands.** The C++ tools are the release zip `VC3D-5416c67-2026-10-22-win64.zip`, run as a user would, starting from the README examples (for example `vc_grow_seg_from_seed` with seed 1408 1408 1024). Python entrypoints come from villa at e2254ebe0, installed with `uv sync --frozen`.
- **Real public data.**
  - Scroll volume: PHercParis4 `20260310173927-45.532um-11.0m-110keV-masked.zarr` on the open-data bucket. Remote runs read it by URL. Local runs use a 49 MB copy of two public chunks (level 1).
  - Segments: grown by `vc_grow_seg_from_seed` from that volume (8 and 50 generations, 78.5 cm2 at 50).
  - Ink data for D5/D6: PHerc0500P2, segment 20250919184428, two published ink prediction TIFFs.
  - Python probes also used a PHercParis4 level-0 crop (256^3) and PHerc1203 level 5 (for the earlier gpupause fix).
- **Pre-registered kill bars.** Each probe states in its file, before any run, what counts as success (at least one new Windows-specific defect, reproducible on real data, with no villa issue or PR in any state) and when it is killed (0 after 75 to 90 minutes). Probes 1, 4 and 5 ended at or past the strict bar. The literal bar was met by probes 1 (D1), 2 (D2), 3 (D3) and the Python probes (D5, D6); D4 came from the bridge work around probe 5. Probes 4 and 5 are reported as kills; they found no defect of consequence and their "checked and fine" lists are in section 4.
- **Hang detection.** I wait on the process handle from plain bash or a script that polls `WaitForExit`, then snapshot the process with CIM and end it by the PID I started. PowerShell and .NET `HasExited` is not reliable here. The exit code is already set while the process is stuck in DLL teardown. My early wrappers reported "exit 0" for runs that were hung, and I nearly dropped D1 because of it. I never kill by image name.
- **Fork CI for builds.** I do not build locally. Fixes are built by GitHub Actions on my fork (Rano200/villa, `workflow_dispatch` plus fork PRs, no upstream PRs). The CI artifact is downloaded and run on the same laptop on the same data.
- **Before and after on the same data.** Before = release 5416c67 and a CI build of unmodified main. After = CI build of the fix branch. Output hashes or byte sizes are compared where the output is deterministic.
- **GUI without clicks.** VC3D was driven through its Agent Bridge (named pipe) with `QT_QPA_PLATFORM=offscreen`, using `qoffscreen.dll` from the MSYS2 Qt 6.11.1-1 package. Its Qt6Core and Qt6Gui DLLs are byte-identical to the release's.

## 3. Defects

All fix branches are on github.com/Rano200/villa (`fix/<name>`). Commit ids are on the stated base.

| # | Symptom | Tools affected | Root cause (file:line) | Fix branch | Proof |
|---|---|---|---|---|---|
| D1 | Correct output is written, then the process never exits (1 thread, 0 CPU). A foreground command, `subprocess.wait()` or CI step blocks forever. | `vc_render_tifxyz`, `vc_zarr_download_region`, `vc_render_video`, `vc_gen_normalgrids generate`, `vc_objrefine` (also hangs the bridge job queue), `vc_grow_seg_from_seed` on its error return. Test binaries in #1686. | `core/src/render/ChunkCache.cpp:731-735`: a function-local static registry holds the process-wide `ChunkCacheService`. On Windows its destructor runs in `DLL_PROCESS_DETACH` after the worker threads are gone, and waits on them. The sibling pools already leak on `_WIN32` (#1554). | `fix/win-render-exit` (d7a8b6e, base 72600fe), +7 lines | About 10 of 10 runs hang on the release. After: the fixed CI build exits by itself with code 0 in 2.1 to 3.6 s, 3 of 3. `00.tif` sha256 9A667B74... identical across all five runs. Also shown on the fixed build: `vc_zarr_download_region` (208 s hang to 2 s, exit 0), `vc_objrefine` (100 s hang to exit 0) and the `vc_grow_seg_from_seed` error return (exit 1, its own EXIT_FAILURE). CI on #1686's tests: 9 of 10 un-hang (section 6). |
| D2 | The whole merge runs (about 17 s on a 78 cm2 segment), then exit 1 with "The system cannot find the path specified." No tifxyz, a stray OBJ is left. | `vc_merge_tifxyz` (so VC3D's merge action when it reaches this tool) | `apps/src/vc_merge_tifxyz.cpp:1521-1538` `rasterizeObjToTifxyz`: `std::system` runs `cmd.exe /c`, which strips the first and last quote of `"exe" "obj" "out" 20`. Second bug behind it, `:1556-1565` `resolveObj2Tifxyz`: sibling probe has no `.exe`. | `fix/win-merge-tifxyz` (1a8638d, base 5f380c3), +11 -1 | Main and release: exit 1 after 17 s. Fixed: exit 0 in 15.5 to 16.4 s, "output: (99, 99), valid=9604". x/y/z.tif identical across runs and equal to running `vc_obj2tifxyz_legacy` by hand. |
| D3 | Local zarr in, exit 0, empty output: "Occupancy LIST: 0 / 32768", "0 chunks (skipped: 4096)", six 65,536-byte empty shards. Silent data loss. | `vc_zarr_recompress` with a local input path (s3/https input is fine) | `apps/src/vc_zarr_recompress.cpp` `LocalBackend::list_chunks` (~159) returns `fs::relative(...).string()`, with backslashes. The occupancy parser (~547-552) splits on `rfind('/')`, finds none, skips every key. | `fix/win-recompress-list` (7025ec6, base 5f380c3), +2 -1 | Main and release: 0 chunks at every level. Fixed: "Occupancy LIST: 8" and "2", level-1 shard 738,566 bytes, equal to the `--one-shard` control. Read-back: 33.5M voxels, MAE 1.094, PSNR 36.60 dB (lossy codec). |
| D4 | VC3D will not start any built-in tool it launches: render returns -32006 "vc_render_tifxyz not found or not executable: <bin>/vc_render_tifxyz"; Run Trace returns -32005 "Failed to start Run Trace." | VC3D `CommandLineToolRunner` users: render, Run Trace, tifxyz2obj, obj2tifxyz, alpha-comp refine, neighbor copy, merge tifxyz, merge patch | `apps/VC3D/CommandLineToolRunner.cpp` `toolName` (~790) builds the path with no `.exe`; `execute()` requires the file to exist. Three pre-checks in `SegmentationCommandHandler.cpp` (~2472, ~3125, ~3975) repeat it. | `fix/win-gui-tools` (0fb5a65, base 5f380c3), +17 -11 | Through the Agent Bridge: release and main fail as above; the fixed build runs `vc_render_tifxyz.exe` to success (1020x1020 tif) and `vc_grow_seg_from_segments.exe` to success (two tifxyz surfaces). |
| D5 | Exit 0, "merged_outputs=1", but every temp `.memmap` stays in `%TEMP%` (full raster size each). The unlink raises `PermissionError [WinError 32]`, swallowed by `except Exception: pass`. | `python -m vesuvius.ink_detection.preprocessing.merge_predictions` | `merge_predictions.py` `write_array_to_temp_memmap`, `prepare_prediction`, `cleanup_prepared_predictions` and the `finally` in `merge_prediction_files`: arrays are still mapped when unlink runs. | `fix/win-merge-preds` (+19 -4 plus a 38-line test) | New test fails 3 of 3 on main, passes on the branch. Real data (two PHerc0500P2 TIFFs): leftover 1,624,000 bytes (1 file) on main, 0 on the branch. Merged output sha256 identical. |
| D6 | `create_label_zarrs` on a folder with a non-ASCII name: "0 written, 1 failed ... Failed to read image data", exit 1. Same file in an ASCII folder works. | `create_label_zarrs`; same pattern, not run: `tifxyz/reader.py:553`, `tifxyz/types.py:303`, `scripts/labels_to_zarr.py:391`, `optimized_inference/processing.py:192`, `entrypoint.py:414` | `create_label_zarrs.py:182` `load_image`: Python `cv2.imread` (opencv 5.0.0 wheel) does not take UTF-8 paths on Windows. | `fix/win-imread-unicode` (6634b4b24): `imread_unicode` helper (`np.fromfile` + `cv2.imdecode`) at 4 sites | New tests fail 3 of 3 (folders "Scan u-umlaut", "Jose" with an accent, CJK) on main; 77 passed on the branch. Real-data-derived PNG: main exit 1, branch exit 0 in both folders, arrays identical. |
| D7 | `labels_to_cc_zarr.py --video`: the zarr outputs are written, then the run never returns at "encoding video". | `lasagna/labels_to_cc_zarr.py` (docstring usage) | `_encode_video` opens ffmpeg with `stderr=PIPE`, writes every frame to stdin, and reads stderr only after `wait()`. ffmpeg's banner fills the 4 KB Windows pipe, so both sides wait. Latent on Linux past 64 KB. | `fix/win-lasagna-video` (cbc1c4f), stderr to a temp file, +6 -8 plus an 86-line test | New test (fake ffmpeg writing 200 KB to stderr) hangs on main on any OS, passes on the branch in 0.13 s. Real PHercParis4 labels: main hung (killed at 60 s); branch exit 0 in 0.8 s, `cc.mp4` 494,405 B, h264, 64 frames. Main's mp4 had the same size, so the stuck step is Python waiting on ffmpeg. |
| D8 | `SHA256SUMS` ends every line in CRLF; `sha256sum -c` fails every row ("<path>\r: No such file"). | `vesuvius/scripts/collect_reviewed_fibers.py` (`collect` and `bake_test_set`) | `Path.write_text` in text mode without `newline`. | `fix/win-sha256sums` (7a9139d), `newline='\n'`, +3 -2 plus a test | `--audit-only` against dl.ash2txt.org/other/annotators (88 reviewed): main `sha256sum -c` 0 of 88 OK; branch 88 of 88 OK. `bake_test_set` covered by the unit test only. |

Each C++ fix was proven on fork CI: Windows build, package and smoke, Linux and macOS compile, CodeQL. Run ids are in `evidence/<name>/` and in each fix's summary. One CI flake (a `test_zarr_chunk_fetcher` temp-dir cleanup) hit D4's first attempt and passed on rerun. Fork PR #3 for D1 tripped the Large PR gate only because I opened it against a stale fork main (the real diff is 1 file).

No fix adds a regression test for the C++ ones. The Windows workflow runs only `ctest -R '^unified_browser_dialog$'`, so a new test would not run on the one platform where these fail. The before/after runs are the proof.

Earlier Windows fixes queued on 10-21, not part of this audit's runs: `win-crossframe`, `win-lasagna`, `win-gpupause` (#2215 is open for the last one).

## 4. Coverage

"D1" in the result column means the tool did its work and then hung at exit.

| Tool or command | Result |
|---|---|
| `--help` on all 51 `vc_*.exe` | Works. Tools with no `--help` handler read it as a path. `vc_tifxyz2rgb --help` aborts, 0xC0000409 (covered by open #2152; not Windows-specific). |
| `vc_grow_seg_from_seed` (README remote example; 8 and 50 generations; two seeds) | Works, exit in 10 to 13 s. Ends with `std::_Exit`, so D1 does not hit its success path. Error return does hang (D1 class). |
| `vc_render_tifxyz` (remote, local; tif, zarr, both; `-g 0/1/3`; `-n 1/3`; forward slashes; `--log-path`; non-ASCII paths) | Output correct. D1 hang on every remote run. Local non-ASCII run exits fine. |
| `vc_zarr_download_region` (dry and real) | Output complete (2 chunks, 49 MB). D1. |
| `vc_render_video` | Remote URL crashes at once (takes local paths only; not Windows-specific). Local mirror works (6.5 MB avi), then D1. |
| `vc_merge_tifxyz` | D2. With two different-seed segments, "real=0" edge drop (algorithmic, not Windows). |
| `vc_zarr_recompress` | D3 (local input). Remote input unaffected. `--one-shard` works. Resume works after the fix. |
| `vc_gen_normalgrids` generate / convert / pyramid | Grids written and renamed OK; generate has D1. convert and pyramid fine, with `--overwrite`. |
| `vc_objrefine` | Output correct, CRLF preserved, 338 faces in and out. Hangs at exit in the bridge run (D1 class). |
| `vc_tifxyz2obj`, `vc_obj2tifxyz`, `vc_obj2tifxyz_legacy` | Fine, including non-ASCII and spaces. obj2tifxyz into an existing dir refuses by design. |
| `vc_tifxyz`, `vc_tifxyz_trim`, `vc_tifxyz_selfcross`, `vc_tifxyz2zarr_sparse`, `vc_transform_geom`, `vc_seg_add_overlap`, `vc_cache_root`, `vc_tifxyz_mmap_prepare`, `vc_zarr_to_tiff`, `vc_obj_uv_lift`, `vc_rgb2tifxyz`, `vc_volpkg_convert` | Worked, including non-ASCII in and out paths and rename-over-existing. |
| `vc_ngrids` | `--vis-ply` on the full extent is very slow (killed); with `--crop` fine. PLY has CRLF line endings (cosmetic). `--fit-normals` throws "Failed to infer volume shape" (not Windows-specific). |
| Static scans | The `/`-only splits (`VolpkgConvert.cpp:34`, `Volume.cpp:57`) split remote URLs, so they are right. Text-mode IO is all text formats; binary files use `std::ios::binary`. `/proc`, `/dev/null`, `sysconf` sites have fallbacks. |
| Agent Bridge, about 60 methods (non-destructive) | Fine: ping, describe, state, catalog read-only, `project.create` (spaces, CJK, https, forward slashes), `volume.open/list`, `segments.attach/list/review/activate`, `tags.set`, `segment.recalc_area/crop_bounds/generate_mask/append_mask`, `render.tifxyz` x6 variants, `flatten.abf`, `points.*`, `viewer.*`, `screenshot.capture` (CJK path), `fiber.list`, `atlas.status`, `segments.rename/delete`, `segmentation.save`. Save-over, rename and delete of mapped dirs leave no residue. |
| Bridge, found | Case-only `segments.rename` is refused (section 5). `segment.refine_alpha_comp` and `segment.reoptimize` hang (D1) and leave their bridge jobs "running" forever, blocking later tool jobs with -32004. |
| Python `surface_preflight` | Works, atomic JSON write OK, exit 2 on a real gate failure. |
| `zarr_tasks` threshold, transpose, recompress, scale, zero-range `--inplace`, export-tifs | Work (spawn, `-n 2`). |
| `clean_labels`, `validate_segments` | Work. |
| `voxelize_objs` (trimesh, numba, dask) | Real merged OBJ to a 6-level label zarr. On Windows it takes the single-process branch because fork is unavailable; slower, handled. |
| `merge_predictions` | D5. |
| `create_label_zarrs` | D6. |
| `composite_from_zarr` (threads and processes), `prepare_9um_isotropic_input`, `download_required_zarr_chunks` (dry run, 1 and 2 workers, resume) | Work. |
| `lasagna/labels_to_cc_zarr.py --video` | D7. |
| `collect_reviewed_fibers.py --audit-only` (live annotators server) | D8; download, temp dir and publish rename work. |
| Other lasagna tools, spiral-fitting, `scripts/fiber_5class`, ink-detection subproject | Not runnable here (torch, cupy, numba-cuda or C++ extensions not installed). |

## 5. Not Windows-specific

Found in the same runs. Not counted in D1-D6.

- **zarr 3 crashes in `zarr_tasks`** (branch `fix/zarr3-tasks`, commits 34c1c651f and e8f483f32 on e2254ebe0). `edt-dilate`, `remap` and `resize` call `Group.create_dataset`, which zarr 3.2.1 removed. All three exit 1 on a 256^3 crop of PHercParis4, exit 0 on the branch. `remap` also dropped the input attributes; the second commit copies them (the real-data output now carries `multiscales`). Tests on zarr 3.2.1: main 3 failed, branch 23 passed. Not run on zarr 2.18.7. This follows the pattern of #2244 for merge. Evidence in `evidence/zarr3-tasks/`.
- **`zarr_tasks threshold` on a group input** hits a worker TypeError. The bot-closed #1431 already fixed this; not re-reported.
- **`vc_tifxyz2rgb --help`** aborts on an uncaught exception. Open #2152 covers it.
- **Case-only `segments.rename`** (for example `Zed` to `ZED`) is refused with -32010 "A segment named ZED already exists". Cause: `SegmentationCommandHandler.cpp` `renameSurfaceHeadless` (~5775) checks `exists(newPath)`, which is true on a case-insensitive filesystem; the same-id check is a case-sensitive string compare. It would also hit macOS. Workaround: rename through a temp name. Not fixed.
- **Paths over 260 characters.** The release exes embed a UTF-8 manifest (#1992) but no `longPathAware`. With `LongPathsEnabled=0`, the Windows 11 default here, every tool fails on a 284-character output path. `vc_tifxyz2obj` prints "Failed to open for write" and exits 0. Adding `longPathAware` is a plausible fix; I did not verify it and did not change the registry.
- **Run Trace with no `paths\` folder** in the project dies with an uncaught `filesystem_error`.
- Smaller items: `render.tifxyz` returns mixed separators in `outputDir`; `project.create "x.volpkg"` gives `x.volpkg.volpkg.json`; `flatten.slim` needs `flatboi`, which is not shipped; `vc_render_video` and `vc_ngrids` points above.

## 6. Relation to existing Windows issues

- **#1686 (open, ten tests hang on Windows).** D1 touches it directly. Part B of that issue describes the mechanism as an inference for test binaries and says apps may behave the same. I confirmed it for apps. I also ran its ten tests on fork CI (Windows, MSYS2 UCRT64, `ctest --timeout 120`, run ids 39834784317 control and 39834788524 fix, the same CI commit on both):
  - Control (72600fe): 10 of 10 time out.
  - With `fix/win-render-exit`: 9 finish, 8 pass (`chunked_plane_sampler_fallback`, `volume_live_s3` 26 s, `lasagna_project_volumes`, `lasagna_normal_sampler`, `atlas`, `volume_local`, `volume_extras`, `volume_pyramid`) and `test_zarr_chunk_fetcher` fails an assertion in 0.41 s. The run log of 39834788524 reads "80% tests passed, 2 tests failed out of 10" (one assertion failure, one timeout). The `test_zarr_chunk_fetcher` failure looks like a timing assumption: the test sleeps 2 ms per fetch inside a 15 ms window, and Windows sleeps are coarser than that.
  - `test_thread_pool_dispatch_lifecycle` still times out. That is the separate ThreadPool race (part A of the issue), not touched by my fix.
  - The new `test_zarr_chunk_fetcher` failure (`CHECK(result.requestedChunks > options.chunkCount)`, `test_zarr_chunk_fetcher.cpp:1026`) was hidden by the hang. It may be a Windows bug or a network or timing issue. I did not investigate.
- **#1979 / #1992 (Windows UTF-8 manifest).** Related, not touched. The manifest works: all C++ tools handled non-ASCII paths. D6 is the Python side of the same problem (Python `cv2.imread`). The long-path point in section 5 is a manifest follow-up.
- **#1839 / #1849 (UNC paths in Attach Segments).** Not touched. The fork CI I used is the project's own Windows workflow.
- **#1259, #1633, #1671, #1583, #1594.** I did not find a defect in these areas, and the fixes here do not change them, with one exception: #1671 (open, temp-file handling in `tifxyz_label_transfer/io.py`) is the same class as D5 in a different module. #2006 and #1724 (closed) were also that module. No overlap in files.
- **Others seen.** #1554 (merged) gave the sibling pools their `_WIN32` leak guards; D1 applies the same pattern to the registry from #1453. #2219 (open) touches both D4 files and adds more extensionless checks; no textual conflict, but they would need the same helper (noted in D4's PR text). #2101 and #2112 (open) have trivial text conflicts with D6. #2240/#2244 are the zarr 3 merge fix.

## 7. Limits and honesty

- **Not run:** GPU training, nnU-Net, the ink-detection subproject beyond the two entrypoints above, spiral C++, `lasagna.*` services, `predict`, `compute_st`, `find_patches` (torch extras not installed). Tools in D1's scope list that I only code-read: `FiberTraceEvaluationCli`, `vc_fiberlets`, `vc_create_segment_mask`, `vc_visualize`, `vc_lasagna_line_probe`, `vc_atlas_*`. A fix in `ChunkCache.cpp` should cover them but I did not show it.
- **D1 on `vc_render_video`:** hangs on the release, but I did not re-run `vc_render_video` on the fixed build. The fix is shown on `vc_render_tifxyz`, `vc_zarr_download_region`, `vc_objrefine`, the `vc_grow_seg_from_seed` error return and, by CI, on the test binaries. I did not run Linux; that Linux does not hang comes from code reading and the #1686 author's gdb data.
- **Derived inputs.** The label PNG for D6 (and the D5 TIFF inputs) come from published predictions of PHerc0500P2 20250919184428. The D6 PNG is a thresholded merge of two published predictions. It is not a published label. The local zarr is two public chunks, not a whole volume. Some tests (D5, D6, zarr 3) are synthetic and prove mechanism only.
- **D2** was shown with a segment merged with a copy of itself. Two segments from different seeds dropped the edge for algorithmic reasons, so merge quality on real overlapping surfaces is not claimed. Paths with `%` or non-ANSI characters through `cmd.exe` are not fixed by D2. VC3D's own merge action is not claimed fixed.
- **D3:** only the occupancy LIST is claimed. The resume LIST uses the same keys and is fixed with it. Recompress output is lossy by design; I do not claim bit-exactness.
- **D4** was proven through the Agent Bridge, offscreen, not by clicking in the GUI. The runner's own "Tool executable not found" text was not captured, so the proof is the differential (fail on main, pass on the fix).
- **D5 and D6** fixes were tested on zarr 3.2.1 and the locked uv environment, not across Python versions.
- **Python wrapper trap.** One of my own early probes misread D1 as a tooling artifact. I list that so a reader knows why handle-waits are in the method.
- **One machine, one network link.** Remote runs were slow here. Nothing about throughput is claimed.
- **Bars.** Probes 4 and 5 were killed against their strict bars. D4 and the case-rename item are in this audit because they were found along the way. The rename item is minor and I would fold it into any future Windows rename or save change rather than a PR of its own.
- **Novelty status: UNCHECKED.** No Windows audit repo was found in GitHub searches "vesuvius windows" and "volume cartographer windows" on 2026-10-22. This is not a claim of being first.
- **Nothing sent.** No issues, comments or upstream PRs. The only public items are branches and PRs on my own fork.

## Layout

- `evidence/<name>/before.txt`, `after.txt`, `tests.txt` for `win-render-exit` (D1), `win-merge-tifxyz` (D2), `win-recompress-list` (D3), `win-gui-tools` (D4), `win-merge-preds` (D5), `win-imread-unicode` (D6), `win-lasagna-video` (D7), `win-sha256sums` (D8) and `zarr3-tasks`.
- `evidence/probes/`: `WIN-VC3D.md` (probes 1 to 4), `WIN-BRIDGE.md` (probe 5), `WIN-PY.md` (Python probes), `CI-1686.md` (fork CI on #1686's tests).
- The fixes' `diff.patch` and `PR.md` files stay in `C:\Users\<user>\vesuvius\deliverables\<name>\` and are not copied here.

Drafted with AI assistance (Claude); the runs and fixes were reviewed by me.
