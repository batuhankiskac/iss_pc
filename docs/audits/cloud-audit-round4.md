# Cloud audit, round 4: correctness bugs outside the frame loop (renderer v2 + guest runtime)

This is the fourth code-only pass over `main` at `fac17b4` (2026-10-26). Rounds 1-3 ([`cloud-audit.md`](cloud-audit.md), [`cloud-audit-round2.md`](cloud-audit-round2.md), [`cloud-audit-round3.md`](cloud-audit-round3.md)) looked at frame time, the CP hot path and the main concurrency primitives. This round looks for bugs in four areas they did not cover:

- **G:** the guest runtime (`port/src/`): memory mapping, files, threads, rwlocks, keys, audio.
- **M:** CPU/GPU memory coherence in renderer v2: map/unmap hooks, page protection, readback write-back, the unified heap.
- **T:** Vulkan resource lifetime and robustness: samplers, the swapchain, the shader and pipeline caches, device-lost handling.
- **L:** the launcher, briefly.

Nothing from rounds 1-3 is repeated. References to them are written R1-n, R2-n and R3-xn. The same rules apply:

- Nothing was run or changed.
- Every line number was read from the file.
- Items not traced end to end are marked **suspected**.

Most of these bugs cost no frame time on the normal path; they show up as crashes, hangs or stale data. Where one does cost time, the entry says so. Paths are relative to `port/` unless they start with `docs/`.

## Ranking (severity × likelihood)

| # | Finding | Main location | Symptom | Default path? | Fix risk |
|---|---|---|---|---|---|
| 1 | Sampler garbage collection destroys `VkSampler`s still referenced by memos, pool jobs and in-flight command buffers | `gpu_v2/gpu/overrides/texture_cache.cpp:1804-1843` | driver crash / device lost | yes, once >3/4 of the sampler limit | low |
| 2 | A `MAP_FIXED` / `sceKernelBatchMap` remap over a live mapping never tells the GPU the old pages are gone | `src/runtime_memory.c:420-438` | stale GPU data, guest memory corrupted by later readbacks | when the game remaps | low |
| 3 | Map/unmap hooks run out of order because any thread may drain the shared hook queue | `src/runtime_memory.c:312-330` | unhandled access violation (crash), images dropped | low-medium | low |
| 4 | Windows: a thread that read-locks a guest rwlock it already holds deadlocks behind a waiting writer | `src/runtime_rwlock.c:96-117` | permanent hang | unknown, Windows only | low/medium |
| 5 | Swapchain recreation during a minimize creates a 0×0 swapchain and asserts | `vk_presenter.cpp:943-945`, `vk_swapchain.cpp:141,206-217,445` | crash on Win+D / minimize | yes | low |
| 6 | One truncated shader-cache file makes every later start abort | `cache_storage.cpp:234-236`, `vk_pipeline_serialization.cpp:186,277,295,621` | persistent crash at boot | after a crash, power loss or full disk | low |
| 7 | `MarkAsMaybeDirty` hashes guest memory under the texture-cache lock and can fault on a GPU-protected page | `texture_cache.cpp:352-360,414,1613,1632` | hang (Windows), process kill (Linux) | suspected, low | low |
| 8 | A binding that straddles the end of unified heap B takes the arena path | `buffer_cache.h:284-286`, `defaults.env:135` | stale or zero data in heap B | suspected | medium |
| 9 | Readback write-back is all-or-nothing per run, but the pages are unmarked anyway | `buffer_cache.cpp:2560-2568,2594-2600,2718-2727`, `bridge.cpp:276-283` | GPU results silently dropped | low | low |
| 10 | The fixed 64-entry GPU hook queue ends the process when it overflows | `src/runtime_memory.c:312-317` | `STOP: GPU hook queue overflow`, `exit(21)` | latent | low |
| 11 | A guest `mprotect` replaces GPU page protection before the invalidate hook runs | `src/runtime_memory.c:609-620` | stale reads; guest writes overwritten by a readback | low | low/medium |
| 12 | `sceKernelAvailableFlexibleMemorySize` always reports 448 MiB | `src/runtime_tlg.c:288-291`, `src/runtime_memory.c:564` | flexible map fails with `NO_MEMORY` | depends on game use | none |
| 13 | The driver pipeline cache is no longer saved past 256 MB; killed checkpoints leave ~217 MB orphan files | `pipeline_checkpoint.cpp:32-33,37` | compile hitches return; disk use grows | likely with play time | low |
| 14 | Device-lost in the acquire/present path asserts before crash diagnostics are written | `vk_swapchain.cpp:289,336,359`, `vk_presenter.cpp:963` | no `gpu-crash.txt` for many GPU crashes | yes (on device lost) | none |
| 15 | Windows `write()` can land at the wrong offset when another thread `pread`s/`pwrite`s the same descriptor | `src/runtime_file.c:488-499,516-520` | file corruption (saves) | low | low |
| 16 | Guest thread stacks, records and TLS blocks are never freed | `src/runtime_thread.c:534-573` | memory growth; stacks above the low window | with thread churn | medium |
| 17 | A deleted pthread key leaves values in other threads, seen again by a re-created key | `src/runtime_kernel.c:328-352` | stale value, destructor on a foreign pointer | low | low |
| 18 | Ajm RIFF parsing can underflow the input size | `src/runtime_ajm.c:250-265` | out-of-bounds read, crash | low | none |
| 19 | `frame_index` is not reset when the swapchain is recreated | `vk_swapchain.cpp:363` | out-of-bounds semaphore index | latent | none |
| 20 | Linux: `sceKernelSleep` returns early on any signal | `src/runtime_gr2.c:92` | short sleeps after `SIGUSR1/2`, `SIGALRM` | Linux only | none |
| 21 | Fault-streak extension discards GPU data the guest never writes | `buffer_cache.cpp:1408-1429`, `memory_tracker.h:332-346` | stale data | **no**: only with `readback_write_discard=1` | low |
| 22 | Suspected: renaming or deleting a file the guest still has open fails on Windows | `src/runtime_file.c:642` | `EACCES` on "write temp, rename over" | suspected | low |
| 23 | Suspected: ImGui's descriptor pool used from several threads; a skipped present can let a frame image be destroyed early | `vk_presenter.cpp:539,600,937-950` | validation errors, rare crash | suspected, narrow | low |

Items 1, 5 and 6 can each be reproduced deterministically (see each entry), and are the cheapest to confirm first.

---

## T. Vulkan lifetime and robustness

### 1. Sampler garbage collection destroys samplers still in use

- **Location:**
  - `gpu_v2/gpu/overrides/texture_cache.cpp:1804-1843` (`GarbageCollectSamplers`). `samplers.erase(hash)` (`:1828`) destroys the `vk::UniqueSampler` at once. Images, by contrast, go through `DeleteImage` → `scheduler.DeferOperation` and `WaitRecordingBefore`.
  - The LRU is touched only by `GetSampler` (`:1448-1456`).
  - Three paths reuse a sampler handle without calling `GetSampler`:
    - the rasterizer's sampler memo (`vk_rasterizer.cpp:4763-4768`: `vk_sampler = sampler_memo->sampler;`);
    - the stage-bind memo (`stage_bind_memo.inl:239`);
    - native templates, which pool threads write into descriptors (`native_passes_rasterizer.inl:1239-1246`). The pool checks that images are alive (`:1009-1011`) but not `SamplersGeneration`.
  - The trigger is `trigger_gc_samplers = max_samplers * 3 / 4` (`texture_cache.cpp:90`), e.g. 3 000 on NVIDIA (limit 4 000).
- **Mechanism:**
  - A sampler used every frame through a memo looks unused to the LRU, so after 16 or more garbage-collector ticks it is a victim.
  - `samplers_generation` is bumped (`:1830`), which makes the CP's memos miss next time. It does not protect against:
    - command buffers already submitted or queued for the recorder thread;
    - a pool job that is recording right now.
- **Symptom:** a destroyed `VkSampler` in a descriptor write or a command buffer. Expect a driver crash in the same class as the documented image-view crash (nvoglv64.dll), a device lost, or garbage sampling.
- **Likelihood:** on the default path (memos on) once the game has more than 3/4 of the device's sampler limit in unique sampler states. Whether it gets there is unknown; log the count first.
- **Fix:**
  1. Defer the destruction: move the `UniqueSampler` into `scheduler.DeferOperation` and wait on `native_passes::RecordingTicket` / `WaitRecordingBefore`, as `DeleteImage` does.
  2. Touch the LRU on memo hits, or skip samplers whose memos are live.
- **Toggle:** none needed. Fix 1 is a pure lifetime fix.
- **How to detect:**
  - log `samplers.size()` at each garbage-collector tick;
  - force the path by lowering `trigger_gc_samplers` to ~64 in a test build, then run under validation layers, looking for "Invalid VkSampler" or "VkSampler in use".

### 5. Swapchain recreation during a minimize creates a 0×0 swapchain and asserts

- **Location:**
  - `gpu_v2/gpu/overrides/vk_presenter.cpp:943-945`: on a failed acquire, `if (!swapchain.NeedsRecreation() || !window.CanPresent()) {…return;} swapchain.Recreate(…)`.
  - `CanPresent()` reads `presentation_available`, which the window thread stores only at the end of `PollEvents` (`gpu_v2/gpu/window.cpp:438`). Windows has already made the window iconic during `SDL_PollEvent`.
  - `vk_swapchain.cpp`:
    - `Create` calls `Destroy()` first (`:141`);
    - it takes `extent = capabilities.currentExtent` (`:445`), which is 0×0 for a minimized Win32 window;
    - it calls `createSwapchainKHR` (`:206`), and `ASSERT_MSG(swapchain_result == vk::Result::eSuccess, …)` aborts (`:216-217`).
  - The same happens with `VK_ERROR_SURFACE_LOST_KHR`, which is mapped to Recreate on the same, lost surface (`swapchain_policy.h:20,27`).
- **Mechanism:** a present or acquire returns `OUT_OF_DATE` inside the gap between the OS minimize and `presentation_available` turning false. The spec forbids a zero `imageExtent` (VUID-VkSwapchainCreateInfoKHR-imageExtent-01689), and drivers return an error or crash.
- **Trigger:** Win+D, Win+M or a taskbar minimize while frames are being presented.
- **Fix:**
  - In `Create`, if `currentExtent` has a zero side, keep `needs_recreation` set, leave the swapchain null and return. `Present` and `AcquireNextImage` then skip frames until a non-zero extent arrives.
  - On `SURFACE_LOST`, recreate the surface before the swapchain.
- **Risk:** low.
- **How to detect:** minimize and restore in a loop under validation layers; look for "Failed to create swapchain" in the log.

### 6. One truncated shader-cache file makes every later start abort

- **Location:**
  - **Writes:** `gpu_v2/gpu/overrides/cache_storage.cpp:234-236`: `IOFile{path, FileAccessMode::Create}` truncates, then `file.Write(data)`, with the result ignored. There is no temporary file and no rename.
  - **Loads:**
    - `vk_pipeline_serialization.cpp:621` checks only `size < 16`;
    - `key.Read(this, sizeof(*this))` (`:277`), the sdata reads (`:295`) and `info.Deserialize` (`:186`) hit `Reader` / `Advance` `ASSERT_MSG` (`common/serdes.h:49,112`). WarmUp does not set `assert_throws`, so the process aborts.
    - Own-cache `.spv` files are not checked for size or SPIR-V magic before `CompileSPV` (`:435,447`).
- **Mechanism:** a short `.key`, `.meta` or `.spv` file is left behind by any of these:
  - a full disk;
  - a power loss or hard reset after a GPU hang;
  - an abort during a write. The game uses `_Exit` after its final checkpoint, and a crash elsewhere can land during an async write.

  Every later boot then aborts in WarmUp.
- **Symptom:** "Invalid or corrupted deserialization container" or an assert at boot, every time, until the player deletes the cache folder.
- **Fix:**
  1. Write to `<name>.tmp`, check the write and close results, then rename over the target.
  2. Parse each blob with `assert_throws = true` inside a try/catch, and skip and delete bad blobs.
  3. Check SPIR-V magic and that the size is a multiple of 4 before compiling.

  The driver cache and the checkpoint already do this (temporary file, `MoveFileExW`, checksum).
- **Risk:** low.
- **How to detect:** truncate one `.key` or `.meta` file to half its size and start the game.

### 13. The driver pipeline cache stops being saved past 256 MB; killed checkpoints leave orphan files

- **Location:**
  - `gpu_v2/gpu/pipeline_checkpoint.cpp:32-33`: `snapshot.value.size() > 256ULL*1024*1024-24` gives `PIPELINE_CHECKPOINT unavailable` and returns. `vk_pipeline_cache.cpp:1316` and `:2363-2364` use the same cap.
  - `:37`: the temporary file is `<cache>.pending-<pid>`.
- **Mechanism:**
  - R2-11 measured the cache at about 217 MB. Once it grows past 256 MB, no new pipeline is persisted again, and the log line does not say why ("unavailable result=0").
  - A checkpoint killed during its write (alt-F4, crash) leaves a ~217 MB `.pending-<pid>` file. The process ID differs each run, so nothing ever reuses or deletes it.
- **Effect:** the cold-start compile hitches that the cache removes come back after enough play time. This is the only round-4 item with a frame-time cost: hitches of tens of ms per new pipeline, not an average ms/frame.
- **Fix:** raise the cap (or make it `ISS_PIPELINE_CACHE_MAX_MB`) and warn once, clearly, when it is hit. Delete `*.pending-*` files at startup.
- **Risk:** low. Check the driver's own limit on `vkCreatePipelineCache` input size.
- **How to measure:** the size of the cache file on a long-played install; count `.pending-*` files in the cache folder.

### 14. Device-lost in the acquire/present path asserts before crash diagnostics are written

- **Location:**
  - `vk_swapchain.cpp:336`: acquire returning `DEVICE_LOST` maps to Fatal, logs "unknown result" and reaches `UNREACHABLE`.
  - `:289` (acquire-fence wait) and `:359` (present) assert.
  - `vk_presenter.cpp:963` (fence reset) asserts.
- **Mechanism:** none of these call `GpuCrashDiag::ReportDeviceLost`, which the scheduler and semaphore paths do. The present thread is often the first to see a device loss, so the process aborts before checkpoints, fault info or `gpu-crash.txt` are written.
- **Fix:** call `ReportDeviceLost(instance, "acquire" / "present")` before asserting.
- **Risk:** none.

### 19. `frame_index` is not reset when the swapchain is recreated (latent)

- **Location:** `vk_swapchain.cpp:363`: `frame_index = (frame_index + 1) % image_count`. `SetupImages` resets `image_count` (`:535`) and `RefreshSemaphores` resizes the semaphore arrays to it, but `frame_index` keeps its old value.
- **Mechanism:** if a recreation returns fewer images, `image_acquired[frame_index]` (`:281`) indexes past the end. The image count can change when the present mode switches between Mailbox and FIFO (entering or leaving exclusive fullscreen, `:148`), or with HDR on some drivers.
- **Fix:** `frame_index = 0;` in `Create`.

### 23. Suspected: unsynchronized ImGui descriptor pool; early frame-image destroy after a skipped present

- **ImGui descriptor pool.** `RecreateFrame` on the CP calls `ImGui::Vulkan::RemoveTexture` / `AddTexture` (`vk_presenter.cpp:539,600`). The texture-manager worker (splash PNG) and the present thread (font atlas) allocate and free sets from the same pool (`imgui_impl_vulkan.cpp:280-298,388,482-487`). Vulkan requires the pool to be externally synchronized. Fix: a mutex around the ImGui pool.
- **Skipped present.** A skipped present calls `free_frame()` without resetting `present_done` (`vk_presenter.cpp:937-950`). `GetRenderFrame` then trusts that fence and may `RecreateFrame`, destroying `frame->image` while the frame's post-process pass is unfinished. The queue drain inside swapchain Recreate usually covers this. Fix: also wait on `frame->ready_tick` before `RecreateFrame`.
- **Minor:** `GetRenderFrame` (`vk_presenter.cpp:1243-1254`) and `Semaphore::Wait` loop forever on an unexpected `VkResult`, such as out-of-host-memory.

---

## M. CPU/GPU memory coherence

### 2. A `MAP_FIXED` / BatchMap remap over a live mapping never tells the GPU the old pages are gone

Two of this round's independent scans found this one.

- **Location:**
  - `src/runtime_memory.c:420-438` (`place`). With `MAP_FIXED_FLAG` it calls `host_map(…)`, then `drop_range(address, address+size)`, then queues only `HOOK_MAP` (`:435`).
  - `direct_unmap`, by contrast, calls `hook_unmap` first (`:521-523`), and `unmap_locked` queues `HOOK_UNMAP` per VMA (`:509-513`).
  - `sceKernelBatchMap` always passes `MAP_FIXED` (`:693-699`).
  - The map hook (`gpu_v2/gpu/bridge.cpp:345-350` → `vk_rasterizer.cpp:5829-5844`) only registers the range, bumps the mapping generation and adds it to `mapped_ranges`. It invalidates no cached buffer or image.
- **Mechanism:** the buffer cache, texture cache and page tracker keep the old mapping's state. `host_map` re-created the view with the guest's protection, which silently removed the tracker's write protection. The tracker still believes those pages are protected, so later Track/Untrack calls compare against stale state and never call Protect again. Two things then go wrong:
  - guest writes to the new memory never fault, so the GPU keeps using the old contents;
  - old GPU-modified pages can later be read back (prefetch windows) into the new mapping's backing, which corrupts it.
- **Trigger:** `sceKernelBatchMap`, `sceKernelMapDirectMemory` / `MapFlexibleMemory`, or `mmap(MAP_FIXED)` over an existing non-reserved mapping (no `MAP_NO_OVERWRITE`).
- **Symptom:** stale textures or buffers after streaming, and corrupted guest data.
- **Fix:** in `place()`, before `host_map`, queue `HOOK_UNMAP` for every non-reserved VMA the range overlaps (as `unmap_locked` does), then `HOOK_MAP`.
- **Toggle:** none (correctness).
- **Risk:** low. The unmap hook already handles partial ranges.
- **How to detect:** log in `place()` whenever `drop_range` removes a non-`KIND_RESERVED` VMA. If it never fires on a long route, the bug is latent for this game.

### 3. Map/unmap hooks run out of order because any thread may drain the shared hook queue

- **Location:** `src/runtime_memory.c:319-330` (`flush_hooks`). It pops `pending[0]` under the lock, then runs the hook **unlocked**. Any thread doing any memory operation, including malloc's `posix_mmap`, drains whatever is queued, including other threads' entries.
- **Mechanism:**
  1. Thread A pops `UNMAP(R)` and enters `Rasterizer::UnmapMemory` (`vk_rasterizer.cpp:5846-5857`). That can block for milliseconds in `buffer_cache.InvalidateMemory` (a readback).
  2. Thread C pops the next `MAP(R)` and finishes `MapMemory` (`mapped_ranges +=`, `:5835`).
  3. A resumes: `texture_cache.UnmapMemory`, then `mapped_ranges -=` (`:5855`). This removes the *new* mapping and its images.
  4. `IsMapped(R)` is now false, so `InvalidateMemory` / `ReadMemory` return "not handled". A guest write to a GPU-protected page of R becomes an unhandled access violation: a crash.

  Related: a caller can return from `munmap` / `mmap` before its own hooks have run, if another thread popped them. And `direct_release` zeroes the memory (`:559`) before its unmap hook runs, unlike `direct_unmap`.
- **Trigger:** unmap then map of the same VA range (BatchMap, release + map) while other threads allocate.
- **Fix:** run hooks to completion one at a time in queue order: hold a dedicated hook mutex across pop and run. Make each caller wait until its own hooks have run. Call the unmap hook before the release's memset, as `direct_unmap` does.
- **Toggle:** none (correctness).
- **Risk:** low. Hooks already run outside the map lock, so a separate hook mutex adds no lock-order edge with the map lock. It must not be held by a thread that faults into a hook.
- **How to detect:** an assert in `MapMemory` / `UnmapMemory` that no other hook for an overlapping range is in flight (an atomic interval set).

### 7. `MarkAsMaybeDirty` hashes guest memory under the texture-cache lock and can fault (suspected)

- **Location:** `gpu_v2/gpu/overrides/texture_cache.cpp:352-360`: `image.hash = XXH3_64bits(addr, image.info.guest_size)` on the guest address. It is called with `mutex` held from the write-fault path (`InvalidateMemory`, `:389`):
  - directly for single-page images (`:414`);
  - through `UntrackImageHead` / `UntrackImageTail` for two-page images (`:1613`, `:1632`).
- **Mechanism:** the hash reads the whole image, including the *other* page. If that page is read-protected because the GPU modified it, the hash faults inside the fault handler, with the texture-cache lock held. The nested fault queues a readback and waits for the CP, and the CP may be waiting for the same lock in `FindImage`. The port's own comment at `:420-427` describes exactly this deadlock ("Native ultrawide streaming freeze, 2026-10-22"), but the fix only disabled the streak path.
- **Symptom:**
  - Windows: a hang.
  - Linux: the handler is installed without `SA_NODEFER` (`src/probe.c:1463`), so a nested SIGSEGV while SIGSEGV is blocked kills the process.
- **Fix:** hash through the backing view (`Core::Memory::HashSparseMemory` / `CopySparseMemory`, `bridge.cpp:275`), which never faults.
- **Risk:** low.
- **How to detect:** the freeze report shows a thread inside `MarkAsMaybeDirty` with `faults_in_progress ≥ 2`.

### 8. A binding that straddles the end of unified heap B takes the arena path (suspected)

- **Location:**
  - `IsUnified` requires full containment (`gpu_v2/gpu/shim/video_core/buffer_cache/buffer_cache.h:284-286`).
  - Heap B is `ISS_UNIFIED_RANGE=0x1048000000:0x20000000` (`defaults.env:135`), so it ends at `0x1068000000`, exactly where heap C begins (`:136-138`).
  - `ClampRangeSize` (`vk_rasterizer.cpp:4243`) clamps across adjacent mappings, so an oversized heap-B descriptor can extend into heap C.
  - `EnableUnified` sets residency bits for heap B (`buffer_cache.cpp:4142-4145`) but never adds its blocks to `resident_ranges`.
- **Mechanism:** a straddling `ObtainBuffer` goes through the arena (`buffer_cache.cpp:2844`, `3017-3053`). Tracking then write-protects heap-B pages, and uploads and GPU writes land in the arena. Two outcomes:
  - **Heap C not fully resident yet:** `EnsureResident` (`:3572`, `3643-3647`) binds arena memory for the heap-B blocks and rewrites their BDA page-table entries away from the imported heap. Every later BDA access to heap B then reads a stale copy.
  - **Otherwise:** the arena's heap-B blocks have no memory bound. The GPU reads zeros, and readbacks write those zeros over heap B.
- **Fix:** split bindings at `unified_base` / `unified_end` (or clamp them there), and add the heap-B blocks to `resident_ranges` as already bound.
- **Risk:** medium; the unified path is performance-critical.
- **How to detect:** log once in `ObtainBuffer` when a range overlaps heap B but `!IsUnified`.

### 9. Readback write-back is all-or-nothing per run, but the pages are unmarked anyway

- **Location:**
  - `TryWriteBacking` (`gpu_v2/gpu/bridge.cpp:276-283`) uses `runtime_memory_backing_view` (`src/runtime_memory.c:845-860`). That returns NULL unless the whole range is `KIND_DIRECT` and physically contiguous.
  - `FinishReadback` (`buffer_cache.cpp:2560-2568`) and `DownloadMemory` (`:2718-2727`) ignore the return value, then unmark the pages as GPU-modified anyway (`:2598`, `:2726`).
- **Mechanism:** the GPU-modified ranges live in a boost `interval_set`, which merges adjacent ranges, so one copy can span two mappings whose backing is not contiguous. Uploads already handle this case by falling back to 16 KB pieces (`sparse_copy.h`, `iss_copy_sparse`). The write path does not.
- **Symptom:** the whole span's GPU data is silently dropped, and the guest reads stale memory.
- **Likelihood:** low.
- **Fix:** write per contiguous sub-range, as `iss_copy_sparse` does, and keep pages GPU-modified when a write fails. Log a failed write once.
- **Risk:** low.

### 11. A guest `mprotect` replaces GPU page protection before the invalidate hook runs

- **Location:** `src/runtime_memory.c:609-620` (`protect_locked`). `host_protect` (`:613`) runs under the map lock. The invalidate hook is only queued (`:617`) and runs later from `flush_hooks`.
- **Mechanism:** a GPU-modified page (protection None) is readable and writable in that window:
  - guest reads see stale data;
  - guest writes are later overwritten by the readback the hook starts (`FinishReadback` writes the GPU copy over them).
- **Likelihood:** low; it needs an `mprotect` on GPU-written memory.
- **Fix:** run the GPU invalidate (readback) before `host_protect`, as `direct_unmap` does for unmaps. Then apply the guest protection combined with the tracker's: the tracker must be told the host protection changed.
- **Risk:** low/medium.

### 21. Fault-streak extension discards GPU data the guest never writes (off the default path)

- **Location:** `buffer_cache.cpp:1408-1429` calls `InvalidateRegion(ahead + off, 16 KB, true, false, []{})` for up to 256 KB ahead of the fault. With `readback_write_discard=1`, `memory_tracker.h:332-346` then discards GPU data in pages the guest may never write.
- **Effect:** those pages lose their GPU state. Their next upload overwrites the GPU copy with stale guest bytes, and CPU reads get stale data.
- **Default path:** discard is off by default (R1-16), so this is harmless today. It matters for anyone benching `ISS_TUNING=readback_write_discard=1`, where it can show up as a "visual bug" of that toggle.
- **Fix:** skip GPU-modified chunks in the streak, treating them as not discardable.

**Lower confidence (memory):** guest kernel calls write output parameters while holding the runtime write lock (`*out` at `runtime_memory.c:476`, `*inout` in `place`). If that page is GPU-modified, the fault waits for the CP, whose readback needs the runtime read lock (`host.backing`): a deadlock. It is unlikely, because these are usually stack variables. Fix: write outputs after `write_unlock()`.

---

## G. Guest runtime

### 4. Windows: re-entrant read lock behind a waiting writer deadlocks

- **Location:**
  - `src/runtime_rwlock.c:96-117`. The holder check (`:97`) allows a second read lock when the thread holds no write lock. The lock is a plain `pthread_rwlock_t` (`:17`), taken with `pthread_rwlock_rdlock` (`:116`).
  - The repo's own `src/rwlock_rp.h:1-9` documents the problem: winpthreads keeps its internal mutex while a writer waits, "so a reader that reads again behind a queued writer blocks forever, and the writer with it". `RpLock` was written for `runtime_memory.c` only; guest `scePthreadRwlock*` still use winpthreads.
  - FreeBSD libthr, the PS4's base, lets a thread that already holds read locks take another one even with writers waiting (its per-thread `rdlock_count`).
- **Trigger:**
  1. Thread A holds rwlock R for reading.
  2. Thread B calls `wrlock(R)` and queues.
  3. Thread A calls `rdlock(R)` again.

  A waits for B and B waits for A. glibc prefers readers, so Linux is not affected.
- **Symptom:** a permanent hang. The freeze report shows "rwlock (read)" on a thread whose holder entry already has `readers > 0`.
- **Likelihood:** unknown. The game imports `scePthreadRwlockRdlock`, `Wrlock` and the timed variants. Of the hangs found this round, this is the most plausible on Windows.
- **Fix:** if the calling thread already holds R for reading (`h->readers > 0`), skip the native lock and only count the recursion. Or move guest rwlocks to `RpLock`, or to a writer-preferring lock with libthr's re-entry rule.
- **Toggle:** `TLG_RWLOCK_RP=1` while it is validated.
- **Risk:** low/medium. Reader preference can starve writers, as with glibc.

### 10. The fixed 64-entry GPU hook queue ends the process when it overflows

- **Location:** `src/runtime_memory.c:312-317`: `if (pending_count == 64) { fputs("STOP: GPU hook queue overflow\n", stderr); exit(21); }`.
- **What fills it while the map lock is held:**
  - `batch_map2` (`:676-688`): one hook per map or protect entry;
  - `unmap_locked` (`:509-513`): one `HOOK_UNMAP` per VMA in the range. `direct_unmap` throws them away only afterwards (`:526-527`), after the overflow check has already run;
  - `direct_release` (`:537-547`): calls it repeatedly.
- **Trigger:** a `sceKernelBatchMap` with more than 64 entries, or an unmap or release across more than 64 VMAs (each `mprotect` can split VMAs).
- **Fix:** a growable array. Also skip queueing in `unmap_locked` when the caller has already notified the GPU (pass a flag instead of resetting `pending_count`).
- **Risk:** low.

### 12. `sceKernelAvailableFlexibleMemorySize` always reports 448 MiB

- **Location:** `src/runtime_tlg.c:288-291` returns a constant. `map_flexible_locked` (`src/runtime_memory.c:564`) enforces `flexible_bytes + size <= FLEXIBLE_SIZE` (448 MiB).
- **Mechanism:** a game that sizes a pool from this call after libc's `mmap` has already used flexible memory asks for more than is left.
- **Symptom:** a flexible map returns `NO_MEMORY`, and the game's allocator fails or asserts.
- **Fix:** return `FLEXIBLE_SIZE - flexible_bytes`, read under the map lock.
- **Risk:** none.

### 15. Windows `write()` can land at the wrong offset

- **Location:** `src/runtime_file.c:488-499` (`do_write`) holds `f->io`, seeks the CRT pointer to `f->offset`, then writes at the CRT pointer. `do_pread` (`:455-462`) and `do_pwrite` (`:516-520`) take no lock. On Windows they go through `positioned()` (`src/compat_win.c:67-97`), which saves the pointer, moves it for `ReadFile` / `WriteFile`, then restores it.
- **Mechanism:** a `pread` or `pwrite` on the same descriptor that runs between `do_write`'s seek and its write makes the data land at the wrong offset. `f->offset` can then be wrong too (`:497-498`).
- **Trigger:** one thread `write`s a descriptor while another thread `pread`s or `pwrite`s it.
- **Likelihood:** low.
- **Fix:** write at `f->offset` through `positioned()`, under `f->io`. For `O_APPEND`, read the size under the lock.
- **Risk:** low.

### 16. Guest thread stacks, records and TLS blocks are never freed

- **Location:**
  - On Linux, stacks come from the bump allocator `runtime_low_map` (`src/runtime_thread.c:571-573`, `src/runtime_memory.c:719-741`) and are never unmapped.
  - On both platforms, `GuestThread`, `tls_block` (`runtime_thread.c:534-550`) and module TLS blocks (`src/runtime.c:47-51`) are never freed.
- **Mechanism:** the game may keep creating short-lived threads (job threads, audio decoders, loaders).
  - Resident memory grows with every one.
  - On Linux, after about 25 000 threads with 1 MiB stacks the 32 GiB low window is used up. Stacks then come from `pthread_attr_setstacksize` above 1 TiB, which breaks guest pointer packing.
- **Fix:** free stacks and records on join, or at exit for detached threads; recycle stacks in a pool.
- **Risk:** medium; a guest may still touch a joined thread's TLS.
- **How to detect:** the `guest threads created=` counter in the exit report, on a long session.

### 17. A deleted pthread key leaves values in other threads

- **Location:** `src/runtime_kernel.c:328-352`. `key_delete` clears only the slot (`used = 0`, `destructor = NULL`). Other threads' `key_values[key]` survive. A key re-created in the same slot then returns the old values from `key_get`, and at thread exit `runtime_thread_keys_cleanup` runs the *new* key's destructor on them.
- **Comparison:** FreeBSD libthr invalidates old values with a per-key `seqno`.
- **Trigger:** `scePthreadKeyDelete` (imported) followed by `scePthreadKeyCreate`.
- **Fix:** keep a generation per key, and store `{value, gen}` per thread. A mismatch reads as NULL and is skipped by cleanup.
- **Risk:** low.

### 18. Ajm RIFF parsing can underflow the input size

- **Location:** `src/runtime_ajm.c:250-265` (`parse_riff`). It does `p += 12` with no size check, and `p += length` for each chunk with no bounds check. Then `*size -= (p - *input)` wraps when `p` passed `end`, and the ATRAC9 decoder reads past the guest buffer.
- **Trigger:** a RIFF header split across Ajm inputs, or a chunk length past the input.
- **Likelihood:** low.
- **Fix:** if `*size < 12`, return; clamp `p` to `end`, and report `PARTIAL_INPUT` if a chunk runs past it.
- **Risk:** none.

### 20. Linux: `sceKernelSleep` returns early on any signal

- **Location:** `src/runtime_gr2.c:92`: `sleep(seconds); return 0;`.
- **Mechanism:** `sleep` returns early on `SIGUSR2` (thread dumps, sent to every thread), `SIGUSR1` or the `SIGALRM` watchdog. The freeze report also does not show the thread as waiting.
- **Fix:** route it through `sleep_ns` (`src/runtime_kernel.c:122`), which retries on `EINTR` and marks the wait.
- **Risk:** none.

### 22. Suspected: renaming or deleting a file the guest still has open fails on Windows

- **Mechanism:** guest files are opened through the CRT, which shares read and write but not delete, as far as we know. `MoveFileExA(…, MOVEFILE_REPLACE_EXISTING)` (`src/runtime_file.c:642`) or `unlink` of a file a guest descriptor still holds then fails with `EACCES`. The PS4 and Linux both allow it.
- **Trigger:** "write temp, then rename over the original" while the original is still open, a common save pattern.
- **Fix:** open with `CreateFileW(…, FILE_SHARE_READ | FILE_SHARE_WRITE | FILE_SHARE_DELETE)` plus `_open_osfhandle`.
- **How to confirm:** check which open call `runtime_file.c` uses, and test rename-over-open on Windows.

### Minor (guest runtime)

- **`sceKernelGetdents`** (`src/runtime_tlg.c:292-294`) returns a raw negative host errno instead of an Orbis error code.
- **`gettimeofday`:** the time zone on Linux comes from the kernel's, usually 0 (`src/runtime_time.c:16-21`). Windows returns the real offset.
- **`munmap` / `mmap` error codes:** `munmap` returns Orbis error codes instead of -1 with errno. `mmap` with size 0 sets `ENOMEM` instead of `EINVAL` (`src/runtime_memory.c:589-595`, `:700`).
- **Audio port reuse:** closing and reopening a port re-runs `pthread_mutex_init` on `p->lock` while `output_port` may be waiting on it (`src/runtime_audio.c:134-135`, `:186-189`).
- **Save data:** writes are not committed atomically on unmount as on PS4, so a crash mid-save tears the save. This is a design gap, not a line bug.

---

## L. Launcher (minor)

- **Settings are written in place.** `save_settings` (`launcher/infamous_run.py:366-369`) and `update_ini` (`:495-505`) use `write_text`, not a temporary file and rename.
  - A crash or power loss during the write leaves a truncated `settings.json`. `load_settings` (`:352-363`) then silently falls back to defaults (`except (OSError, ValueError)`), and the next save overwrites the player's settings.
  - A `settings.json` that parses to a non-object (e.g. a list) raises `TypeError` in `settings.update`, which is not caught, so the launcher does not start.
  - `userdata.py` and `savemanager.py` already use temporary files and rename; use the same helper here.
- The save import and export paths (`savemanager.py:485-555`) were checked: staging folder, rename into place, backup before replacing, zip-slip checks. They look correct. One edge: if the `.sce_sys` rename fails after the slot folder was moved in for a slot that did not exist before, the rollback leaves a slot without `.sce_sys`.

---

## Checked, no issue found

- **Scheduler:**
  - command-pool `Commit` is single-threaded (`SyncRecorder` runs before the CP takes the buffer);
  - `submitted_tick` stays monotonic under `submit_mutex`;
  - `bindSparse` runs under `submit_mutex`.
- **Image lifetime:**
  - `DeleteImage` defers to the CP tick and uses `WaitRecordingBefore`;
  - the pool checks `ImageAlive`;
  - `TexturesValid` catches freed images.
- **Native passes:**
  - segments are recycled only after `executed_tick` ≤ the GPU tick;
  - each pool thread has its own lane and command pool;
  - templates are held by `shared_ptr`.
- **Staging and streams:**
  - stream-buffer wrap-around and watch waits are correct;
  - large staging buffers are tick-gated;
  - `DescriptorHeap` reuses pools only after their tick.
- **Caches:**
  - the driver cache and checkpoint use a temporary file plus `MoveFileExW`, with checksum, vendor/device and UUID checks;
  - the library is SHA-verified;
  - the write queue drains before join.
- **Shutdown:** the final checkpoint and `DataBase::Close` run, then `_Exit`, so destructor-order bugs never run on the default path.
- **Memory tracker:**
  - an L2 page-table block is one 16 MB region, so watcher updates see a consistent state;
  - page locks are taken in ascending order;
  - every Protect runs under page locks;
  - `ExtendWriteFault` stays inside one region and excludes cpu, gpu, hot and gpu-owned pages;
  - `AddPendingReadback` / `RemovePendingReadback` are balanced;
  - the `PublishGpuWrite` memo is invalidated on subtract.
- **Guest runtime:**
  - the RTC epoch and `days_from_civil` are correct (checked numerically);
  - semaphore FIFO order, timeouts, cancel and delete;
  - mutex and condition-variable type mapping, beyond the R3 items;
  - event-flag cancel and timeout, beyond the R3 pulse item;
  - clock IDs and `pthread_once`;
  - AudioOut format tables and pacing;
  - Ajm job and chunk bounds, apart from item 18;
  - pad open/read, PlayGo headers, `getdirentries` layout;
  - path translation rejects `..`;
  - FP control words are copied into new threads;
  - the Windows main-thread stack is committed.
- **userfaultfd:** disabled (`emulator_settings.h:72` returns false), so Linux uses SIGSEGV. Its `OnMap(lower, upper)` passes an end address as a size, which only matters if it is ever re-enabled.

## Suggested order

1. **Confirm with the deterministic reproductions:**
   - item 5: minimize in a loop;
   - item 6: truncate one cache file;
   - item 1: lower the sampler GC trigger in a test build.

   Each fix is small and local.
2. **Add the one-line detectors and run a long route,** to learn whether the latent items fire in this game:
   - item 2: log in `place()`;
   - item 4: log a nested read lock behind a waiting writer;
   - item 8: log heap-B straddles;
   - item 12: log the flexible-memory size the game asks for;
   - item 16: the thread counter.
3. **Correctness fixes with no performance cost:**
   - items 3 and 10 (hook queue: a mutex and a growable array);
   - item 7 (hash through the backing view);
   - item 9 (per-sub-range write-back);
   - item 14 (device-lost diagnostics);
   - item 19 (`frame_index`);
   - items 17, 18, 20.
4. **Then the rest:**
   - item 13 (cache cap and orphan cleanup);
   - item 11;
   - item 15;
   - item 22;
   - item 23.
