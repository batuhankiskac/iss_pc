# Cloud audit, round 6: the open player issues

This round starts from players' reports, not from the code. It reads every open issue on the project's public tracker, sorts them into clusters, and traces each cluster to code where the evidence allows. For every cluster it says what causes it, whether our branch still has the problem, and what the fix is.

**Tracker and branches.** This repository (`batuhankiskac/iss_pc`) has no issues of its own. Players report through the launcher's "Report a problem" button, which opens issues at [HayatoG/iss_pc](https://github.com/HayatoG/iss_pc/issues). That is the project this repository mirrors: our `main` (`fac17b4`) is upstream `developer` at `88e912f` (2026-10-23) plus 71 of our own commits.

**Snapshot.** There were 326 open issues on 2026-10-26, numbered #7 to #703. All 326 were read. Nothing was posted to GitHub, and no code was changed.

**Method.**
1. Every issue page was read, comments included. Attached logs were downloaded where the page linked them, about 30 logs.
2. One structured record was extracted per issue: build, renderer, GPU, exit code, signature line, maintainer replies.
3. Records were clustered by signature.
4. Each cluster was traced in the code at `fac17b4` and checked against the release history with `git merge-base`. Every `file:line` below was read from the file; earlier audit items are cited as R*n*-*x*.

Paths are relative to `port/` unless they start with `docs/`.

| Build | What it is |
|---|---|
| `cde38af` | release **v1.0** (2026-10-22). Most reporters run it. Renderer v2 has been the Windows default since `da39b0b` (2026-10-21). |
| `da39b0b`, `d3995fe`, `27f8bba`, `a88bbc1`, `9c24de1` | older releases (2026-10-16 to 2026-10-21) |
| `88e912f` | upstream `developer` HEAD, beta-2026.10.24. It carries fixes made after v1.0. |
| `fac17b4` | our `main`: `88e912f` plus 71 commits (performance work, diagnostics, lean defaults) |

How to tell the renderers apart in a v1.0 log: `built Oct 22 2026 07:01:31` is `tlg-probe.exe` (renderer v1), and `07:07:40` is `tlg-probe-v2.exe`.

## Summary

| Kind | Count |
|---|---|
| Crash | 100 |
| Performance | 61 |
| Freeze | 51 |
| Feature request | 24 |
| Visual | 18 |
| Black screen | 14 |
| Stutter | 10 |
| Launcher | 8 |
| Input | 7 |
| Setup | 5 |
| Audio | 2 |
| Info / other | 26 |

Every issue is in exactly one cluster (Appendix A). The main conclusions:

1. **Many of the most-reported blockers are already fixed on `developer` but not released.** These clusters total about 40 issues:
   - **The freeze when absorbing a power (20 issues).** Fixed by `8471638`; one reporter confirmed it on the beta.
   - **The overlay crash at `tlg-probe-v2.exe+0x177b4e` (8 issues).** Fixed by `50607e5`.
   - **The #632 deadlock.** Fixed by `3dd8887`.
   - **The freeze report never firing under renderer v2.** Fixed by `0f7d045` and, in our branch, `cfdbabf`.

   A release from `88e912f`, or better from our branch, closes most of them. Each fix is listed in [section 1](#1-fixed-after-v10-not-released-yet).
2. **New causes traced to code that are still present in `fac17b4`:**
   - the Polaris driver crash (#31 and its 35 closed duplicates);
   - a crash at load on cards without `VK_EXT_vertex_input_dynamic_state`;
   - the `0x800da08c0` boot crash;
   - mod `mask` patches written 4 MB too low (#653, #213);
   - the flicker that came back with renderer v2;
   - the renderer v1 FPS collapse after warm-up;
   - the problem report attaching another person's log;
   - a key-binding bug, the 01.00 import stop, and the PSN request hang.
3. **Several big clusters cannot be diagnosed from the reports, because of how the reports are made.** The report keeps only `Fault|STOP|FREEZE` lines, drops stacks and exit codes, and uncaught C++ exceptions are swallowed before they can say what they were ([section 4](#4-diagnostic-gaps-that-hide-the-causes)). These clusters are:
   - the guest debug exception `0xa0020013` (10 issues);
   - the uncaught C++ exceptions (7);
   - the dark world (10);
   - the crashes with no log line (18).

   Fixing the diagnostics is the cheapest way to resolve them.

## Ranking

The clusters are ranked by players affected × severity. In the Status column:

- **released** means fixed in v1.0 or earlier;
- **fixed, unreleased** means fixed after v1.0, in `88e912f` and `fac17b4`;
- **present** means the problem is still in `fac17b4`.

| # | Cluster | Issues | Root cause | Status | Confidence |
|---|---|---|---|---|---|
| F1 | Freeze or white screen when absorbing a power or core relay | 20 | Guest Wwise condition wait on a mutex it does not hold; the host wait returned still holding it | fixed, unreleased (`8471638`) | high |
| C4 | Crash `read 0x101 at tlg-probe-v2.exe+0x177b4e` (overlay, saves, Seattle) | 8 | 4 FPS labels for 5 FPS values, read out of bounds | fixed, unreleased (`50607e5`) | high |
| D1 | Polaris/GCN crash `write 0 at amdvlk64.dll+0xebbbb` after the logos (renderer v1) | 12 (+35 closed) | 9,217 image barriers in one `vkCmdPipelineBarrier2`; barrier merging is opt-in | present | high |
| C3 | `execute access to 0 at 0` at load with native passes on (RX 580) | 4 | `setVertexInputEXT` called on devices without the extension (null function pointer) | present | high |
| C5 | Boot crash `read 0x800da08c0` | 5 | Draw-distance thread reads the image during the whole-image `PAGE_NOACCESS` window | present | high |
| R3 | Flickering smoke, fog, lights and fire; single-frame flashes | 8 | Renderer v2 uses upstream's wave64 pass without v1's inFAMOUS fixes | present | high (gap), medium (all reports) |
| P1 | Renderer v1 FPS collapses when "loading shaders" reaches 100% (the Oct 19 regression) | 18 | `e3f7e99` removed the early return for lazy programs, so a 5-10 min background stage load runs | present for v1 (Linux default); avoided on Windows by v2 | high (trigger) |
| L4 | Mods not applied or patched wrong; Paper Trail blocked; v1 cache race | 6 | `mask` offset bug; HTTP wait never returns; unlocked counter | present | high |
| C1 | `STOP: guest raised debug exception code=0xa0020013` | 10 | Raised by the game itself; no caller is recorded | present, cause unknown | low |
| C2 | `Fault: exception 0x20474343` (uncaught C++ exception) | 7 | The fault filter swallows it before `std::terminate` | reporting gap present, cause unknown | high (gap) |
| R1 | World goes dark after a story point and stays dark in that save | 10 | Save-dependent; not a port cache. CPU exposure from GPU tiles is the leading suspect | present, cause unknown | low |
| F3 | Cutscene picture freezes, long stalls | 33 | v1: pipeline compiles on the GPU thread. v2: mostly F1; long v2 stalls unexplained | mostly fixed | medium |
| M1, C6 | Out of VRAM, thrashing and FPS decay on 4-6 GB cards and iGPUs | 15 | 512 MB texture floor, arena never shrinks or spills, out-of-memory on residency is fatal | out-of-memory crash fixed in v1.0; thrashing present | high |
| L1 | The problem report attaches another person's log | 4 | The newest `run-*.log` in `out/logs` is preselected whoever wrote it | present | high |
| L3 | Game 01.00 stops at `FMUirmwfWxg#dlcldr`; EU features off; language locked | 12 | 01.00 only gets a warning; `dlcldr.prx` is never linked; title-ID gates | present | high |
| I1 | Input: `r2=none` also unbinds the mouse; mouse too slow; pad lag | 10 | `apply_button_map` removes mouse sources; mouse drives the stick; 15.6 ms window poll | present | high / medium |
| D2 | RDNA read faults in amdvlk64.dll at startup (renderer v1) | 10 | Driver fault during concurrent pipeline creation; wave stats ruled out | avoided by v2 default | medium (scope) |
| F2 | #632 deadlock; freeze report blind under v2 | 2 | Fault-streak path under the texture lock; zeroed frame counters | fixed, unreleased (`3dd8887`, `0f7d045`/`cfdbabf`) | high |
| L2 | Launcher setup and UX | 13 | Detection shows one problem at a time; pad probe on the UI thread; shared settings file; garbled Portuguese | present | high / medium |
| F5 | GPU hang at the intro hatch | 1 | A self-sampling draw without the feedback-loop dynamic state | present | low-medium |
| C8 | Guest address-range reservation fails (exit 21) | 1 | Something already occupies part of 64 GiB-1008 GiB | present | medium |
| R4 | `output_res` 4K breaks character lighting; "Fast" darkens; BC 1D | 5 | Proxy/native resolution mixing; Fast ignored under v2 | present | medium |
| R2, F4, C7, C9, D3 | Black screen at 60 FPS; main thread spin; small-address guest faults; crashes with no log; other drivers | 40 | Mostly unsupported dumps, below-target GPUs or missing data | needs data | low |
| P2, P3, A1, X1-X3 | RX 580 FPS, general performance, audio, requests, empty reports | 72 | No defect traced | — | — |

---

## 1. Fixed after v1.0, not released yet

These fixes are on `developer` (`88e912f`), or only on our branch where noted. Nothing more is needed in code: ship them, then ask the listed reporters to retest.

| Fix | In | Closes | What it fixes |
|---|---|---|---|
| `8471638` (PR #637) | `88e912f` | **F1:** #646 #652 #662 #674 #695 #698 #584 #575 #576 #643 #559 #543 #480 #665 #544 #507 #534 #623 #442 #586 | Wwise's job wait (guest `0x803e90`) unlocks a mutex and then condition-waits on it. In v1.0, `wait_cond` called `pthread_cond_wait` anyway, returned holding a mutex the guest never unlocks, and the next lock blocked forever. The absorb effect's white flash stays on screen. Now it returns `EPERM`, as the PS4 does (`src/runtime_mutex.c:236`). #507 confirmed the fix on the beta. #586's thread list shows the deadlock directly. |
| `50607e5` | `88e912f` | **C4:** #367 #502 #537 #556 #608 #677 #476 #605 | In v1.0 `tlg_overlay.cpp:499` had `fps_labels[] = {"30","120","240","Unlimited"}` for `FpsLimits = {30,60,120,240,0}`. "Unlimited" (index 4) read past the array, giving `ImGui::RenderTextClipped` a 0x101 pointer (#367 symbolized it). The labels were also shifted, so 60 showed as "120". Now there are 5 labels and a `static_assert` (`gpu/shim/tlg_overlay.cpp:518-520`). |
| `3dd8887` (PR #617) | `88e912f` | **F2:** #632 | The texture fault-streak path hashed GPU-protected pages under the texture-cache lock. The nested fault waited for the CP, and the CP waited for the lock. The default is now off (`gpu_v2/gpu/overrides/texture_cache.cpp:424-431`). The per-page variant remains: R4-7, below. |
| `0f7d045`; `cfdbabf` (ours) | `88e912f`; `fac17b4` | **F2:** #451; every v2 "no FREEZE line" report | In v1.0, v2's `tlggpu_frame_counters` was `*out = {}` (`cde38af:gpu_v2/gpu/gpu_host.cpp:139-141`). The freeze monitor arms only on the first counted frame (`src/runtime_freeze.c:169-170`), so it never fired under v2. #662 waited 30 minutes with nothing logged. `cfdbabf` adds full counters and Frame stats lines (`gpu_v2/gpu/gpu_host.cpp:180-187`, `overrides/videoout_driver.cpp:419-422`). |
| `28be8d4` + `f0d64fb` (ours) | `fac17b4` | #632 (side report) | The upload worker's 1 ms `wait_for` was rounded down by libc++, so it spun about 95% of a core. Now `ISS_UPLOAD_IDLE=sleep`. |
| `23b8771`, `a367a0d`, `f0d64fb` (ours) | `fac17b4` | #572, likely #607 #579 | About 90,000 unbuffered `BindBuffers: Clamped size …` lines on the CP thread after about 25 minutes (FPS 40 → 16). Now minimal and buffered logs by default. |
| `ca1aab1` (ours) | `fac17b4` | C2 candidate (#333) | The pipeline checkpoint no longer copies and hashes 54-250 MB every 30 s when no pipeline was created. That copy is a plausible `bad_alloc` source on 7 GB machines. |
| `b0669b8` (ours) | `fac17b4` | #160 #237 (Linux) | The update check failed on Linux because the bundled OpenSSL looked for CA certificates in the wrong place. |
| `96103ef` (ours) | `fac17b4` | #610 #439 #435 (partly) | Separate vertical sensitivity, sliders up to 10. It cannot lift the cap described in I1. |

Already released (in v1.0), so these reports are stale:

- #424, #475: out-of-memory on arena migration. Fixed by `990038b`, `9c4082b`, `bc6a98d` and `3f4b62e`; #424 confirmed.
- #28: save write error, "disc corrupt" dialog. Fixed by `98c6b9a` and `6b668a0`.
- #75: 137 MB log from the AMD 1D trace. Capped by `a43e085`.
- #169: backports ship; BP08 on by default since `a85781c`.
- #125: swipe emulation works.
- The d3995fe FPS regression on Windows (P1): avoided because v2 is the default.

---

## 2. Present in `fac17b4`, cause traced

### D1. Polaris/GCN driver crash: 9,217 image barriers in one call (#31 and 35 closed duplicates)

**Issues:** #31, #91, #94, #119, #129, #133, #203, #385, #603, #655 (`amdvlk64.dll+0xebbbb`), #315 (RX 550, older driver: `+0x48ed8`), #538.

**Evidence:**
- All are renderer v1, right after the logos, on RX 470/550/560/570/580 and the 5600GT iGPU.
- The v1.0 log of #603 and the log of #315 both end with `FlushBarriers: 9217 image barriers`, then `pipelineBarrier2 start`, then the fault, with no "done" line.
- 9,216 of those barriers are single-subresource TransferDst→ShaderRead barriers on one image with 1,536 layers × 6 mips.

**Cause:**
- After a partial transition, `Image::Transit` emits one barrier per mip × layer (`gpu/shadps4/video_core/texture_cache/image.cpp:409-470`, the per-subresource push at `:441`).
- The merge that would fold them is gated on `MergeBarriersEnabled()` (`:516-517`), which accepts only exactly `"1"` (`barrier_merge.h:13-15`).
- The whole list goes to the driver in one call (`gpu/shadps4/video_core/renderer_vulkan/vk_runtime.cpp:985-1044`).

**Why the RX 580 profile helps:** it is the only preset with `TLG_BARRIER_MERGE='1'` (`launcher/presets_rx580.py:16`; the default `presets_current.py:40` sets `'0'`). #385 ran more than 10 minutes without the crash with merging on.

**Fix:**
1. Turn barrier merging on by default. It merges only barriers with the same stages, access and layouts and adjacent subresources, so it is semantically identical. Keep `TLG_BARRIER_MERGE=0` as the opt-out.
2. Split `FlushBarriers` into calls of at most about 512 image barriers.
3. Renderer v2 has the same per-subresource walk (`gpu_v2/gpu/overrides/image.cpp:320-382`) and no batching (`gpu_v2/.../vk_runtime.cpp:935-958`). Apply the same split there; it is latent, since no v2 report shows it yet.

**Risk:** low. **Confidence:** high.

### C3. `execute access to 0 at 0` at load with native passes on (#694, #472, #627, likely #660)

**Evidence:** all are RX 580 2048SP with native passes on. #694 runs fine with them off. Its log shows `Extension VK_EXT_vertex_input_dynamic_state unavailable.` about 20 s before the fault, and `rsp` shows a call had just pushed its return address.

**Cause:**
- `cmd.setVertexInputEXT(bindings, attributes)` is called unconditionally in the native-pass `Record` (`gpu_v2/gpu/native_passes_rasterizer.inl:1258`) and in `RecordBatch` (`gpu_v2/gpu/native_recorder.inl:632`).
- Only `NativePassExecute` rejects devices without the extension (`native_passes_rasterizer.inl:1594-1600`, `!instance.IsVertexInputDynamicState()`). The pool records earlier, at submit time.
- Without the extension, the dispatcher entry is null, so the call jumps to address 0.

**Fix:** in `NativePassesSetup`, check `instance.IsVertexInputDynamicState()` and stay off with a log line. Guard `RecordBatch` the same way. Have the launcher turn native passes off on such GPUs.

**Diagnostic:** for a fault at rip 0, print `describe_address(*(uintptr_t*)Rsp)`, the caller.

**Risk:** none. **Confidence:** high for #694, #472 and #627.

### C5. Boot crash `Fault: read access to 0000000800da08c0` (#618, #583, #288, #115; maybe #489)

**Evidence:** the same address shows up on both renderers (`tlg-probe.exe` and `tlg-probe-v2.exe`), on US 01.07, EU and 01.00. `rcx=0x800000000` is the image base, and in #115 the fault comes right after the `Red zone:` line, before the first frame. `0x800da08c0` is `image + TLG_VIEW_LOD_TABLE` (`src/view_hook.h:17`).

**Cause:** a race at boot.
1. `install_view_tuning` (`src/probe.c:747-775`) calls `tlg_view_attach`, which starts a detached thread that runs `Apply()` every 100 ms. `Apply` reads and writes the LOD table through `image` (`gpu/shim/tlg_view.cpp:23-29,66-71`).
2. Later, the main thread makes the **whole** image `PAGE_NOACCESS` (`src/probe.c:1731`), then restores each segment's protection one by one (`:1732-1734`).
3. If the thread's tick lands in that window, its read faults.

The image is about 2 GB here (`rdx=0x7f000000`), so `VirtualProtect` on it is slow on low-RAM and older machines. That matches who reports it: 8 GB laptops, Xeon E5 v4, Ryzen 2200G, RX 580 systems.

#489 (`read 0x8003f3eff` in `iss_gpu_install_image`, `gpu_v2/gpu/bridge.cpp:506`) may be the same window. Low confidence.

**Fix:** start the draw-distance thread only after the final protection pass (`boot_step("image mapped and relocated")`, `src/probe.c:1740`). Better, apply the protections without the whole-image `NOACCESS` step: protect only the gaps between segments.

**Also check:** whether the LOD table's segment is writable. Round 5 item 22 noted that `install_view_tuning`, unlike the mouse installer, does not require a writable segment.

**Risk:** none. **Confidence:** high.

### L4-a. Mod `mask` patches land 0x400000 too low (#653, explains #213)

**Cause:** `scripts/mod_patches.py:138`: `offset = matches[0] + relative - EBOOT_BASE`.
- The match comes from the ELF segment's `vaddr` plus its offset. That is already image-relative: the loader places vaddr 0 at image offset 0 (`scripts/patches.py`, header).
- Literal `Address` lines are absolute PS4 addresses and correctly subtract `EBOOT_BASE` (`:201`); mask matches must not.
- So the built-in "Skip intro movie" writes `ret` at `0x1073b0`, inside an unrelated function, instead of at `0x5073b0`.

**Effect on #213:** every entry of shadPS4's `InfamousSecondSon.xml` is `Type="mask"`. Each one is either rejected as "outside the eboot" or written 4 MB too low. The v1.0 `0xc0000096` crash in #213 is plausibly one of those writes (low confidence).

**Fix:** `offset = matches[0] + relative`. The game-side patch runs `scripts/mod_patches.py` from disk, so replacing it fixes application; the launcher's frozen copy only affects the Mods list.

**Test gap:** the only mask fixture (`tests/test_patches.py:29`) has no `Address` and is "skipped as a whole". Add a synthetic ELF with an executable `PT_LOAD` and assert the returned image offset.

**Confidence:** high.

### R3. Flicker came back with renderer v2: v2 lacks v1's wave64 fixes (#7, #96, #557, #569; maybe #83, #215)

**Cause:**
- Renderer v2 runs upstream shadPS4's `LowerWave64BallotPass` (`gpu_v2/shadps4/shader_recompiler/ir/passes/lower_wave64_pass.cpp`, 199 lines, called from `gpu_v2/gpu/overrides/recompiler.cpp:215`). No override replaces it.
- Renderer v1's pass (`gpu/shadps4/.../lower_wave64_pass.cpp`, 492 lines; commits `4448ef2`, `5ed8725`, `b37381d`, all released) has three inFAMOUS fixes that v2 lacks:
  1. **ReadFirstLane lowering** (v1 `:384-391`, `:470-480`; comment: "inFAMOUS: flickering lights and steam").
  2. **Ballots inside loops that are not lane-divergent.** v1 lowers them at `:101-158` ("flickering smoke and fire"). v2 skips any ballot inside a loop (`:74`, `loops.empty()`).
  3. **Divergent ballots in their own 32-lane half** (v1 `:405-422`: "particle and light lists flickered").
- The #651 log (v2, v1.0) shows `Physical device subgroup size 32` and repeated v2 warnings `lower_wave64_pass.cpp:126 Ballot instruction in non uniform control flow`. So the unlowered ballots run on 32-wide hardware.

**Also missing in v2:** `b37381d`'s profile subgroup size. v2 sets `profile.subgroup_size = instance.SubgroupSize()` (`gpu_v2/gpu/overrides/vk_pipeline_cache.cpp:1236`), which can disagree with a pipeline's `requiredSubgroupSize=64`.

**Fix:** add v1's pass as a gpu_v2 override (list it in `gpu_v2/CMakeLists.txt`), adapting it so v2's `MaskedBitCount32` counts as lane-varying. Take the subgroup size from what the pipeline requests.

**Confidence:** high that the gap exists; medium that it explains each report. #557 also reproduces on v1, so check its 778k texture deletions as well.

### P1. Renderer v1: FPS collapses when "loading shaders" reaches 100% (#377 and 17 more)

**Evidence:**
- #377 bisected it to `3c19758` → `d3995fe` (39 → 6-10 FPS).
- In the #364 log, FPS falls from 57 to 8.5 exactly when `Pipeline warm-up: 0 of 25431 pipelines in 319.29 s` prints, and "not prepared" jumps from about 1k to 167k.
- #385 shows the same at 565-579 s.

**Cause:** `e3f7e99` (between `3c19758` and `d3995fe`) removed the early return in `StartBackgroundWarmUp` that printed "Pipeline warm-up: none; programs load from the cache on first use" (`gpu/shadps4/.../vk_pipeline_serialization.cpp:1203-1207` at `3c19758`). Every start with a precompiled cache now runs a 5-10 minute background "stages only" load that merges stages on the GPU thread. Why draw preparation collapses when that load ends is not established.

**Status:**
- Windows players were "fixed" by v1.0 only because `da39b0b` made renderer v2 the default; v2 has no background warm-up.
- Renderer v1 is still the Linux default (`launcher/backports_table.py:255`).

**Fix:** restore the early return when `stages_only && !build_cached`. `LoadProgramFromCache` already runs on every path (`vk_pipeline_cache.cpp:2014`).

**Diagnostic:** split the draw-prep "not prepared" count by reason; A/B with `ISS_BACKGROUND_WARMUP=0`.

**Confidence:** high on the trigger.

### L1. The problem report attaches another person's log (#247, #325, #372, #373)

**Evidence:** four different machines attached the same `run-20261016-195519.log` (a88bbc1, CUSA00004, 25.4 FPS). Each of those reports shows "Game: unknown".

**Cause:**
- `report.list_logs` takes every `run-*.log` in `<port>/out/logs` and sorts by mtime (`launcher/qt/report.py:88-96`).
- The Support screen preselects the newest one (`launcher/qt/support.py:66-72,100-101`), with no check that this PC or this build wrote it.
- These players never ran the game ("Game: unknown" means `game_dir` was never set, `report.py:342-348`), so a log inside a redistributed port folder was the newest. The official a88bbc1 zip has no logs. #325 fits an in-place update that kept `out/logs`.

**Fix:**
- Preselect only logs the launcher itself wrote (record them in the per-user settings in `new_log()`, `launcher/infamous_run.py:836-848`), and never one whose `Build:` line differs from the running version.
- Flag a log whose game differs from the configured one.

**Confidence:** high (code); medium (where the log came from).

### L3. Game version 01.00 and EU dumps

**01.00 (#135, #313, #433):**
- The base game stops with `STOP: first unsupported PS4 import: FMUirmwfWxg#dlcldr (index 245)` (exit 20).
- Only `libc.prx` and `libSceFios2.prx` are linked (`launcher/infamous_run.py:319`).
- `sceKernelLoadStartModule` prints "linked ahead of time" for any `/app0/` path, even one that was never linked (`src/runtime_gr2.c:78-89`). That message is misleading.
- The launcher only warns: `other_eboot` and `old_param` are in `WARNINGS` (`infamous_run.py:403`), so Play stays enabled.

**Fix:**
- Block the known base layout with an "install update 1.07" message.
- Make `runtime_gr2.c` say "not linked" for modules that were not.
- Optionally stub `FMUirmwfWxg` (its semantics are unknown, so test it).

**EU CUSA00004 (#127, #469, #485, #545, #648, #699, #47, #78, #89):**
- The EU 1.07 ELF is byte-identical to the US one (`docs/CUSA00004_CAMERA.md`). Yet these are gated on the US title ID:
  - direct mouse aim (`src/probe.c:1006`);
  - karma (`probe.c:923-928`);
  - native resolution (`src/iss_native_resolution.h:121`);
  - HUD (`probe.c:849,878`).

  Use signature gates as the camera hooks do (`src/iss_camera_hook.h:16-18`). Keep karma US-only until EU saves are handled.
- FTP dumps without `playgo-chunk.dat` are offered only the six CUSA00223 languages (`infamous_run.py:188-189,273-305`), so EU players cannot pick Italian or Polish.
- `Red orbs: multiplier unavailable` (#78) is a God of War III gate (`probe.c:588`) and means nothing for this game.

### I1. Input

- **`r2=none` also removes `MOUSE_LEFT` (#699).**
  - Cause: `apply_button_map` drops every source of the PS4 button, mouse sources included (`src/runtime_pad.c:300`). The default `{MOUSE_LEFT, BTN_R2}` (`:243`) is lost, and the launcher table hides mouse defaults (`launcher/padsettings.py:17-24`).
  - Fix: when the replacement names no mouse source, purge only keyboard and pad sources.
  - Confidence: high.
- **Mouse camera too slow (#439, #435, #610).**
  - Cause: the mouse drives the right stick, and deflection is clamped to ±1 (`src/mouse_stick.h:18,49-51`), so the turn rate can never exceed the game's top stick speed, whatever the DPI.
  - `96103ef` (sliders) helps within that cap. The real fix is direct mouse aim, which is opt-in (`ISS_DIRECT_MOUSE_AIM=1`, `probe.c:994`) and not in the launcher, plus the 1.5x camera mod.
- **DualSense lag and overshoot at 70-90 FPS (#686).**
  - The window thread polls input with `sleep_for(2ms)` (`gpu_v2/gpu/bridge.cpp:740`). That is about 15.6 ms on Windows unless `ISS_TIMER_1MS=1`, so input updates at about 64 Hz, below the frame rate (R1-21).
  - Separately, the touch `id` is the finger slot (`src/runtime_pad.c:755`), not a rolling ID, so a lift and retouch between reads is invisible to the game.
  - A/B with `ISS_TIMER_1MS=1`. Medium confidence.
- **InputPlumber (#505) and Steam Controller via its wireless dongle (#639).** Hypothesis: SDL's HIDAPI driver competes for hidraw. Test `SDL_JOYSTICK_HIDAPI=0` and `SDL_JOYSTICK_HIDAPI_STEAM=0`. Low confidence.

### L4-b. Paper Trail missions, and the v1 cache race (#654, #141)

**#654, Paper Trail:**
- NP always reports signed out (`src/runtime_services.c:160-163`).
- `sceHttpSendRequest` fails (`:141`, `:682`), and `sceHttpSetEpoll` is a no-op (`:688`), so `sceHttpWaitRequest` (`:149-157`) never returns an event. The game's non-blocking request thread waits forever.
- Fix: record requests in `SetEpoll`, and have `http_wait` return a socket-error event so the game takes its own offline path.
- Do not fake a PSN sign-in globally, since that starts other online flows. The reporter's offline patch could ship as a built-in **bytes**-type XML, off by default (bytes type avoids #653).

**#141 / PR #625 (open, not merged):** renderer v1 only, which is the Linux default.
- `num_requests` is incremented under `submit_mutex` (`gpu/shadps4/video_core/cache_storage.cpp:257-258`) but decremented without it (`:142`).
- The busy loop is at `:126-131`.
- v2 uses a locked queue and is fine.

### L2. Launcher setup and UX (#208, #393, #672, #58, #150, #106, #524, #359, #204)

| Issue | Finding | Fix |
|---|---|---|
| "Missing eboot.bin" / "still missing something" (#208, #393, #672) | `check_game` checks three files in the chosen folder (`infamous_run.py:457-488`), and the chip shows only the first problem. | List every problem; search 1-2 subfolders for `eboot.bin` + `sce_sys/param.sfo`; recognise `.pkg` with "extract first". |
| Patch-only folder shows Ready (#58) | The update folder alone passes all three checks. | Also check for a base-game-only file. |
| White-screen hang (#150) | `Backend.refresh()` runs `tlg-probe --gamepad` synchronously on the UI thread: at least 300 ms, up to a 10 s timeout (`launcher/qt/backend.py:239-240`, `logic.py:187-193`). It has eleven callers. | Move pad detection to a worker. |
| Folder not remembered (#106, #524) | The First Light launcher writes the same `%APPDATA%/infamous-launcher/settings.json` (`infamous_run.py:342-344`). | Per-title settings folder; `portable.txt` for portable mode. |
| Folder picker does nothing on Linux (#359) | `chooseFolder` returns silently on an empty path (`backend.py:426-428`); the portal registration fails without an installed `.desktop` file. Low confidence. | Log it, show a message, offer a paste-path field. |
| Turkish ignored (#204) | The wiring exists (`ISS_LANGUAGE` → `system_param`, `runtime_services.c:74-77`). Unexplained. | Log the language returned and the localisation archive opened. |
| **Regression in `88e912f`/`fac17b4`** | `72c8509` garbled the Portuguese folder messages in `launcher/infamous_run.py:381-399` (7 `Ã£`-style sequences; `cde38af` has none). | Re-save as UTF-8 from `cde38af`'s text. |

### M1 / C6. VRAM on 4-6 GB cards and integrated GPUs (#696, #424, #475, #80, #72, #75, #118, #607, #579, #535)

**Where the budget comes from:** the launcher sets `ISS_GC_BUDGET_MB` (3500 by default, or `auto` on v2; `infamous_run.py:311,783-784`). `src/probe.c:1151-1175` aliases it to `TLG_`.
- **Renderer v1** caps it at the driver budget on iGPUs (`gpu/.../texture_cache.cpp:1832-1842`).
- **Renderer v2** computes a device guard of the card minus 768 MB (cards up to 12 GB) and an automatic budget of `max(guard − (usage − images), 512 MB)` (`gpu_v2/gpu/overrides/texture_cache.cpp:162-205`).

**Still wrong in `fac17b4`.** From #424's v1.0 log (GTX 1650 Ti, 4 GB): usage 3,588 MB against a budget of 3,349 MB, of which images 702, arena 896 and "other" 1,989 MB. Only about 660 MB of that is tracked through VMA.
- The 512 MB texture floor sits below the working set, so the cache thrashes: 55,872 deletions, about 12 FPS.
- The buffer arena never shrinks. Its sparse buffer allows no system-memory type on NVIDIA (`ARENA_SPILL type=-1`), so it cannot spill either (`buffer_cache.cpp:714-724,3588`).
- `AllocateResidency` uses `Vulkan::Check` (`buffer_cache.cpp:3509`), so a real out-of-memory there is fatal.
- On v1 iGPUs (#696, Ryzen 7520U with 7.2 GB RAM), the image collector cannot evict buffers, which are most of the use: "3110 of 850 MiB". The image allocation then asserts (`gpu/.../texture_cache/image.cpp:199-209`) and prints `STOP: GPU library assertion failed`.

**Fix:**
- Scale the floor with the card (e.g. card/4) and a smaller headroom on 4 GB cards.
- Retry residency in a spill memory type, or evict images first, instead of `Check`.
- Find the untracked ~1.3 GB (VMA block slack is a suspect).
- On v1 out-of-memory, run an aggressive collection and retry without the device-local preference.

**Confidence:** high.

### F5. Intro-hatch GPU hang (#422)

**Evidence:** reproducible on `da39b0b` and `cde38af`, with native passes on or off. The black box shows a draw (vs `3f94c5dc`, ps `a43fe5bd`, 960×540) that samples its own color target (uid 1770). It never finishes, and the device is then lost.

**Cause (hypothesis):**
- The CP path moves the image to the feedback-loop layout and calls `setAttachmentFeedbackLoopEnableEXT` (`gpu_v2/gpu/overrides/vk_scheduler.cpp:1718-1722`).
- But graphics pipelines never declare the `eAttachmentFeedbackLoopEnableEXT` dynamic state (`gpu_v2/gpu/overrides/vk_graphics_pipeline.cpp:228-251`). The draw then samples its own attachment outside a declared feedback loop, which is undefined behaviour.
- Only one player reports it, so something system-specific may matter too: a 120 FPS engine, Optimus, an external monitor.

**Fix:** add that dynamic state when `IsAttachmentFeedbackLoopLayoutSupported()`, or copy the target to a temporary image for self-sampling draws (behind a toggle).

**Diagnostic:** a run with `BB_VK_VALIDATION=1`, `ISS_GPU_BLACKBOX=1` and `engine_fps=60`.

### C8. Address-range reservation fails (#145, still on v1.0, exit 21)

**Evidence:** `runtime_memory_reserve` reserves `[0x1000000000, 0xfc00000000)` as one placeholder with `VirtualAlloc2` (`src/runtime_memory.c:40-42,176-179`); it is the first call in `main`. The attached v1.0 log shows error **487** (`ERROR_INVALID_ADDRESS`), not 87. So something already occupies part of that range before `main`: an injected overlay or security DLL, or an Exploit-protection override forcing ASLR.

**Fix:** on failure, walk the range with `VirtualQuery` and print each occupant (base, size, type, mapped file) plus the process ASLR policy, then reserve around the holes; the region table supports several placeholders.

**Workaround:** remove any per-program ASLR override for `tlg-probe*.exe`.

### R4. Settings that break the image (#144, #75, #72, #602)

- **`output_res=3840x2160` breaks character lighting (#144, v1, RADV).** With the upscaler off, the scene is supersampled through 1920×1080 and 960×540 proxies (`gpu/.../vk_scene_resolution.cpp:73-94`). Compute shaders and integer loads read native copies resolved with nearest-neighbour blits (`:205-215`). Hypothesis: data crossing between native and proxy resolution misaligns at silhouettes. Fix: clamp `output_res` to 1080p unless an upscaler is active. Medium-low confidence.
- **"Fast (no GPU particles)" darkens the image (#75, maybe #602).** Without readbacks, the game's CPU auto-exposure (eboot `+0x1717c0`, `docs/PERFORMANCE_DIAGNOSTICS.md`) sums stale luminance tiles. Under v2 the option is also ignored: v2 reads `BB_READBACKS` (`gpu_v2/gpu/shim/core/emulator_settings.h:30`), while the launcher sets `ISS_READBACKS` (R3-C3). Fix: keep the exposure and particle sites precise in Fast mode; hide the option under v2.
- **BC 1D textures refused on NVIDIA (#72, v1 only).** v2 creates every 1D image as 2D (`gpu_v2/gpu/overrides/image.cpp:55-63`). Workaround `ISS_BC_TINY=1`; consider `auto` on NVIDIA for v1.

---

## 3. Present, cause not established

### C1. `STOP: guest raised debug exception code=0xa0020013 argument=0x0` (10 issues)

**Issues:** #189, #210, #425, #433, #512, #563, #594, #634, #692, plus #603 (second run) and #84 (exit 23).

**Pattern:** it happens at boot, right after shader preparation, or on "Continue", and once it starts it repeats on every launch. It occurs on both renderers, with native passes on and off, on AMD, NVIDIA and Intel, and on game 01.00 and 01.07.

**What the code does:** the game raises it itself through `sceKernelDebugRaiseException` (`src/runtime_kernel.c:370-371`). `raise_exception` (`:183-188`) prints the code and calls `_exit(23)` without recording the caller. `68f4960` (#189) only removed the follow-on `ntdll` fault.

**Hypotheses:**
- a fatal path in the guest after a runtime call returned something unexpected, e.g. the flexible memory size (R4-12);
- persistent bad state, most likely a torn save: savedata has no commit on unmount;
- a modified dump.

**Diagnostic:** in `raise_exception`, print:
- the guest return address relative to the image, and which module it is in;
- the guest frame-pointer chain;
- the thread name;
- a short ring of the last imports called.

Then ask reporters to move savedata aside and retest. This is the single most valuable diagnostic in this round.

### C2. `Fault: exception 0x20474343 at KERNELBASE.dll+0xc408a`: uncaught C++ exception (7 issues)

**Issues:** #333, #528, #539, #593, #671, #688, #691. Mostly v1.0 with renderer v2, but #333 is v1, so it is not v2-only.

**Why nothing useful is logged:**
- `SetUnhandledExceptionFilter(unhandled_fault)` (`src/probe.c:1443`) replaces the CRT handler that lets libc++abi reach `std::terminate`.
- `unhandled_fault` (`:286`) goes to `fatal_exception`, which prints the line at `:214` and calls `TerminateProcess(…, 3)` (`:230`).
- So v2's terminate hook (`gpu_v2/gpu/host_hooks.cpp:39`) never runs. It would not print the type or `what()` anyway.

**Throw candidates** (threads with no handler):
- the detached checkpoint thread (`gpu_v2/gpu/bridge.cpp:739`), whose 54-250 MB copies could raise `bad_alloc` on 7 GB machines (reduced by `ca1aab1`);
- the CP command rethrow (`gpu_v2/gpu/overrides/liverpool.cpp:1018-1024`);
- the guest-thread submit rethrows (`:4539-4541`, `:4583-4585`).

**Fix:**
1. In `unhandled_fault`, for code `0x20474343`, print the thread name and an `RtlVirtualUnwind` stack, then let the exception continue to `std::terminate`.
2. Make the terminate hook print `STOP: uncaught C++ exception <type>: <what()>`, and add one to the v1 exe.
3. Wrap the checkpoint thread's body in try/catch.

### R1. The world goes dark after a story point and stays dark in that save (10 issues)

**Issues:** #222, #223, #253, #681, #645, #651, #666, #363, #531, #555 (about 12 players with commenters).

**Pattern:**
- It starts mid-session: after "Chasing the Light", the Fetch boss, the Neon altar interior or the subway chase.
- Sky, ambient, sun and street lighting go to zero (80% of pixels exactly 0 in #223). HUD, particles, neon and the player's power still light the scene.

**What it is not:**
- It is not tied to one renderer, vendor or OS.
- These changes do not help: shader and driver caches deleted, presets, occlusion modes, `TLG_BACKPORTS=0`, native passes off, Precise accuracy.
- A new game is fine (#651), and #681 recovered by dying and relaunching. So the carrier is the save's story or lighting state, not a port cache. The port writes only caches, `tlg.ini`, `settings.json` and the karma store.

**Leading hypothesis (low confidence):** the CPU auto-exposure that sums GPU-written luminance tiles (eboot `+0x1717c0`) gets a wrong or stale result that only matters in night and interior lighting environments.

**Diagnostic:**
- get a dark save from #223 or #651, since it reproduces on every load;
- log the floats read at `0x1717c0` and the resulting exposure, comparing a dark save with a day save;
- take a frame capture to see whether the HDR light buffer itself lacks sun and ambient.

### Others

| Cluster | Notes |
|---|---|
| **F3 (v2 part)** | #564 (`88e912f`): a 44 s stall that is neither compiles nor readback. The CP consumed command buffers for about 10 s, then stopped, with no host stack. A 230 MB `PIPELINE_CHECKPOINT` took 6.4 s at the same moment, so schedule checkpoints after a quiet period. The v1 stalls (#364 and others) are pipeline compiles on the GPU thread plus warm-up contention; v2 compiles on workers. |
| **F4** (#104, #199) | Main thread at `guest+0x53b342`, the engine's microsecond `rdtsc` busy-wait (`src/spin_hook.h:3-10`), polling something the GPU writes. #104 ran renderer v1 with `TLG_READBACKS=0` ("Fast"), which cannot work: remove or warn about that option. #199 is an Intel HD 630 driver or GPU-thread hang, below target. |
| **C7** | Small-address guest faults: #641 (Reggie's death, 0x6d1fd, blocks progress), #700/#701 (EU), #450, #389, #214. Unknown. #193 was self-inflicted by a modified file. |
| **D2** | RDNA `amdvlk64.dll+0x206xxxx/+0x2076d89/+0x221affa` read faults, renderer v1 only. The fault address is `rdx+idx*8` with idx a zero-extended -3. Wave stats are ruled out (off by default and absent in several crashing logs). So is the 1D BC image (#270 logs 3 and 4 crash with no image created). Likely a driver bug under concurrent pipeline creation. Diagnostic: record the in-flight `vkCreate*Pipelines` key per thread and print it when the faulting module is a driver DLL. Workaround: renderer v2, the default. |
| **D3** | Intel Arc `igvk64.dll` (#427, #342: both unpatched 01.00 dumps); `nvoglv64.dll+0xdcd151` at core absorption (#577: retest on `88e912f`); RTSS overlay (#667: third party); Quadro M5000 (#70: below target). |
| **R2** | Black screen with the log at 60 FPS: #418 (the same dump is black on two PCs, so the dump), #448 (EU 01.00), #463 (version unknown), #323, #343, #224. Diagnostic: log the mean luminance of the presented image and draws per frame every 5 s. |
| **C9** | 18 crashes with no log line; see [section 4](#4-diagnostic-gaps-that-hide-the-causes). |

---

## 4. Diagnostic gaps that hide the causes

These are cross-cutting, and each is small. Together they would turn most "no log line" and "unknown" reports above into traceable ones.

1. **The report drops everything but `Fault|STOP|FREEZE` lines** (`launcher/qt/report.py:113`, `_EVENT_RE`). Stack frames (`  #N`), `GPU_FATAL`, `Assertion Failed!` with its message, and `Caller return offset` are all lost. Keep those, plus the 3 lines after a `Fault:`.
2. **The exit code is read but never logged** (`launcher/qt/backend.py`). Append `Exit: 0x…` to the run log.
3. **Renderer v2 assertions print no `STOP:` line.** `host_hooks.cpp` calls `abort()` and the SIGABRT hook prints only `GPU_FATAL` frames. Print `STOP: GPU library assertion failed: <message>` first.
4. **C++ exceptions are swallowed** (C2 above).
5. **No linker map is published**, so `tlg-probe-v2.exe+0x…` offsets cannot be symbolized. #367's offset was only resolved because a player had symbols. Generate `-Wl,-Map` per release and keep it with the release.
6. **Renderer v2's wait hooks are no-ops** (`gpu_v2/gpu/gpu_host.cpp:580`, `tlggpu_set_wait_hooks`). A thread waiting in a fault handler or `SendCommand` does not show as waiting in the freeze report. The freeze report also walks only frame pointers (`src/runtime_freeze.c:53,282`); use `RtlVirtualUnwind` for host threads.
7. **The GPU black box is off by default since `f0d64fb`** (`gpu_v2/defaults.env:44`), so GPU hangs leave no `gpu-stall` file. Turn it on automatically after a device loss, or keep a cheap ring always on.
8. **Every v2 report from v1.0 lacks freeze data** (fixed: `0f7d045`/`cfdbabf`). Ask F1/F3 reporters to retest on the beta before triaging further.

## 5. Not code bugs, or no defect traced

| Cluster | Notes |
|---|---|
| **P2** (9) | RX 580 / Polaris at 8-25 FPS with the GPU at 100%: the hardware limit. PR #574 (RX 580 profile) is open. D1's barrier merge is part of that profile and should become default anyway. |
| **P3** (25) | Performance reports on supported hardware with no defect traced. Many are on pre-v1.0 builds; ask for a retest on the beta with Frame stats. |
| **A1** (2) | #399 (audio cut out, no data), #467 (First Light, another game). |
| **X1** (16) | Feature requests: ultrawide and resolution, DLSS/FSR4/HDR, motion blur toggle, macOS, Steam Deck questions. |
| **X2** (3) | HayatoG's tracking tasks #8, #9, #11. #8 (readback-dominated FPS) and #9 (first-use shader stutter) overlap our rounds 1-3. |
| **X3** (17) | Empty auto-reports, a legal suggestion, and #197 (resolved by the reporter). The launcher could refuse to open an issue whose symptom field is empty. |

## Suggested order

1. **Release.** Cut a release from `88e912f`, or better from our `fac17b4`, which adds the v2 freeze report, lean logging and the upload-worker fix. Ask the F1, C4 and F2 reporters to retest. Expected to close about 30 issues.
2. **One-line fixes:**
   - #653 mask offset, plus the test;
   - C3 vertex-input guard;
   - C5 start the view thread after protection;
   - D1 barrier merge on by default, plus a batch limit;
   - I1 `r2=none`;
   - the Portuguese mojibake.
3. **Diagnostics** ([section 4](#4-diagnostic-gaps-that-hide-the-causes)) and the C1 caller print. Without these, C1, C2, R1 and C9 (45 issues) stay unknown.
4. **Then:**
   - R3: port v1's wave64 pass to v2;
   - P1: restore the early return for v1/Linux;
   - L1: report log selection;
   - L3: block 01.00, signature-gate the EU features;
   - M1: VRAM on 4 GB cards;
   - L4-b: Paper Trail HTTP wait;
   - F5: feedback-loop dynamic state;
   - C8: reservation diagnostics.

---

## Appendix A: every open issue by cluster

All 326 open issues on 2026-10-26, each in exactly one cluster.

| ID | Cluster | Count | Issues |
|---|---|---|---|
| F1 | Freeze/white screen at power absorb (Wwise cond wait) | 20 | #698, #695, #674, #665, #662, #652, #646, #643, #623, #586, #584, #576, #575, #559, #544, #543, #534, #507, #480, #442 |
| F2 | #632 texture-cache fault-streak deadlock; blind v2 freeze report | 2 | #632, #451 |
| F3 | Cutscene freezes and long stalls | 33 | #683, #664, #657, #636, #629, #622, #620, #609, #589, #587, #582, #581, #578, #571, #568, #564, #562, #551, #522, #478, #470, #465, #464, #453, #398, #375, #347, #344, #236, #230, #220, #201, #161 |
| F4 | Main thread spins at guest+0x53b342 | 3 | #199, #104, #28 |
| F5 | Intro hatch GPU hang (self-sampling draw) | 1 | #422 |
| C1 | Guest debug exception 0xa0020013 | 10 | #692, #634, #594, #563, #512, #433, #425, #210, #189, #84 |
| C2 | Uncaught host C++ exception 0x20474343 | 7 | #691, #688, #671, #593, #539, #528, #333 |
| C3 | Native passes without VK_EXT_vertex_input_dynamic_state: call to address 0 | 4 | #694, #660, #627, #472 |
| C4 | Overlay FPS labels out of bounds (0x101) | 8 | #677, #608, #605, #556, #537, #502, #476, #367 |
| C5 | Draw-distance thread vs image protection race (0x800da08c0) | 5 | #618, #583, #489, #288, #115 |
| C6 | GPU library assertion / out of device memory | 9 | #696, #663, #475, #455, #424, #411, #271, #212, #137 |
| C7 | Guest null/small-address faults | 7 | #701, #700, #641, #450, #389, #214, #193 |
| C8 | Guest address-range reservation fails | 1 | #145 |
| C9 | Crashes without a log line | 18 | #689, #684, #649, #588, #549, #540, #503, #473, #459, #457, #454, #429, #414, #291, #234, #229, #218, #159 |
| D1 | Polaris/GCN driver null write (9217 barriers in one call) | 12 | #655, #603, #538, #385, #315, #203, #133, #129, #119, #94, #91, #31 |
| D2 | RDNA amdvlk64 read faults in renderer v1 | 10 | #357, #355, #285, #280, #270, #252, #146, #123, #121, #120 |
| D3 | Other driver faults (Intel Arc, NVIDIA, overlays, old GPUs) | 5 | #667, #577, #427, #342, #70 |
| M1 | VRAM budget on 4-6 GB cards / FPS decay | 6 | #607, #579, #572, #535, #118, #80 |
| P1 | Oct 19 warm-up regression in renderer v1 | 18 | #413, #403, #392, #391, #387, #382, #381, #377, #374, #370, #369, #366, #364, #360, #354, #339, #317, #311 |
| P2 | RX 580 / Polaris low FPS | 9 | #690, #640, #580, #570, #513, #491, #486, #390, #346 |
| P3 | Performance on supported hardware (no defect traced) | 25 | #703, #693, #670, #668, #661, #650, #635, #615, #611, #536, #533, #519, #444, #410, #409, #407, #351, #331, #248, #188, #182, #181, #138, #126, #98 |
| R1 | Dark world after a story point (save-dependent) | 10 | #681, #666, #651, #645, #555, #531, #363, #253, #223, #222 |
| R2 | Black screen while the game runs at 60 FPS | 7 | #463, #448, #418, #343, #338, #323, #224 |
| R3 | Flicker and single-frame flashes (v2 wave64 gap) | 8 | #569, #557, #215, #116, #96, #83, #66, #7 |
| R4 | Settings that break the image (output_res, Fast accuracy, BC 1D) | 5 | #602, #415, #144, #75, #72 |
| A1 | Audio | 2 | #467, #399 |
| L1 | Problem report attaches a foreign log | 4 | #373, #372, #325, #247 |
| L2 | Launcher setup and UX | 13 | #672, #550, #524, #408, #393, #376, #359, #237, #208, #204, #160, #150, #106 |
| L3 | Unsupported or partly supported game versions | 12 | #648, #573, #545, #485, #469, #313, #135, #127, #89, #78, #58, #47 |
| L4 | Mods, patches, PSN and caches | 6 | #654, #653, #468, #213, #169, #141 |
| I1 | Input | 10 | #699, #686, #678, #639, #610, #505, #439, #435, #187, #125 |
| X1 | Feature requests and questions | 16 | #679, #669, #523, #520, #446, #412, #397, #383, #341, #240, #239, #219, #131, #117, #54, #27 |
| X2 | Maintainer tracking tasks | 3 | #11, #9, #8 |
| X3 | Empty reports, not bugs, resolved by reporter | 17 | #687, #685, #675, #658, #647, #631, #630, #604, #592, #590, #388, #362, #361, #335, #198, #197, #191 |

## Appendix B: data and limits

- **Records.** The per-issue records (one JSON object per issue: build, renderer, GPU, driver, exit code, signature, key log lines, maintainer replies) were extracted from the issue pages. Logs attached to about 30 issues were downloaded and read: #104, #145, #199, #270, #315, #364, #385, #422, #424, #475, #476, #502, #535, #539, #564, #572, #586, #603, #651, #691, #694 and others.
- **Logs not read.** Logs behind external links (Google Drive, MediaFire), and zips that were not fetched, were not read. Those records hold only what the page says.
- **Maintainer replies.** Most replies are HayatoG's 2026-10-23 note asking reporters on builds older than v1.0 to retest, and triage by collaborator Andretuta. Where a reporter confirmed a fix, the "Fixed after v1.0" and "already released" lists say so.
- **Fix status.** "Fixed, unreleased" was checked with `git merge-base --is-ancestor <fix> cde38af` (false) and `… 88e912f` / `… fac17b4` (true).
