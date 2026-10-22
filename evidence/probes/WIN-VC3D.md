# WIN-VC3D branch

## Pre-registered kill bar (written before any run)
Success = >= 1 Windows-specific defect (crash, hang, nonzero exit, empty/wrong output vs documented behaviour) reproducible on real public data, with no villa issue or PR in any state about it. 0 such defects after ~2 h = KILL.

(log below)

## Log so far (paused ~12:55)
Release: VC3D-5416c67-2026-10-22-win64.zip, extracted to C:\Users\<user>\vesuvius\winvc3d\app\...\bin (51 vc_*.exe).
| tool | command | exit | result | Windows-specific? |
|---|---|---|---|---|
| all 51 | --help | mostly 0 | tools without a --help handler treat it as a path; vc_tifxyz2rgb exits 0xC0000409 (uncaught std::runtime_error "Cannot open: --help\meta.json", terminate) | NO (uncaught exception on bad input, same on Linux) |
| vc_grow_seg_from_seed | README remote example, PHercParis4 URL, seed 1408 1408 1024, gens 8, thread_limit 2 | 0 | segment written, 1.41 cm2 | works |
| vc_render_tifxyz | -v URL -s seg --scale 1 -g 0 --cache-gb 1 -n 3 --tif-output + --zarr-output | outputs complete (3 tifs, zarr pyramid) | correct | works |
| vc_render_tifxyz | tif only, -g 0 / -g 3 / forward-slash paths | 0 | OK | works |
| vc_tifxyz2obj / vc_obj2tifxyz | non-ASCII + spaces paths ("t ëst ß 日本\seg ü1") | 0 | OK; re-run of obj2tifxyz into existing dir: "path already exists!" exit 1 | NO (deliberate guard, platform-neutral) |
Code scan: fs::rename sites exist, but build is MSYS2 GCC (libstdc++ replaces files); HOME falls back to USERPROFILE; "/dev/null" in vc_render_tifxyz.cpp:1891 is unused when no zarr output (no C:\dev artifact created by me).

## Open lead (UNRESOLVED, probably tooling artifact)
vc_render_tifxyz processes I started via PowerShell Start-Process/.NET Process stayed listed after finishing output (Win32_Process: 1 thread, ~500 handles, WS 0, taskkill "no running instance"; CIM Terminate cleared them). Two foreground runs (-g 0 with --zarr-output + --log-path; -g 1 with trailing-slash tif path) reported "TIMEOUT 300s" in my wrapper after printing 100%, but hangprobe runs (-g 3, -g 0) saw HasExited=true, code 0 at 5 s. Most likely zombie process objects kept alive by handles my host leaked, not a villa defect. Not proven either way. Do not count as a defect.

## Defects so far: 0

## RESUME FROM HERE
- Tools done: --help on all; grow_seg_from_seed; render_tifxyz (tif, zarr, log-path, fwd slash); tifxyz2obj; obj2tifxyz (unicode paths).
- Next: (1) resolve the open question: rerun the exact failing command (-g 0 -n 3 --zarr-output --log-path "log ü.txt") via cmd.exe /c with output to files (no handle leak) and check if it exits; (2) vc_zarr_download_region (needs --project *.volpkg.json; vc_volpkg_convert can make one), vc_zarr_to_tiff, vc_tifxyz (in-place save/rename over existing, QuadSurface.cpp:2586-2660), vc_tifxyz_trim (rename dance), vc_merge_tifxyz (rename 2214), vc_tifxyz2zarr_sparse, vc_render_video (ffmpeg DLLs).
- Helpers: C:\Users\<user>\vesuvius\winvc3d\runtool.ps1 (-exe NAME -a @(args) -timeoutSec N), hangprobe.ps1. Kill only by PID via CIM Terminate. Check free RAM >= 1500 MB first (it dipped to 330 MB once, from the user's load).
- Prior-art search (gh issue/pr list --state all) not yet done since no defect found.
- Kill bar: >= 1 Windows-specific reproducible defect with no villa issue/PR in any state; 0 after ~2 h = KILL. Elapsed so far ~1h20m of wall clock, 0 defects.
- Processes I started: all vc_* terminated (PIDs 14108, 36440, 14296, 17464, 31664, 32280, 18780 gone).

## RESUMED RUN: lead settled, FINAL VERDICT

### Defect D1: vc_render_tifxyz.exe never terminates after finishing (Windows release, remote OME-Zarr)
- Repro (plain bash, no PowerShell wrapper; real public data):
  `cd C:\Users\<user>\vesuvius\winvc3d; app\VC3D-5416c67-2026-10-22-win64\bin\vc_render_tifxyz.exe -v https://vesuvius-challenge-open-data.s3.amazonaws.com/PHercParis4/volumes/20260310173927-45.532um-11.0m-110keV-masked.zarr -s run1\out\auto_grown_20261022113620655 --scale 1 -g 0 --cache-gb 1 -n 1 --tif-output lead\sync1\tif`
  (segment grown by vc_grow_seg_from_seed from the README example, seed 1408 1408 1024, 8 gens).
- Result: 00.tif is written correctly (360x360) and the log reaches 100%, then the process never ends. The foreground command did not return for 60 s+ (background run for 20+ min) until I killed it. tasklist shows it alive with ~47 MB. Win32_Process: ThreadCount 1, ~500 handles, ~0 CPU (CPU counter unchanged over 5 s); the one thread is in a Wait state. taskkill /F says "no running instance"; only CIM Terminate (by PID) removes it. 100% of ~10 runs on this machine (tif-only, tif+zarr, +--log-path, -g 0/1/3, -n 1/3, forward-slash paths, cached and uncached).
- Why my earlier wrappers saw "exit 0": .NET/PowerShell Process.HasExited reads the exit code, which ExitProcess has already set while the process is stuck in DLL teardown. WaitForExit(ms) / handle-wait never signals. Anyone waiting on the handle (subprocess.wait(), bash `wait`, a foreground shell command, CI, batch scripts) blocks forever, and the leftover process sits in the table (each shows ~2.9 GB private commit in Get-Process).
- Other tools exit fine: vc_grow_seg_from_seed, vc_tifxyz2obj, vc_obj2tifxyz, vc_zarr_to_tiff, vc_tifxyz, vc_tifxyz_trim, vc_merge_tifxyz --help, vc_zarr_download_region (error path), all REAL EXIT in ~1 s. So it is specific to tools that use the process-wide ChunkCacheService (render tool).
- Code location: core/src/render/ChunkCache.cpp:731-735 `processChunkCacheServiceRegistry()` holds `static ProcessChunkCacheServiceRegistry registry` containing a shared_ptr<ChunkCacheService> (created by processChunkCacheService() line 785 / configureProcessChunkCacheService() line 796). On Windows with shared vc_core the destructor runs in DLL_PROCESS_DETACH after ExitProcess killed the scheduler worker threads, and ~ChunkCacheService waits for them. No `_WIN32` "leak instead of join" branch (the siblings got one in #1554: ChunkedPlaneSampler.cpp, SurfaceCache.cpp). Linux runs static destructors with live threads, so it does not hang there. apps/src/vc_render_tifxyz.cpp ends with `return EXIT_SUCCESS` (line ~2173) after remoteCache->waitForPendingChunks(); no explicit service shutdown.
- Windows-specific: yes (mechanism). I did not run Linux; the Linux non-hang is from code reading plus the #1686 author's Linux gdb data.

### Prior art (gh search, --state all, issues and PRs): PARTIAL, this is the catch
- Issue #1686 "Ten tests hang indefinitely on Windows, producing no output" (OPEN): part B of the comment by Bullo27 describes exactly this mechanism for 8 test binaries ("process-wide ChunkCacheService alive when main returns ... single thread, ~0 CPU"), flagged *inferred*, "needs Windows to verify", and says "apps may exit the same way on Windows (untested)". Related PRs: #1688 (CLOSED, only the ThreadPool part A), #1986 "Fix Windows ThreadPool shutdown hang" (CLOSED), #1687/#1312 (CLOSED). No PR or issue was found that fixes the ChunkCacheService static teardown, and none reports vc_render_tifxyz (or any app) hanging at exit. Searches: "vc_render_tifxyz hang", "does not exit", "hang on exit Windows", "DLL_PROCESS_DETACH", "processChunkCacheServiceRegistry", "leak rather than join", "ChunkCacheService exit Windows hang", "waitForPendingChunks".
- Related but different: #1611 (CLOSED, remote chunk streaming stalls), #1809/#1817 (failed fetch SIGABRT, merged).

### Verdict against the pre-registered bar
Bar text: ">= 1 Windows-specific defect ... reproducible on real public data, with no villa issue or PR in any state about it."
- Defect part: MET. D1 is a Windows-specific hang (process never terminates; exit code set, wait never returns) on the project's own release with real public data, reproduced ~10/10, mechanism located.
- "No issue or PR in any state" part: NOT cleanly met. #1686 (open) names the mechanism and speculates apps are affected, but only for tests, as an unverified inference. No tool-level report, no fix PR, no Windows confirmation exists.
- Strict reading: 0 qualifying defects, so KILL. Literal-bar reading counting #1686 as covering it: also not a pass. Honest call: the value left is a Windows confirmation comment on #1686 (apps too; here is the repro) and/or a small fix (leak the process-wide ChunkCacheService on _WIN32, or call an explicit shutdown before main returns / use quick_exit-style `std::_Exit` after flushing in the CLI tools). Both are outward actions and were NOT taken. The fix is plausible but I could not build it (rule: no builds), so it is unverified; do the prior-art check on #1686's current state before anything outward.
- Other results: tools table above stands; 0 other defects. vc_tifxyz2rgb / vc_zarr_to_tiff abort on bad args (uncaught exceptions, platform-neutral).
- Elapsed ~2 h.
- All processes I started are terminated by PID (14712, 34604, 19212, 30076, 33820, 33388, plus earlier ones); none left.

## Probe 2
Pre-registered kill bar (written before any run, started $(date)): success = >= 1 NEW Windows-specific defect beyond D1, reproducible on real public data, with no issue or PR in any state. 0 after ~90 min of runs = KILL.
Targets: vc_render_video, vc_merge_tifxyz, vc_zarr_download_region, untested exes; D1-class static-with-threads in shipped DLLs lacking a _WIN32 leak guard, reached by a shipped exe.
Hang test = wait on process handle from plain bash 60 s after final output.
(log below)

### Probe 2 log (~55 min of runs, all with plain-bash handle waits; files under C:\Users\<user>\vesuvius\winvc3d\p2)
Bar met: D2 below is a NEW Windows-specific defect, real public data, no issue/PR in any state.

| tool | command (abridged) | result |
|---|---|---|
| vc_grow_seg_from_seed | README example, params gens=50 (p2/params50.json), seeds 1408 1408 1024 and 1500 1440 1024 | exit 0 in 10-13 s, 78.5 cm2 each. Exits cleanly: it ends with std::_Exit (vc_grow_seg_from_seed.cpp:1563,1646), so the D1 class does not hit it |
| vc_zarr_download_region | --project min.volpkg.json (file containing `{}` is enough) --url <PHercParis4 zarr> --level 1 --zmin 1000 --zmax 1100 --ymin 1300 --ymax 1500 --xmin 1300 --xmax 1500 (-n dry run and real) | output complete ("Done. requested=2 cached=2 ..."), 2 chunks (49 MB incl. metadata) in C:\Users\<user>\.VC3D\remote_cache, then the process NEVER EXITS: dry run PID 34456 and real run PID 26752 both alive >100 s, 1 thread, 332 handles, CPU frozen. Terminated by PID. Same D1 signature (main returns, static ProcessChunkCacheServiceRegistry destroyed in DLL_PROCESS_DETACH). Reached via configureProcessChunkCacheService, vc_zarr_download_region.cpp:332 |
| vc_render_video | <remote URL> out.avi seg: crashes at once (std::filesystem_error "directory iterator cannot open directory": Volume(path) takes a local path only; platform-neutral, tool never supported URLs). On the local mirror dir from the download step + the grown segment | works (FFMPEG backend, 6.5 MB out.avi), then NEVER EXITS (PID 10520, same signature). Also writes col.tif into the CWD (neutral) |
| vc_merge_tifxyz | two overlapping segments from two seeds: edge dropped ("real=0", surfaces not same-sheet/normals filter; algorithmic, neutral). Then a segment merged with a copy of itself (vp3/paths segA, segB; merge.json {"rows":["segA segB"]}) | runs steps 0-6 (17 s) then FAILS exit 1: "rasterizing OBJ -> ..." / "The system cannot find the path specified." / "vc_obj2tifxyz_legacy failed (rc=1)"; no tifxyz written; stray vp3/paths/segA_merged.obj left behind. Fails identically with default sibling lookup, with --obj2tifxyz given as absolute .exe path (forward-slash and backslash forms), and with the bin dir on PATH |

### D2 vc_merge_tifxyz can never finish on Windows (std::system quoting)
- Code: apps/src/vc_merge_tifxyz.cpp:1521-1538 rasterizeObjToTifxyz builds `"exe" "obj" "out" 20` and calls std::system. On Windows system() runs `cmd.exe /c <string>`; the string starts with a quote and ends with a digit, so cmd's quote-stripping rule removes the first and the LAST quote on the line and mangles the command. Direct proof (p2 .bat): `cmd.exe /c "exe" "obj" "out" 20` (the villa string) -> "The filename, directory name, or volume label syntax is incorrect." rc 1; the same with one extra outer pair of quotes `cmd.exe /c ""exe" "obj" "out" 20"` -> rc 0 and a valid tifxyz (x/y/z.tif, meta.json); calling the exe directly -> rc 0.
- Second, separate Windows bug, by code reading only (masked by the first): resolveObj2Tifxyz (line 1556-1565) tests fs::exists(self.parent_path()/"vc_obj2tifxyz_legacy") with no ".exe", false on Windows, so the sibling is never found and the code falls back to PATH lookup.
- Fix shape: wrap the command in an extra pair of quotes on _WIN32 (or use CreateProcess/ _wspawnv / std::filesystem + QProcess-free spawn without a shell), and append ".exe" in the sibling probe. Not built or pushed.
- It also breaks VC3D's merge GUI (#891) on Windows since that shells out to this tool (not run).
- Whole merge compute (minutes on real surfaces) is discarded at the last step; exit code 1.
- Only std::system site in shipped apps/core/utils (tests have 3 more, neutral). utils/src/aws_auth.cpp popen: not exercised.
- Prior art (gh --state all, issues+PRs, queries: vc_merge_tifxyz, obj2tifxyz_legacy, std::system, cmd.exe, "merge Windows", rasterizeObjToTifxyz, "system cannot find the path"): nothing on this. #1312 (closed, Windows tests) touched CommandLineToolRunner only; #1978/#1977 are obj2tifxyz_legacy scale, unrelated. Open #2232/#2161/#2225 touch ChunkCache/volcodec/GPU render, not merge_tifxyz.

### D1 scope extension (not a new root cause)
vc_zarr_download_region and vc_render_video hang at exit exactly like vc_render_tifxyz. Static scan: the unguarded process-wide static with owning threads is only ProcessChunkCacheServiceRegistry (ChunkCache.cpp:733); the thread pools in ChunkCache.cpp:453/470, ChunkedPlaneSampler.cpp:187/207, SurfaceCache.cpp, PersistentZarrCacheBudget.cpp:254 already have _WIN32 leak guards; g_logFlushThread in vc_render_tifxyz.cpp is joined by a guard object. Other exes that reach processChunkCacheService without _Exit: FiberTraceEvaluationCli, ObjAlphaCompRefinement, vc_fiberlets, vc_gen_normalgrids, vc_create_segment_mask, vc_visualize, vc_lasagna_line_probe, vc_atlas_* (code reading only, not run). A fix in ChunkCache.cpp (leak the registry on _WIN32, same pattern as the pools) covers all of them; deliverables/win-render-exit already does that for the render tool.

### Verdict: BAR MET by D2 (one new, independent defect). Processes started and ended by PID: 34456, 26752, 10520 (CIM Terminate); none left. Downloads ~50 MB.

## Probe 3
Pre-registered kill bar (written before any run, started 2026-10-22 PM): success = >= 1 NEW Windows-specific defect beyond D1/D2, reproducible on real public data, with no issue or PR in any state. 0 after ~90 min = KILL.
Step A static scan (fs::rename over existing, text-mode binary IO, '/'-only splitting, /tmp, HOME, popen, mmap, UTF-8 path from std::string, open handles during delete). Step B runs on top candidates + shipped vc_*.exe not in tables. Step C prior art.
(log below)

### Probe 3 log (~65 min; files under C:\Users\<user>\vesuvius\winvc3d\p3)
Step A candidates (static, origin/main): vc_zarr_recompress.cpp list_chunks (fs::relative(..).string() then rfind('/') key parsing, lines ~157-159 / ~547); vc_gen_normalgrids.cpp:1641 fs::rename tmp->out; vc_tifxyz_mmap_prepare.cpp:206 rename over existing; vc_rgb2tifxyz.cpp:51; ObjAlphaCompRefinement.cpp:224 text-mode OBJ copy; vc_ngrids.cpp PlyWriter text mode (CRLF in ASCII PLY); VolpkgConvert.cpp:34 and Volume.cpp:57 '/'-only splits (not run); RemoteCacheSettings.cpp:88 HOME (has fallback). MemMap.cpp and aws_auth.cpp popen have Windows branches.
Step B runs (all with the local mirror p3/vol.zarr = copy of the 49 MB public PHercParis4 chunks from probe 2):
| tool | result |
|---|---|
| vc_gen_normalgrids generate (-g xy, --sparse-volume 32, with/without --verify-grid-save, preview) | grids written and renamed OK, fresh dirs; hangs at exit = D1 (killed by PID 4484, 28964, 28076) |
| vc_gen_normalgrids convert / pyramid (+--overwrite) | OK |
| vc_ngrids --vis-ply (full extent) | very slow (CPU-bound, killed PID 19864); with --crop OK; PLY is written with CRLF line endings (text mode), cosmetic |
| vc_ngrids --fit-normals | uncaught runtime_error "Failed to infer volume shape" (neutral) |
| vc_tifxyz_mmap_prepare (dry, real rewrite via LZW tif, rerun) | OK, rename over existing works |
| vc_tifxyz_selfcross, vc_cache_root, vc_seg_add_overlap, vc_transform_geom, vc_tifxyz2zarr_sparse | OK |
| vc_zarr_recompress local zarr -> zarr | DEFECT D3 below |

### D3 vc_zarr_recompress: local-input run silently produces an EMPTY output on Windows (exit 0)
- Repro: `vc_zarr_recompress.exe p3\vol.zarr rc.zarr --jobs 2 --inner-jobs 2` where vol.zarr is a local copy of public PHercParis4 chunks (level 1 has 2 chunks, 1/3/5/5 and 1/4/5/5). Every level prints "Occupancy LIST: 0 / 32768 chunks present (100.0% sparse)", "Processed: 1 shards, 0 chunks (zero: 0, skipped: 4096)", exit 0, and the level-1 shard is a 65536-byte empty index (rc.zarr/1/c/0/0/0).
- Control (same input, same exe, skips the occupancy LIST): `--levels 1 --one-shard 1/0/0/0` -> "2 chunks (zero: 4094)", 738,566-byte shard. So the data is readable; only the occupancy LIST finds nothing.
- Cause (code reading): LocalIO::list_chunks returns `fs::relative(entry.path(), root).string()`, which has backslashes on Windows; the occupancy parser (vc_zarr_recompress.cpp ~547-552) splits keys with rfind('/'), finds no separator, `continue`s for every key, so the mask is all false and every chunk is skipped as absent. The resume LIST uses the same keys. Fix shape: `.generic_string()` in list_chunks. Not built.
- Impact: silent data loss, no error, exit 0. Only local-path input is affected (S3/https backend builds keys with '/').
- Prior art (gh --state all, issues+PRs; queries vc_zarr_recompress, "zarr_recompress Windows", "recompress occupancy", "list_chunks backslash", "LocalIO list_chunks", "recompress local input", "Occupancy LIST", "backslash path separator", "generic_string", "native separator zarr key"; plus all PRs listing vc_zarr_recompress.cpp among files): nothing about this. Hits are Python zarr_tasks recompress (#2186/#2199/#2156/#2069/#1670/#1711, different code) and PRs merely touching the file (#832, #1554 merged; #2232 draft volcodec_ng), none about key separators.

### Probe 3 verdict: BAR MET by D3 (new, Windows-specific, real public data, no issue/PR found). Processes started and ended by PID: 4484, 28964, 28076, 19864 (CIM Terminate); none left.

## Probe 4
Pre-registered kill bar (written before any run, started 2026-10-22 15:08): success = >= 1 NEW Windows-specific defect beyond D1-D3, reproducible on real public data, with no issue or PR in any state. 0 after ~75 min = KILL.
Targets: '/'-only splitting (vc_rgb2tifxyz:51, VolpkgConvert:34, Volume:57), ObjAlphaCompRefinement:224 text-mode OBJ copy, other non-binary ifstream/ofstream of binary formats, non-ASCII paths.
(log below)

### Probe 4 log (~60 min; files under C:\Users\<user>\vesuvius\winvc3d\p4; data = p3 local copy of public PHercParis4 chunks + segment grown from README seed)
Static findings: the "/"-only splits flagged in probe 3 are NOT defects. VolpkgConvert.cpp:34 and Volume.cpp:57 split REMOTE URL/prefix strings (so '/' is correct); vc_rgb2tifxyz.cpp:51 is a rename-or-copy helper, no split. The text-mode IO list is all text formats (JSON, OBJ, PLY ascii, shard-file list); binary spool/zarr/chunk files are opened std::ios::binary. /proc/meminfo, /dev/null, sysconf sites all have _WIN32 or non-Linux fallbacks (vc_lasagna_maxflow_graph uses "NUL").
| tool | test | result |
|---|---|---|
| vc_volpkg_convert | local dir "Scan ü" relative and absolute, output "out ä\*.volpkg.json" | exit 0, names OK, locations written with backslashes ("..\\Scan ü\\paths"); resolveLocalPath uses fs::path so consumers are fine; no defect |
| vc_objrefine | real OBJ from vc_tifxyz2obj (non-ASCII dir), local zarr | exit OK (no hang observed), 196 v/vt/vn, 338 f, 926 lines in and out, non-vertex lines identical, CRLF preserved; no defect |
| vc_render_tifxyz | local non-ASCII volume + segment + tif out + --log-path "log ö.txt" | OK |
| vc_rgb2tifxyz | non-ASCII png/bounds/out/name | OK (cv::imread handles UTF-8 via the activeCodePage manifest; only Python cv2 fails there) |
| vc_tifxyz_trim, selfcross, tifxyz2zarr_sparse, transform_geom (obj + tifxyz), vc_tifxyz -r, zarr_to_tiff, obj_uv_lift, obj2tifxyz_legacy | non-ASCII in/out paths | all exit 0 with correct outputs |
| vc_tifxyz2obj, vc_transform_geom | OUTPUT path of 284 chars (6 x 40-char dirs under p4), LongPathsEnabled=0 | tifxyz2obj prints "Failed to open for write" and EXITS 0 (neutral: vc_tifxyz2obj.cpp ofstream check returns void); transform_geom exit 4 "cannot create directories" |

### Candidate C1 (weak, NOT counted as a pass): no tool opts into long paths
- The release exes embed the UTF-8 manifest (cmake/windows/utf8.manifest, #1992) but no <longPathAware>; with LongPathsEnabled=0 (the Windows 11 default here) every tool fails on any path > 260 chars. Repro above, real data, deterministic. Fix shape: add longPathAware to the manifest (plus users need LongPathsEnabled=1; fix NOT verified, I will not change the system registry).
- Prior art: gh issue/pr --state all for "long path", "MAX_PATH", "longPathAware", "260 characters", "path too long": nothing relevant.
- Why not a pass: environment-dependent, fix unverified, impact limited to deep paths; also the exit-0-on-failed-write in tifxyz2obj is platform-neutral (related closed PR #1389 "Report failures in the exit code in three CLI tools").

### Probe 4 verdict: KILL by the pre-registered bar (0 clean new Windows-specific defects; 1 weak candidate C1). Hang at exit seen in tools reaching the chunk cache = D1, not re-reported. Processes started by me: all exited or terminated by PID (29636, 6364 gone); none left. No downloads.
