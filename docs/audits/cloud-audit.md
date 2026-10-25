# Cloud audit: frame-time and latency sources (renderer v2)

This is a code-only audit of `main` at `fac17b4`, done on 2026-10-25. Nothing was run or changed. Every `ms/frame` figure below is an estimate made by reading the code, not a measurement, unless it cites a number recorded in the code or the docs. Each finding says how to measure it with counters the build already has, or with a counter to add.

## Starting point

These facts come from `docs/PERFORMANCE_DIAGNOSTICS.md` ("Findings" and "Runtime defaults") and the README:

- The GPU is about 51% busy, so the game is not GPU-bound.
- The command processor (CP, `shadPS4:GpuCommandProcessor`) limits the frame rate. Giving it a P-core took the frame rate from 54.5 to 58.7 FPS. That is about 17 ms per frame, of which the CP spends about 10-11 ms in cycles.
- Making the pool and the CP cheaper did not raise FPS. Removing stalls did: unbuffered logging, and libc++ timed waits on Windows.

Two kinds of cost therefore matter:

1. **CP blocked or idle while the frame has work.** The CP waits on another thread, on the GPU or on a lock, or it has nothing to do because the guest cannot hand work over. Any of these adds directly to frame time.
2. **Hops in the guest → CP → GPU → readback → guest loop.** Each one adds to a wait that the CP, or the guest thread feeding it, sits in: readback round-trips, fences, thread wake-ups.

Windows behaviour these findings rely on (MSYS2 clang64, libc++, winpthreads, timer resolution 15.6 ms because `ISS_TIMER_1MS` is off):

- A libc++ `condition_variable::wait_for` truncates the remaining time to whole milliseconds. A wait shorter than 2 ms turns into a busy loop of 0 ms waits that lasts until the deadline. A wait of 2 ms or more that nobody notifies sleeps until the next 15.6 ms tick.
- `std::this_thread::sleep_for`, `Sleep(n)` and winpthreads timed waits round up to the 15.6 ms tick.
- The high-resolution waitable timers (`bridge.cpp` `IssPreciseSleepUs` / `IssIdleWait*`, `compat_win.c` `sleep_timer`) are accurate to about 0.5 ms.

## Ranking

Ranked by expected value: the estimate weighted by how sure the mechanism is to fire on the bench route.

| # | Finding | Main location | Est. ms/frame | Toggle | Risk |
|---|---|---|---|---|---|
| 1 | CP waits for the recorder thread to finish the whole frame at every flip | `vk_presenter.cpp:706-721`, `vk_scheduler.cpp:383-396` | ~1-1.5 (code comment: ~14% of the CP) | `ISS_PRESENT_STREAM` (exists, broken), new `ISS_PRESENT_CMDBUF=own` | medium |
| 2 | Submit gate lockstep: the CP idles at each guest frame boundary | `gnmdriver.cpp:112-127,2612-2615,2631-2640` | 0.2-2 | `ISS_SUBMIT_GATE` (exists, unmeasured) | medium |
| 3 | CP waits ~0.9 ms/frame for native-pass pool threads that run at normal priority, may sit on E-cores and are woken all at once | `native_passes.cpp:1411-1463`, `bridge.cpp:119-121` | 0.2-0.6 | new `ISS_NATIVE_PASSES_PRIO`; existing `ISS_NATIVE_PASSES_CPU`, `_WAKE` | low |
| 4 | Every async readback makes two CP round-trips; the second is often redundant | `buffer_cache.cpp:1674-1716` | 0.2-1.5 | new `ISS_READBACK_GUEST_FINISH` | low/medium |
| 5 | Recorder thread falls asleep between batches; each batch pays a wake-up before the GPU sees it | `vk_scheduler.cpp:467-485`, `bridge.cpp:563` | 0.2-1 | `ISS_TUNING=recorder_spin_us=N` (exists) | low |
| 6 | One static `Scheduler::submit_mutex` makes the CP wait behind `vkQueuePresentKHR` and upload copies | `vk_scheduler.cpp:510-511,820-826,929-935`, `vk_presenter.cpp:1186` | 0.1-1 | new `ISS_SUBMIT_LOCK=split` | medium |
| 7 | GPU batches go out late, so readbacks miss the readback queue and wait for everything recorded before them | `vk_rasterizer.cpp:155-187`, `buffer_cache.cpp:2107-2108,2189-2195` | 0.3-2 | `ISS_SUBMIT_INTERVAL_US` (exists, unset) | low |
| 8 | Deferred unified fences wait up to 1 ms, or a whole task, before they are submitted | `vk_rasterizer.cpp:5596-5637`, `defaults.env:122-123` | 0-1 | `ISS_UNIFIED_FLUSH_IRQ/_US` (exist) | low |
| 9 | CP drains the GPU for label waits that are already satisfied (`SyncUnifiedRead` runs first, with 64 KB stamps) | `liverpool.cpp:3362,3307,3454,4030`, `buffer_cache.cpp:4209-4230` | 0-2 | new `ISS_UNIFIED_WAIT_ORDER`, `ISS_UNIFIED_CHUNK_KB` | low |
| 10 | Pending-ops runner: blocking wait plus a thread wake for every label or EOP the guest waits on | `vk_scheduler.cpp:1238-1260` | 0.1-1 | new `ISS_PENDING_OPS_SPIN_US` | low |
| 11 | Flip-done label is released only after `Present()` returns (acquire fence, ImGui, queue present) | `videoout_driver.cpp:420-473` | 0 to one refresh, *if* the CP waits on flips | new `ISS_FLIP_EARLY_LABEL` | low/medium |
| 12 | VO-label wait: 1 ms busy spin, commands served about once per ms, open work not submitted (the doc's follow-up) | `liverpool.cpp:3376-3398`, `driver.h:60-63` | 0-1 | new `ISS_VO_WAIT`, `ISS_VO_WAIT_FLUSH` | low |
| 13 | Guest memory-map table lock: one winpthreads mutex taken twice per GPU backing lookup | `runtime_memory.c:75,108-114,845-860`, `rwlock_rp.h:24-38` | 0.1-1 | new `ISS_MAP_LOCK=atomic` | medium |
| 14 | Synchronous readbacks wait for the whole GPU to go idle on the CP thread | `buffer_cache.cpp:1708-1733,1791-1806,2706-2711` | 0 to the `GPU idle waits` ms | new `ISS_READBACK_SYNC=tick` | medium |
| 15 | Helper threads the CP waits on (recorder, pending ops, present) have no P-core placement | `bridge.cpp:1008-1015` | 0.1-0.5 | new `ISS_GPU_HELPERS_CPU=pcore` | low |
| 16 | Write-fault readbacks of pages that are about to be overwritten whole | `buffer_cache.cpp:1341-1365,1442-1462` | 0-1 (mostly stutter) | `ISS_READBACK_DISCARD` (exists, unmeasured) | medium |
| 17 | Havok critical sections spin ~1.3 ms on Raptor Lake before blocking | `havok_spin.h:24-27`, `probe.c:811` | 0-0.5 (1% lows) | `ISS_HAVOK_SPIN` (exists) | low |
| 18 | Relaxed readback sites still pay a full CP round-trip (the `0x1717c0` test was confounded) | `buffer_cache.cpp:1657-1667,1527-1585` | 0-0.3 | new `ISS_RELAXED_ASYNC` | medium |
| 19 | 15.6 ms timer: guest timed waits, `sceKernelWaitEqueue` timeouts | `bridge.cpp:579-591`, `equeue.cpp:196-204,277-305`, `runtime_mutex.c:241` | 0-0.5 | `ISS_TIMER_1MS` (re-bench), new `ISS_EQUEUE_WAIT=hr` | low |
| 20 | `binary_semaphore` waits fall back to `Sleep()` backoff on libc++ older than LLVM 22 | `liverpool.h:54-58,258` | 0 now; several ms on an old toolchain | build check | none |
| 21 | Window thread's `sleep_for(2 ms)` sleeps ~15.6 ms, so input is polled at ~64 Hz | `bridge.cpp:740` | 0 FPS; up to 15 ms input latency | new `ISS_WINDOW_POLL_US` | low |
| 22 | `VLog` formats twice and does a hash lookup under one global mutex even for suppressed repeats | `logging.cpp:9-20` | <0.1 | new `ISS_LOG_FAST_REPEAT` | none |
| 23 | Guest semaphores, rwlocks and `sched_yield` built on winpthreads | `runtime_sema.c:33,153,184`, `runtime_rwlock.c:90,128,145`, `runtime_kernel.c:175` | <0.1 each | `ISS_SEMA_SRW` (exists) | low |

The table lists more toggles to bench than code fixes to write. Items 2, 5, 7, 8, 16, 17 and 19 can be A/B-tested on `bench.ps1` today without code changes:

```text
.\scripts\bench.ps1 run -Exe '...' -Session gate `
    -Variants ([ordered]@{ base = @{}; gfx = @{ ISS_SUBMIT_GATE = 'gfx' } })
# then, one per session: ISS_SUBMIT_GATE=off; ISS_TUNING='async_written_uploads=1,scout=1,scout_fast=1,bind_prefetch=1,recorder_spin_us=300';
# ISS_SUBMIT_INTERVAL_US=500; ISS_UNIFIED_FLUSH_IRQ=1; ISS_UNIFIED_FLUSH_US=250; ISS_READBACK_DISCARD=1;
# ISS_HAVOK_SPIN=256; ISS_TIMER_1MS=1
```

When you override `ISS_TUNING` in a variant, repeat the default keys from `defaults.env:121`. The variant's value replaces the whole variable, not single keys.

## Findings

### 1. CP waits for the recorder thread to finish the whole frame at every flip

- **Location:** `port/gpu_v2/gpu/overrides/vk_presenter.cpp:706-721` (`PrepareFrame`: unless `ISS_PRESENT_STREAM=1`, it takes `draw_scheduler.CommandBuffer()`), `port/gpu_v2/gpu/overrides/vk_scheduler.cpp:383-396` (`CommandBuffer()` → `SyncRecorder()`), `:446-452` (`recorder_idle_cv.wait(... recorder_jobs.empty() && !recorder_busy)`). `PrepareFrame` runs on the CP: `videoout_driver.cpp:527-531`.
- **Mechanism:** with threaded recording (`ISS_PARALLEL_RENDER=1`), the CP hands batches of commands to `shadPS4:GpuCommandRecorder` and keeps going. At each flip, the presenter wants a direct command buffer for the frame copy and post-process pass. Getting one makes the CP wait until the recorder has recorded *and submitted* everything queued before it. `recorder_busy` covers `SubmitToQueue`, which includes `vkQueueSubmit` and the upload-dependency wait (`vk_scheduler.cpp:929-935`). The code's own comment (`vk_presenter.cpp:707-712`) measures this at "~14% of the command processor in the city, the next frame started only then". The CP is the limiter, so all of that is frame time.
- **Proposed fix:** keep the present commands off the recorder's command buffer without stopping the CP:
  - (a) Record the barrier and post-process pass into a command buffer from the presenter's own pool. Submit it from the present thread, waiting on `frame->ready_tick` (as `vk_presenter.cpp:1171` already does) once the recorder has *submitted* that tick. The present thread, not the CP, then blocks on `draw_scheduler.SubmittedTick() >= frame->ready_tick`.
  - (b) Or debug `ISS_PRESENT_STREAM=1`. Its black screen (2026-10-18, not debugged) is plausibly a wait-before-signal on one `VkQueue`: the present submission waits for a tick that the recorder thread submits later on the same queue. The `SubmittedTick` documentation (`port/gpu_v2/gpu/shim/video_core/renderer_vulkan/vk_scheduler.h:664-667`) warns against waiting on a tick that is not submitted yet. The same present-thread wait as in (a) would remove it.
  - FSR keeps state across frames and stays on the old path in both variants.
- **Toggle:** `ISS_PRESENT_CMDBUF=own` (a), or a fixed `ISS_PRESENT_STREAM=1` (b).
- **Expected effect:** **~1-1.5 ms/frame**, from ~14% of a ~10 ms CP frame. This is the largest CP-blocking item that has a number in the code.
- **Risk:** medium.
  - The image copy must still be ordered after the frame's rendering (it is, through the timeline wait), and the frame-pacing fence (`present_done`) must still be signalled.
  - In (a), the layout transition of the guest image (`runtime.Transit`, `vk_presenter.cpp:798`) goes through the texture cache's tracked layout state. It must stay in the CP's stream, or the separate command buffer has to use barriers that leave the tracked state consistent.
  - Check screenshots, HDR, and FSR on and off.
- **How to measure:** `SHADPS4_FRAME_CSV` column `recorder_sync_ms`; `RecorderSyncs` / `RecorderSyncNs` in the `Perf:` line. Bench A/B, and compare `cpu_cp_ms/f` with `-CpuSplit`.

### 2. Submit gate lockstep: the CP idles at each guest frame boundary

- **Location:**
  - `port/gpu_v2/gpu/overrides/gnmdriver.cpp:112-127`: `WaitGpuIdle`.
  - `:2567-2570` and `:2612-2615`: every graphics submission waits.
  - `:2631-2640`: `sceGnmSubmitDone` sets `submission_lock` while the CP is busy.
  - `:371-374`: DingDongs wait too in `idle` mode.
  - The lock is released only by the GpuIdle interrupt at `liverpool.cpp:1496`, which fires when the CP has run out of work.
  - The mode switch is in `port/gpu_v2/gpu/submit_gate.h`.
- **Mechanism:** with the default `ISS_SUBMIT_GATE=idle`, the guest's first submission of frame N+1 cannot reach the CP until the CP has drained all of frame N. The CP then sits idle until three things have happened: the guest's render thread wakes from `cv_lock.wait`, it finishes `PerformSubmit`, and it notifies `submit_cv`. Asynchronous compute DingDongs are held the same way. Every waiter waits while holding `m_submit_lock` (`SubmitLockGuard`, `:2612`, `:371`), so a job thread's DingDong queues behind the render thread's blocked submission. The gap occurs at least once per frame, and on a CP-limited frame all of it is frame time. The lockstep is inherited from shadPS4's handling of `sceGnmSubmitDone`.
- **Proposed fix:** none in code. The toggle exists (commits `0171e29` and `4719bc5`, both 2026-10-25) but has not been benchmarked. Bench `gfx` first, then `off`, which forces `ISS_FRAME_END_MARK=1`.
- **Toggle:** existing `ISS_SUBMIT_GATE=idle|gfx|off` and `ISS_FRAME_END_MARK=1`.
- **Expected effect:** **0.2-2 ms/frame**.
  - Floor: two thread wake-ups per frame (GpuIdle → guest, guest → CP, each roughly 20-100 µs on Windows), plus the rest of `PerformSubmit`.
  - Ceiling: the `idle->admit gap ms/frame` of the `Submit gate:` line.
- **Risk:** medium.
  - With `off`, the CP can fall a frame behind. The game's own frame-end label wait bounds this, and `max_gpu_lag_frames` can bound it more.
  - The README marks both modes experimental. Watch for hangs in menus and loading screens, and for particle readbacks that are one frame old.
- **How to measure:** `ISS_FRAME_STATS=1` prints the `Submit gate:` line: `wait ms/frame`, `idle->admit gap ms/frame`, `lock contention ms/frame`, `lag frames avg`. Then bench A/B.

### 3. CP waits ~0.9 ms/frame for native-pass pool threads that run at normal priority

- **Location:**
  - `port/gpu_v2/gpu/native_passes.cpp:1426-1463`: `WaitRecorded`, `submit.done_cv.wait`.
  - `:1411-1418`: `WaitJob`. It is called per command buffer from `liverpool.cpp:1868` and at job boundaries from `:1926` and `:3234`.
  - Measured wait, in the comment at `native_passes.cpp:1513-1514`: "the command processor's wait for the pool 1.47 -> 0.89 ms per frame in the city".
  - Priority list: `bridge.cpp:119-121`. It has no `ISS:NativePassesPool` or `ISS:NativePassesScan`.
  - Wake-up: `native_passes.cpp:1608-1617` uses `notify_all` by default (`ISS_NATIVE_PASSES_WAKE=all`), and each finished job calls `done_cv.notify_all()` (`:1176-1179`).
- **Mechanism:** the CP, running above normal on its own P-core, blocks until six normal-priority pool threads have recorded its next segment. Those threads compete with the guest's ~7 busy cores and may run on E-cores (`ISS_NATIVE_PASSES_CPU=off`). Waking all six on every queue, and waking the CP on every finished job, adds context switches. While the CP waits it also does not run `ProcessCommands()`, so guest readback round-trips (finding 4) that arrive meanwhile wait too.
- **Proposed fix:**
  1. Add both pool names to `raise_gpu_threads`.
  2. Bench `ISS_NATIVE_PASSES_CPU=pcore` and `ISS_NATIVE_PASSES_WAKE=one` (both exist).
  3. Notify `done_cv` only when the job the CP is waiting for completes (record the awaited index).
  4. In the wait loop, also wake on `num_commands`, and run `ProcessCommands()` for commands that record no GPU work (round-trip 2 of finding 4).
- **Toggle:**
  - New: `ISS_NATIVE_PASSES_PRIO=1` (fix 1) and `ISS_NATIVE_PASSES_SERVE_CMDS=1` (fix 4).
  - Existing: `ISS_NATIVE_PASSES_CPU`, `ISS_NATIVE_PASSES_WAKE`.
- **Expected effect:** about a quarter to two thirds of the measured 0.89 ms, so **0.2-0.6 ms/frame**.
- **Risk:** low for fixes 1-3. Fix 4 is low as long as it only admits commands that do not record into the open command buffer.
- **How to measure:** the `NATIVE_PASSES_SUBMIT` line: `cp_waits`, `cp_wait_us/submit`, `pool_wakes`, `pool_idle_wakes`. Then bench A/B.

### 4. Every async readback makes two CP round-trips; the second is often redundant

- **Location:**
  - `port/gpu_v2/gpu/overrides/buffer_cache.cpp:1674-1683`: round-trip 1, `SendCommand<true>` → `BeginReadback`.
  - `:1685`: the guest waits on the timeline semaphore.
  - `:1688-1716`: round-trip 2, repeated up to 4 times: `SendCommand<true>` → `CompleteReadbacks(false)` + `IsRegionGpuModified`.
  - `SendCommand<true>` itself: `port/gpu_v2/gpu/shim/video_core/amdgpu/liverpool.h:234-264`.
  - Entry from the fault handler: `page_manager.cpp:883-965` → `vk_rasterizer.cpp:5502` (write) and `:5785` (read).
  - The CP retires finished readbacks on its own only at `liverpool.cpp:1384` (between task resumes), in `TickFrame` (`buffer_cache.cpp:1277`) and at fences (`:2311`).
- **Mechanism:** a guest fault on GPU-written memory blocks the faulting thread for this whole sequence:
  1. CP pickup.
  2. `BeginReadback`.
  3. GPU copy.
  4. **A second CP pickup.**
  5. `FinishReadback`: a memcpy into guest memory and the page-state change.

  Once the semaphore is signalled, the data is already in a host-cached staging buffer. The second pickup only exists because that last step runs on the CP. It costs microseconds while the CP is decoding packets (`ProcessCommands()` runs per packet). It costs milliseconds while the CP is in a blocking section: findings 1, 3, 6 and 12, a `Flush`, or a lazy compile. `ReadbackQueueNs` only counts round-trip 1, so round-trip 2 does not show up in today's counters.
- **Proposed fix:**
  - (a) Cheap: give each `PendingReadback` a shared `std::atomic<u8> finished`. `FinishReadback` sets it and `BeginReadback` returns it. The guest skips round-trip 2 when the flag is set; the CP's `RetireReadbacks` often gets there first.
  - (b) Full: let the waiting thread run `FinishReadback` for its own readback under a readback mutex that `AssertGraphicsReadbackOwner` accepts. When an earlier overlapping readback is still pending (the ordering rule at `:2519-2529`), fall back to the CP.
- **Toggle:** `ISS_READBACK_GUEST_FINISH=0|1|2` (old / skip-if-finished / guest finishes).
- **Expected effect:** one CP pickup per async readback, about 20-200 µs while the CP is decoding and 1 ms or more while it is blocked. At ~8-30 critical-path readbacks a frame, that is **0.2-1.5 ms/frame** on the faulting threads. The share on the threads the CP waits for becomes frame time.
- **Risk:** (a) low: it skips a round-trip whose answer is already known. (b) medium: `pending_readbacks`, `memory_tracker` and `staging_pool.FreeDeferred` are single-owner today, so they need a lock audit.
- **How to measure:** run with `ISS_READBACK_DIAG=1 ISS_READBACK_DIAG_LOG=<csv>`.
  - Per request, `request_end − wait_end` is round-trip 2 plus `FinishReadback`.
  - The `guest_enqueue_done` duration is round-trip 1.
  - Also check the `readback_queue_ms` and `readback_async_wait_ms` columns of `SHADPS4_FRAME_CSV`.
  - Diagnosis runs cost ~16% FPS. Compare them only with each other, and use plain runs for the FPS A/B.

### 5. Recorder thread falls asleep between batches; each batch pays a wake-up before the GPU sees it

- **Location:** `port/gpu_v2/gpu/overrides/vk_scheduler.cpp:467-485` (spin `recorder_spin_us`, then `recorder_cv.wait`), `:420-441` (`DispatchStream` wakes it with a system call), `bridge.cpp:563` (`recorder_spin_us=20` with `ISS_PARALLEL_RENDER=1`). Measured pattern in the comment at `vk_scheduler.cpp:405-406`: "the recording thread is asleep for ~150 of ~160 batches a frame, each one a wake up system call".
- **Mechanism:** a 20 µs spin is shorter than the gap between 256-command batches, so the recorder sleeps between almost all of them. Each batch then waits for a `SleepConditionVariableSRW` wake-up before it is recorded and submitted. On a laptop core leaving a C-state that is roughly 20-150 µs, and on an E-core it is longer. The GPU work the guest's readbacks and fences wait on starts that much later, and the CP pays a wake-up system call per batch.
- **Proposed fix:** bench `recorder_spin_us=200`, `=500` and `=1000` in `ISS_TUNING`. The recorder already runs above normal and spinning is cheap compared with the latency it saves. If the knob works, adapt the spin to the observed batch gap (EWMA) instead of using a constant.
- **Toggle:** existing `ISS_TUNING=...,recorder_spin_us=N`.
- **Expected effect:** **0.2-1 ms/frame**, through earlier GPU start and fewer wake-ups on the CP.
- **Risk:** low. One more busy core; watch `cpu_gpu_helpers_ms/f`.
- **How to measure:** `SHADPS4_FRAME_CSV` columns `recorder_wakes`, `recorder_work_ms`. In the bench, `-CpuSplit` (`gpu_helpers_ms/f`), FPS and `low1_fps`.

### 6. One static `Scheduler::submit_mutex` makes the CP wait behind `vkQueuePresentKHR` and upload copies

- **Location:**
  - `port/gpu_v2/gpu/overrides/vk_scheduler.cpp:70` and `vk_scheduler.h:717`: the mutex is a single static for all schedulers.
  - `:820-826`: the CP takes it on every threaded flush, only to run `on_submit`.
  - `:510-511`: the recorder holds it across `SubmitToQueue`, which includes `submit_dependency_wait` at `:929-935`: `upload_worker->Wait`, which runs queued upload jobs inline (protections plus memcpys).
  - `:883`: non-streamed submits hold it too.
  - `vk_presenter.cpp:1186`: the present thread holds it for `swapchain.Present()`, i.e. `vkQueuePresentKHR`.
- **Mechanism:** the lock is needed so that each `VkQueue` gets externally synchronised submits. But the CP only takes it for `on_submit`, and `on_submit` needs the queue only when sparse binds are pending (`buffer_cache.cpp:4282-4293` returns early otherwise). As a result the CP can block behind:
  - a present call, which takes from ~0.1 ms to more than 1 ms depending on present mode and compositor;
  - the recorder's `vkQueueSubmit`;
  - the upload jobs the recorder executes under the lock.

  This happens roughly 20-30 times a frame.
- **Proposed fix:**
  1. Run `submit_dependency_wait` before taking the lock. It touches no queue.
  2. On the CP, call `on_submit` without the lock, and take the lock only inside `SubmitPendingArenaBinds` when `pending_binds` is non-empty.
  3. Use one mutex per `VkQueue` instead of one static mutex.
- **Toggle:** `ISS_SUBMIT_LOCK=global|split`.
- **Expected effect:** **0.1-1 ms/frame** of CP blocking, more with FIFO present or a slow compositor.
- **Risk:** medium. Submits to one queue must stay in order with respect to the timeline values they signal. Keep `NextTick()` and the submit under the per-queue lock, and check the `G4` asserts with `ISS_CP_SPLIT=0`.
- **How to measure:** time the lock acquisitions: add `SubmitLockWaitNs` per caller (CP, recorder, present) next to the `scoped_lock`s, and print it in the `Perf:` line. Also `PRESENT_ACQUIRE call_ms/fence_avg_ms` (`vk_swapchain.cpp:310`). Then bench A/B.

### 7. GPU batches go out late, so readbacks miss the readback queue

- **Location:** `port/gpu_v2/gpu/overrides/vk_rasterizer.cpp:155-187` (`PeriodicSubmit`: `ISS_SUBMIT_EVERY=1000`, `ISS_SUBMIT_INTERVAL_US` unset by default), `buffer_cache.cpp:2107-2108` (`use_queue = queue_enabled && writers.submitted`), `:2189-2195` (otherwise: copy into the open graphics command buffer, then `scheduler.Flush()`).
- **Mechanism:** the comment at `vk_rasterizer.cpp:158-160` records that with ~150 native segments a frame (each counts as one draw), batches went out ~13 times a frame, up to 2.4 ms apart. A readback whose writer is still in the open batch cannot use the readback queue. Its copy goes to the end of the graphics command buffer, so the guest waits for that whole batch and for everything submitted before it. Meanwhile the GPU sat idle while the batch was being recorded.
- **Proposed fix:** bench `ISS_SUBMIT_INTERVAL_US=500` and `=1000` (it exists). If one of them helps, make it the default. Optionally, also count native segments by their draw count in `ISS_SUBMIT_EVERY`.
- **Toggle:** existing `ISS_SUBMIT_INTERVAL_US`, `ISS_SUBMIT_EVERY`.
- **Expected effect:** **0.3-2 ms/frame** of readback wait and GPU start latency, minus ~20-50 µs of CP time per extra submit.
- **Risk:** low.
- **How to measure:** the `Perf:` line `readback producers done/submitted/queued/current` and `queue reads/fallbacks`; `SHADPS4_FRAME_CSV` `readback_queue_fallbacks`. Then bench A/B.

### 8. Deferred unified fences wait up to 1 ms, or a whole task, before they are submitted

- **Location:**
  - `port/gpu_v2/gpu/overrides/vk_rasterizer.cpp:5596-5615`: `DeferUnifiedFence` submits only if `irq && flush_irq`, or if at least `ISS_UNIFIED_FLUSH_US` has passed since the last flush.
  - `:5619-5637`: `FlushUnifiedFences`.
  - `port/gpu_v2/defaults.env:122-123`: `ISS_UNIFIED_FLUSH_IRQ=0`, `ISS_UNIFIED_FLUSH_US=1000`. Both were inherited from the Castlesea profile (commit `f5793f7`), and no A/B is on record.
  - `liverpool.cpp:1386`: the 1 ms check runs only in the outer loop, between task resumes.
  - `liverpool.cpp:1483`: the idle flush.
- **Mechanism:** with `ISS_UNIFIED_MEMORY=1`, guest threads wait on EOP/ReleaseMem labels and interrupts: job-system fences and `sceKernelWaitEqueue` on GfxEop. The pending-ops runner writes those labels only after the GPU finishes the command buffer that contains them, and that buffer is submitted only at the next `Flush`.
  - With `FLUSH_IRQ=0`, even interrupt fences are deferred.
  - The 1 ms deadline is checked only when another fence is deferred, or when the CP returns to its outer loop. A whole-frame graphics task does not return until it yields or completes.
  - So a fence early in a long command buffer can stay unsubmitted until the next readback flush, `ISS_SUBMIT_EVERY` batch or frame end.
- **Proposed fix:** first bench the existing knobs: `ISS_UNIFIED_FLUSH_IRQ=1`, then `ISS_UNIFIED_FLUSH_US=250`. In code, also check the deadline per packet: call `FlushUnifiedFences(false)` from `ProcessCommands()` or from `YIELD`. When nothing is pending, that costs one load and one compare.
- **Toggle:** existing `ISS_UNIFIED_FLUSH_IRQ`, `ISS_UNIFIED_FLUSH_US`; new `ISS_UNIFIED_FLUSH_POLL=packet`.
- **Expected effect:** up to 1 ms (more inside a long task) per fence that a critical-path thread waits on. **0-1 ms/frame**. Extra submits cost CP time, so keep the interval at or above ~250 µs.
- **Risk:** low: more, smaller submits, same ordering.
- **How to measure:** the `UNIFIED_STATS` line (`fences_deferred`, `fences_flushed` per frame); with `ISS_TIMELINE=1`, the `fence_defer` → `fence_write` durations (`vk_rasterizer.cpp:5573-5584`). Then bench A/B.

### 9. CP drains the GPU for label waits that are already satisfied

- **Location:**
  - `port/gpu_v2/gpu/overrides/liverpool.cpp:3362` (graphics `WaitRegMem`), `:3307` (`MemSemaphore`), `:3454` (`CondExec`), `:4030` (compute `WaitRegMem`).
  - These call `buffer_cache.cpp:4209-4230` (`SyncUnifiedRead`: host barrier + `sched.Wait(tick)`).
  - Write stamps: `buffer_cache.cpp:4161-4166`. Every *writable* binding in the unified heap stamps its whole 64 KB chunks with the current tick. The chunk size is `UNIFIED_CHUNK_SHIFT = 16` in `port/gpu_v2/gpu/shim/video_core/buffer_cache/buffer_cache.h:811`.
- **Mechanism:** before testing a memory wait, the CP waits for every unfinished GPU write to the label's 64 KB chunk. If that write is in the open tick, `Scheduler::Wait` submits and blocks until the GPU catches up. This happens even when:
  - (a) the label already holds the awaited value, or
  - (b) `PendingUnifiedLabel`, which is tested *after* the sync (`:3364-3372`), would have satisfied the wait from the CP's own deferred fence.

  Labels usually sit next to the data they guard. A label that shares 64 KB with any GPU-writable binding therefore costs a full CP → GPU drain. That drain is frame time, and it also starves the GPU of the next batch.
- **Proposed fix:**
  - Reorder the checks: `Test(current value)` → `PendingUnifiedLabel` → only then `SyncUnifiedRead`. A wait that is already satisfied needs no ordering against later GPU writes, and the hardware CP also just reads memory.
  - Separately, stamp at 4 KB (`UNIFIED_CHUNK_SHIFT = 12`: 128 K entries for the 512 MB heap, 1 MB of RAM) or with exact ranges.
- **Toggle:** `ISS_UNIFIED_WAIT_ORDER=sync_first|test_first`; `ISS_UNIFIED_CHUNK_KB=64|4` (the shift must become a runtime value).
- **Expected effect:** each avoided sync saves the GPU time left in the open batch plus a submit, 0.3-3 ms. How often it happens is unknown. `UNIFIED_STATS cp_read_wait/frame` is the ceiling. Plausibly **0-2 ms/frame**.
- **Risk:** low for the reorder. Suppose a label is satisfied now, and unfinished GPU work would overwrite it with a *non*-satisfying value. That is a guest race on hardware too.
- **How to measure:** `UNIFIED_STATS` `cp_read_syncs/frame` and `cp_read_wait/frame` (`unified_memory.h:41-42`), before and after. Then bench A/B.

### 10. Pending-ops runner: a blocking wait and a thread wake per guest-visible label

- **Location:** `port/gpu_v2/gpu/overrides/vk_scheduler.cpp:1238-1260` (`PriorityPendingOpsThread`: `priority_pending_ops_cv.wait`, then `work_semaphore.Wait(op.gpu_tick)`, which is `vkWaitSemaphores`, `port/gpu_v2/shadps4/video_core/renderer_vulkan/vk_semaphore.cpp:51-81`). Deferred fences are recorded at `vk_rasterizer.cpp:5578-5584` and handed to the runner at `:748` (`DeferPriorityOperationsAt`).
- **Mechanism:** each deferred label, EOP or interrupt costs two hops before the guest thread can wake:
  1. The GPU completes and the driver interrupt wakes the runner from `vkWaitSemaphores`.
  2. The runner writes the label or raises the IRQ, which wakes the guest.

  The runner is FIFO and serial. A large `TryWriteBacking` from an async image download (`texture_cache.cpp:341-347`, only when an image is downloaded at GC) delays every label queued behind it.
- **Proposed fix:**
  - For a tick that was just submitted, spin on `getSemaphoreCounterValue` (cheap on current drivers) for up to ~50-100 µs before blocking.
  - Give bulk downloads their own runner.
  - Longer term: for labels inside the unified range, write the value on the GPU (`vkCmdFillBuffer` / `vkCmdUpdateBuffer` after the host barrier), so no CPU hop is needed. Keep the runner for interrupts only.
- **Toggle:** `ISS_PENDING_OPS_SPIN_US=N`; `ISS_UNIFIED_GPU_LABELS=1` (long term).
- **Expected effect:** 20-150 µs per label that a critical-path thread waits on. **0.1-1 ms/frame**.
- **Risk:** low for the spin (one helper core busier). Medium for GPU-written labels: the guest must see the label only after the data it guards, which needs a host-visible barrier before the fill.
- **How to measure:** with `ISS_TIMELINE=1`, compare `fence_write` timestamps against the GPU completion time (`ISS_GPU_PROFILE=1` gives submission completion). Then bench A/B.

### 11. Flip-done label is released only after `Present()` returns

- **Location:** `port/gpu_v2/gpu/overrides/videoout_driver.cpp:420` (`presenter->Present(req.frame)`), then `:434-449` (flip status), `:457-467` (flip events), `:469-474` (`buffer_labels[previous_index] = 0; SignalVoLabel()`). Inside `Present`: `vk_swapchain.cpp:280-293` (acquire plus `waitForFences(acquire_fence)`, on by default through `ISS_PRESENT_ACQUIRE_FENCE`), ImGui, `vk_presenter.cpp:1177` (submit) and `:1186-1187` (`vkQueuePresentKHR` under `submit_mutex`, see finding 6).
- **Mechanism:** a guest's `WaitUntilSafeForRendering` / `InsertWaitFlipDone` waits on the label of the buffer being replaced. That buffer's content was copied to a host `Frame` by `PrepareFrame`, on the GPU queue, before any later guest rendering into it, so nothing reads it during `Present`. The label still waits for the acquire fence, the ImGui pass, the submit and the present call. In Mailbox (the v2 default, `emulator_settings.h:26` `BB_PRESENT_MODE`), the acquire is usually short. In FIFO (exclusive fullscreen, `vk_swapchain.cpp:148`), the code's own comment measures "~10.5 ms after the acquire returns" (`:252`).
- **Proposed fix:** in `Flip()`, update the flip status, raise the flip events and reset the previous label *before* `Present()`, not after it.
- **Toggle:** `ISS_FLIP_EARLY_LABEL=1`.
- **Expected effect:** the `Present()` duration per frame in which the CP actually waits for a flip. That is 0 if the CP never waits on flips, and up to one refresh in FIFO. The `hr` vblank clock gave no FPS change, which suggests flip waits are short on the bench route, so measure before expecting much in Mailbox.
- **Risk:** low/medium. The game sees the flip event slightly before the image reaches the swapchain, as it would at a vblank on PS4. Check frame pacing (`Frame pacing` line) and that `sceVideoOutGetFlipStatus` users still behave.
- **How to measure:**
  - `PRESENT_ACQUIRE call_ms/fence_avg_ms`.
  - With `ISS_TIMELINE=1`, the gap between `vo_flip_begin` and `vo_flip_events_labels_done`, and the CP's VO-wait time from finding 12.

### 12. VO-label wait: 1 ms busy spin, commands served about once per ms, open work not submitted

This is the follow-up in `PERFORMANCE_DIAGNOSTICS.md` ("Runtime defaults").

- **Location:** `port/gpu_v2/gpu/overrides/liverpool.cpp:3376-3398`, `port/gpu_v2/gpu/shim/core/libraries/videoout/driver.h:60-63` (`vo_cv.wait_for(lk, 1 ms, pred)`), signalled from `videoout_driver.cpp:472-473`.
- **Mechanism:** this is the same libc++ truncation that `ISS_UPLOAD_IDLE=sleep` fixed for the upload worker. Commit `28be8d4` describes it as "mostly returned at once … when it did round up to 1 ms it slept the 15.6 ms timer tick instead". It has three effects:
  1. The pinned CP core spins at 100% and keeps re-taking `vo_mutex`. On a laptop that takes power from the guest's cores, and the present thread (`videoout_driver.cpp:719`) and `sceVideoOutGetVblankStatus` contend for that mutex.
  2. `ProcessCommands()` (`:3385`) runs only between two `WaitVoLabelFor` calls: about once per ms, and once per 15.6 ms when the rounding goes the other way. A readback round-trip (finding 4) that arrives during the wait waits that long.
  3. Nothing is submitted before the CP blocks. Deferred fences (`FlushUnifiedFences` runs only at `:1386`/`:1483`, outside the task) and the draws recorded since the last `Flush` stay in the open command buffer for the whole wait. The GPU can go idle while work is waiting.
- **Proposed fix:**
  1. Replace the timed wait with an event wait: `SendCommand` sets an `std::atomic<bool> cp_in_vo_wait` flag and calls `vo_cv.notify_all()`, or reuse `IssIdleWait*` (`bridge.cpp:432-468`). Wait without a timeout on the predicate `label || num_commands || (yield_to_compute && asc_submits)`, and keep a 50 ms high-resolution safety timeout.
  2. Before the first wait, treat the CP as idle: call `FlushUnifiedFences(true)` and `RetireReadbacks()`, and call `Flush()` if the open command buffer holds work.
- **Toggle:** `ISS_VO_WAIT=poll|event` (fix 1) and `ISS_VO_WAIT_FLUSH=1` (fix 2), benched separately.
- **Expected effect:** 0 if the CP never waits on a flip label. Otherwise, for each wait: GPU starvation for up to its length (fix 2), plus ~0.5 ms on average for each readback that falls inside it (fix 1). **0-1 ms/frame**, and better 1% lows once the occasional 15.6 ms sleep is gone.
- **Risk:** low. A missed wake becomes a hang, hence the safety timeout. Fix 2 only submits earlier.
- **How to measure:** `EpochVoWaitsSkipped` (`vo` in the `Perf:` epochs) counts satisfied VO waits. Add a VO-wait ns/calls counter next to `blocked_ns` (`:3390`); today that time is folded into `GpuSpinNs`. Readback diag `guest_enqueue_done` durations that cluster around 1 ms are the queueing signature.

### 13. Guest memory-map table lock: one winpthreads mutex taken twice per GPU backing lookup

- **Location:**
  - `port/src/runtime_memory.c:75`: on Windows, a single global `RpLock`.
  - `port/src/rwlock_rp.h:24-38`: each read enter and each read exit is a `pthread_mutex_lock`/`unlock` pair on that lock's internal mutex.
  - `runtime_memory.c:108-114`: `read_lock` = `try_read_lock` plus `read_unlock`.
  - `:845-860`: `runtime_memory_backing_view`, which has no thread-local cache. Compare `runtime_memory_clamp` (`:898`), which has one (`:878-880`).
  - Callers: `host.backing` in `bridge.cpp:270-292` (`CopySparseMemory` through `sparse_copy.h:10-15`: at least one lookup per copy, one per 16 KB of a sparse copy; `TryWriteBacking` for every readback finish) and `bridge.cpp:367,379` (runtime readback/invalidate hooks).
  - Writers: guest map, protect and release calls take it exclusively. A direct-memory release also zeroes the range under it (`runtime_memory.c:534-559`, `host_discard`).
- **Mechanism:** the CP's stream copies, the upload worker, the native pool, readback finishes and the fault handlers all serialise on one mutex, with four atomic RMWs per lookup on one cache line. winpthreads mutexes do not spin, so any collision goes to a kernel wait (~10-50 µs). A guest `mmap`/`mprotect` holding the lock exclusively stalls every GPU lookup. Linux uses glibc's rwlock fast path.
- **Proposed fix:**
  - Readers: an atomic reader count with one CAS to enter, and `WaitOnAddress` only while a writer holds it. Keep the reader preference that the nesting comment at `rwlock_rp.h:1-9` depends on.
  - Add a generation-tagged thread-local cache to `backing_view`, as `clamp` has.
  - In `runtime_memory_visit` (`:863-873`), reuse a snapshot keyed by `table_generation` instead of allocating and copying the table on every call.
- **Toggle:** `ISS_MAP_LOCK=mutex|atomic`.
- **Expected effect:** **0.1-1 ms/frame**, scaling with lookups per frame (thousands with stream copies) and with guest map/protect activity during play.
- **Risk:** medium. A hand-written rwlock has to preserve the reader nesting that prevents the fault-handler deadlock described in `rwlock_rp.h`. Use the existing `port/tests/test_rwlock_rp.c` as the base for a stress test.
- **How to measure:** add a counter of `backing_view` calls and of contended `rp_*` acquisitions (`pthread_mutex_trylock` failing first). `ISS_RUNTIME_COUNTERS=1` is the place to report them. Bench `cpu_cp_ms/f` and FPS.

### 14. Synchronous readbacks wait for the whole GPU to go idle on the CP thread

- **Location:**
  - `port/gpu_v2/gpu/overrides/buffer_cache.cpp:1718-1734`: a fault on the CP thread itself (`assume_locks`).
  - `:1708-1713`: an async retry after the GPU rewrote the window, or after 4 rounds.
  - `:1791-1806`: `ReadbackSync`. At `:1794`, `CompleteReadbacks(true)` waits for *every* pending readback in FIFO order (`:2486-2501`).
  - `:2706-2711`: `DownloadMemory` → `FinishAllQueues` (`:1102-1125`), which submits and waits for idle on every queue.
- **Mechanism:** when the CP touches GPU-modified guest memory, or an async readback is retried, the CP first waits for unrelated pending readbacks and then for the GPU to go completely idle. `async_readbacks` removed this pattern for guest faults: the comment at `:1669-1671` describes the CP waiting "for the GPU to become idle, hundreds of times per second". It remains on these two paths, and each occurrence serialises CP recording with GPU execution.
- **Proposed fix:**
  - For the CP-thread fault, call `BeginReadback`, wait on the returned tick (readback queue when the writers are submitted), then `CompleteReadbacks(false)`. This waits for this window's writers only, not for idle.
  - In `ReadbackSync`, wait only for pending readbacks that overlap the window (`FirstUnfinishedReadback`).
- **Toggle:** `ISS_READBACK_SYNC=idle|tick`.
- **Expected effect:** 0.5-4 ms per occurrence. The `Readback stats:` line reports `GPU idle waits N (x ms)` per frame, and x is the ceiling. Likely small on average, but a stutter source when it fires.
- **Risk:** medium. The CP-thread fault happens in the middle of a packet. `BeginReadback` flushes the open command buffer just as `FinishAllQueues` does, so ordering is unchanged, but this path is rarely exercised. Validate with `ISS_READBACK_DIAG`.
- **How to measure:** `Readback stats:` `GPU idle waits` (and its ms); `SHADPS4_FRAME_CSV` `readback_retries`, `readback_sync_ms`.

### 15. Helper threads the CP waits on have no P-core placement

- **Location:** `port/gpu_v2/gpu/bridge.cpp:1008-1015` (`IssPinCommandProcessor`: gives the CP its own core and removes that core from the process default CPU set, nothing else), `:116-138` (priorities only). The threads concerned: `shadPS4:GpuCommandRecorder`, `shadPS4:GpuSchedPriorityPendingOpsRunner`, `shadPS4:PresentThread`, `shadPS4:GpuUploadWorker`.
- **Mechanism:** on a hybrid CPU (the i7-13700H used for the numbers), the recorder (finding 5), the pending-ops runner (finding 10) and the present thread (findings 1 and 11) can be scheduled on E-cores. There they wake more slowly and run a few times slower, while the CP or a guest wait sits on them. Pinning the CP to a P-core was worth +7.7% FPS, and the threads on the same chain were left to the scheduler.
- **Proposed fix:** give these four threads CPU sets of the highest efficiency class, excluding the CP's core. Reuse the class logic of `ISS_NATIVE_PASSES_CPU=pcore` (`native_passes.cpp:1136`).
- **Toggle:** `ISS_GPU_HELPERS_CPU=off|pcore`.
- **Expected effect:** **0.1-0.5 ms/frame**. Zero on non-hybrid CPUs.
- **Risk:** low. On parts with few P-cores, the helpers then compete with the game's P-core threads. Bench on 6P+8E and 8P+16E if possible.
- **How to measure:** the `Bench CPU threads:` log line (busiest threads, cycle ms/frame). For readiness, an ETW/WPA trace (CPU usage precise, ready time) of these threads. Then bench A/B.

### 16. Write-fault readbacks of pages that are about to be overwritten whole

- **Location:** `port/gpu_v2/gpu/overrides/buffer_cache.cpp:1341-1365` (a write fault → `ReadMemory(is_write=true)`: the two round-trips and GPU wait of finding 4), `:1442-1462` (`DiscardOverwrite`), `vk_rasterizer.cpp:5521-5535`.
- **Mechanism:** a guest `memcpy`, `memset` or file read over GPU-written pages reads each page back before overwriting it, although only the edge pages need it. The toggle exists (commit `2270b76`, 2026-10-25), is off by default, and has no A/B on record. The periodic 11 MB copy (`buffer_cache.cpp:1404-1406`: ~2870 faults, a 45 ms frame) is the extreme case.
- **Proposed fix:** bench it. The `Readback stats:` line already estimates `likely whole-page overwrites` per frame.
- **Toggle:** existing `ISS_READBACK_DISCARD=1`.
- **Expected effect:** about `likely whole-page overwrites × (write-fault ms / write faults)` per frame. **0-1 ms/frame** on average, and more on `over33` and `worst_ms`.
- **Risk:** medium. Correctness depends on each hooked writer really covering whole pages. Compare visuals.
- **How to measure:** `Readback stats:` `write faults (ms)` and `discard: calls / discarded / read back anyway`. In the bench, `low1_fps`, `over33`, `worst_ms`.

### 17. Havok critical sections spin ~1.3 ms on Raptor Lake before blocking

- **Location:** `port/src/havok_spin.h:3-9,24-27` (11 sites with counts of 1000-4000; "about 1,500 cycles an attempt on Raptor Lake, so 4000 spins for about 1.3 ms before blocking"), `port/src/probe.c:811-814` (original counts unless `ISS_HAVOK_SPIN` is set), with `ISS_MUTEX_SPIN_US=20` at the end (`runtime_mutex.c:118-135`).
- **Mechanism:** the counts were tuned for the PS4's Jaguar cores, where an attempt is far cheaper. Since Skylake, Intel's `PAUSE` takes ~140 cycles. A contended Havok lock burns up to 1.3 ms of a core, and its `trylock` CAS bounces the holder's cache line. On an SMT sibling this slows the lock holder, so the frame's physics critical path gets longer.
- **Proposed fix:** bench `ISS_HAVOK_SPIN=256` and `=64`. The doc already has the `h256` recipe.
- **Toggle:** existing `ISS_HAVOK_SPIN`.
- **Expected effect:** **0-0.5 ms/frame**, mostly in 1% lows.
- **Risk:** low: too low a count only means blocking earlier.
- **How to measure:** bench A/B (`low1_fps`, `p99_ms`, `cpu_guest_ms/f`).

### 18. Relaxed readback sites still pay a full CP round-trip

- **Location:** `port/gpu_v2/gpu/overrides/buffer_cache.cpp:1657-1667` (`SendCommand<true>` → `BeginRelaxedReadback`), `:1527-1585` (which may `scheduler.Flush()` at `:1565`).
- **Mechanism:** a relaxed site lets the guest read stale data instead of waiting for the GPU. The faulting thread still waits for the CP to pick up the command and record, and possibly flush, a download. `PERFORMANCE_DIAGNOSTICS.md` reports that relaxing `0x1717c0` "gave no measurable FPS gain", but that run still paid this round-trip, so it does not show that relaxing is useless.
- **Proposed fix:** on the guest thread, unmark the page (`UnmarkRegionAsGpuModified`, already page-locked) and return. Queue the download with `SendCommand<false>`, capturing by value, and keep the `relaxed_pages` refresh on the CP.
- **Toggle:** `ISS_RELAXED_ASYNC=1`.
- **Expected effect:** one CP pickup per relaxed fault, only while relaxed sites are configured. **0-0.3 ms/frame**. Its main value is making the `relaxed_site` experiment conclusive.
- **Risk:** medium. Guest-thread access to `memory_tracker` page state must not race `CollectReadback` on the CP.
- **How to measure:** `ISS_TUNING=...,relaxed_site=0x1717c0` with and without the toggle; `Perf:` `relaxed readbacks`.

### 19. 15.6 ms timer resolution: guest timed waits and `sceKernelWaitEqueue` timeouts

- **Location:**
  - `port/gpu_v2/gpu/bridge.cpp:579-591`: `ISS_TIMER_1MS`, off by default. The 2026-10-18 A/B behind that default was 2 runs per side.
  - `port/gpu_v2/shadps4/core/libraries/kernel/equeue.cpp:196-204`: a `sceKernelWaitEqueue` with a timeout uses libc++ `wait_for`.
  - `equeue.cpp:277-305`: `WaitForSmallTimer` spins on `yield()` for HR timers below the spin threshold.
  - Guest primitives on winpthreads: `port/src/runtime_mutex.c:192,241`, `runtime_sema.c:169`, `runtime_tlg.c:224`.
  - `sceKernelUsleep` is already correct: it uses the high-resolution timer at `compat_win.c:121-133`.
- **Mechanism:**
  - A timed equeue wait below 2 ms spins and keeps re-taking `m_mutex`, which `TriggerEvent` needs. A timed wait that nobody signals sleeps until the 15.6 ms tick.
  - A guest timed wait that expires (for example a short timed wait used as a poll) lasts a tick instead of its PS4 duration.
  - Signalled waits wake promptly, so this only costs anything when a frame-critical thread times out regularly.
- **Proposed fix:**
  - Re-bench `ISS_TIMER_1MS=1` on today's paired bench (~1.2% noise).
  - In `EqueueInternal`, use a per-queue auto-reset event set in `TriggerEvent`, plus the high-resolution timer (the `IssIdleWait*` pattern), instead of `wait_for`.
- **Toggle:** existing `ISS_TIMER_1MS`; new `ISS_EQUEUE_WAIT=hr`.
- **Expected effect:** probably small: **0-0.5 ms/frame**, mostly in 1% lows of other scenes.
- **Risk:** low. `timeBeginPeriod(1)` costs a little idle power.
- **How to measure:** count timeouts. Add counters at `equeue.cpp:487` (the `ETIMEDOUT` return) and at the guest cond/sema `ETIMEDOUT` paths, reported through `ISS_RUNTIME_COUNTERS=1`. Then bench A/B.

### 20. `binary_semaphore` waits fall back to `Sleep()` backoff on libc++ older than LLVM 22

- **Location:** `port/gpu_v2/gpu/shim/video_core/amdgpu/liverpool.h:54-58` (`CommandCompletion::sem`), `:258` (`completion->sem.acquire()`, on every `SendCommand<true>`, so on every readback round-trip).
- **Mechanism:** libc++ implements `atomic::wait` and the semaphores on Windows with `WaitOnAddress` only since llvm/llvm-project#163524 (merged 2025-11-13, so in LLVM 22). Before that, it polls, then yields, then calls `sleep_for(elapsed / 2)`, which on Windows becomes `Sleep(1)`: ~15.6 ms at the default timer resolution. CI builds against an up-to-date MSYS2 (`.github/workflows/build.yml:153-156`, `update: true`). A local MSYS2 that was never updated would insert a 15.6 ms sleep into any round-trip that takes longer than ~64 µs. `bench.json` records exe hashes but not the toolchain.
- **Proposed fix:** print `_LIBCPP_VERSION` in the `PORT_TUNING` block, and warn when it is below 220000. Optionally use an event (`IssIdleWait*`) here so the code no longer depends on libc++ at all.
- **Toggle:** none (a build check).
- **Expected effect:** 0 on a current toolchain; several ms/frame and very poor 1% lows on an old one.
- **Risk:** none.
- **How to measure:** run `clang --version` in the MSYS2 shell that built `tlg-probe-v2.exe`.

### 21. Window thread's `sleep_for(2 ms)` sleeps ~15.6 ms, so input is polled at ~64 Hz

- **Location:** `port/gpu_v2/gpu/bridge.cpp:740` (game loop of the window thread: `PollEvents` at `:726`), and `:711` (loading loop, `5 ms`).
- **Mechanism:** `sleep_for` rounds up to `Sleep(2)`, which lasts until the next 15.6 ms tick. SDL events, mouse motion and gamepad state are therefore taken in at ~64 Hz. That beats against the ~58 FPS game: some frames get no new input and some get two batches. This is latency and smoothness, not FPS.
- **Proposed fix:** `IssPreciseSleepUs(2000)` (`bridge.cpp:413`), or `SDL_WaitEventTimeout` with a 2 ms timeout.
- **Toggle:** `ISS_WINDOW_POLL_US=2000`.
- **Expected effect:** 0 ms/frame. Mouse/pad latency down by up to ~14 ms, and smoother mouse aim.
- **Risk:** low (slightly more wake-ups on the window thread).
- **How to measure:** `ISS_MOUSE_LOG=1` timestamps between events; subjective mouse judder.

### 22. `VLog`: global mutex, two `fmt` formats and a hash lookup per suppressed line

- **Location:** `port/gpu_v2/gpu/logging.cpp:9-20`. Hot callers include `liverpool.cpp:3283` (`LOG_WARNING` for every unhandled `IT_COPY_DATA`) and the stream-out warning at `:3436`.
- **Mechanism:** repeat suppression (only powers of two are printed) happens *after* formatting the text and a key string and looking the key up in an `unordered_map<std::string>`, all under one process-wide mutex. A warning issued per packet costs a few µs of CP time and serialises every thread that logs.
- **Proposed fix:** a per-call-site `static std::atomic<u64>` counter in the macro, checked before anything is formatted.
- **Toggle:** `ISS_LOG_FAST_REPEAT=1`.
- **Expected effect:** **<0.1 ms/frame**, unless a warning fires per packet.
- **Risk:** none.
- **How to measure:** read the `GPU_REPEAT count=` lines in a bench `game.log` and divide by frames.

### 23. Guest semaphores, rwlocks and `sched_yield` built on winpthreads

- **Location:**
  - Semaphores: `port/src/runtime_sema.c:33` (one global lock and a list scan), `:153` and `:184` (each blocking wait creates and destroys a winpthreads condvar, the destroy under the global lock).
  - Rwlocks: `port/src/runtime_rwlock.c:90,128,145` (three trips through `registry_lock` per read lock/unlock, and a `calloc` per first acquire at `:102`).
  - Yield: `port/src/runtime_kernel.c:175` and `runtime_thread.c:615`, where `sched_yield` becomes winpthreads `Sleep(0)`.
- **Mechanism:** each blocking semaphore wait costs ~5-15 µs of setup, serialised against every signal. Guest rwlock reads serialise on a global registry. `Sleep(0)` is a system call that never yields to lower-priority ready threads.
- **Proposed fix:**
  - Make `ISS_SEMA_SRW=1` the default and use a lock per semaphore.
  - Rwlocks: per-lock state, a thread-local holder record and an SRWLOCK fast path.
  - Implement the yield with `SwitchToThread()`.
- **Toggle:** existing `ISS_SEMA_SRW`, `ISS_EF_SRW`; new `ISS_RWLOCK_SRW=1`.
- **Expected effect:** **<0.1 ms/frame** each; the SRW toggles measured within noise.
- **Risk:** low.
- **How to measure:** `ISS_RUNTIME_COUNTERS=1` (semaphore blocking waits; add rwlock registry contention).

## Checked, no issue on the default path

- **Upload worker idle wait** (`buffer_cache.cpp:820-848`): fixed by `ISS_UPLOAD_IDLE=sleep` (event plus high-resolution timer).
- **CP main idle wait** (`liverpool.cpp:1283-1290`): an untimed `CondvarWait`, woken by `notify_all`.
- **Submit-gate wait itself** (`gnmdriver.cpp:119`): untimed (the cost is the gate, finding 2).
- **Timed waits that run only with `ISS_CP_SPLIT=1`:** `WaitCpSeal` and the split loop's `wait_for(1 ms)` (`liverpool.cpp:1129,1393`), and `ProcessAsc` (`:1537,1549,1612`). They have the same libc++ truncation; fix them with finding 12 if the split is ever enabled. The G4 waits and yields in `vk_rasterizer.cpp:836,2740,2812,2894,5486-5491` are also unreachable because `PrepareCpSplit` returns false.
- **Max-lag and sync-frame-end waits** (`liverpool.cpp:4457,4479`): only with the `max_gpu_lag_frames` / `gpu_sync_frame_end` tunings.
- **Lazy pipeline wait** (`vk_rasterizer.cpp:1299-1321`): pause, then yield, then 100 µs high-resolution sleeps, during compile bursts only. Steady state does not block the CP on compiles: `CompileAsync` uses `wait_us=0`, and `PipelineCompiler::WaitFor` is unused.
- **`ProcessDownloadImages`** (`texture_cache.cpp:220-226`, `scheduler.Finish()` per image): empty unless `BB_READBACK_LINEAR=1`.
- **Draw pipe** (`ISS_DRAW_PIPE=0`): `DrainDrawPipe` returns at once.
- **Scout:** never blocks the CP.
- **Fault-burst bookkeeping and stack scans** (`page_manager.cpp:743-760`): only with `ISS_FAULT_BURST_LOG=1`.
- **`vk_semaphore.cpp` waits:** `vkWaitSemaphores`, no polling. No `vkGetFenceStatus` polling anywhere. `getSemaphoreCounterValue` in `RetireReadbacks` and `CompleteReadbacks` is a cheap driver read.
- **Swapchain:** `WaitForSwapchainIdle` runs only on recreate, HDR change and destroy.
- **Present:** `PrepareLastFrame` fence (pause only); `GetRenderFrame` back-pressure is normally already signalled with 3 frames.
- **`spin_lock.cpp`:** a pure `_mm_pause` spin with no back-off. Its only user is the page manager, for short sections. A holder preempted mid-section would make waiters burn a quantum. Watch for it, but not on the hot path today.
- **`shared_first_mutex.h` timed locks:** use `wait_until` (truncated on Windows) but have no callers.
- **Logging:** `ISS_LOG_BUFFER=1` flushes every 200 ms to a file (`probe.c:1260-1275`). Crash paths do not wait on the stream locks.
- **Guest side:**
  - `sceKernelUsleep` uses the high-resolution timer (`compat_win.c:121-133`).
  - Time calls (`runtime_kernel.c:79-100`) are QPC/rdtsc in user mode.
  - The guest mutex spin (`ISS_MUTEX_SPIN_US=20`, `runtime_mutex.c:118-135`) was measured at +2.3%.
  - `TLG_THREAD_PRIO` is off, so no guest thread is lowered. Keep it off: Windows has no priority inheritance.
  - The fault handler (`probe.c:240-257`) does no syscalls or locks before `tlggpu_handle_fault`, apart from finding 13.
  - The freeze watchdog, wait-stats and toggle-file threads sleep or are opt-in.
  - The pad replay `stat()` every 20 ms is bench-only.
- **Small leftovers worth a cleanup but not measurable:**
  - `getenv` on every call in `runtime_karma.c:212` (karma tick) and `runtime_file.c:68` (every file read).
  - Two `SetFilePointerEx` per `compat_pread` on the streaming threads.
- **Vblank clock:** the bursts of `ISS_VBLANK_TIMER=legacy` (`thread.cpp:113-131,236-252`) are real, but `hr` was measured with no FPS change. `SetCurrentThreadRealtime` is a no-op on Windows (`thread.cpp:79-81`); the present thread gets only `ABOVE_NORMAL` from `raise_gpu_threads`.

## Suggested order of work

1. **Bench-only items** (no code): finding 2 (`ISS_SUBMIT_GATE=gfx`, then `off`), 5 (`recorder_spin_us`), 7 (`ISS_SUBMIT_INTERVAL_US`), 8 (`ISS_UNIFIED_FLUSH_IRQ`/`_US`), 3 (`ISS_NATIVE_PASSES_CPU=pcore`, `_WAKE=one`), 16, 17 and 19. Each is one `bench.ps1 run` session of ~10-17 min with the fast A/B recipe.
2. **One diagnosis run** with `ISS_FRAME_STATS=1`, `SHADPS4_FRAME_CSV`, `ISS_READBACK_DIAG=1`/`_LOG` and `ISS_TIMELINE=1`. It answers the open questions above: `recorder_sync_ms`, the `Submit gate:` and `UNIFIED_STATS` lines, round-trip 2 durations, how often VO waits happen.
3. **Small code changes, each behind its toggle, in this order:** 1 (present command buffer), 9 (wait reorder), 4a (skip the redundant round-trip), 12 (VO wait event plus flush), 6 (split `submit_mutex`), 3 (pool priority and serving commands), 11 (early flip label), 10 (pending-ops spin), 13 (map lock).
