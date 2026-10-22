# WIN-BRIDGE (probe 5) - pre-registered kill bar, written before any run (2026-10-22)

Success = >= 1 NEW Windows-specific defect beyond D1-D4, reproducible on real public data through the VC3D Agent Bridge, with no issue/PR in any state.
Kill = 0 after ~90 min.
Platform-neutral bugs are listed separately and not counted.

## Run log (probe 5, ~90 min, fixed build gui-39832809875 = VC3D-0fb5a65, offscreen, origin/main e2254ebe0 for source)
Script: winvc3d/guiproof/bridge_probe5.py (phases all/c/d). Transcripts: winvc3d/guiproof/p5/run1 (sweep), run2 (rename/delete/remote), results.jsonl per run.
Data: copies of the real PHercParis4 mirror p3/vol.zarr, segA/segB from p3/paths, run1/out grown seg; all under dirs with spaces + u-umlaut + CJK.
PIDs started and ended by CIM Terminate: 29304, 6212, 18528 (run1); 25404, 10932 (run2); 27860 (run3). Discovery records removed. Nothing else touched.

### Calls that worked on Windows (no defect)
ping, rpc.describe, state.get, catalog.list_samples/describe_sample (read-only), project.create (spaces, CJK, https URL, fwd slashes), volume.open (incl. lowercased path, remote project), volume.list, segments.attach (idempotent by normalized path incl. fwd slash/UPPERCASE/case variants), segments.list/review/activate, tags.set, segment.recalc_area/crop_bounds/generate_mask/append_mask, render.tifxyz x6 (tif_stack, zarr, numSlices 3, groupIdx, relative/fwd-slash/no outputDir), flatten.abf, points.* json and segment-path round trips, viewer.*, screenshot.capture (CJK path), fiber.list, atlas.status, segments.rename/delete/segmentation.save on loaded segments (rename of a mapped dir, delete of an active loaded segment, edit+save-over via replaceDirectoryContents: all fine, no residue .tmp_/.vc-save-rollback), rename to Windows device names con/aux (accepted by VC3D and also deletable by VC3D; not a VC3D failure).

### Findings
Counted candidate (minor): C1 segments.rename case-only rename refused on Windows.
 - Call: segments.rename {segmentId:"Zed", newName:"ZED"} -> error -32010 "A segment named ZED already exists" (data.reason "target name already exists"); same on a fresh project.
 - Code: SegmentationCommandHandler.cpp renameSurfaceHeadless (~5775) `if (std::filesystem::exists(newPath)) return fail("name exists")`; newPath == currentPath on a case-insensitive FS, and the oldId==newId check is a case-sensitive string compare. Linux renames fine. (Also macOS.) GUI Rename Surface shares the core.
 - Prior art: gh issue/pr search "case-only rename", "case insensitive rename", "rename different case", "reserved name", "rename segment Windows": no hit about this. Open #2222/#2223 (saveWithUuid subfolders) touch the Windows save path but not rename. #2219 not on rename.
 - Severity: cosmetic limitation (workaround: rename via a temp name). Not a crash/data loss.

D1-class confirmations (not counted, same root cause as D1):
 - segment.refine_alpha_comp: vc_objrefine.exe finished (output x/y/z/meta written, "took 11.499 s") then never exits (PID 6212, 1 thread, 12 s CPU) on a LOCAL zarr; bridge job-10 stays "running" forever and blocks every later tool-source job with -32004 JOB_RUNNING. (D1 scope list had it code-read only.)
 - segment.reoptimize: vc_grow_seg_from_seed prints "ERROR: volume reports no usable voxel size" and `return EXIT_FAILURE` (vc_grow_seg_from_seed.cpp ~line 484, a plain return, unlike the _Exit success path) -> process never exits (PID 18528), job-11 stuck "running" forever. Same static-teardown hang on the error exit path.
 - Remote render via bridge works (volume.open remote project + render.tifxyz) but is slow (8 bands, >200 s on this link); not a defect.

### Platform-neutral things hit (not counted)
 - flatten.slim -> -32006 flatboi not found (external tool not shipped; lookup uses QStandardPaths::findExecutable so .exe is fine). Prior art not searched further.
 - flatten.straighten on a tiny 98x98 segment: vc_straighten "overlap-pairs: spacing estimate found too few cross-wrap neighbors" exit 2 (data-size dependent).
 - segment.recalc_area on a segment without mask.tif returns success:false "no mask.tif" (works after generate_mask).
 - segments.attach / project.create require absolute paths (relative rejected by design); project.create "x.volpkg" -> "x.volpkg.volpkg.json" and name "proj.volpkg.json2" quirks.
 - reoptimize/grow need voxel size (min_area_cm 0.3) -> error on the p3 mirror lacking voxel-size metadata; Run Trace paths\ filesystem_error known.
 - segments.create_editable_copy is catalog-only (-32009 on local segments); fiber.export with no fibers errors; lasagna.* need a service.
 - render.tifxyz outputDir returns mixed separators (".../t2\layers") when given forward slashes: cosmetic.

## Verdict
Strict reading (a defect of real consequence, as D1-D4 are): 0 -> KILL. Literal reading: C1 is a new, reproducible, Windows-specific, un-reported defect (low severity), so the bar is met only trivially. Recommend: do not spend a PR slot on C1 alone; fold it into any future Windows rename/save PR. The bridge surface was swept (~60 methods, all non-destructive paths) and the Windows-specific code (replaceDirectoryContents save, rename/delete of mapped dirs, Unicode/space paths, tool launch via the fixed build) holds up.
