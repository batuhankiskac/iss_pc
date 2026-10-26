# Cloud audit, round 2: more frame-time and latency sources (renderer v2)

This is a second code-only pass over `main` at `fac17b4` (2026-10-26). It covers the areas that round 1 ([`cloud-audit.md`](cloud-audit.md)) did not reach, and repeats nothing from it. The rules are the same: nothing was run or changed, every `ms/frame` figure is an estimate from reading the code unless it quotes a number recorded in the code or the docs, and each finding says how to measure it. References to findings in round 1 are written as **R1-n**.

The same model of the frame applies. The command processor (CP) limits the frame rate and the GPU is about 51% busy. Frame time comes from three things: the CP being blocked, the CP sitting idle while the guest has work it cannot hand over, and extra hops in the guest → CP → GPU → readback → guest loop. Plain CPU work on threads nobody waits for does not count.

Windows facts used here:

- A libc++ timed wait shorter than 2 ms spins.
- `Sleep` and winpthreads waits round up to the 15.6 ms tick.
- A Windows exception round trip costs about 5-15 µs.
- A `VirtualProtect` that removes access sends TLB-shootdown IPIs to every core running a thread of the process. The CP's core always runs one.

## Ranking

| # | Finding | Main location | Est. ms/frame | Toggle | Risk |
|---|---|---|---|---|---|
| 1 | **Linux only:** `ISS_CP_CPU=auto` pins the CP to logical CPU 14, often an E-core on hybrid CPUs, and keeps no other thread off it | `bridge.cpp:788-804` | +4-8 on Linux hybrid (Windows unaffected) | new `ISS_CP_CPU=auto` topology pick, `ISS_CP_EXCLUSIVE` | low |
| 2 | Every fence makes the CP run the upload worker's queued jobs (protections and copies) | `buffer_cache.cpp:2272-2281`, `upload_worker.h:58-60` | 0.3-2 | new `ISS_FENCE_UPLOAD_WAIT` | medium |
| 3 | Compute fairness yields to compute tasks that can only yield back, and each yield walks every mapped queue | `liverpool.cpp:1914-1921,1372-1420,3511-3514,4040-4042` | 0.2-1.5 | `ISS_COMPUTE_FAIRNESS=0` (exists); new `ISS_ASC_FAIRNESS=runnable`, `ISS_CP_QUEUE_BITMAP` | low |
| 4 | Locks held across `VirtualProtect`: the 16 MB region lock and the texture-cache lock, both shared by guest fault handlers and the CP | `region_manager.h:129-131,168-171`, `page_manager.cpp:455-466`, `texture_cache.cpp:389`, `texture_cache.h:227-230` | 0.1-1 | `late_protect` (exists, off, unmeasured); new `ISS_TEX_LATE_PROTECT` | low/medium |
| 5 | Readback prefetch rescans every hot window at every fence (5.6% of the CP in the forest, measured) | `buffer_cache.cpp:2282-2331` | 0.3-0.5 (forest) | new `ISS_PREFETCH_PER_WORK` | low |
| 6 | The guest's DingDong scans its compute ring for native ASC while holding the global submit lock | `gnmdriver.cpp:371`, `liverpool.cpp:4554-4558`, `native_passes.cpp:1805-1845` | 0.1-0.5 | new `ISS_DINGDONG_LOCK=queue` | low/medium |
| 7 | One readback-queue submission, recorded and submitted on the CP, per guest fault | `buffer_cache.cpp:912-974,2185,1559` | 0.1-0.5 | new `ISS_READBACK_SUBMIT_BATCH` | low |
| 8 | Renderer v2 ignores the runtime's write hooks, so large memcpys and file reads into tracked memory fault window by window (launcher's `ISS_WRITE_MERGE` and `iss_gpu_invalidate` are dead) | `gpu_host.cpp:170-178`, `bridge.cpp:374-381`, `runtime.c:189-210`, `runtime_file.c:396-405` | 0.05-0.3, plus streaming stutter | new `ISS_COPY_INVALIDATE_KB` | medium |
| 9 | CP spins at full power, with no `PAUSE`, while a guest-written label is unsatisfied | `liverpool.cpp:3396-3398,3308-3310,4040-4042` | 0 desktop; a few % on power-limited laptops | new `ISS_CP_WAIT=pause\|umwait` | low |
| 10 | Idle guest job workers poll in 1 µs RDTSC spins and never block | `spin_hook.h:3-10`, `probe.c:778-797` | 0-1.5 (clocks and placement) | new `ISS_SPIN_WAIT=tpause\|yield` | low |
| 11 | Driver pipeline-cache checkpoint (~217 MB) every 30 s after new pipelines: hitch source | `bridge.cpp:737-739`, `pipeline_checkpoint.cpp:21-34` | 0 average; occasional 20-150 ms hitch (estimate) | new `ISS_PIPELINE_CHECKPOINT_IDLE_S` | low |
| 12 | A guest memory release zeroes the whole range while holding the exclusive memory-map lock | `runtime_memory.c:534-561,276,344-347` | 0 unless the game releases during play | new `ISS_MAP_ZERO=lazy` | low |
| 13 | Any guest map or unmap invalidates every GPU memo keyed on the global mapping generation | `buffer_cache.cpp:267-282`, `stage_bind_memo.inl:102` | <0.3 if maps happen during play | new `ISS_MAPPING_GEN=region` | low/medium |
| 14 | The guest thread that submits graphics work has no placement or priority; guest affinity masks are recorded but never applied | `runtime_thread.c:504-507,627-630` | 0.1-0.7 | new `ISS_GUEST_SUBMIT_PCORE` | low |
| 15 | SSE4A `EXTRQ` sites use the slow C stub; a tested inline stub exists but is off | `isa_rewrite.c:293-321`, `probe.c:1690-1693` | 0.15-0.25 of guest CPU | `ISS_EXTRQ_INLINE=1` (exists) | low |
| 16 | Guest sleeps below 1 ms round up to the high-resolution timer's ~0.5 ms grain | `compat_win.c:120-133` | <0.1 | new `ISS_USLEEP_PRECISE` | low |

Bench-only items (no code change needed): 3 (`ISS_COMPUTE_FAIRNESS=0`, as a first probe), 4 (`late_protect=1` in `ISS_TUNING`) and 15 (`ISS_EXTRQ_INLINE=1`). When you override `ISS_TUNING`, repeat its default keys (`defaults.env:121`).

## Findings

### 1. Linux only: `ISS_CP_CPU=auto` pins the CP to logical CPU 14

- **Location:** `port/gpu_v2/gpu/bridge.cpp:788-804`. The code is `long wanted = choice.mode == CpMode::Auto ? 14 : choice.lp;` followed by `pthread_setaffinity_np(pthread_self(), …)`. The README documents this ("On Linux `auto` behaves like the old default `14`"), and the Windows path picks the core by topology (R1 context).
- **Mechanism:**
  - On Intel hybrid CPUs, Linux numbers the P-core hyperthreads first. On the i7-13700H that is CPUs 0-11 for the P-cores and 12-19 for the E-cores, so CPU 14 is a Gracemont E-core. This is the same core the README says was wrong on Windows ("the old default `14` was an E-core").
  - Linux also keeps no other thread off that CPU: the comment at `:789-790` notes that CPU sets have no Linux equivalent.
  - On a non-hybrid SMT CPU with 8 cores, CPU 14 is the hyperthread sibling of CPU 6, which other threads use.
- **Proposed fix:**
  1. In `auto`, pick the CP's CPU from sysfs: `/sys/devices/cpu_core/cpus` on hybrid machines, otherwise the highest `cpufreq/cpuinfo_max_freq`, and use `topology/thread_siblings_list` to take a whole core. Never pick CPU 0.
  2. Before any guest thread is created (in `iss_gpu_initialize`), set the process's main-thread affinity to all CPUs except the CP's core and its sibling, so every later thread inherits the mask. The CP sets its own single-CPU mask afterwards.
- **Toggle:** `ISS_CP_CPU=auto` with the topology pick (keep the old behaviour behind `ISS_CP_CPU=14`), and `ISS_CP_EXCLUSIVE=1` for step 2.
- **Expected effect:** on Windows, moving the CP to its own P-core gave 54.5 → 58.7 FPS. Running ~10 ms of CP cycles per frame on an E-core at ~60% of P-core speed would cost **+4-8 ms/frame** on a Linux hybrid laptop.
- **Risk:** low. Check `lscpu -e` on the machines you test.
- **How to measure:** the log line `PORT_TUNING cp_cpu=14 mode=auto(linux: 14)`, `lscpu -e` (MAXMHZ per CPU), then an A/B of `ISS_CP_CPU=14` against `ISS_CP_CPU=<a P-core>` on Linux.

### 2. Every fence makes the CP run the upload worker's queued jobs

- **Location:**
  - `port/gpu_v2/gpu/overrides/buffer_cache.cpp:2272-2281`: `PrefetchReadbacks` calls `upload_worker->WaitIdle()` whenever `async_uploads && async_written_uploads`, before any of its early returns.
  - It is called from `Rasterizer::OnFence` (`vk_rasterizer.cpp:3309-3313`), which runs at every EOP, EOS, WriteData and compute ReleaseMem fence (`liverpool.cpp:3005`, `:3081`, `:3258`, `:4055`).
  - `WaitIdle` (`port/gpu_v2/shadps4/video_core/buffer_cache/upload_worker.h:58-60`) calls `Wait` (`buffer_cache.cpp:818-826`). If the worker has not taken the jobs, the CP takes `exec_mutex` and runs `ExecuteQueued` itself: `RefreshDeferredProtect` (VirtualProtect) plus the guest → staging copies.
  - Defaults: `ISS_UPLOAD_WORKER=1`, and `async_written_uploads=1` in the default `ISS_TUNING`.
- **Mechanism:** with `ISS_UPLOAD_IDLE=sleep`, the default since 2026-10-25, the worker wakes only every 1 ms or once 8 jobs are queued (`buffer_cache.cpp:790`, `:841`). Fences come much more often: the compute rings alone fence "thousands of times a second" (comment at `:2286-2288`).
  - So most jobs probably run on the CP, the frame-limiting thread, in small batches, which also gives fewer merged protect runs.
  - When the worker happens to be in the middle of a batch, the CP blocks on `exec_mutex` instead.
- **Proposed fix:** the wait exists so that the read protection of GPU-written ranges is in place before a label lets the guest read them. That matters only for fences the CP writes *immediately*.
  - A deferred unified-memory fence is written by the pending-ops runner after its submission. That submission already waits for the jobs it depends on: `upload_dependency` is set at `vk_scheduler.cpp:814-815` and waited on in `SubmitToQueue` (`:929-935`, through `SetSubmitDependency` at `buffer_cache.cpp:746`).
  - Record the sequence number of the last job that carries a protection in `Enqueue`. Call `Wait(that_seq)` only from fences that are written immediately.
- **Toggle:** `ISS_FENCE_UPLOAD_WAIT=all|immediate` (`off` for diagnosis only).
- **Expected effect:** most of the upload worker's work moves off the CP: **0.3-2 ms/frame** of CP time. This interacts with `ISS_UPLOAD_IDLE=sleep`: the measured +5.3% bundle already included the change of who runs the jobs.
- **Risk:** medium. A label written before its range's read protection lets the guest read stale memory without faulting. First check that every job carrying a protection is covered by the submission dependency.
- **How to measure:** add `AsyncUploadFenceWaitNs` around the `WaitIdle` and count jobs run by the CP versus the worker. `AsyncUploadBusyNs` is kept per thread (`perf_stats.h:606-614`). Then bench A/B.

### 3. Compute fairness yields to compute tasks that can only yield back

- **Location:**
  - `port/gpu_v2/gpu/overrides/liverpool.cpp:1914-1921`: with `ISS_COMPUTE_FAIRNESS=1` (`asc_yield_packets=16`, `bridge.cpp:564-565`, inherited from the Castlesea profile), the graphics task yields every 16 packets while `asc_submits != 0`.
  - `:3511-3514`: a compute task whose ring the native pool is still recording yields straight back (`while (!AscRecorded(…)) YIELD_ASC_QUEUE`).
  - `native_passes.cpp:1895-1919`: `AscRecorded` takes `submit.pool_mutex` on every call and gives up after 50 ms.
  - `:4040-4042`: a compute `WaitRegMem` on an unsatisfied label also yields straight back.
  - The outer loop (`liverpool.cpp:1372-1420`) runs its whole service block on every slot visit, including empty slots (`:1407-1419`: `curr_qid = (curr_qid + 1) % num_mapped_queues`, then `continue` when the slot is empty). The block is `ProcessCommands`, `RetireReadbacks` (which may call `vkGetSemaphoreCounterValue`), `FlushUnifiedFences`, the `m_access` lock, and so on.
- **Mechanism:** `asc_submits` counts every compute submission that has not completed, including ones that cannot run: waiting for the pool to record them (native ASC, default on since 2026-10-18, after fairness was set up), or waiting for a graphics label.
  - While one is pending, graphics gives up the CP every 16 packets. The loop then walks every mapped queue slot to reach the compute task, which re-tests and yields, and walks them all again to get back to graphics.
  - Each pass takes the pool mutex as well, against six pool threads that need it to pop jobs and report completion.
  - `liverpool.h:65` (`static_assert(NumTotalQueues < 64u); // need to fit into u64 bitmap for ffs`) shows that upstream intended a bitmap of non-empty queues. The loop here scans with a modulo instead.
- **Proposed fix:**
  1. Count only *runnable* compute submissions for the fairness trigger. Set a per-queue "blocked" bit when a task yields from `AscRecorded` or from a label wait. Clear it on DingDong, on pool completion, or when the awaited label tests true (the task publishes the address, mask, reference and function of the wait).
  2. Keep a `u64` bitmap of queues that have submissions, pick the next one with `std::countr_zero`, and run the service block once per round instead of once per slot.
  3. In `AscRecorded`, read `work->jobs_left` (an atomic) before taking `pool_mutex`.
- **Toggle:** bench the existing `ISS_COMPUTE_FAIRNESS=0` first. New: `ISS_ASC_FAIRNESS=all|runnable` (fix 1) and `ISS_CP_QUEUE_BITMAP=1` (fix 2).
- **Expected effect:** each useless yield cycle costs ~2-5 µs: two coroutine switches, two `SetAsyncCompute` calls, and one service pass per mapped queue. At up to ~1000 yields a frame while a compute submission stays blocked, that is **0.2-1.5 ms/frame** of CP time.
- **Risk:** low for fixes 2 and 3. For fix 1, every wake source of a blocked queue must be covered. Keep a fallback yield every N×16 packets, so a missed wake only delays compute work.
  - With fairness off, compute results may arrive later for the guest (particle readbacks). Watch the readback waits in the A/B.
- **How to measure:**
  - The `Perf:` line: `GfxYields`, the ASC start delay (`AscStartDelayNs/AscStarts`, `perf_stats.h:933`), the ASC busy share.
  - `NATIVE_ASC waits / wait_us/wait / timeouts` (`native_passes.cpp:1968-1979`). A non-zero `timeouts` means a 50 ms stall of that compute queue.
  - `ISS_CP_PATH_CYCLES=1`.

### 4. Locks held across `VirtualProtect`, shared by guest fault handlers and the CP

- **Location:**
  - **Region lock:**
    - `port/gpu_v2/gpu/shim/video_core/buffer_cache/region_manager.h:129-131` and `:168-171`: `ChangeRegionState` / `ForEachModifiedRange` take the region `std::mutex`, then `tracker->BeginLateProtect()`.
    - `page_manager.cpp:455-466`: `Protect` runs at once unless `late.active`, with the page locks held (`:387`, `:476`).
    - `late_protect` is `false` by default (`port/gpu_v2/shadps4/common/perf_stats.h:541`) and is not in the default `ISS_TUNING`.
  - **Texture cache:**
    - Guest side: a write fault on an image page takes `TextureCache::mutex` and runs `UntrackImage` → `Protect` under it (`texture_cache.cpp:389-412`).
    - CP side, same mutex: `UpdateImage` (`shim/video_core/texture_cache/texture_cache.h:227-230`: `scoped_lock{mutex}`, `TrackImage` (write-protect), `RefreshImage` (upload)) and `FindImage` on memo misses (`texture_cache.cpp:977`).
- **Mechanism:** every `VirtualProtect` is a system call, and one that removes access also triggers a TLB shootdown, while one of these locks is held.
  - The CP's upload path (`ForEachUploadRange`) and a guest write fault on the same 16 MB region wait for each other's system call, plus an SRW wake.
  - In the texture cache, a guest write fault on an image page holds the lock that the CP needs for every texture-memo miss and every image refresh.
  - Hypothesis to check: the shootdown IPIs land on the CP's core whichever thread calls `VirtualProtect`, so protect churn costs the CP even when the upload worker does the protecting.
- **Proposed fix:**
  1. Bench `ISS_TUNING=…,late_protect=1`. It already exists: it collects the protect runs and applies them after the region lock is released, keeping the page locks. It has no A/B on record.
  2. Do the same in the texture cache: collect the runs under `mutex` and apply them after unlocking.
  3. Separately, protect less often: once per submission, not per fence (finding 2).
- **Toggle:** existing `late_protect`; new `ISS_TEX_LATE_PROTECT=1`.
- **Expected effect:** 5-30 µs per contended fault or upload, so **0.1-1 ms/frame**. More in fault bursts: the periodic 45 ms frames had ~2875 faults each (`page_manager.cpp:743-746`).
- **Risk:** low for `late_protect`, since the page locks stay held until the protection is applied. Medium for the texture cache: image (un)tracking must not race a new `TrackImage` on the same pages.
- **How to measure:** `LateProtects` (`perf_stats.h:237-238`) to confirm the path runs. Add `RegionLockWaitNs` and `TexLockWaitNs` (time spent after a failed `try_lock`). For the shootdown hypothesis, count interrupts on the CP's core with ETW (DPC/ISR, IPI). Then bench A/B.

### 5. Readback prefetch rescans every hot window at every fence

- **Location:** `port/gpu_v2/gpu/overrides/buffer_cache.cpp:2282-2331` and following (`PrefetchReadbacks` walks `hot_windows` at each `OnFence`).
  - The cost is measured in the comment at `:2285-2291`: "the forest: ~110k scans for ~11k prefetches recorded, 5.6% of the command processor".
  - `ISS_PREFETCH_PER_SUBMIT=1` was measured and rejected because guest readback waits grew (Aftermath 1.4 → 4.8 ms a frame).
- **Mechanism:** most fences follow no new GPU work that could write a hot window. `PER_SUBMIT` skipped too much because it keyed on *submissions*, and work recorded into the open command buffer also produces new results.
- **Proposed fix:** rescan only when GPU work was *recorded* since the last scan, using a counter bumped by every recording path that writes guest-visible memory. `MemoryTracker::BumpWriteSeq` (`buffer_cache.cpp:4167`) and the GPU-write publication path are candidates. A fence after no new recorded work cannot see new GPU writes, and the first fence after new work still prefetches.
- **Toggle:** `ISS_PREFETCH_PER_WORK=1`.
- **Expected effect:** most of the measured 5.6% of the CP in the forest: **0.3-0.5 ms/frame** there, less in the city. Readback waits should not grow the way they did with `PER_SUBMIT`.
- **Risk:** low, as long as the counter covers native segments too.
- **How to measure:** `FOREST_SKIPS prefetch_scans=` (`buffer_cache.cpp:1266-1272`) and the `Perf:` line's `readback … prefetch … hits`. Bench on a forest route as well as the city route.

### 6. The guest's DingDong scans its compute ring while holding the global submit lock

- **Location:**
  - `port/gpu_v2/gpu/overrides/gnmdriver.cpp:371`: `SubmitLockGuard` in `sceGnmDingDong`, held until it returns.
  - `liverpool->SubmitAsc(…)` runs under it and reaches `liverpool.cpp:4554-4558` (`iss::native_passes::BeforeAscSubmit`).
  - `native_passes.cpp:1805-1845` runs on the submitting thread: it copies the queue's shadow registers, walks the ring and its indirect buffers, allocates the work (`make_shared`) and queues the pool jobs under `pool_mutex`. `scan_us` is accumulated at `:1843`.
- **Mechanism:** `m_submit_lock` is a single process-wide mutex. Every graphics submission, every `sceGnmSubmitDone` and every DingDong of every compute queue takes it.
  - A job thread's DingDong scans its compute submission while holding it. The city has ~300 tiny dispatches a frame and the forest ~1300 (comment at `native_passes.cpp:1710-1712`).
  - The render thread's `sceGnmSubmitCommandBuffers`/`sceGnmSubmitDone` waits behind that scan, and so does the CP, which needs the submission to continue.
  - Ordering only matters per compute queue: `asc_next_offs_dw[vqid]`, the per-queue shadow under `AscShadow::mutex`, the per-queue FIFO in `SubmitAsc`.
- **Proposed fix:**
  - In `sceGnmDingDong`, use a per-`vqid` mutex for the offset update and `SubmitAsc`. Keep `m_submit_lock` only around the gate check (`WaitGpuIdle`) and the debug-pause hook.
  - Give `sceGnmUnmapComputeQueue` (`gnmdriver.cpp:2650-2665`), which takes `m_submit_lock` to fence against DingDongs, the per-queue lock as well.
- **Toggle:** `ISS_DINGDONG_LOCK=global|queue`.
- **Expected effect:** **0.1-0.5 ms/frame** on the render thread. The ceiling is the `Submit gate:` line's `lock contention ms/frame` (`SubmitLockWaitNs`, `SubmitLockGuard` at `gnmdriver.cpp:130-145`).
- **Risk:** low/medium. `SubmitAsc` must still reject a closing queue (`g4_ledger.CanIssue`, under `submit_mutex`).
- **How to measure:** the `Submit gate:` line's `lock contention ms/frame` and the `NATIVE_ASC scan_us/submit` line. Then bench A/B.

### 7. One readback-queue submission per guest fault, recorded and submitted on the CP

- **Location:**
  - `port/gpu_v2/gpu/overrides/buffer_cache.cpp:912-974` (`ReadbackQueue::Submit`): commits a command buffer, records begin, copy, barrier and end, then calls `GetReadbackQueue().submit`.
  - It is called once per request from `:2185` (`BeginReadback`) and `:1559` (`BeginRelaxedReadback`).
  - Only `PrefetchReadbacks` batches several windows into one submission.
  - The readback queue is its own `VkQueue` (queue index 1 of the family, `vk_instance.cpp:725-731`), so this path takes no lock.
- **Mechanism:**
  - A `vkQueueSubmit` costs ~20-40 µs on Windows drivers, because each submission goes through the kernel.
  - Readbacks come in clusters: several job threads fault on the particle and feedback buffers right after the same fence.
  - Each fault pays its own submission on the CP, inside the round-trip that the faulting threads are waiting for.
- **Proposed fix:** collect the readback-queue jobs of one `ProcessCommands()` drain, which can hold several `SendCommand` callbacks, and submit them together at the end of the drain, under one tick.
- **Toggle:** `ISS_READBACK_SUBMIT_BATCH=1`.
- **Expected effect:** **0.1-0.5 ms/frame** of CP time: `ReadbackQueueSubmits`/frame × ~20-40 µs, minus one submission per cluster. Round-trips also get shorter for the later faults of a cluster.
- **Risk:** low: order within a drain is unchanged.
- **How to measure:** `ReadbackQueueSubmits` (`Perf:` `queue … flushes`), and with `ISS_TIMELINE=1` the `readback_copy` durations.

### 8. Renderer v2 ignores the runtime's write hooks, so large memcpys and file reads fault window by window

- **Location:**
  - `port/src/runtime.c:189-210`: guest memset/memcpy/memmove of 64 bytes or more call `tlggpu_cpu_write`. The comment promises "no fault and no GPU wait with TLG_WRITE_MERGE=1".
  - `runtime_file.c:396-405`: file reads go through a bounce buffer, then `tlggpu_cpu_write`.
  - In renderer v2, `port/gpu_v2/gpu/gpu_host.cpp:170-178` does nothing unless `ISS_READBACK_DISCARD=1`. It returns 0, so the runtime writes normally and every page faults.
  - The launcher still sets `ISS_WRITE_MERGE=1` (`port/launcher/infamous_run.py:787`), but no v2 code reads it.
  - `iss_gpu_invalidate` (`port/gpu_v2/gpu/bridge.cpp:374-381`) has no caller in `port/src`. It is Castlesea's measured fix for file reads, behind `ISS_RUNTIME_WRITE_INVALIDATE`: "the game streams ~5 MB of textures every ~2 s, ~1470 exceptions, part of the periodic 45 ms frames".
- **Mechanism:** an exception round trip (5-15 µs) is paid for each 64 KB window of clean tracked pages (`ISS_FAULT_WINDOW_KB=64`). GPU-modified pages fault every 4 KB until the 3-fault streak logic starts (`buffer_cache.cpp:1384-1420`), and each of those also reads the page back (R1-4, R1-16). The streaming thread and job threads pay this during texture streaming.
- **Proposed fix:**
  - In v2's `tlggpu_cpu_write`, for ranges of 64 KB or more that contain tracked pages, make one `InvalidateMemory(range)` call before the copy, as `iss_gpu_invalidate` does.
  - Skip the call when nothing in the range is tracked: `BumpBindEpoch` resets the CP's bind memos.
  - Combine it with `ISS_READBACK_DISCARD` for whole pages that the GPU has written (R1-16).
  - Log once that `ISS_WRITE_MERGE` has no effect in v2, so the launcher option stops being misleading.
- **Toggle:** `ISS_COPY_INVALIDATE_KB=64` (0 = off).
- **Expected effect:** **0.05-0.3 ms/frame** on average, plus fewer of the periodic streaming stutters.
- **Risk:** medium. A range invalidation reads GPU-modified pages back once, for the whole range, through the async path. Check that it cannot run on the CP thread (`IsCurrentCpThread`).
- **How to measure:** `Frame stats … write faults/s`, `Readback stats:` `stream entries` and `likely whole-page overwrites`, the `FAULT_WINDOW` counters. Bench `worst_ms`/`over33` on a route that streams.

### 9. The CP spins at full power while a guest-written label is unsatisfied

- **Location:**
  - `port/gpu_v2/gpu/overrides/liverpool.cpp:3396-3398`: graphics `WaitRegMem` (`while (!Test) YIELD_GFX();`).
  - `:3308-3310` (`MemSemaphore`) and `:4040-4042` (compute).
  - The outer loop at `:1372-1420`, which repeats its service block with no `_mm_pause`, back-off or sleep.
- **Mechanism:** the CP is not doing frame-limiting work here; it is waiting for the guest. But a pinned P-core runs a full-speed loop that re-takes `queue.m_access` and may call `vkGetSemaphoreCounterValue` on every pass. On a power-limited laptop (the reference machine is an H-series part), a P-core held at full turbo costs several watts of the cores' shared budget, so the guest threads run at lower clocks.
- **Proposed fix:** after N failed tests, when no command is queued and no other queue is runnable:
  - With WAITPKG (CPUID.(EAX=7,ECX=0):ECX[5], Alder Lake and later, possibly hidden under Hyper-V/VBS): `_umonitor(label)`, re-test, then `_umwait(1, tsc_now + ~10 µs)`. This wakes on the guest's store to that cache line, and the deadline bounds `SendCommand` latency.
  - Without it: `_mm_pause` with exponential back-off up to ~2 µs.
  - In both cases, skip `RetireReadbacks` on passes where `pending_readbacks` is empty.
- **Toggle:** `ISS_CP_WAIT=spin|pause|umwait`.
- **Expected effect:** 0 ms/frame on a desktop. Possibly a few % FPS on a power- or thermally-limited laptop. Speculative, needs the bench.
- **Risk:** low, given the deadline.
- **How to measure:**
  - `GpuSpinNs` (`Perf:` `gpu thread … spin`).
  - Package power and per-core clocks (HWiNFO) during the bench.
  - A/B on mains power.

### 10. Idle guest job workers poll in 1 µs RDTSC spins and never block

- **Location:** `port/src/spin_hook.h:3-10`: two engine functions (`0x53b200`, `0x53b2b0`) busy-wait on RDTSC for waits up to 500 µs. Their callers include "job workers waiting 1 us for work (0x1f2e40, 0x243cec), waits on counters in 50 us steps". `probe.c:778-797` only adds a `PAUSE` (`ISS_SPIN_PAUSE`, on by default; the README reports +0.3%, within noise).
- **Mechanism:** idle job workers never sleep. They stay runnable at normal priority, and this has two effects:
  1. They hold P-cores, so the threads the CP waits on drift to E-cores: the native-pass pool, the recorder and the pending-ops runner (R1-3, R1-15).
  2. On an H-series laptop, 6-7 cores kept in C0 lower the turbo clock left for the CP.

  `PAUSE` only helps the SMT sibling.
- **Proposed fix:** route the two functions to a host wait. Use `tpause` to the TSC deadline (WAITPKG; `mwaitx` on AMD, `PAUSE` otherwise). After ~50-100 µs of back-to-back calls from one thread, call `SwitchToThread()` / `sched_yield()` on each call. Wake-up latency stays in microseconds.
- **Toggle:** `ISS_SPIN_WAIT=pause|tpause|yield` (default `pause`, today's behaviour).
- **Expected effect:** **0-1.5 ms/frame**, unmeasured. The upper bound assumes most guest busy time is idle polling and the CP's clock rises 10-15%.
- **Risk:** low with `tpause`. `yield` can add scheduling latency to job pickup.
- **How to measure:** the `Bench CPU threads:` log line (guest threads at ~17 ms of cycles per frame each are spinning), `cpu_guest_ms/f`, the CP core's effective clock. Then bench A/B.

### 11. Driver pipeline-cache checkpoint every 30 s after new pipelines: a hitch source

- **Location:** `port/gpu_v2/gpu/bridge.cpp:737-739` (every 30 s, a detached thread runs `CheckpointDriverCache()`), `port/gpu_v2/gpu/pipeline_checkpoint.cpp:21-34` (`getPipelineCacheData`, ~217 MB per the comment at `:21`, then `XXH3_64bits` over the data). It runs only when pipelines were created since the last checkpoint (commit `ca1aab1`).
- **Mechanism:**
  - The driver probably holds its cache lock while it serialises. A compile on a compiler thread waits for it, the recorder waits for that compile (lazy binds, `WaitPipelineReady`), and the CP then waits for the recorder at the next flip (R1-1).
  - Allocating, copying and hashing 200-400 MB costs page faults and memory bandwidth at the same moment.
  - It fires right after new pipelines, which is exactly when the game is streaming new content.
- **Proposed fix:** checkpoint only after ≥10 s with no pipeline creation, reuse one buffer, and run the thread at `THREAD_MODE_BACKGROUND_BEGIN`.
- **Toggle:** `ISS_PIPELINE_CHECKPOINT_IDLE_S=10`.
- **Expected effect:** 0 on average. An estimated occasional 20-150 ms hitch when a compile overlaps a checkpoint.
- **Risk:** low: a crash loses at most the pipelines compiled since the last idle period.
- **How to measure:** with `TLG_PERF_TRACE=1`, look at captures triggered by a frame of 100 ms or more and check whether a `DriverCacheSave` thread was active at the same time.

### 12. A guest memory release zeroes the whole range while holding the exclusive map lock

- **Location:** `port/src/runtime_memory.c:534-561` (direct-memory release: `write_lock()` … `host_discard(start,size)` at `:559` … `write_unlock()`), `:276` (`host_discard` = `memset(backing_base+phys, 0, size)` on Windows), `:344-347` (`flex_free` → `host_discard`, reached from munmap at `:413`, with the lock held).
- **Mechanism:** every GPU backing lookup waits behind the exclusive map lock (R1-13): the CP's stream copies, readback finishes and fault handlers. Zeroing tens of MB takes several ms. Linux punches a hole in the file instead (`:308-310`), which is cheap.
- **Proposed fix:** zero after `write_unlock()`. The range is already removed from the tables and cannot be reused before the zeroing ends if the release path keeps the physical range marked busy until then. Alternatively, zero lazily when the range is next allocated.
- **Toggle:** `ISS_MAP_ZERO=locked|after|lazy`.
- **Expected effect:** 0 if the game releases nothing during play. Otherwise a stall of several ms per large release, on every thread that does GPU lookups.
- **Risk:** low with `after`, as long as the physical range stays reserved until it is zeroed.
- **How to measure:** compare the exit report's `direct memory allocations=…, maps=…` and `flexible maps=…` (`runtime_memory.c:950-952`) between a 60 s and a 120 s run of the same route: releases during play show up as the difference.

### 13. Any guest map or unmap invalidates every GPU memo keyed on the global mapping generation

- **Location:** `port/gpu_v2/gpu/bridge.cpp:345-350` (`iss_gpu_map` → `RegisterMemory` + `MapMemory`), `vk_rasterizer.cpp:5829-5858` (`MapMemory`/`RegisterMemory`/`UnmapMemory` → `NoteMappingChange`), `buffer_cache.cpp:267-282` (`mapping_generation.fetch_add(1)`), consumers such as `stage_bind_memo.inl:102` (`ISS_STAGE_MEMO=1` default) and `native_draw_run_rasterizer.inl:66-71`.
- **Mechanism:** the generation is a single global counter. Any guest map anywhere, even far from a GPU resource, makes every stage-bind memo entry stale, so the next draw of each stage takes the slow path on the CP. `UnmapMemory` also runs the buffer and texture cache invalidations on the guest thread, under the locks the CP takes for every draw.
- **Proposed fix:** keep the generations per 16 MB region (the `prepared_read_mapping_versions` scheme at `buffer_cache.cpp:273-278` already does this) and compare the generations of the regions a memo entry covers. Start with the stage memo.
- **Toggle:** `ISS_MAPPING_GEN=global|region`.
- **Expected effect:** 0 unless the game maps during play. Otherwise about one frame of memo misses per map, **<0.3 ms/frame**.
- **Risk:** low/medium: 62 call sites use `MappingGeneration()`.
- **How to measure:** as in finding 12, plus the `stage_last_memo_checks/hits` columns of `SHADPS4_FRAME_CSV`.

### 14. The guest's submitting thread gets no placement or priority, and guest affinity is ignored

- **Location:** `port/src/runtime_thread.c:504-507` and `:627-630` (`a->affinity=mask; return 0;`, `t->attr.affinity=mask; return 0;`: recorded, never applied). On Windows, the only placement rule is that the process default CPU sets exclude the CP's core (`bridge.cpp:1013`).
- **Mechanism:** the guest thread that calls `sceGnmSubmit*` and `sceGnmSubmitDone` runs at normal priority among ~7 busy guest threads, on any of the remaining P- and E-cores. When it lands on an E-core, or shares a core with a spinning worker (finding 10), the submit-gate gap (R1-2) and the DingDong lock (finding 6) get longer.
- **Proposed fix:** at its first graphics submission, give the calling thread P-core CPU sets (without the CP's core) and `THREAD_PRIORITY_ABOVE_NORMAL`. Optionally, map guest affinity masks onto P-core sets.
- **Toggle:** `ISS_GUEST_SUBMIT_PCORE=1`; `ISS_GUEST_AFFINITY=ignore|pcore`.
- **Expected effect:** **0.1-0.7 ms/frame** (speculative).
- **Risk:** low. Windows has no priority inheritance, so raise only the thread that submits.
- **How to measure:** the `Submit gate:` `idle->admit gap` and `lock contention`; ETW ready time of the submitting thread.

### 15. SSE4A `EXTRQ` sites use the slow C stub; a tested inline stub exists but is off

- **Location:** `port/src/isa_rewrite.c:293-321` (the default stub: `pushfq`, 11 GPR pushes, two XMM spills, `call isa_emulate_sse4a`, `popfq`), `probe.c:1690-1693` (`ISS_EXTRQ_INLINE`, default 0), `port/docs/isa_rewrite.md:20-27` (all 96 sites are jumps and none trap; the top ten each run ~3.2 M times per 60 s).
- **Mechanism:** each hit costs ~70-100 cycles against ~15 for the inline stub, and there are ≥11 k hits a frame. This is guest CPU time, so it shows in frame time only if the routine (`0x890a77-0x89475c`) is on the path to a submission.
- **Proposed fix:** bench `ISS_EXTRQ_INLINE=1` (covered by `test_isa_sse4a`). If it is neutral or better, set it in `defaults.env`.
- **Toggle:** existing `ISS_EXTRQ_INLINE=1`.
- **Expected effect:** **0.15-0.25 ms/frame** of guest CPU; FPS effect likely small.
- **Risk:** low.
- **How to measure:** `ISS_RUNTIME_COUNTERS=1` (`ISA: most executed sites`), bench `cpu_guest_ms/f`.

### 16. Guest sleeps below 1 ms round up to the high-resolution timer's ~0.5 ms grain

- **Location:** `port/src/compat_win.c:120-133` (each `usleep`/`nanosleep` waits on the per-thread high-resolution timer).
- **Mechanism:** the engine's own waits of 500 µs or less spin in guest code (finding 10), so only middleware that polls with something like `usleep(100)` is affected: it sleeps ~0.5 ms instead of 0.1 ms.
- **Proposed fix:** below 1 ms, sleep for (ns − 0.5 ms) on the timer, then spin with `PAUSE` until the deadline.
- **Toggle:** `ISS_USLEEP_PRECISE=1`.
- **Expected effect:** **<0.1 ms/frame**.
- **Risk:** low.
- **How to measure:** `TLG_WAIT_STATS=1` (records sleep waits with the requested µs).

## Corrections to round 1

- **R1 "Checked" list, `spin_lock.cpp`:** `Common::SpinLock` is not used by the page manager's page locks. `page_manager.cpp` only includes the header; its page locks are `std::mutex` (`page_manager.cpp:590-600`). The only `SpinLock` users are the `top_locks` of `multi_level_page_table.h:58-80`, for first-touch allocation with `NULL_CHECK`, which is cold.
- **R1-6 (`submit_mutex`):** the CP's `on_submit` callback (`vk_rasterizer.cpp:745-756`) does more than sparse binds: `runtime.FlushBarriers()`, handing the deferred unified fences to the pending-ops runner (`DeferPriorityOperationsAt`), and the unified host barrier. Moving it out of the lock must keep that hand-off ordered with the submission that signals the fences' tick. The runner is per scheduler, so it probably is ordered, but check it.

## Dead toggles found in round 2

These are set but have no effect in renderer v2. Remove them, or log that they are ignored, so A/B runs do not test nothing:

| Variable or key | Set in | Why it has no effect |
|---|---|---|
| `ISS_TLS_PATCH=1` | `defaults.env:119` | Nothing reads it. The TCB loads are always patched (`probe.c:288-302`). |
| `parallel_protect` (in `ISS_TUNING`) | `bridge.cpp:608` | Parsed, read nowhere in `port/gpu_v2`. The runtime's protect path already takes the map lock shared (`runtime_memory.c:928-939`). |
| `ISS_WRITE_MERGE` | `infamous_run.py:787` | Not read by v2 (finding 8). |
| `ISS_RUNTIME_WRITE_INVALIDATE` | — | Guards `iss_gpu_invalidate` (`bridge.cpp:374-381`), which nothing calls (finding 8). |

## Checked, no issue on the default path

- **Exception paths:** all 96 SSE4A sites are jumps, none int3 (int3 is only a logged fallback). Unpatched sites would stop the game rather than trap. TLS reads are rewritten to the TEB slot, with no traps. `rdtsc` runs natively. `cpuid` sites were never counted; under VBS each one is a VM exit, so a one-time scan is worthwhile. All hooks are jmp/call bridges.
- **Guest memcpy hooks:** in v2 they are a static-bool check (`gpu_host.cpp:170-178`), with no lock or lookup.
- **Fault dispatch:** a `std::set` walk of two handlers (`shim/core/signals.h:24-28`).
- **`TextureCache::InvalidateMemory`:** answered without the lock when the range has no images (`texture_cache.cpp:384-387`); see finding 4 for when it does lock.
- **Perf counters:** per thread, so they cost no shared atomics (`perf_stats.h:606-614`).
- **Staging pool:** 16 MB ring blocks, and `Reserve(…, false)` never waits (a full ring grows by a block, logged `STAGING_RING`).
- **Command pools:** never wait.
- **Stream buffer:** 128 MB; it waits only if a lap moves more than that (watch `GpuWaits`).
- **Scheduler:** `Flush`, `BeginSession` and `EndSession` with threaded recording are cheap apart from R1-6.
- **Scout:** `Submit` only queues and `Begin` does not block; a register-file copy runs on the CP after a sequence gap.
- **`ProcessDownloadImages`:** empty without `BB_READBACK_LINEAR`.
- **Hot-page cooling:** off (`hot_page_faults=0`).
- **Gnm census and engine markers:** opt-in.
- **Present path:** RenderDoc start/end capture are no-ops. Vulkan validation is off by default.
- **Audio:** paced on the high-resolution timer.
- **AJM:** decodes synchronously on audio threads only.
- **Video decode:** cutscenes only.
- **Pad:** reads take their own lock plus SDL's; with no gamepad attached, `SDL_GetGamepads` runs on every read (µs).
- **File reads:** synchronous on the streaming thread, with a 4 MB bounce buffer per thread. One more `getenv` per file open (`runtime_file.c:288`), cheap.
- **Save data:** opens and closes the file on every call, but only while saving.
- **Time:** QPC and rdtsc only; clock ids 2 and 15 make a `GetProcessTimes` system call.
- **Red-zone stubs:** 5,898 jmp patches, with an A/B documented as within noise.
- **Guest heap census, pause guard:** opt-in or cheap.

## Suggested order

1. **Bench only:**
   - `ISS_COMPUTE_FAIRNESS=0` (finding 3; watch readback waits);
   - `late_protect=1` (finding 4);
   - `ISS_EXTRQ_INLINE=1` (finding 15);
   - on Linux, `ISS_CP_CPU=<P-core>` against `auto` (finding 1).
2. **Small code changes, each behind its toggle:**
   - finding 1 (Linux topology pick);
   - finding 2 (immediate-fence-only upload wait);
   - finding 3 (runnable-only fairness and queue bitmap);
   - finding 5 (prefetch per recorded work);
   - finding 7 (batched readback submits);
   - finding 6 (per-queue DingDong lock);
   - finding 8 (range invalidate for large writes);
   - finding 12 (zero after unlock).
3. **Measurement first:** findings 9, 10, 11, 13 and 14. Their effect depends on platform power limits or on how often the game maps memory, so collect the counters named above before writing code.
