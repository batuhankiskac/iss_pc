# Cloud audit, round 3: bugs, optimizations and tooling (renderer v2)

This is the third code-only pass over `main` at `fac17b4` (2026-10-26). It looks in three directions the first two rounds ([`cloud-audit.md`](cloud-audit.md), [`cloud-audit-round2.md`](cloud-audit-round2.md)) did not:

- **Optimizations:** CPU cost on the threads that set the frame time. Pinning the command processor (CP) to a P-core was worth +7.7% FPS (54.5 → 58.7), so the CP's own execution speed matters. So does the speed of the native-pass jobs the CP waits for (0.89 ms/frame in the city, `native_passes.cpp:1513-1514`).
- **Bugs:** concurrency errors that cause hangs, crashes, lost events or stale data.
- **Configuration and tooling:** toggles that do not do what they say, and benchmark or analysis code that could mislead an A/B decision.

Nothing from rounds 1-2 is repeated. References to them are written R1-n and R2-n. The same rules apply:

- Nothing was run or changed.
- Every `ms/frame` figure is an estimate from reading the code, unless it quotes a number recorded in the code or the docs.
- Every item says how to measure or reproduce it.

Paths are under `port/` unless they start with `docs/` or `scripts/`. `NP` stands for `port/gpu_v2/gpu/native_passes.cpp`, and `NPR` for `port/gpu_v2/gpu/native_passes_rasterizer.inl`.

One caution from `docs/PERFORMANCE_DIAGNOSTICS.md` applies to every pool item (O4, O8, O9): a branch that cut the pool's CPU time by about 15% lost 2.4% FPS. Judge pool changes by the CP's `cp_wait_us/submit` and by FPS, not by how much pool CPU they save.

## Ranking

The tables are ranked separately. Optimizations are ranked by expected ms/frame, bugs by severity × likelihood, and tooling by how easily it corrupts an A/B result.

### Optimizations

| # | Finding | Main location | Est. ms/frame | Toggle | Risk |
|---|---|---|---|---|---|
| O1 | `ISS_ASC_NATIVE_QUEUE` would move the particle compute that the main guest readback waits for off the graphics queue. It is off, undocumented and unmeasured | `vk_rasterizer.h:565-573`, `vk_rasterizer.cpp:5642-5649` | 0.5-2 (if it holds) | `ISS_ASC_NATIVE_QUEUE=1` (exists) | medium-high |
| O2 | Renderer v2 is built at `-O2` with no LTO or PGO. Renderer v1 measured −4.8% CP µs/draw and +4.2% FPS from them | `gpu_v2/CMakeLists.txt:211`, `gpu/CMakeLists.txt:17-44` | 0.2-0.6 | new `-DTLG_V2_LTO`, `-DTLG_V2_PGO` | low (LTO), medium (PGO) |
| O3 | The CP sleeps on `done_cv` while the job it needs is still queued; it could record that job itself | `NP:1411-1418,1426-1463,1148-1180` | 0.1-0.5 | new `ISS_NATIVE_PASSES_CP_STEAL` | medium |
| O4 | Every pool draw takes four `shared_mutex` locks, copies a `shared_ptr` and bumps shared counters | `NPR:976-1001,1394-1398`, `NP:943` | 0.1-0.25 (CP wait) | new `ISS_NATIVE_PASSES_POOL_MEMO` | medium |
| O5 | The segment check hashes 4.6 KB per segment for masks that are all ones in submit mode | `NP:812`, `NPR:180-194,1650-1668` | 0.1-0.3 | new `ISS_NATIVE_PASSES_FAST_KEYS=2` | low |
| O6 | `rec_targets` looks up render and depth targets that `BeginRendering` just resolved; the stencil path takes the texture-cache lock before its memo | `NPR:1887-1902`, `texture_cache.cpp:1266-1290` | 0.1-0.4 | new `ISS_NATIVE_PASSES_FORMAT_REUSE`; `ISS_TEXTURE_LOOKUP_MEMO=1` (exists) | low |
| O7 | The submit-gate release waits for the CP's per-frame housekeeping | `liverpool.cpp:1485-1497,1323-1331` | 0.05-0.3 | new `ISS_GATE_EARLY_IDLE` | low/medium |
| O8 | Per pool draw: an `unordered_map` per register write, a heap-allocated descriptor vector, a 2 KB memset hash | `NP:171,291-312`, `native_material_rasterizer.inl:255-261` | 0.05-0.2 (CP wait) | new `ISS_NATIVE_PASSES_FLAT_FINAL` | low |
| O9 | Submit mode still does work that only the draw-time mode reads: `latest[]` stores and `PublishSpanStart` | `NP:464-465,1369-1380`, `liverpool.cpp:1866` | 0.05-0.15 | new `ISS_NATIVE_PASSES_LATEST`, `ISS_NATIVE_PASSES_SPAN_STARTS` | none |
| O10 | Small per-draw items: scout entries read one by one, native compute re-looks up its images, out-of-line per-packet calls | `scout.cpp:101-118`, `native_compute_rasterizer.inl:661-689,887` | ≤0.1-0.3 together | several, see O10 | low |

### Bugs

| # | Finding | Main location | Symptom | Likelihood | Fix risk |
|---|---|---|---|---|---|
| B1 | A native-pass pool or scan thread that faults on a GPU-modified page deadlocks the CP | `NP:897-900,1040-1043,1415,1442,1460`, `buffer_cache.cpp:1341-1361,1674` | permanent hang | low | low |
| B2 | A guest condition wait on a recursive mutex held at depth > 1 keeps the mutex locked | `src/runtime_mutex.c:58,235-245` | permanent hang | unknown, cheap to check | low |
| B3 | Guest-thread unmaps free texture-cache images that the CP is still using | `gpu_host.cpp:97-101`, `texture_cache.cpp:479-487` | `ReportDoubleFree`, rare crash | low-medium | low/medium |
| B4 | `Image::flags` is read-modify-written by the CP without the lock and by fault handlers with it | `texture_cache.cpp:1210,1224,1253` vs `:358,398,474` | stale textures, stale CPU-visible render targets | low per event, many events | low |
| B5 | The default `idle` submit gate waits with no limit and needs every queue idle, compute included | `gnmdriver.cpp:112-119,2635-2638` | latent permanent hang | low | low |
| B6 | Polling an event queue skips its mutex | `equeue.cpp:177-180,233-246` | lost flip/EOP events, use-after-free | depends on guest polling | none |
| B7 | `kqueues`, the flip/vblank event lists and the IRQ table are changed with no lock while the present and CP threads read them | `equeue.cpp:25-33`, `video_out.cpp:64,79,106,121`, `platform.h:41,49,60,68,91` | rare crash at register/unregister time | low | low |
| B8 | Event-flag "pulse" (set then clear) wakeups are lost | `src/runtime_tlg.c:111-125,155-163,255-259` | stall or hang if the game pulses | unknown, cheap to check | low |
| B9 | `frame_end(); submit_done = false;` can erase a SubmitDone that arrives during `frame_end` | `liverpool.cpp:1485-1491,4418` | one frame's flush delayed | low | none |
| B10 | Minor: the 50 ms `AscRecorded` give-up leaves queued jobs running; `mutex_unlock` restores owner fields on a failed unlock | `NP:1911-1916`, `src/runtime_mutex.c:157-167` | spurious EPERM; token writes into passed ring memory | very low | low |

### Configuration and tooling

| # | Finding | Main location | How it misleads |
|---|---|---|---|
| C1 | A variant's `ISS_TUNING` drops the four default tuning keys | `defaults.env:121`, `infamous_run.py:823-826` | an `ISS_TUNING=x=1` variant also loses scout, bind prefetch and async written uploads |
| C2 | The bench cannot guarantee VSync off under v2 | `vk_swapchain.cpp:148,430-435`, `window.cpp:317`, `bench.ps1:785` | a saved exclusive-fullscreen mode caps every run at the refresh rate |
| C3 | defaults.env overwrites the player's switches; most launcher switches do nothing in v2 | `infamous_run.py:822-826` | `ini:bp08=0`, `ISS_READBACKS=0` and similar variants are A/A runs |
| C4 | Dead or always-overwritten keys in `defaults.env` | `defaults.env:10,33,40,49,56-57,69-70,79,84,103,119,124` | toggling them measures nothing |
| C5 | Value conventions that break the usual 0/1 reading | `bridge.cpp:775-781,558-560`, `vk_pipeline_cache.cpp:1296-1299` | `ISS_CP_CPU=0` pins to CPU 0, `ISS_AOT_COMPUTE=0` does not turn AOT off |
| C6 | `TLG_` names in variants are ignored by v2; stale shell `TLG_` twins beat defaults.env | `bench.ps1:1099,1122`, `probe.c:1166` | `TLG_FAST_DRAW=0` runs with fast draw on |
| C7 | `compare`'s `~` mark uses the control's unpaired spread | `scripts/round_analyze.py:630-638` | shows noise as a win, or a real +2.3% as `~` |
| C8 | The run order is unbalanced at the default (odd) round counts | `bench.ps1:1494` | position effects leak into the difference |
| C9 | Smaller tooling issues: warm-up asymmetry, slot signature, `bench_cpu` window start | `bench.ps1:1271-1295,1478-1485`, `bench_cpu.cpp:63-83` | see C9 |

**Bench-only items** (no code change needed): O1 (`ISS_ASC_NATIVE_QUEUE=1`) and O6 (`ISS_TEXTURE_LOOKUP_MEMO=1`). Before benching anything, apply C1, C2 and C5. When you override `ISS_TUNING`, repeat its default keys.

**Already covered:** a fourth report of the 1 ms `WaitVoLabelFor` spin (`liverpool.cpp:3382`) is R1-12, and is not repeated here.

---

## A. Optimizations

### O1. `ISS_ASC_NATIVE_QUEUE` targets the main guest readback wait, and it is off, undocumented and unmeasured

- **Location:**
  - The comment at `gpu_v2/gpu/shim/video_core/renderer_vulkan/vk_rasterizer.h:565-573` describes the target: "the game's thread waits each frame for its particle compute, which on the graphics queue waited behind the rendering submitted before it".
  - Gate: `vk_rasterizer.cpp:5642-5649` (`AscQueueEnabled`: needs `ISS_ASC_NATIVE_QUEUE=1`, a compute-only queue family, no CP split, no draw pipe).
  - Queue family setup: `vk_instance.cpp:443-462`.
  - Routing: `native_compute_rasterizer.inl:733-743` (a compute ring's buffer-only segment runs on the compute queue).
  - Fences: `vk_rasterizer.cpp:5712-5743` (`AscRelease` writes the labels when the compute queue reaches them).
  - The variable is absent from `defaults.env` and the README. It came in with the Castlesea renderer (commit `f5793f7`) and was never A/B tested in this repo.
- **Mechanism:**
  - With the default `ISS_ASYNC_COMPUTE=0` and this toggle off, the particle simulation's compute dispatches go on the graphics queue, behind every draw submitted before them.
  - The guest's precise particle-feedback readback (`guest+0x3f3eff`, the top readback site per `PERFORMANCE_DIAGNOSTICS.md`) waits for that whole queue.
  - On a compute-only queue, the compute work runs alongside the rendering, as it does on the PS4. Its fence and readback then wait only for the compute work. The readback queue also moves to the compute family's second queue (`vk_instance.cpp:451-458`).
- **Proposed fix:** bench `ISS_ASC_NATIVE_QUEUE=1`. If it holds up, add it to `defaults.env` and document it.
- **Toggle:** `ISS_ASC_NATIVE_QUEUE=1` (exists).
- **Expected effect:** the guest waits for the GPU on the frame's critical path. Suppose the particle readback waits about 1-3 ms behind rendering today. That is the same order as the 2.3-2.6 ms the code measured for such waits (`buffer_cache.cpp:1951-1957`, `vk_instance.cpp:465-468`). The gain would then be **0.5-2 ms/frame**. This is the largest unmeasured lever found in round 3.
- **Risk:** medium-high.
  - It adds cross-queue ordering: graphics waits for compute at every compute flush (`vk_rasterizer.cpp:5664-5675`).
  - Segments with images, GDS or CPU uploads fall back to graphics (`AscSegmentOnGraphics`).
  - Check fire, smoke and particles visually, and watch for hangs in menus and loading screens.
- **How to measure:**
  - the `ASC_QUEUE segments=… flushes=… compute_waits=… label_waits=…` line (`vk_rasterizer.cpp:5664-5675`) and `ASC_STATS` (`:3213-3228`);
  - with `ISS_READBACK_DIAG_LOG`, the `wait_start` → `wait_end` time of the `rip=0x3f3eff` requests;
  - `readback_async_wait_ms` in `SHADPS4_FRAME_CSV`;
  - a bench A/B.

### O2. Renderer v2 is built at `-O2` with no LTO or PGO

- **Location:**
  - `port/gpu_v2/CMakeLists.txt:211`: `target_compile_options(issgpu PRIVATE -O2 -g1 …)`. It comes after the Release flags, so its `-O2` replaces the Release `-O3`.
  - The file has no IPO, LTO or PGO setting. The only `INTERPROCEDURAL_OPTIMIZATION` line turns it *off* for `atrac9_v2` (`:240`).
  - Renderer v1 has both: `TLG_LTO` (default on) and `TLG_PGO=generate|use` (`port/gpu/CMakeLists.txt:17-44`).
- **Mechanism:**
  - On v1 the measured results were (`port/docs/parallel_gpu.md:244-252`):
    - LTO: 73.2 → 74.4 FPS, −1.8% GPU-thread µs/draw.
    - LTO + PGO: 76.3 FPS, −4.8% µs/draw.
  - v2's hot paths have even more small functions called across files that only LTO can inline:
    - per packet: `BeforeDrawPipePacket`, the `thread_local next_job_boundary` access (`native_passes.h:181`);
    - per draw: `native_passes::Enabled()`, `pause_guard::Blocked()`, `feedback_join::*`.
  - XXH3, used in every segment check (O5), is called through the libxxhash DLL (`PkgConfig::XXHASH`) instead of being inlined.
- **Proposed fix:**
  1. Add `TLG_V2_LTO` (ThinLTO with lld, default on after an A/B), and drop the `-O2` override or A/B `-O3`.
  2. Add `TLG_V2_PGO=generate|use`, as on v1. The game often ends through `_exit`, so the generate build must write its profile periodically (`__llvm_profile_write_file` from the stats thread) or at a fixed point.
  3. Link xxhash statically, or use the header-only `XXH_INLINE_ALL`.
- **Toggle:** CMake `-DTLG_V2_LTO=ON`, `-DTLG_V2_PGO=generate|use`.
- **Expected effect:** 2-5% of about 10.5 ms of CP time, i.e. **0.2-0.6 ms/frame**. The pool jobs gain the same share.
- **Risk:** low for LTO: check that the probe's patch offsets and exported symbols do not depend on non-inlined functions. Medium for PGO: profiles go stale, and a stale profile is quietly worse than none.
- **How to measure:** `bench.ps1` `cpu_cp_ms_per_frame` and `cpu_gpu_helpers_ms_per_frame`, the `ISS_CP_PATH_CYCLES` totals, and FPS in the same scene.

### O3. The CP waits for native-pass jobs that no pool thread has started; it could record them itself

- **Location:**
  - `NP:1411-1418` (`WaitJob`: `submit.done_cv.wait(lock, done)` with `pool_mutex` held) and `:1426-1463` (`WaitRecorded`).
  - Jobs sit in the global FIFO `submit.jobs` until a pool thread pops them (`PoolThread`, `:1148-1180`).
  - Recording a job is `RecordCommandBuffer(*lane, job.begin, job.end, job.start)` (`:906-…`) on a per-thread lane from `NewJobLane()` (`:794-800`). `running_jobs[slot]` publishes which job each thread records, so that image frees can wait (`WaitRecordingBefore`).
- **Mechanism:** the CP's wait for the pool (0.89 ms/frame in the city, `:1513-1514`, R1-3) includes the case where the job the CP needs is still queued. That happens when all six pool threads are busy with *later* jobs, asleep and slow to wake (E-cores, R1-3), or not woken yet. The CP then sleeps on `done_cv` while its next job waits in the queue. The port already steals work in two places:
  - the upload worker's `Wait` runs queued jobs on the caller (`buffer_cache.cpp:818-826`);
  - `WaitPipelineReady` compiles a queued pipeline itself (`CompileNowIfQueued`, `vk_rasterizer.cpp:1305-1307`).
- **Proposed fix:**
  1. In `WaitJob`, with `pool_mutex` held, look for job *k* of `work` in `submit.jobs`. Jobs are FIFO by command buffer, so the CP's job is normally at or near the front.
  2. If it is there, erase it, unlock, and record it on a CP-owned lane (`NewJobLane()` once).
  3. Publish it in a dedicated `running_jobs[MaxPoolThreads]` slot, and do the same `job_done` / `jobs_left` / notify bookkeeping as `PoolThread`.
  4. Wait only when the job has already started.
- **Toggle:** `ISS_NATIVE_PASSES_CP_STEAL=1`.
- **Expected effect:** the part of the 0.89 ms/frame in which the needed job had not started: **0.1-0.5 ms/frame** of CP time (estimate). It also makes the CP less sensitive to pool threads landing on E-cores.
- **Risk:** medium.
  - The native recording callbacks (`cb.record_*`) already run on six threads at once, so a seventh caller is the same contract.
  - `WaitRecordingBefore` (image frees) must see the CP's slot.
  - The CP must not trip over its own thread-local state while recording a job, such as the "compute bindings active" flags. Check the callbacks for `thread_local`, and keep `Shader::Info::num_ud_snapshots` (`NPR:994`) per lane.
- **How to measure:** `NATIVE_PASSES_SUBMIT cp_waits` and `cp_wait_us/submit`, plus a new `cp_steals` counter, then a bench A/B.

### O4. Every pool draw takes four shared locks, copies a `shared_ptr` and bumps shared counters

- **Location**, per recorded draw on a pool thread:
  - `IsUnlearnable` (`NPR:1394-1398`, called through `cb.unlearnable` at `NP:954`): a `shared_lock` on `Rejected().mutex`;
  - `NPR:976-980`: a `shared_lock` on `LearnedKeys().mutex`;
  - `NPR:987-991` and `:997-1001`: two `shared_lock`s on `NativeMaterial::State().mutex`, plus a `std::shared_ptr<const Template>` copy (`:1000`);
  - shared counters on one cache line: `stats.draws` and `stats.recorded` (`NP:943`, `:1020`), and the rasterizer's `stats.draws` (`NPR:974`).
- **Mechanism:** libc++'s `std::shared_mutex` is a mutex plus two condition variables. `lock_shared` and `unlock_shared` both take the inner exclusive mutex, which is an SRW lock on Windows. Each shared lock therefore costs two exclusive acquire/release pairs on one cache line that all six pool threads write. A `shared_ptr` copy adds two more atomic read-modify-writes on the template's control block, which every thread drawing that material shares. That is about 20 contended atomic operations per draw. Contention grows with the pool size, which may be part of why more pool threads did not help.
- **Proposed fix:**
  1. Keep a per-thread memo from key to (shape, `const Template*`), checked against a table generation number. Bump the generation on every erase, including `native_compute_rasterizer.inl:655` and the material drop path.
  2. Hold a raw pointer while recording, and copy the `shared_ptr` only when appending to `r.templates`.
  3. Keep the counters per thread and sum them at report time.
- **Toggle:** `ISS_NATIVE_PASSES_POOL_MEMO=1`.
- **Expected effect:** 0.5-1.5 µs per pool draw. With CP-wait-bound jobs, that is **0.1-0.25 ms/frame** of CP wait.
- **Risk:** medium. A missed generation bump would use a freed template. Keep the `shared_ptr` held in `r.templates` as the lifetime anchor.
- **How to measure:** `NATIVE_PASSES_SUBMIT pool_busy_us/job` and `cp_wait_us/submit`; a profiler's contention view on `SRWLock`.

### O5. The segment check hashes 4.6 KB per segment for masks that are all ones in submit mode

- **Location:**
  - Every submit-mode job starts with `l.written.fill(~0u); // every word is exact: nothing is assumed` (`NP:812`). So, as far as the code shows, every key's mask is all ones.
  - CP side, per segment (`NPR:1650-1668`):
    - `MaskedKeyWords` (`:180-187`): an `ExpandedMask` lookup (XXH3 of the mask, `unordered_map` probe, memcmp), an AND pass over 1 153 words, then a `NativeMaterial::Hasher` that value-initializes a 2 KB block (`native_material_rasterizer.inl:124-127`) and copies the 4.6 KB through it in three XXH3 calls;
    - `InheritedHashWords` (`:189-194`): `state & ~mask`, which is all zeros, then XXH3 over 4.6 KB. With an all-ones mask the result is a constant.
  - Pool side: `cb.key` (`NP:945`) runs `MaskedKey` (`NPR:137-145`) whenever the key is dirty. With an all-ones mask its bit-test loop over 1 153 words does nothing (about 0.5 µs).
- **Mechanism:** about 0.7-1 µs of CP time per segment buys nothing when the mask is full. The code comment says the CP checks about 16 000 segments a second (`NPR:155`), which is about 275 per frame.
- **Proposed fix:**
  1. When a key is recorded, set a "full mask" bit next to it.
  2. For full masks, use a precomputed constant for the inherited hash (XXH3 of 4 612 zero bytes).
  3. Hash `state_words` directly with no AND or copy, so the result stays bit-identical to today's `MaskedKeyWords` for a full mask.
  4. On the pool side, skip the bit loop when the mask is full.
- **Toggle:** `ISS_NATIVE_PASSES_FAST_KEYS=2`, validated with the existing `ISS_NATIVE_PASSES_FAST_KEYS_CHECK=1`.
- **Expected effect:** **0.1-0.3 ms/frame** of CP time.
- **Risk:** low. Count full masks first: if some keys still carry partial masks, they take today's path.
- **How to measure:** the `rec_chain` µs per call in `ISS_CP_PATH_CYCLES`. A one-off counter of full vs partial masks confirms the premise.

### O6. Render and depth targets are looked up again after `BeginRendering`; the stencil path locks before its memo

- **Location:**
  - `NPR:1887-1902`: the format check calls `texture_cache.FindRenderTarget(…).info.format` per colour target and `FindDepthTarget(…).info.format`, right after `BeginRendering` resolved the same views (`vk_rasterizer.cpp:4812`, `:4861`).
  - `texture_cache.cpp:1266-1290`: with a stencil, `FindDepthTarget` takes `TextureCache::mutex` (`:1269`) before it checks the stencil memo (`:1277`). The memo is off by default (`ISS_TEXTURE_LOOKUP_MEMO=0`, `defaults.env:115`), so the code walks the page table under the lock each time.
  - The same mutex is shared with the guest fault and unmap handlers (R2-4, and the comment at `:1267-1268`).
  - `FindRenderTarget` and `FindDepthTarget` also do the unlocked `flags |=` writes of B4 a second time.
- **Mechanism:** extra lookups and one extra lock hold per segment on the CP. When a guest fault handler holds the texture-cache lock across `VirtualProtect` (R2-4), the CP waits for it here too.
- **Proposed fix:**
  1. Have `BeginRendering` keep the attachment formats it just resolved, and compare against those.
  2. Move the stencil memo check ahead of the lock.
  3. A/B `ISS_TEXTURE_LOOKUP_MEMO=1`.
- **Toggle:** `ISS_NATIVE_PASSES_FORMAT_REUSE=1`; `ISS_TEXTURE_LOOKUP_MEMO=1` (exists).
- **Expected effect:** 0.5-1.5 µs per segment, so **0.1-0.4 ms/frame**.
- **Risk:** low. The formats must be the ones of the views actually bound, which `BeginRendering` already has.
- **How to measure:** `rec_targets` and `begin_render` in `ISS_CP_PATH_CYCLES`; `ISS_TEXTURE_LOOKUP_MEMO_STATS=1` for the memo hit rate.

### O7. The submit-gate release waits for the CP's per-frame housekeeping

- **Location:** `gpu_v2/gpu/overrides/liverpool.cpp:1485-1497`.
  - When the CP runs out of work, it first runs `frame_end()` (`:1323-1331`: `rasterizer->OnSubmit()` and `rasterizer->Flush()`).
  - Only then does it call `Platform::IrqC::Instance()->Signal(GpuIdle)` (`:1497`).
  - That signal releases the default `ISS_SUBMIT_GATE=idle` lock (`gnmdriver.cpp:100-107`, `ResetSubmissionLock`).
  - `OnSubmit` (`vk_rasterizer.cpp:3203-3307`) is the per-frame housekeeping: `occlusion.OnFrameEnd`, `buffer_cache.TickFrame`, `texture_cache.ProcessDownloadImages`, `RunGarbageCollector`, `runtime.TickFrame` and `scheduler.CollectProfile`.
- **Mechanism:** the guest's render thread waits in `sceGnmSubmitCommandBuffers` for that signal (R1-2). Everything before the signal is serial:
  1. the housekeeping;
  2. the guest's wake-up (~20-100 µs);
  3. the guest's `PerformSubmit`;
  4. only then does the CP get work again.

  If the signal is raised as soon as the frame's commands are flushed, the guest's wake-up and submission overlap the housekeeping.
- **Proposed fix:** with the gate in `idle` mode, reorder the idle path: `Flush()`, then `Signal(GpuIdle)`, then the `TickFrame` part of `OnSubmit`. The CP still picks up the next submission only after the housekeeping, so nothing runs concurrently with it.
- **Toggle:** `ISS_GATE_EARLY_IDLE=1`.
- **Expected effect:** min(housekeeping time, guest wake-up + submit), about **0.05-0.3 ms/frame**. It becomes 0 once the gate is `gfx` or `off` (R1-2).
- **Risk:** low/medium.
  - `IsGpuIdle()` must keep meaning "no submission left" (`gnmdriver.cpp:2635-2637`).
  - The garbage collector must not free something the next frame's first submission uses before that submission runs. This already holds, because the CP is single-threaded.
- **How to measure:** time `frame_end()` with a new `FrameEndNs` counter. In the `Submit gate:` line, the `idle->admit gap` should then reflect only the guest side. Bench A/B with `ISS_SUBMIT_GATE=idle`.

### O8. Containers, a heap copy and a large memset on every pool draw

- **Location:**
  - `MergePending` (`NP:291-312`) does a `std::unordered_map<u32,u32> final_index` `try_emplace` (`:171`) for every pending register word, 30 or more per draw including the 32 user-data words. It allocates map nodes on a segment's first draws and frees them all at every `clear()` (`:970`).
  - `BuildWrites` (`native_material_rasterizer.inl:255-261`) does `out = t.writes`. `Pipeline::DescriptorWrites` is a `std::vector<vk::WriteDescriptorSet>` (`vk_pipeline_common.h:63`), declared fresh per draw (`NPR:1181`), so this is a heap allocation and a copy on every draw.
  - `MaterialHash` builds its hash with a value-initialized 2 KB `Hasher` block plus about 40 small `Put` calls (the same `Hasher` as O5).
- **Mechanism:** this is allocator traffic on six threads at once, and the CRT heap lock is shared with the CP.
- **Proposed fix:**
  1. Replace `final_index` with a flat register-to-slot array stamped per segment (a generation per entry, so clearing costs nothing).
  2. Reuse a thread-local `DescriptorWrites` scratch vector, or make the type a `small_vector<…, 16>`.
  3. Hash a stack array once instead of streaming through `Hasher`.
- **Toggle:** `ISS_NATIVE_PASSES_FLAT_FINAL=1`.
- **Expected effect:** 0.6-1.2 µs per pool draw, so **0.05-0.2 ms/frame** of CP wait.
- **Risk:** low.
- **How to measure:** `pool_busy_us/job` and `cp_wait_us/submit`.

### O9. Submit mode still does work that only the draw-time mode reads

- **Location:**
  - `Apply` stores every tracked register write into the global `latest[t]` and `latest_set[t]` atomics (`NP:464-465`; arrays at `:196-197`). They are read only with `ISS_NATIVE_PASSES_PREDICT=1` on the draw-time path (`:675-678`). Submit mode, the default (`defaults.env:82`), returns before installing any draw-time hook (`:1300-1304`).
  - `PublishSpanStart` (`NP:1369-1380`, called from `liverpool.cpp:1866` for every command buffer and indirect buffer) allocates two 5.2 KB `SpanStart`s, copies the tracked registers twice, takes `span_mutex` and inserts into a map. Its only readers, `LoadStart` (`:134-138`) through `Restart` (`:568-583`), are called only from the draw-time hooks (`:595`, `:638`, `:640`).
- **Mechanism:**
  - The `latest` stores from the async scan thread and all six pool threads write the same hot cache lines, for example the user-data words every draw sets. The scan sits in front of the first job of each command buffer.
  - `PublishSpanStart` runs on the CP.
- **Proposed fix:** wrap the stores in `if (predict_latest)`, and skip `PublishSpanStart` when `SubmitMode()`.
- **Toggle:** `ISS_NATIVE_PASSES_LATEST=1` and `ISS_NATIVE_PASSES_SPAN_STARTS=1` restore today's behaviour.
- **Expected effect:** 0.1-0.4 µs per draw on the scan and the pool; 1-2 µs per command buffer on the CP. Together about **0.05-0.15 ms/frame**.
- **Risk:** none on the default path. Both stay on when the draw-time mode is used.
- **How to measure:** `scan_us/submit`, `cp_wait_us/submit`, and `ISS_PASS_COST` class 12 (setup).

### O10. Small per-draw and per-packet items on the CP

Each item is small; together they are about ≤0.1-0.3 ms/frame. LTO (O2) covers part of the call overhead.

- **Scout entries read one at a time for executed segments.** `begin_draw(packet)` (`NPR:1977-1981`) goes through `ScoutSubmission::Next` (`gpu_v2/shadps4/video_core/amdgpu/scout.cpp:101-118`) for every draw of a natively executed segment. Each call reads the `published` counter and the `packet` field of an entry written by the scout thread, costing one or two cross-core cache misses per draw for entries the CP does not use. Fix: `Skip(n, last_packet)`, which advances the cursor by n and checks only the last entry. Toggle `ISS_SCOUT_BULK_SKIP=1`. Measure: `rec_draws`.
- **Native compute re-looks up its images on each run.** `native_compute_rasterizer.inl:661-689` calls `FindImage` (texture-cache lock, address index, `TouchImage`), `GetAssociatedDepth` and `FindTexture` for every template image, after `TexturesValid` has already validated them. `:887` copies the `DescriptorWrites` vector per replayed dispatch. Fix: a memo keyed on (`Generation()`, image generation) that still touches images for the garbage collector, and a scratch vector. Toggle `ISS_NATIVE_COMPUTE_IMAGE_MEMO=1`. Measure: `cs_textures`, `cs_record`.
- **Per-packet calls.**
  - `extern thread_local next_job_boundary` (`native_passes.h:181`) has a dynamic initializer (`NP:1399`), so every access goes through a TLS wrapper call. Fix: `constinit` with a nullptr sentinel.
  - `BeforeDrawPipePacket` is an out-of-line call per packet even with `ISS_DRAW_PIPE=0` (`liverpool.cpp:1946`). Fix: hoist a local bool before the loop.
- **Repeated work in `TryFastDraw`.** It repeats the same program memcmps up to about eight times per draw (`fast_draw.inl:1853-1994`).
- **A store into shared data.** `TexturesValid` stores `cache_generation` unconditionally (`native_material_rasterizer.inl:299`) into a template that the pool threads read, which dirties the line. Store only when the value changes.

---

## B. Bugs

### B1. A native-pass pool or scan thread that faults on a GPU-modified page deadlocks the CP

- **Location:**
  - Pool threads write tokens into the guest command buffer through the normal, protected guest mapping:
    - `NP:1039-1043`: `auto* w = const_cast<u32*>(next); w[1] = TokenMagic; …` (draws);
    - `:896-900` (dispatches).
  - The async scan thread reads the command buffers directly.
  - The CP waits for those threads with no timeout while holding `pool_mutex`:
    - `:1415` (`WaitJob`);
    - `:1442` (waiting for the scan);
    - `:1460`.
  - A guest fault on a non-CP thread: `page_manager.cpp:883-892` → `PrepareDrawPipeFault` returns false off the CP (`draw_pipe_rasterizer.inl:121-134`) → `BufferCache::InvalidateMemory(…, assume_locks=false)` (`buffer_cache.cpp:1341-1361`) → `ReadMemory` → `liverpool->SendCommand<true>` (`:1674`). This waits for the CP to run the command.
- **Mechanism:** commands are drained only by the CP's `ProcessCommands`, which never runs while the CP sits in `done_cv`. If the page a worker writes or reads is GPU-modified, the worker's fault waits for the CP, and the CP waits for the worker. `readback_write_discard` is off by default, so a write fault reads back instead of discarding. Two layouts trigger it:
  - GPU-written data embedded in the command buffer's pages (`allocateFromCommandBuffer`-style indirect arguments or a compute output);
  - a frame allocator that puts both on one 4 KB page.
- **Symptom:** a permanent hang. The freeze report shows `faults_in_progress ≥ 1` (`gpu_host.cpp:158-185`) on a `NativePassesPool` or scan thread, and the CP inside `done_cv`.
- **Likelihood:** low; it depends on memory layout. There is no way out once it happens.
- **Proposed fix:** write the tokens, and read the command buffers, through the backing view (`Core::Memory::TryWriteBacking`, as `liverpool.cpp:3013-3041` and `texture_cache.cpp:337-344` already do). They then never fault. A defensive alternative: turn the three waits into a `wait_for(1 ms)` loop that runs `ProcessCommands()`, as the VO-label wait does.
- **Toggle:** none needed for the backing-view writes. `ISS_NATIVE_PASSES_WAIT_PUMP=1` for the defensive loop.
- **Risk:** low.
- **How to detect:** set a `thread_local` "native worker" flag, and have `GuestFaultSignalHandler` log once when it sees the flag on a GPU-modified page. Even without a hang, the count shows how close the layout comes.

### B2. A guest condition wait on a recursive mutex held at depth > 1 keeps the mutex locked

- **Location:**
  - `port/src/runtime_mutex.c:58`: guest mutex type 2 becomes `PTHREAD_MUTEX_RECURSIVE`.
  - `:235-245` (`wait_cond`): `m->owner = NULL; m->depth = 0;` then a single `pthread_cond_wait(&c->native, &m->native)`.
- **Mechanism:** a host condition wait releases one level of a recursive mutex, on both glibc and winpthreads. With depth 2 the mutex stays locked through the wait, so the thread that would signal blocks on the lock forever. The PS4's libthr (`_mutex_cv_unlock`) saves the recursion count, releases the mutex fully, and restores the count after the wait. The freeze report shows the owner as NULL (cleared at `:239`), which points away from the cause.
- **Symptom:** a permanent hang in a condition wait, with the waker blocked on the same mutex.
- **Likelihood:** unknown. It depends on whether the game (Wwise, Havok, its job system) waits on a recursive mutex locked twice. A one-line log answers it.
- **Proposed fix:** in `wait_cond`, unlock `depth - 1` extra times before the wait, relock that many times after it, and restore `depth`.
- **Toggle:** none (correctness). Keep a `TLG_COND_RECURSIVE=0` escape while it is new.
- **Risk:** low.
- **How to detect:** log once per mutex when `wait_cond` sees `depth > 1`.

### B3. Guest-thread unmaps free images that the CP may be using: a likely source of the known "double free"

- **Location:**
  - `gpu_v2/gpu/gpu_host.cpp:97-101`: the runtime's unmap hook calls `Rasterizer::UnmapMemory` on the guest thread that unmapped. It runs from `runtime_memory.c` `flush_hooks()`, after the map lock is dropped.
  - That reaches `vk_rasterizer.cpp:5846-5857` and then `texture_cache.cpp:479-487` (`UnmapMemory`), which calls `FreeImage` for every image in the range under `TextureCache::mutex`.
  - On the CP, `FindImage` (`texture_cache.cpp:972-…`) takes the mutex only to look up the id. `FindTexture`, `FindRenderTarget`, the fast path of `UpdateImage` (`texture_cache.h:222-226`) and the bind code then use `slot_images[image_id]` and `Image&` references after the lock is released, for the rest of the draw.
  - `texture_cache.h:520-532` (`FreeImage`) already detects and skips a second free: "A second free of an image (a race not found yet) would assert in UnregisterImage: it is logged with both callers and skipped." `texture_cache.cpp:1267-1268` also names unmaps as a source of double collection.
- **Mechanism:** while the CP is between `FindImage` and the end of a draw that binds image *I*, a guest thread that unmaps memory overlapping *I* frees it: `UntrackImage`, `UnregisterImage`, `DeleteImage`, and the slot becomes reusable. The CP then writes `flags`, touches the LRU and creates views on a freed or reused slot. If the slot was reused, the CP's unlocked `flags |= …` (B4) writes a stale snapshot over the new image's flags. A stale or missing `Registered` bit gives exactly the double-free report. Other interleavings give corruption or a crash.
- **Symptom:** an occasional `ReportDoubleFree` line in `game.log`; rarely, a crash or Vulkan validation errors around image views after streaming or level transitions.
- **Proposed fix:** run texture-cache unmaps only on the CP. When `UnmapMemory` is called from a non-CP thread, queue it with `liverpool->SendCommand<true>(…)`. It then runs between packets, when no `Image&` is held, and the guest's unmap returns after the images are freed, keeping today's ordering. The buffer-cache part (`InvalidateMemory`) can stay on the guest thread.
- **Toggle:** `ISS_UNMAP_ON_CP=1` while it is validated.
- **Cost:** one CP round trip per guest unmap that overlaps GPU memory, which is rare during play.
- **Risk:** low/medium. The queued unmap must run before the guest reuses the range, which the synchronous `SendCommand<true>` guarantees.
- **How to detect:**
  - count `ReportDoubleFree` lines (they print both free sites, `texture_cache.cpp:1897-…`);
  - assert that `FreeImage` runs on the CP (`liverpool->IsCurrentCpThread()`) and see whether it fires on a long route with streaming.

### B4. `Image::flags` is read-modify-written by the CP without the lock and by guest fault handlers with it

- **Location:**
  - `Image::flags` is a plain `ImageFlagBits` (`gpu_v2/shadps4/video_core/texture_cache/image.h:139`).
  - The CP writes it **without** `TextureCache::mutex`:
    - `gpu_v2/gpu/overrides/texture_cache.cpp:1210` (`FindTexture`, storage binding: `image.flags |= ImageFlagBits::GpuModified`);
    - `:1224` (`FindRenderTarget`);
    - `:1253` (`FindDepthTarget`).
  - Guest threads write it **with** the mutex, from the write-fault path:
    - `:398` (`InvalidateMemory`: `image.flags |= ImageFlagBits::CpuDirty`);
    - `:358` (`MarkAsMaybeDirty`);
    - `:474` (`InvalidateMemoryFromGPU`: `GpuDirty`).
  - The code already knows about this race for one bit. `shim/video_core/texture_cache/texture_cache.h:400-402` says the command processor updates the flags without the lock and a stale `Picked` bit written back would hide the image, and `Picked` was moved out of `flags` for that reason.
- **Mechanism:** `flags |= X` is a load, an OR and a store. Two directions can lose a bit:
  - **Lost `CpuDirty`.** A guest write fault sets `CpuDirty` between the CP's load and store, and the CP's store erases it. The fault handler has also just called `UntrackImage`, so the image's pages are no longer write-protected and later CPU writes do not fault again. The image stays stale until something else retracks it.
  - **Lost `GpuModified`.** This direction is worse: a GPU-written target is not downloaded or protected for a guest read, so the guest reads stale data.
- **Symptom:** rare, timing-dependent stale textures or stale CPU-visible render-target data: UI or dynamic textures that stop updating, or flicker. It is not a stall.
- **Likelihood:** low per event, but not zero. There are thousands of binds and image-page faults per frame, the window is a few instructions, and play sessions last hours.
- **Proposed fix:** make every modification of `Image::flags` an atomic RMW, for example `std::atomic_ref<u32>(reinterpret_cast<u32&>(image.flags)).fetch_or(u32(bit), std::memory_order_relaxed)` behind `SetFlag` / `ClearFlag` helpers. Plain reads stay as they are. This costs one `lock or` per bind and keeps the lock-free fast path.
- **Toggle:** none (correctness).
- **Risk:** low.
- **How to detect:** a stress test that binds an image as a render target in a loop on one thread while another thread write-faults its pages, checking that `CpuDirty` is never lost. ThreadSanitizer on the Linux build also flags it.

### B5. The default `idle` submit gate waits with no limit, and needs every queue idle

- **Location:**
  - `gpu_v2/gpu/overrides/gnmdriver.cpp:112-119` (`WaitGpuIdle`): `cv_lock.wait(lock, [] { return submission_lock == 0; });`, with no timeout. It is reached from DingDong (`:371-374`, when `AscWaits()`, i.e. in `idle` mode, `submit_gate.h:88-90`), from both submit functions and from SubmitDone.
  - The lock is set at `:2637-2638` when `!liverpool->IsGpuIdle()`. `IsGpuIdle()` is `num_submits == 0` (`liverpool.h:119-121`), and `SubmitAsc` increments `num_submits` too (`liverpool.cpp:4580`).
  - It is cleared only by the GpuIdle interrupt (`liverpool.cpp:1497`), which needs every queue drained.
  - The port's own comment at `gnmdriver.cpp:2425-2429` names the hazard for the gate-off seed path, and bounds that wait at 50 ms: "queued work may wait on a label only a later DingDong or a guest CPU write satisfies, so an unbounded wait could hang the guest". The gate itself has no such bound.
- **Mechanism:** a queued compute submission that waits on a label written only by later work (a later DingDong, or a guest CPU write the blocked thread would make) keeps `num_submits` above zero, so GpuIdle never comes. On a PS4, DingDong never blocks.
- **Symptom:** a permanent hang, with the guest in `WaitGpuIdle` and the CP in its yield loop (R2-3).
- **Likelihood:** low. The game runs with this gate today, so no common path hits it. The R1-2 move to `gfx` or `off` removes it for DingDongs.
- **Proposed fix:** bound the wait (e.g. 50 ms, then log and continue, as the seed path does). Or base the lock on graphics work only (`num_gfx_submits`). R1-2 (`ISS_SUBMIT_GATE=gfx`) also stops DingDongs from waiting.
- **Toggle:** new `ISS_SUBMIT_GATE_TIMEOUT_MS` (0 = unbounded, today).
- **Risk:** low. A timeout only changes behaviour on a path that would otherwise hang.
- **How to detect:** the `SubmitGateWaits*` / `SubmitGateWaitNs*` counters (`gnmdriver.cpp:121-…`); log the longest single gate wait.

### B6. Polling an event queue skips its mutex

- **Location:** `gpu_v2/shadps4/core/libraries/kernel/equeue.cpp:177-180`. With `timo != nullptr && *timo == 0`, `WaitForEvents` returns `GetTriggeredEvents(ev, num)` without `m_mutex`. `GetTriggeredEvents` (`:233-246`) copies triggered events, clears them and erases one-shot entries from `m_events`. This is the `sceKernelWaitEqueue` that v2 registers (`:651`).
- **Mechanism:** `TriggerEvent` (`:209-229`) runs under `m_mutex` on the present thread (vblank and flip events) and on the priority-ops thread (the GfxEop interrupt). If a trigger lands between the poll's copy and its `Clear()`, it is wiped: that flip or EOP event is lost. If an `AddEvent` reallocates `m_events` during the poll, the poll reads freed memory.
- **Symptom:** a missed flip or EOP event, so a guest wait that never ends or a skipped frame; rarely a crash.
- **Likelihood:** depends on whether the game polls with a zero timeout. A counter answers it.
- **Proposed fix:** take `m_mutex` on the poll path (`std::scoped_lock lock{m_mutex}; return GetTriggeredEvents(ev, num);`).
- **Toggle:** none (correctness).
- **Risk:** none.
- **How to detect:** count zero-timeout calls per equeue.

### B7. Containers shared with the present and CP threads have no lock

- **Location:**
  - **`kqueues`:** `equeue.cpp:25-33` (`std::unordered_map<s32, EqueueInternal*>`). It is looked up on every vblank and flip (`GetEqueue` at `videoout_driver.cpp:459` and `:723`), while guest threads insert (`equeue.cpp:332`, `:445`) and delete (`:458-459`). A rehash during a lookup, or a delete, leaves a dangling pointer.
  - **Flip and vblank event lists:** `flip_events` and `vblank_events` change through `push_back` and `erase` with no lock (`video_out.cpp:64,79,106,121`). The present thread iterates them at `videoout_driver.cpp:458` (no lock) and `:719-722` (under `vo_mutex`, which the writers do not take).
  - **The IRQ table:** `platform.h:41,49,60,68` call `irq_contexts.try_emplace` on a plain `std::unordered_map` (`:91`) before taking the per-context lock. The first `Signal` or `Register` of each interrupt id can come from different threads at once: the CP's GpuIdle, the priority-ops thread's GfxEop, the guest's `sceGnmAddEqEvent`.
- **Symptom:** a rare crash or a lost registration, mostly at boot, level loads, or when the game re-registers its events.
- **Proposed fix:** a `shared_mutex` around `kqueues`; `vo_mutex` around every change of the event lists; a fixed array of IRQ contexts created up front, indexed by `InterruptId`.
- **Toggle:** none (correctness).
- **Risk:** low.
- **How to detect:** ThreadSanitizer on the Linux build. These are upstream shadPS4 patterns, so check upstream for fixes first.

### B8. Event-flag "pulse" wakeups are lost

- **Location:** `port/src/runtime_tlg.c`:
  - `ef_set` (`:155-163`) ORs the bits and calls `ef_wake` (`:111-125`), which only signals;
  - `ef_clear` (`:164-170`) clears them;
  - the waiter re-tests `ef_match(ef->bits, …)` only after it wakes and retakes the lock (`:255-259`).
- **Mechanism:** with a set followed by a clear before the waiter runs, the waiter finds the bits gone and sleeps again. The setter clearing right after the set almost always wins the race to the lock. PS4 event flags release the waiters whose condition is met at the moment of the set, and apply their clear mode then.
- **Symptom:** a stall until the next set, or a hang if no further set comes.
- **Likelihood:** unknown. It needs a set-then-clear pattern on a flag with a waiter.
- **Proposed fix:** in `ef_set`, hand off to satisfied waiters in FIFO order: evaluate each waiter's pattern, record its result bits, apply its clear mode, and mark it done. `runtime_sema.c` already uses this handoff pattern for semaphores.
- **Toggle:** `TLG_EF_HANDOFF=1` while it is validated.
- **Risk:** low.
- **How to detect:** count waiters that wake to an unmatched pattern right after a set (a new `ef_rewaits` counter next to `ef_wakeups` and `ef_skipped`, `:93`).

### B9. `frame_end(); submit_done = false;` can erase a SubmitDone

- **Location:** `liverpool.cpp:1485-1491`. With the default `ISS_FRAME_END_MARK=0`, the idle path runs `frame_end()` and then stores `submit_done = false`. `SubmitDone` stores `submit_done = true` under `submit_mutex` (`:4418`). The flag is a `std::atomic<bool>` (`liverpool.h:508`), but the check, the work and the clear are not one step.
- **Mechanism:** a SubmitDone that lands during `frame_end` (OnSubmit + Flush) is erased. That frame's OnSubmit and Flush are skipped until the next frame end. This only happens when the guest's whole next frame plus its SubmitDone fits inside one `frame_end`, which the `idle` gate makes rare: when the GPU was busy at SubmitDone, the gate holds the guest until GpuIdle.
- **Symptom:** about one frame of delayed flush and readback retirement, now and then.
- **Proposed fix:** `if (submit_done.exchange(false)) frame_end();`
- **Toggle:** none.
- **Risk:** none.
- **How to detect:** count frame ends against SubmitDones in the `Perf:` line; they should match.

### B10. Minor

- **`AscRecorded` give-up does not cancel pool jobs** (`NP:1911-1916`). After 50 ms it drops the entry, but pool jobs already queued or running keep writing tokens into ring memory the CP has already passed. The state and generation checks in `Claim` make a false match unlikely. Fix: an "abandoned" flag checked before the token writes. A nonzero `asc_stats.timeouts` means the fallback was taken.
- **`mutex_unlock` failure path** (`src/runtime_mutex.c:157-167`). After a failed unlock by a thread that does not own the mutex, it writes back the owner and depth it read before the call, while the real owner may be changing them. The stale owner then makes `wait_cond` (`:236`) return a spurious EPERM. Fix: touch owner and depth only when `owner == holder()`.

---

## C. Configuration and tooling

These do not cost frame time. They make A/B runs measure something other than intended, which is how a real win gets recorded as "no change", or the reverse.

### C1. A variant's `ISS_TUNING` silently drops the four default tuning keys

- **Location:**
  - `port/gpu_v2/defaults.env:121` sets `ISS_TUNING=async_written_uploads=1,scout=1,scout_fast=1,bind_prefetch=1`.
  - `port/launcher/infamous_run.py:823-826` applies a defaults.env key only `if key not in os.environ`, so a variant's `ISS_TUNING` replaces the whole list.
  - The code defaults of those four keys are off (`perf_stats.h:503,506,521,527`; parser `bridge.cpp:599-626`).
  - The bench's own example uses `B = @{ ISS_TUNING = 'x=1' }` (`scripts/bench.ps1:60,103`; `docs/PERFORMANCE_DIAGNOSTICS.md:125,172`), and the README suggests the "`recorder_spin_us` key of `ISS_TUNING`" as a bench candidate (`README.md:228`).
- **How it misleads:** that variant measures `x=1` **plus** four regressions (no async written uploads, no scout, no bind prefetch), and the result reads as "x hurts". Rounds 1 and 2 of this audit already warn about this in their bench recipes, but the tool should not depend on a warning.
- **Fix:** in `bench.ps1` (or the launcher), merge `ISS_TUNING` per key: start from defaults.env, and let the variant's keys override only themselves. Log the effective string; the probe already prints `PORT_TUNING` lines.

### C2. The bench cannot guarantee VSync off under renderer v2

- **Location:**
  - v2 does not read `tlg.ini`'s `vsync` or `fps_limit`. The present mode comes from `BB_PRESENT_MODE` (default Mailbox, `gpu_v2/gpu/shim/core/emulator_settings.h:26`), and the frame limit from `BB_FPS_LIMIT` (`:52-53`).
  - Exclusive fullscreen forces FIFO, i.e. VSync (`vk_swapchain.cpp:148`). The display mode comes from `tlg.ini` `display_mode` (`window.cpp:317`), which `launch()` rewrites from the player's settings.json (`infamous_run.py:958-962`).
  - An unsupported Mailbox also falls back to FIFO (`vk_swapchain.cpp:430-435`).
  - `bench.ps1:785` copies only `vsync` into the run's settings.json.
- **How it misleads:** if the player's saved display mode is exclusive fullscreen, every bench run is capped at the monitor's refresh rate, and every toggle measures "no change". The `ini:vsync` and `ini:fps_limit` variants are A/A runs under v2.
- **Fix:** set `display_mode=windowed` (or borderless) in the run's settings.json, and `BB_PRESENT_MODE=Mailbox` (or `Immediate`) in the run environment. Fail the run when the log shows `falling back to Fifo` or a FIFO swapchain, and record the present mode in `bench.json`.

### C3. The player's "Draw pipeline" switch is overwritten by defaults.env, and most launcher switches do nothing in v2

- **Location:**
  - `infamous_run.py:822` sets the backports and A/B switches (`env.update(backports_table.environment(…))`).
  - Then `:823-826` applies defaults.env for every key not in `os.environ`. So `ISS_DRAW_PIPE` from the Advanced screen is replaced by `defaults.env:22` (`ISS_DRAW_PIPE=0`).
- **Switches no v2 code reads**, from a scan of the v2 sources (this adds to the R2 dead-toggle table):
  - BP01-BP12, and about 55 `TLG_*` rows of the Fixes and Advanced screens;
  - `ISS_READBACKS` (v2 reads `BB_READBACKS`, `emulator_settings.h:30`);
  - `ISS_EAGER_READBACK`, `ISS_DEFERRED_READBACK`, `ISS_PIPELINE_BINARIES`, `ISS_UPSCALER`, `ISS_JITTER` and `ISS_OBJECT_MOTION`.
- **How it misleads:** variants such as `ini:bp08=0`, `ini:adv_*` or `ISS_READBACKS=0` are A/A runs. Players also see switches that do nothing.
- **Fix:**
  - apply defaults.env first and the player's switches after it;
  - mark the v1-only switches in the launcher when v2 is selected;
  - have `bench.ps1` reject `ini:` and env keys that v2 does not read, using a list generated from a `getenv` scan.

### C4. Dead or always-overwritten keys in `defaults.env`

- **Location:**
  - Never read by v2: `defaults.env` lines 10 (`ISS_CLAMP_CACHE_FIRST`), 33 (`ISS_FEEDBACK_ARENA`), 40 (`ISS_GPU`), 49 (`ISS_INDIRECT_GUARD`), 56-57 (`ISS_MAPPING_CACHE`, `ISS_MAPPING_INDEX`), 69-70 (`ISS_NATIVE_GEOMETRY`, `_STATS`), 84 (`ISS_NATIVE_REDUCTION`), 103 (`ISS_PROFILE_CITY_DRAW_GPU`), and 119 (`ISS_TLS_PATCH`, R2).
  - Always overwritten by `renderer_v2_env` (`infamous_run.py:176-179`): line 79 (`ISS_NATIVE_PASSES=1`) and line 124 (`ISS_UNIFIED_MEMORY=1`). The comment at `backports_table.py:251-253` says defaults.env has unified memory off, but the file says 1.
- **Fix:** delete them, and add a unit test that every defaults.env key has a `getenv` reader in the v2 sources.

### C5. Value conventions that break the usual 0/1 reading

- **`ISS_CP_CPU=0` pins the CP to logical CPU 0.** Off is spelled `off`, `none` or `-1` (`gpu_v2/gpu/bridge.cpp:775-781`). A "pinning off" variant written as `0` would make pinning look like a large win, or put the CP on the CPU that takes the interrupts.
- **`ISS_AOT_COMPUTE=0` does not turn AOT off.** Any non-empty value keeps the dynamic-stride variant and calls `StartAotCompute("0")`, which logs `index not found` (`vk_pipeline_cache.cpp:1296-1299,1355,1631-1634`). While `ISS_AOT_COMPUTE` is set, `ISS_DYNAMIC_STRIDE=0` has no effect either.
- **`ISS_UPLOAD_IDLE=1` means the old poll mode** (`buffer_cache.cpp:766`).
- **`ISS_READBACK_WINDOW_KB` accepts only 64 or 128.** Anything else, 256 included, becomes 512 (`bridge.cpp:558-560`).
- **An empty value in a bench variant means "the defaults.env value", not the code default** (`bench.ps1:802-813` with `infamous_run.py:825`). So `ISS_GPU_BLACKBOX=''` gives 0, and `ISS_UNIFIED_FLUSH_IRQ=''` gives 0, although the code default when unset is on (`vk_rasterizer.cpp:5600-5603`).
- **Fix:** accept `0` / `off` uniformly for boolean-like switches, warn on values a parser silently replaces, and document in `PERFORMANCE_DIAGNOSTICS.md` what "unset" means for a bench variant.

### C6. `TLG_` names in variants are ignored by v2, and stale shell `TLG_` twins beat defaults.env

- **Location:** v2 reads `ISS_` names directly: 345 `getenv("ISS_…")` calls against 9 for `TLG_`.
  - `bench.ps1:1099` accepts `TLG_X` variant keys and leaves `ISS_X` alone. The launcher then adds the defaults.env `ISS_X`.
  - In the other direction, a `TLG_` value left in the shell (e.g. `TLG_MUTEX_SPIN_US`, `TLG_LOG_BUFFER`) beats defaults.env, because `probe.c:1166` copies `ISS_X` to `TLG_X` only when `TLG_X` is unset. `bench.ps1:1122` warns only about twins of keys a variant sets.
- **How it misleads:** `TLG_FAST_DRAW=0` in a variant still runs with fast draw on. A stale shell variable silently changes every variant.
- **Fix:** rewrite `TLG_X` to `ISS_X` in variants, and warn about every shell `TLG_` twin of a defaults.env key.

### C7. `compare`'s `~` and "better" do not follow the paired statistics

- **Location:** `scripts/round_analyze.py:630-638`, and the table printed without Python (`bench.ps1:1755`). The noise mark is `abs(d) <= noise_k * stdev(control runs)`. It uses the control's unpaired spread, ignores the variant's own spread, the number of runs and the pairing, and has no floor. The adaptive stop (`bench.ps1:1340-1380`) uses paired CIs with a 0.5% SD floor.
- **How it misleads:**
  - Three control runs at 60.0 / 60.1 / 60.05 FPS give a 0.17% bar, so a variant at +0.4% from its own noise shows as a win.
  - With a 1.5% control CV, a real +2.3% (CI +0.8 to +3.7%, the documented `ISS_MUTEX_SPIN_US` result) shows as `~`.
- **Fix:** derive `~`, "better" and "worse" from the same paired CI and equivalence rule as the adaptive stop, and print the CI next to the mean difference.

### C8. Run order is unbalanced at the default round counts

- **Location:** `scripts/bench.ps1:1494` reverses the order on even rounds (A B / B A / A B). `-MinRounds 3` and `-MaxRounds 5` are odd, so the control runs first in 2 of 3 (or 3 of 5) rounds.
- **How it misleads:** any position effect (thermal state, disk cache) leaks into the difference as effect/n. A 2% penalty for running second shows as about 0.7% at n=3, close to the 1% equivalence margin.
- **Fix:** use ABBA blocks and stop only after an even number of rounds, or model the position as a factor and report it.

### C9. Smaller tooling issues

- **Warm-up asymmetry.** When every slot is warm, only the control gets a warm-up run (`bench.ps1:1478-1485`). A test variant's first measured run therefore starts with a cold OS file cache while the background pipeline preload runs (`vk_pipeline_serialization.cpp:662-672`).
- **Slot signature.** It leaves out defaults.env and the launcher (`bench.ps1:1271-1295`), so editing defaults.env keeps slots "warm".
- **`bench_cpu.cpp:63-83,132-137`.** A thread that could not be opened, or had no creation time, at window start is counted from zero, so its whole lifetime lands in the window. The class names `VkRecorder`, `tlg:Present`, `Shader*` and `DriverCacheSave` match no thread today; this is harmless.
- **Run status.** A run killed after its `Bench:` line stays `ok` even if the kill truncated a cache save (`slot.json` then says ok).

---

## Checked, no issue on the default path

- **Lost wakeups.** In each of these, the notifier takes the waiter's mutex or changes the state under it:
  - the flip-label reset and `SignalVoLabel`;
  - `vblank_cv` (count and notify under `vo_mutex`);
  - `submission_lock` / `cv_lock` (under `m_wait_idle`; the unlocked read in `sceGnmAreSubmitsAllowed` is benign);
  - equeue `TriggerEvent` with several waiters;
  - `submit_cv` (`num_submits`, `num_commands` and the queue all under `submit_mutex`);
  - `CommandCompletion` (CAS plus a binary semaphore);
  - the upload worker's `idle_woken` with its auto-reset event;
  - native passes `work_cv` / `done_cv` in both wake modes;
  - the scheduler's `recorder_cv`, `recorder_idle_cv`, `submitted_cv` and `priority_pending_ops_cv`;
  - `runtime_sema.c` (per-waiter condvar under the registry lock); `RpLock` reader preference.
- **The memory tracker.** The pending-readback count and the discard check are both done under the region lock. `ExtendWriteFault` marks pages CPU-modified before unprotecting them.
- **Disabled diagnostics on the CP.** `cp_path::Scope` / `Switch` cost one static-bool test when off, `diagnostics::Scope` never calls `NowNs`, `LOG_*` are guarded by `ShouldLog`, and the censuses and `FULL_PATH_STATS` are static-bool gated.
- **Per-draw bookkeeping on the CP.**
  - `PeriodicSubmit` makes no `NowNs` call.
  - `PopPendingOperations` returns early on an atomic count.
  - `Perf::Add` counters are per thread.
  - `ReportDue` costs one QPC per segment.
- **Recording.**
  - `CommandStream` allocates from an arena.
  - Bind, push and dynamic state are deduplicated (`BindCache`, `DynamicState::Commit`).
  - The graphics pipeline key is memoized.
  - `Claim` is a lock-free probe.
  - The `std::function` arguments to `NativePassExecute` fit the small-buffer optimization.
- **Not audited deeply:** the CP-split ("G4") paths, which are dead because `DecideCpSplit` always ends in `worker_page_fault_route_unavailable`, and the draw pipe (`ISS_DRAW_PIPE=0`).

## Suggested order

1. **Fix the bench first** (C1, C2, C5, C7). Every measurement after this depends on them, and they are script and launcher changes only.
2. **Cheap correctness fixes with no performance risk:** B6 (one lock), B9 (one `exchange`), B4 (atomic flag helpers), B2 (recursive unlock), B5 (bounded gate wait).
3. **Bench the existing toggles:** O1 (`ISS_ASC_NATIVE_QUEUE=1`) with a visual check, and O6 (`ISS_TEXTURE_LOOKUP_MEMO=1`).
4. **Build:** O2 (LTO first, then PGO).
5. **CP-side changes,** one A/B each: O5, O6 (format reuse), O7, O3.
6. **Pool-side changes,** judged by `cp_wait_us/submit` and FPS, not pool CPU: O9, O8, O4.
7. **Hang hardening:** B1 (backing-view token writes), B3 (unmaps on the CP), B7, B8.
