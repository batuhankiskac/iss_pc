# Cloud audit, round 7: review of the 35 commits after `fac17b4`

Rounds 1-6 reviewed `main` at `fac17b4`:

- [`cloud-audit.md`](cloud-audit.md)
- [`round 2`](cloud-audit-round2.md)
- [`round 3`](cloud-audit-round3.md)
- [`round 4`](cloud-audit-round4.md)
- [`round 5`](cloud-audit-round5.md)
- [`round 6`](cloud-audit-round6.md)

Since then `main` has moved to `d7c30e3` (2026-10-26). The 35 commits in between (`fac17b4..d7c30e3`, 52 files, +2217/-221) do three things:

- They implement fixes from those audits.
- They add toggles: `ISS_SUBMIT_LOCK=split`, `ISS_VO_WAIT=event`, `ISS_FLIP_EARLY_LABEL`, `ISS_OVERLAP_AGE`, `ISS_PERF_LOG`.
- They change defaults: `ISS_SUBMIT_INTERVAL_US=500` and `ISS_EXTRQ_INLINE=1` in `defaults.env`.

This round reviews those commits. It looks for four things:

- bugs the commits introduced;
- audit fixes that leave the original failure path reachable;
- performance regressions in the new code;
- tests that do not exercise what they claim.

It also covers the two player issues opened since round 6 (#704, #705).

Rules, as before:

- Nothing was run or changed.
- Every line number refers to `d7c30e3` and was read from the file. The core claims of items 1-6, 8 and 10 were re-read a second time.
- Items not traced end to end are marked **suspected**.

Abbreviations:

| Short | Path |
|---|---|
| `G` | `port/gpu_v2/gpu` |
| `O` | `port/gpu_v2/gpu/overrides` |
| `S` | `port/gpu_v2/shadps4` |
| `NP` | `port/gpu_v2/gpu/native_passes.cpp` |
| `NPR` | `port/gpu_v2/gpu/native_passes_rasterizer.inl` |
| `TC` | `O/texture_cache.cpp` |
| `VS` | `O/vk_pipeline_serialization.cpp` |

Item IDs from earlier rounds are written `R<round>-<item>`, for example `R4-1` or `R3-B1`.

## Summary

- **Two regressions in the new cache code can stop every launch.**
  - **Item 1:** 8f793cd narrowed a catch, so a corrupt `.meta` length now escapes as `std::bad_alloc` and GPU initialization fails.
  - **Item 2:** 152bc9a's SPIR-V check accepts exactly the truncation shape the old in-place writer left behind.
  - In both cases the bad file is never deleted, so every launch repeats the failure.
  - Each fix is a few lines.
- **Four audit fixes are partial; the original failure is still reachable on the default path:**
  - R4-1, sampler lifetime in native pool jobs (item 3);
  - R3-B1, native-pass read faults deadlocking the CP (item 4);
  - R4-5, `SURFACE_LOST` (item 10);
  - R4-6, cache corruption (items 1 and 2).
- **`ISS_PERF_LOG=1` prints nothing (item 5).** The `Perf:` line is logged in a class that is filtered at `Warning`. The docs and the 2026-10-26 findings describe a line that never appears. Field evidence: a v1.0 log with `TLG_GPU_LOG=debug` (#705) has 0 `Perf:` lines.
- **The measured `ISS_EXTRQ_INLINE=1` gain reaches only renderer v2 (item 6).** The code default is still 0, and `defaults.env` applies only to v2. So Linux (renderer 1 by default) on Intel CPUs keeps the slow stub.
- **The paired-CI change to `round_analyze` can mark a significant result with the wrong sign (item 7).** This happens when a round is unbalanced, for example when one control run crashed. Bench verdicts after 6c2a858 should be re-checked against the "paired avg_fps diff" lines.
- **Most of the new code checks out** (see "Verified correct"):
  - the split submit lock, the bounded tick waits, the early flip label;
  - `ISS_VO_WAIT=event`, the occlusion hold, the AscRelease parse-time label;
  - the atomic image flags, the queued LRU touches;
  - the event-flag handoff, the guest mutex/rwlock fixes;
  - the SSE4A inline stubs (bit-exact);
  - the defaults.env precedence for players.

## Ranking (severity × likelihood)

| # | Finding | Main location | Symptom | Default path? | Commit | Fix risk |
|---|---|---|---|---|---|---|
| 1 | A corrupt `.meta` string length escapes the preload as `std::bad_alloc` | `S/common/serdes.h:133-138`, `VS:412-417` | "GPU initialization failed" on every launch | yes, with a corrupt or format-mismatched `.meta` | 8f793cd | none |
| 2 | `IsValidSpirv` accepts a `.spv` truncated at 4096·k bytes | `G/cache_blob.h:22-30`, `VS:437-445` | pipeline creation abort or driver crash on every launch | yes, after an old build was killed mid-write | 152bc9a | low |
| 3 | Native pool jobs can write a destroyed `VkSampler` into a descriptor (R4-1 incomplete) | `NPR:1006-1011`, `TC:1910-1919`, `NP:1210-1216` | NVIDIA driver crash class | yes, once ≥3/4 of the sampler limit is used | a1458bd | low |
| 4 | Native scan and pool threads still read command buffers through the guest mapping (R3-B1 incomplete) | `NP:935,1585`, `NP:1439-1450` | permanent CP hang | yes (pump off by default) | 43f89bf | low |
| 5 | `ISS_PERF_LOG=1` never prints the `Perf:` line | `S/common/perf_stats.h:918`, `G/logging.cpp:9` | diagnostics blind; docs wrong | yes | edcfebc | none |
| 6 | `ISS_EXTRQ_INLINE=1` reaches only renderer v2 | `port/src/probe.c:1690-1694`, `port/gpu_v2/defaults.env:27` | ~2% FPS lost on Linux/Intel and renderer 1 | yes | a085047 | low |
| 7 | `round_analyze` takes the sign from all runs but significance from paired rounds | `scripts/round_analyze.py:626-650,699-703` | a significant change shown with the wrong sign | bench tooling | 6c2a858 | none |
| 8 | The driver-cache checkpoint copies before the cap check, has no exception guard, does not retry the rename | `G/pipeline_checkpoint.cpp:52-75`, `G/bridge.cpp:759-760` | a hitch every 30 s, `Fault: exception 0x20474343`, a leftover `.pending` file | above the cap / under memory pressure | 8f793cd (not a regression) | low |
| 9 | Every boot snapshots, FNV-hashes and rewrites the whole driver cache before the game starts | `G/pipeline_checkpoint.cpp:39,46,63`, `G/bridge.cpp:741-742` | +0.7-1.2 s per launch (seconds on HDD) | yes | pre-existing | low |
| 10 | `SURFACE_LOST` still recreates on the dead surface and asserts (R4-5 incomplete) | `G/swapchain_policy.h:20,27`, `O/vk_swapchain.cpp:146,471-473` | abort on display-driver reset or RDP switch | yes | 152bc9a | low |
| 11 | Linux CP placement: no core exclusion, `14` fallback off hybrid Intel, README stale (R2-1 partial) | `G/bridge.cpp:813-816,852-866`, `README.md:225` | SMT sibling contention on the frame-limiting core | Linux | 647cadb | low |
| 12 | Sampler LRU touches cover only the graphics memo, so native-path samplers are evicted every frame above the trigger | `O/vk_rasterizer.cpp:4831-4837` | native segments with samplers fall back every frame | above the trigger | a1458bd | low |
| 13 | Six Advanced switches have no reader in v2 but are not marked; bench `ini:` variants on v2-ignored keys are A/A | `port/launcher/advanced_table.py:44,61,85,98,173,178`, `port/tests/test_launch_env_v2.py:142-158` | switches that do nothing; A/A bench runs | yes | 0fe5f49, 2f95d3e | none |
| 14 | A non-owner can unlock a NORMAL (type 3) guest mutex | `port/src/runtime_mutex.c:59,158-171` | lost mutual exclusion | if the game does it | pre-existing | low |
| 15 | `IssPreciseSleepUs` busy-spins the window thread without a high-resolution timer | `G/bridge.cpp:434-438` | one core at 100% | Windows before 10 1803, or a timer failure | 647cadb | none |
| 16 | `ISS_ASC_NATIVE_QUEUE=1` with unified memory keeps the backwards-fence hang | `O/vk_rasterizer.cpp:5714-5723,5792-5803` | guest spins forever | **no** (opt-in) | 6d254a5 | low |
| 17 | Tests that cannot fail where it matters | `port/tests/test_runtime.c:322-342`, `port/tests/test_cache_blob.cpp:41-133`, `port/tests/test_sema.c:212-219` | no regression guard | CI | c355870, 152bc9a, d372769 | none |
| 18 | Bench and launcher environment residuals | `scripts/bench.ps1:812-813,897-908`, `port/launcher/infamous_run.py:858,874` | A/A bench runs; shell overrides ignored | shell users and bench | 87daebe, 2f95d3e | none |
| 19 | `sceKernelSleep` is invisible to the freeze report (R4-20 half done) | `port/src/runtime_gr2.c:94-102` | freeze reports show sleeping threads as running | yes | c355870 | none |
| 20 | Smaller items: present-stream skip path, overlap age in menus, R3-B3, cancel bits, plain `Image` copies, token-write lock, cache semantics | see section 20 | various, low | mixed | mixed | low |

Fix items 1, 2 and 3 before the next release. They are the only ones that can turn a working install into one that crashes on every launch or in normal play, and each is a few lines.

---

## 1. A corrupt `.meta` string length escapes the preload as `std::bad_alloc` (R4-6 incomplete)

- **Location:**
  - `Reader::Read(std::string&)` at `S/common/serdes.h:133-138` calls `s.resize(length)` (`:136`) before the bounds check. The check only runs in `Read(ptr, size)` at `:111`.
  - `Read(auto& v)` (`:124-131`) has the same shape: it `emplace_back`s `num_elements` times and only fails when an element read runs out.
  - The catch around `LoadShaderMeta` turns only `std::logic_error` into `iss::AssertFailure` (`VS:412-417`).
  - The callers catch only `iss::AssertFailure` (`VS:278`, `VS:739`).
  - Commit 152bc9a had `catch (const std::exception&)` there; 8f793cd narrowed it ("narrow catches").
- **Mechanism:**
  - The WarmUp preload reads each shader's metadata: `preload_key` (`VS:735-737`) → `LoadCachedPipelineStage` (`:277`) → `LoadPipelineStage` (`:413`) → `LoadShaderMeta` → `StageSpecialization::Deserialize` (`spec.Read(bits)`).
  - A `length` above `std::string::max_size()` throws `length_error`, a `logic_error`, which is handled.
  - A `length` between free memory and `max_size()` (about 2^63 with libc++) makes `resize` throw `std::bad_alloc`. That is not a `logic_error`, so it passes every catch and reaches the GPU init thread's catch (`G/bridge.cpp:766`), which logs "GPU initialization failed: std::bad_alloc".
  - `DropCorruptBlob` never runs, so the same blob fails the next launch too.
  - Mid-range lengths (0.1 GB up to free memory) zero-fill gigabytes before the check fires: a multi-second stall at boot, after which the blob is dropped.
- **Realistic triggers:**
  - A change to the metadata layout without a `ShaderMetaVersion` bump. The length is then read from neighbouring bytes, for example the ASCII `'0'`/`'1'` of the bitset, which gives a value around 0x3130313000000000.
  - A bit flip in the high bytes of a length field.
- **Effect:** the game cannot start until the player deletes the shader cache folder. No frame-time cost.
- **Proposed fix:**
  - In `Read(std::string&)`, add `ASSERT_MSG(length <= ar.container.size() - ar.offset, "...")` before `resize`.
  - In `Read(auto&)`, add `ASSERT_MSG(num_elements <= ar.container.size() - ar.offset, "...")` before the loop. Every element reads at least one byte, so a valid count never exceeds the bytes left.
  - Also catch `std::bad_alloc` next to `std::logic_error` at `VS:414`, so any other allocation from a corrupt size is dropped too.
  - Valid blobs always pass.
- **Toggle:** none (correctness).
- **Risk:** none.
- **How to measure:** in a copy of a cache, set the length field of one `.meta` bitset to `0x3130313000000000`.
  - Before the fix, init fails.
  - After it, the log says "Corrupt cached shader metadata ... skipped and deleted" and the game boots.
  - Add this case to `test_cache_blob` (item 17).

## 2. `IsValidSpirv` accepts the truncation shape the old writer produced (R4-6 item 3 incomplete)

- **Location:** `G/cache_blob.h:22-30` checks only three things: at least 20 bytes, a multiple of 4, and the magic number. It is used at `VS:437-445`. Commit 152bc9a.
- **Mechanism:**
  - Builds before 152bc9a (including v1.0, which every player has run) wrote `.spv` files in place. They used `IOFile::WriteSpan`, which is UCRT `fwrite`.
  - `fwrite` writes `floor(N/4096)·4096` bytes straight through and buffers the tail until `fflush`.
  - So a process killed during the write leaves a file of 0 bytes or of 4096·k bytes. The latter passes all three checks.
  - `CompileSPV` succeeds, because drivers do not validate modules at `vkCreateShaderModule`.
  - The compiler thread then builds a pipeline from a module cut mid-instruction, without `OpFunctionEnd`. The driver either returns an error (`ASSERT_MSG` abort at `O/vk_graphics_pipeline.cpp:515` or `O/vk_compute_pipeline.cpp:148`) or crashes.
  - The blob is not dropped, so every launch repeats this.
  - Older releases that share the cache folder still write this shape.
- **Effect:** a crash on every launch, or on the first use of that shader, for any player whose cache has such a file. Round 6 found many crashes during shader compile and early boot; this is a candidate cause for the ones that persist across launches.
- **Proposed fix:** walk the instruction stream in `IsValidSpirv`:
  1. Require the ID bound (word 3) to be non-zero.
  2. For `i = 5` while `i < n`: let `count = w[i] >> 16`; reject `count == 0` or `count > n - i`; then `i += count`.
  3. Require the last instruction's opcode (`w[last] & 0xffff`) to be 56 (`OpFunctionEnd`).
  - Cost is about 1 µs per KB, roughly 10-30 ms over a full preload (estimate).
- **Toggle:** none.
- **Risk:** low. A module whose last instruction is not `OpFunctionEnd` is invalid by the SPIR-V spec when it has functions, and every shader here has an entry point.
- **How to measure:** a unit test where every 4-byte prefix of a real module must fail. Also log `rejected_spirv` once at the end of the preload.

## 3. Native pool jobs can write a destroyed `VkSampler` into a descriptor (R4-1 incomplete)

- **Location:**
  - The pool job checks only that the template's images are still alive (`NPR:1006-1011`). It never checks the sampler generation.
  - It then writes the template's sampler infos (`NPR:1239-1246` via `G/native_material_rasterizer.inl:218-221,460-466`) with `pushDescriptorSetKHR` / `updateDescriptorSets`.
  - The sampler GC defers the destroy until the GPU passes the current tick and the jobs started before `RecordingTicket()` finish (`TC:1910-1919`).
  - `WaitRecordingBefore(ticket)` waits only for jobs with `started <= ticket` (`NP:1210-1216`).
  - Commit a1458bd.
- **Mechanism:**
  1. At frame end the CP's `GarbageCollectSamplers` evicts sampler S, which only a native template T still uses (item 12 explains why such samplers are the victims).
  2. It bumps `samplers_generation`, takes ticket K, and defers the destroy.
  3. A pool job J starts after the GC (sequence K+1) for a DCB the guest just submitted. It finds T, which stays in the tables until the CP's own check drops it (`G/native_material_rasterizer.inl:347`, `NPR:1712`).
  4. J's image check passes, and J writes S's handle into its descriptors.
  5. `WaitRecordingBefore(K)` does not wait for J. With `ISS_SUBMIT_INTERVAL_US=500` the tick completes within a few ms, so `vkDestroySampler` can run before or during J's write.
  - The CP rejects J's segment at claim time (`NPR:1712`), so the command buffer is never executed. But the destroyed handle has already been passed to the driver while recording. That is the class of the NVIDIA crash the image check at `NPR:1006-1008` was added for.
  - The compute (`native_compute_rasterizer.inl:647`) and recorder (`native_recorder.inl:842,900`) paths do check the generation. Only the graphics pool job does not.
- **Effect:** a crash inside the driver during recording. It can only happen once the cache holds at least 3/4 of `maxSamplerAllocationCount` (about 3000 on NVIDIA's 4000), because the GC does not evict below that. Nothing logs the sampler count today, so how often players reach it is unknown.
- **Proposed fix:**
  - After `NPR:1011`, add `if (t->has_samplers && g.texture_cache->SamplersGeneration() != t->samplers_generation) return miss(MissSamplers);`.
  - Load the generation with `seq_cst`, to pair with the `seq_cst` store of `running_jobs` (`NP:1188`) and the fence in `RecordingTicket()`. That is the same handshake the image path relies on: a job that sees the old generation started before the ticket, so the destroy waits for it.
  - Add `samplers=` and `trigger=` to the 600-tick `TEXTURE_GC` line.
- **Toggle:** none (correctness).
- **Risk:** low. A miss only sends the draw to the full path.
- **How to measure:** a test build with the sampler GC trigger lowered (for example 64) under validation layers, on a route with many materials. Before the fix, validation reports a destroyed `VkSampler` handle used by native pool threads; after it, none.

## 4. Native scan and pool threads still read command buffers through the guest mapping (R3-B1 incomplete)

- **Location:**
  - 43f89bf moved the token writes to the backing view (`NP:105-114`, `G/bridge.cpp:290-298`).
  - The scan thread still reads the DCB directly: `const u32 header = *p;` at `NP:1585`, run by `ScanThread` (`NP:1684`).
  - The pool threads also read it directly in `RecordCommandBuffer` (`NP:935`).
  - The CP's wait for the jobs is untimed unless `ISS_NATIVE_WAIT_PUMP=1`, which is off by default (`NP:1439-1450`). The waiting call sites are `O/liverpool.cpp:1891` (`WaitRecorded`) and `:1949` (`WaitRecordedAt`).
- **Mechanism:**
  1. The GPU writes into a page that a later command buffer shares (occlusion results or another GPU write next to a DCB), so the page is read-protected.
  2. A scan or pool thread reads that page and takes a read fault.
  3. The fault handler's `ReadMemory` path (`page_manager.cpp:965` → `buffer_cache.cpp:1674`) is off the CP, so it calls `SendCommand<true>` and blocks until the CP serves the readback.
  4. The CP is in `WaitDone`, inside `submit.done_cv.wait`, waiting for that job. It never runs `ProcessCommands`.
  - The new sampler destroy adds one more caller of the same blocking wait. `WaitRecordingBefore` is a yield spin that runs inside `PopPendingOperations` with `pending_ops_mutex` held (`O/vk_scheduler.cpp:752-753`). It is reached from `DeleteImage` (`TC:2158-2159`) and now also from the sampler destroy (`TC:1916-1917`).
  - R3-B1's proposed fix covered both the token writes and the command-buffer reads. Only the writes were done.
- **Effect:** a permanent hang (the CP and one worker wait for each other). The likelihood is low and depends on memory layout. In a freeze report it would show as a `NativePassesScan` or pool thread inside a fault while the CP sits in `done_cv`.
- **Proposed fix:**
  - **Real fix:** read the command buffer through `host.backing(begin, bytes)` in `ScanCommandBuffer`, `RecordCommandBuffer` and `RecordDispatch`. Translate the pointer once per job, and keep guest virtual addresses for registry keys and token addresses.
  - **Cheap interim fix:** turn `ISS_NATIVE_WAIT_PUMP=1` on in `defaults.env` after an A/B run. The pump unlocks `pool_mutex` before serving commands (`NP:1451-1454`), so it adds no lock order, and it serves them at packet boundaries where `ProcessCommands` already runs.
  - **For the deferred destroys:** when jobs before the ticket are still running, move the op to a CP-owned "after recording" list and poll that list after `PopPendingOperations` has dropped its mutex. Re-deferring from inside the callback would self-deadlock on `pending_ops_mutex`.
- **Toggle:** `ISS_NATIVE_WAIT_PUMP` exists; the backing-view reads need none.
- **Risk:** low. The backing view is what the token writes already use.
- **How to measure:** add a thread-local "native worker" flag and log it on read faults in `GuestFaultSignalHandler`. Count `native_worker_read_faults` in the exit summary. Any non-zero count is a potential hang.

## 5. `ISS_PERF_LOG=1` never prints the `Perf:` line

- **Location:**
  - `S/common/perf_stats.h:918` writes the line with `LOG_INFO(Debug, "Perf: ...")`.
  - `G/logging.cpp:9` sets every log class to `Warning`, except `Render` and `Render_Vulkan` at `Info`.
  - `LOG_GENERIC_AT` checks `ShouldLog` before formatting (`S/common/logging/log.h:34-36,50-53`).
  - Nothing raises `g_class_levels`: `UpdateLogLevels` is declared but not defined, and `TLG_GPU_LOG` does not touch it.
  - The toggle is at `G/bridge.cpp:592-594,642`, and `docs/PERFORMANCE_DIAGNOSTICS.md:444` documents it.
  - Commit edcfebc added the toggle; fc533c0 fixed its format string. The level filter is older.
- **Mechanism:** `OnFlip` sees `perf_log`, sums ~190 per-thread counters and calls `LOG_INFO(Debug, ...)`. `ShouldLog(Debug, Info)` is `Info >= Warning`, which is false, so nothing is written. Only the `LOG_WARNING` "Perf slow frame" lines (`perf_stats.h:880`) get through.
- **Field evidence:** the #705 log (v1.0, `TLG_GPU_LOG=debug`) has 0 `Perf:` lines. The only GPU log classes in it are 100 and 101 (`Render`, `Render_Vulkan`).
- **Effect:**
  - No FPS change; about 10 µs per flip is spent building a line nobody sees.
  - The new `vo_waits`, `cp_label_stalls` and `submit_mutex_wait` totals are visible only in the frame CSV.
  - Every earlier audit item whose "how to measure" names the `Perf:` line cannot be measured that way.
  - The "Audit integration (perf log, ...)" finding and the toggle table describe output that does not exist.
- **Proposed fix:**
  - Print the line directly with `fmt::format` and `std::fputs(..., stderr)`, bypassing `VLog`.
  - `VLog` keys its dedup table by the full text and caps it at 4096 keys (`G/logging.cpp:15-18`). A unique ~2.5 KB line per second fills that table in about 68 minutes. After that, every new repeating message is printed on every occurrence instead of at powers of two, which brings back the log flood of round 6 (L2).
  - The alternative, raising `g_class_levels[Debug]` to `Info` when `perf_log` is set, has that side effect. Prefer the direct print.
- **Toggle:** `ISS_PERF_LOG` (exists).
- **Risk:** none.
- **How to measure:** run with `ISS_PERF_LOG=1` and `grep -c '^Perf:'` the log: expect about one line per second. Today you get only `PORT_TUNING perf_log=1` and the slow-frame lines.

## 6. `ISS_EXTRQ_INLINE=1` reaches only renderer v2

- **Location:**
  - `port/src/probe.c:1690-1694` reads `TLG_EXTRQ_INLINE`; the stubs are on only when the value is `"1"`.
  - The comment and `port/docs/isa_rewrite.md:21` both say "default 0".
  - Only `port/gpu_v2/defaults.env:27` sets it, and the launcher applies `defaults.env` only when renderer v2 runs (`port/launcher/infamous_run.py:157-171,853-859`).
  - The Linux renderer default is `1` (`port/launcher/backports_table.py:256`).
  - Commit a085047.
- **Mechanism:** the inline SSE2 stubs are a CPU-side rewrite of the game's SSE4A `extrq`/`insertq` sites and do not depend on the renderer. They are applied only on CPUs without SSE4A, which means every Intel CPU. These setups still run all 96 sites through the C emulator call (pushfq, 11 pushes, call), 3.2 million times per 60 s:
  - Intel Linux players (renderer 1 by default);
  - Windows players who pick renderer 1;
  - direct probe runs.
- **Bit-exactness:** checked against `isa_emulate_sse4a` (`port/src/isa_emulate.c:141-148`); the stubs match it:
  - length 0 means 64;
  - when index + length > 64, the high bits are zero-filled;
  - the register form computes `(64-len)&63`;
  - the `INSERTQ` mask is cut at bit 63, and the high qword is zeroed;
  - the REX/prefix order and the RIP-relative constants at slot+224 are correct;
  - a stub that would reach the constants falls back to the C stub (`port/src/isa_rewrite.c:428`).
- **Effect:** the measured gain was +2.2% FPS (95% CI +1.1..+3.3) and -5.5% guest CPU. At 60 FPS that is about 0.35 ms per frame, missed by those players.
- **Proposed fix:**
  - Make the code default on: `isa_table->inline_sse4a = !(extrq_inline && extrq_inline[0] == '0');`.
  - Keep `ISS_EXTRQ_INLINE=0` as the way out, and update the comment and `isa_rewrite.md`.
  - The alternative, `'ISS_EXTRQ_INLINE': '1'` in the launcher's `DEFAULT_ENV` (`infamous_run.py:338`), would miss direct runs.
- **Toggle:** `ISS_EXTRQ_INLINE=0` to turn it off.
- **Risk:** low. `test_isa_sse4a.c` covers the stubs.
- **How to measure:** on an Intel Linux renderer-1 run, check that the log shows "SSE4A inline stubs: N sites", then run one paired bench session on v1.

## 7. `round_analyze` can mark a significant change with the wrong sign

- **Location:**
  - In `scripts/round_analyze.py`, each cell's mean and `diff_pct` use all of a variant's runs (`:626-627`).
  - The `~` (noise) decision uses `paired_diffs` (`:631,643`), which keeps only rounds present on both sides (`:699`).
  - `better` takes its sign from the unpaired `d` (`:650`).
  - Commit 6c2a858 (the R3-C7 fix). 2f95d3e added a fallback to the unpaired test only for rounds whose control value is 0 (`dropped`, `:701-702`).
- **Mechanism:**
  - `bench.ps1` does not re-run a failed run, `cmd_bench` drops failed runs, and `--exclude-drift` drops drifted ones.
  - So one control crash leaves the variant with an extra, unpaired run. That run moves the unpaired mean, while significance comes from the paired rounds only.
  - Reproduced on the snapshot module:
    - A = 60.0 / 60.6 / 61.0 FPS (round 4 crashed); B = 61.0 / 61.6 / 62.0 / 55.0.
    - The cell shows B −1.0%, `noise_test=paired`, `noise=False`, `better=False`: a significant regression.
    - The "paired avg_fps diff" line for the same data says +1.65%, 95% CI [+1.62, +1.69].
  - A variant that hangs now and then, like `ASC_NATIVE_QUEUE`, produces exactly this pattern.
- **Effect:** wrong keep/drop decisions on tuning toggles. No runtime cost.
- **Proposed fix:**
  - In `paired_diffs`, add `unpaired = sum(k not in ctl for k in v) + sum(k not in v for k in ctl)` to the result.
  - At `:643`, also require `not e["unpaired"]`.
  - When the paired test applies, take `better` from `e["mean"]` and print the paired mean in the cell.
- **Risk:** none.
- **How to measure:** add a session with one failed control run to `port/tests/test_round_analyze.py`. Re-check the 2026-10-26 table entries whose sessions had a failed or excluded run.

## 8. Driver-cache checkpoint: copies before the cap check, no exception guard, no rename retry

- **Location:** `G/pipeline_checkpoint.cpp:52-75`. The periodic caller is a bare detached thread at `G/bridge.cpp:759-760`.
- **Mechanism:**
  - **Copy before the cap check.** `getPipelineCacheData` copies the whole driver cache (`:52`) before the cap test (`:53`). Over the cap, the function returns without updating `checkpointed_creates`. So once the cache passes 512 MB, every 30 s checkpoint that follows a new pipeline creation does three things:
    - copies more than 512 MB;
    - holds the driver's cache lock during the copy (the R2-11 hitch on compiler threads);
    - throws the copy away.
  - **No exception guard.** Nothing catches `std::bad_alloc` from that copy. On the detached thread this terminates the process, which a player report shows as `Fault: exception 0x20474343` (round 6 cluster C2, fix item 3, still open). 8f793cd halved the peak by dropping the packed copy; the guard is still missing.
  - **No rename retry.** `MoveFileExW` (`:72`) is not retried. The blob writer got a retry for antivirus sharing violations (`G/cache_blob.cpp:18-27`), but this rename did not. On failure, a 200-500 MB `.pending-<pid>` file stays on disk until the next start.
  - **Over-cap cache files are never replaced.** A cache file over the cap is skipped at load (`O/vk_pipeline_cache.cpp:1320-1324`), and since nothing over the cap is saved, it is never replaced. That happens if the file was written under 152bc9a's 1024 MB default or before `ISS_PIPELINE_CACHE_MAX_MB` was lowered. Every launch then compiles everything again. The one-time message ("new pipelines will compile again next session") understates this.
- **Effect:**
  - Over the cap: a 100-300 ms copy every 30 s while new pipelines appear, plus compile hitches.
  - Under memory pressure: a crash on the checkpoint thread.
  - How many players exceed 512 MB is unknown; #564 had 230 MB.
- **Proposed fix:**
  - Query the size first with `vkGetPipelineCacheData(dev, cache, &size, nullptr)`. Once a size over the cap is seen, set `static bool over_cap` and return early from then on, since the cache only grows.
  - Wrap the body in `try { ... } catch (const std::exception& e) { log; return false; }`.
  - Reuse the retrying `Replace` from `cache_blob.cpp`, and call `DeleteFileW(temporary)` on failure.
  - At load, apply the user cap to writes only and a hard ceiling (4000 MB) to reads; the envelope checksum already guards integrity.
- **Toggle:** none.
- **Risk:** low.
- **How to measure:** the `PIPELINE_CHECKPOINT ... elapsed_ms` and `over cap` lines, and a test that fills memory before a checkpoint.

## 9. Every boot rewrites the whole driver cache before the game starts (pre-existing)

- **Location:** `G/pipeline_checkpoint.cpp:39,46,63`, called at `G/bridge.cpp:741-742` before `ready=1`. The FNV checksum is in `S/video_core/renderer_vulkan/driver_cache_data.h:24-28`.
- **Mechanism:**
  - `previous_hash` starts at 0, and the preload counts as pipeline creations.
  - So the first checkpoint always does the full job on the window thread, before the game is released: it snapshots the cache, hashes it with XXH3 (`:63`), computes the byte-serial FNV checksum for the envelope (~4 cycles per byte), and writes and flushes about 217 MB.
  - The load path also computes the FNV checksum once (`O/vk_pipeline_cache.cpp:1328`).
- **Effect:**
  - About 2×0.2-0.3 s of FNV plus 0.2-0.5 s of write and flush on an SSD: roughly 0.7-1.2 s added to every launch.
  - Seconds on slow disks: #564 showed 6.4 s for 230 MB.
  - No in-game frame cost.
- **Proposed fix:**
  - Use an envelope v2 with the XXH3 that is already computed, and still accept v1 on read.
  - Remember the XXH3 of the payload loaded at boot, and skip the boot write when the snapshot hashes the same. **Suspected:** drivers return identical bytes when nothing new was compiled. Log both hashes once to confirm.
- **Toggle:** `ISS_PIPELINE_CHECKPOINT_BOOT=0` (skip the boot checkpoint).
- **Risk:** low.
- **How to measure:** the boot `PIPELINE_CHECKPOINT ... elapsed_ms` line and the time from start to `ready`.

## 10. `SURFACE_LOST` still recreates on the dead surface and asserts (R4-5 incomplete)

- **Location:**
  - `G/swapchain_policy.h:20,27` maps `-1000000000` (`VK_ERROR_SURFACE_LOST_KHR`) to `Recreate` for both acquire and present.
  - `Recreate` → `Create` → `SetSurfaceProperties` (`O/vk_swapchain.cpp:146`).
  - `SetSurfaceProperties` asserts when `getSurfaceCapabilitiesKHR` fails (`:471-473`), which it does on a lost surface.
  - Commit 152bc9a fixed the 0×0 (minimized) case of R4-5 but not this one.
- **Mechanism:** after a surface loss, every recreate queries the same dead surface. The first query fails, and `ASSERT_MSG` aborts.
- **Effect:** the game aborts instead of recovering. Triggers: a display-driver reset (TDR), a monitor hot-unplug on some drivers, or an RDP session switch.
- **Proposed fix:**
  - On `SURFACE_LOST`, destroy the swapchain and the surface, call `CreateSurface(instance, window)` again, then `Create`.
  - Minimum fix: treat a failed capabilities query like 0×0. Keep no swapchain, set `needs_recreation`, and retry on the next frame.
- **Toggle:** none.
- **Risk:** low.
- **How to measure:** start the game, then connect over RDP and back; or unplug the monitor with validation layers on.

## 11. Linux CP placement: no core exclusion, `14` fallback off hybrid Intel, README stale (R2-1 partial)

- **Location:**
  - `G/bridge.cpp:813-816`: "the other threads keep their affinity".
  - `G/bridge.cpp:852-866`: the `auto` choice.
  - `README.md:225` still says "On Linux `auto` behaves like the old default `14`".
  - Commit 647cadb.
- **Mechanism:**
  - On hybrid Intel, `auto` now picks the first logical CPU of the last P-core; that part is correct. But no other thread is kept off that core, so guest spinner threads can run on its SMT sibling.
  - On non-hybrid CPUs, Zen 5/5c, and kernels without the `cpu_core` PMU, `auto` still pins logical CPU 14:
    - on an 8-core/16-thread CPU that is the sibling of core 6;
    - on a 7950X3D it is on CCD1, while Windows `auto` picks the V-cache CCD.
  - R2-1 asked for an inherited affinity mask, and for ranking cores by `cpuinfo_max_freq` / `acpi_cppc/highest_perf` where there is no hybrid PMU. Neither was done.
- **Effect (suspected):** with a busy sibling, CP throughput drops by 15-30%, which is about 1.5-3 ms of a 10 ms CP frame. Linux only.
- **Proposed fix:**
  - In `iss_gpu_initialize`, before any guest thread starts, call `sched_setaffinity` on the process with every CPU except the CP core's `thread_siblings_list`.
  - Where there is no `cpu_core` PMU, fall back to the frequency ranking.
  - Fix `README.md:225` and its Portuguese twin.
- **Toggle:** `ISS_CP_EXCLUSIVE=1`.
- **Risk:** low. Other threads lose one core.
- **How to measure:** `lscpu -e`, `perf sched timehist` on the sibling CPU, and a paired bench with and without the exclusive mask.

## 12. Sampler LRU touches cover only the graphics memo

- **Location:** the only touches are at `O/vk_rasterizer.cpp:4831-4837`, on the graphics path and not on ASC. These sampler users never touch:
  - stage-memo hits, which compare the S# only (`G/stage_bind_memo.inl:239-244`);
  - native segment claims (`NPR:1712`, `G/native_recorder.inl:842,900`);
  - material templates (`G/native_material_rasterizer.inl:347`).
  - Commit a1458bd.
- **Mechanism:**
  1. Above the GC trigger, the collector evicts the least-recently-touched samplers. These are exactly the 10-40 per frame that only native paths and memo hits use.
  2. Each eviction bumps `samplers_generation`, so every template with samplers is rejected.
  3. The full path re-learns and re-creates those samplers, so the count stays above the trigger and the cycle repeats every frame.
  4. Each repeat also reopens the item 3 window.
- **Effect:** while it lasts, native segments with samplers fall back to the CP every frame, which costs the native-pass gain (not measured). It only applies above the trigger.
- **Proposed fix:** keep the samplers' `lru_id`s in templates, and touch them once per GC tick and template on a validated claim (under `samplers_mutex`). Log `samplers.size()` first (item 3) to see whether players reach the trigger.
- **Toggle:** none.
- **Risk:** low.
- **How to measure:** the extended `TEXTURE_GC` line, and the `NATIVE_PASSES ... miss[...]` sampler counter rising together with the GC evictions.

## 13. Six Advanced switches have no reader in v2; bench `ini:` variants on v2-ignored keys are A/A (R3-C3 incomplete)

- **Location:**
  - Six rows in `port/launcher/advanced_table.py` have no reader in renderer v2:

    | Line | Switch |
    |---|---|
    | 44 | `ASYNC_FENCES` |
    | 61 | `COPY_THREADS` |
    | 85 | `WRITE_MERGE` |
    | 98 | `FAULT_WINDOW` |
    | 173 | `REGION_TRACE` |
    | 178 | `OCCLUSION_LOG` |

  - None of them is in `NO_EFFECT_V2`, so the UI does not say "(no effect in renderer v2)".
  - Example: `adv_fault_window=1024` produces `ISS_FAULT_WINDOW=1024`, but v2 reads `ISS_FAULT_WINDOW_KB` (`O/buffer_cache.cpp:1388`).
  - The other five have only comments or writers in v2 (`port/src/runtime_memory.c:753-755` comments, `port/src/probe.c:1204` writer, `G/gpu_host.cpp:581` no-op), or live in files v2 does not build (`tlg_copy.cpp`, `tlg_region.cpp`; v2's `OURS_SOURCES` is at `port/gpu_v2/CMakeLists.txt:186-189`).
  - `scripts/bench.ps1:1180` accepts any `ini:` key. The trace for `ini:adv_draw_pipe=1`: tlg.ini → `backports_table.environment` → popped at `infamous_run.py:863-865` → the run uses `defaults.env`'s 0, the same as the control. The same happens for `ini:bp01..bp12` and every `NO_EFFECT_V2` row.
  - `port/tests/test_launch_env_v2.py:142-158` scans comments, writers and every `gpu/shim/*.cpp`, not only the files v2 builds. So it would fail if any of the six were marked, and it cannot catch missing marks.
  - Commits 0fe5f49, 2f95d3e.
- **Effect:** switches that silently do nothing, and A/A bench sessions that report "neutral" for a change that was never applied. No runtime cost.
- **Proposed fix:**
  - Add the six IDs to `NO_EFFECT_V2`.
  - In the test, strip comments, scan only `OURS_SOURCES` (parsed from `CMakeLists.txt`), and add a completeness check: every row not in `NO_EFFECT_V2` must have a `getenv` reader in v2.
  - In `bench.ps1`, fail (or print a note) on `ini:` or env keys in `NO_EFFECT_V2` or in the v2 `defaults.env` keys the launcher overrides.
- **Risk:** none.
- **How to measure:** the new test fails on the current table and passes after the fix.

## 14. A non-owner can unlock a NORMAL (type 3) guest mutex (pre-existing)

- **Location:** `port/src/runtime_mutex.c:59` maps PS4 type 3 to `PTHREAD_MUTEX_NORMAL`. `mutex_unlock` (`:158-171`) calls the host unlock without an owner check.
- **Mechanism:**
  - Neither glibc nor winpthreads checks the owner when a NORMAL mutex is unlocked. So if T2 unlocks a mutex that T1 holds, the host returns 0, and the R3-B10b write-back branch (`:169`) never runs.
  - `owner`/`depth` still name T1 while the mutex is free. T3 can then lock it while T1 believes it still holds it, and T1's later unlock can release T3's hold.
  - FreeBSD libthr (PS4) returns `EPERM` for any unlock by a non-owner, whatever the type.
- **Effect:** lost mutual exclusion, so a rare crash or data corruption. **Suspected:** whether inFAMOUS ever does this is unknown.
- **Proposed fix:**
  - At the top of `mutex_unlock`: `if (!m->depth || m->owner != holder()) return orbis_error(EPERM);`. This also makes the R3-B10b write-back unnecessary.
  - To check first, log once per mutex when a non-owner unlock happens.
- **Toggle:** `TLG_MUTEX_OWNER_CHECK=0` to restore the old behaviour.
- **Risk:** low. It returns what PS4 returns.
- **How to measure:** the one-time log line over a long play session.

## 15. `IssPreciseSleepUs` busy-spins without a high-resolution timer

- **Location:** `G/bridge.cpp:434-438`. Since 647cadb, the window loop calls it every iteration (`:733`, `:764`) instead of `sleep_for(2ms)`.
- **Mechanism:**
  - `CreateWaitableTimerExW(..., CREATE_WAITABLE_TIMER_HIGH_RESOLUTION, ...)` returns NULL on Windows before 10 1803.
  - The thread-local `timer` then stays NULL, and every call does `SwitchToThread(); return;`. The same happens when `SetWaitableTimer` fails.
  - `SwitchToThread` returns at once when nothing else is ready on that CPU, so the window thread polls `PollEvents` in a tight loop.
  - `IssVblankSleepNs` in the same file already falls back to a plain waitable timer (`:496-499`).
- **Effect:** one logical CPU at 100% on affected systems. **Suspected:** 0-3% FPS on power- or thermal-limited laptops. No crash.
- **Proposed fix:** mirror `IssVblankSleepNs`: `if (!t) t = CreateWaitableTimerW(nullptr, TRUE, nullptr);`. If the handle is still null, or `SetWaitableTimer` fails, call `Sleep(std::max(1u, us / 1000))` instead of `SwitchToThread()`.
- **Toggle:** none.
- **Risk:** none.
- **How to measure:** force `timer = nullptr` in a test build and watch the window thread's CPU in Process Explorer.

## 16. `ISS_ASC_NATIVE_QUEUE=1` with unified memory keeps the backwards-fence hang

- **Location:**
  - `O/vk_rasterizer.cpp:5792` writes the label at parse time only when unified memory is off.
  - With it on (`:5796-5803`), the code logs once, and the compute ring's label goes to the compute scheduler's priority operations. Graphics-path labels go to the graphics scheduler (`DeferUnifiedFence`, `:5626-5637`).
  - Unified memory is on in `defaults.env:129`, which `play_v2.sh` uses, and in the launcher's Linux default.
  - Commit 6d254a5.
- **Mechanism:**
  1. Ring R's release N goes to the compute scheduler.
  2. R's next work runs on the graphics queue, and release N+1 goes to the graphics scheduler.
  3. Two independent threads write the two labels. If N lands after N+1, the guest's monotonic fence goes backwards, and the guest spins: the hang in `docs/PERFORMANCE_DIAGNOSTICS.md` ("Keep off").
- **Effect:** a guaranteed eventual hang with that toggle. Opt-in only.
- **Proposed fix:** have `AscQueueEnabled()` (`:5714-5723`) return false when `iss::unified::enabled`. Or, better if the queue is ever worth it, route a compute ring's labels through the graphics scheduler after `AddCrossWait(compute tick)`, so one timeline retires all of a ring's labels.
- **Risk:** low.
- **How to measure:** the same 55 s gameplay run that hung, with `ISS_UNIFIED_MEMORY=1`.

## 17. Tests that cannot fail where it matters

- **R4-4 test (c355870):** `rw_reentry` (`port/tests/test_runtime.c:322-342`) is meant to fail with `TLG_RWLOCK_REENTRANT=0`.
  - glibc's default rwlock prefers readers, so the nested timed read succeeds behind the queued writer even with the fix off.
  - CI runs `--test-only` only on Linux (`.github/workflows/build.yml:89`); the Windows job builds without tests. So the test cannot fail in CI.
  - Fix: add a test-only `TLG_RWLOCK_PREFER_WRITER=1` that sets `PTHREAD_RWLOCK_PREFER_WRITER_NONRECURSIVE_NP` in `create_locked`. Have the test expect a pass by default and a failure with the toggle off.
- **Event-flag SRW path (d372769):** `port/tests/test_sema.py:17-18` sets `TLG_EF_SRW=1`, but `ef_srw()` returns 0 everywhere except Windows (`port/src/runtime_tlg.c:102-106`). So the SRW branch of `ef_hand_off` (`:180-184`) never runs in CI.
  - Fix: run `runtime-test` and `sema-test` in the Windows job.
- **Cancel test timing:** the cancel+set loop (`port/tests/test_sema.c:212-219`) relies on `nap_ms(20)` for the waiter to enrol. On a loaded runner the waiter can enrol after the cancel+set, match `0x1`, return 0 and fail the assertion.
  - Fix: `do { nap_ms(5); assert(ef_cancel(ef, 0, &waiting) == 0); } while (!waiting);` before the set.
  - Still untested: the delete-after-handoff guard (`runtime_tlg.c:163`) and the timeout-versus-handoff override (`:301`).
- **Cache test (152bc9a, 8f793cd):** `port/tests/test_cache_blob.cpp:41-83` tests a hand-written copy of `preload_key`, not the real loader. It never runs `LoadShaderMeta` or `LoadPipelineStage`, the `logic_error` conversion, or `DropCorruptBlob`.
  - Corruption is tested only as prefix truncation (`:115-122`). There is no corrupt length field, which would have exposed item 1, and no near-`SIZE_MAX` size.
  - The SPIR-V cases (`:124-133`) check only the header and the multiple of 4, so item 2's shape passes.
  - Fix: drive the real `Deserialize` code, and add those three cases.

## 18. Bench and launcher environment residuals

- **A stale shell `TLG_` twin leaks into the control (R3-C6 residual, 87daebe).**
  - `StaleTlg` (`scripts/bench.ps1:897-908`) covers only the twins of keys in `defaults.env`.
  - Example: a shell with `TLG_SEMA_SRW=1`, and variants A = `{}`, B = `{ISS_SEMA_SRW=1}`. A keeps `TLG_SEMA_SRW=1`, because the probe never overwrites a `TLG_` name that is already set (`port/src/probe.c:1166`). The run is A/A, while the note at `:1225` says the variable is unset for the runs.
  - Fix: in `Resolve-VariantDefaults`, add to `$stale` the `TLG_` twin of every `ISS_` key that any variant sets.
- **Launcher precedence comments that do not hold (87daebe, 2f95d3e).**
  - A shell `TLG_X` does not win in v2: the defaults loop checks only the `ISS_` name (`port/launcher/infamous_run.py:858`). Example: `TLG_SUBMIT_INTERVAL_US=0` still gives `ISS_SUBMIT_INTERVAL_US=500`.
  - An exported empty value does not "leave the default in place" (`:874`). `ISS_TUNING=''` drops all four default tuning keys, the R3-C1 failure reached another way.
  - Fix: if `'TLG_' + key[4:]` is set in the environment, use it instead of the default; treat `''` like unset.
- **`ISS_OVERLAP_AGE=0` (ef0963c)** means "1 frame", the most aggressive freeing (`TC:56-58`), not "off". The old tick rule is `ISS_OVERLAP_AGE=ticks`. A bench "off" variant written as `0` measures the wrong thing. Warn on 0, or treat 0 as `ticks`.
- **Effect:** these only bias A/B results toward 0. None can inflate the measured +19.2% (submit interval) or +2.2% (EXTRQ): neither key was in `defaults.env` when they were measured.

## 19. `sceKernelSleep` is invisible to the freeze report (R4-20 half done)

- **Location:** `kernel_sleep` (`port/src/runtime_gr2.c:94-102`) now retries on `EINTR`, but does not call `runtime_wait_begin`/`runtime_wait_end`. Commit c355870.
- **Effect:** a thread in `sceKernelSleep` shows as running in freeze reports, the second half of what R4-20 named.
- **Proposed fix:** export `sleep_ns` from `port/src/runtime_kernel.c:122-133` and call `sleep_ns((uint64_t)seconds * 1000000000ull, RUNTIME_GUEST_FRAME())`.
- **Risk:** none.

## 20. Smaller items

| Item | Location | Note | Fix |
|---|---|---|---|
| Present-stream skip path frees a frame before its ready tick (R4-23 extended) | `O/vk_presenter.cpp:940-955` (skips) before the wait at `:1188-1191`; `RecreateFrame` at `:1274` | Only with `ISS_PRESENT_STREAM=1` (off) plus a size or HDR change. `GetRenderFrame` may then call `RecreateFrame` while the recorder or GPU still uses the image. | Call `WaitFlushedSubmitted(frame->ready_tick)` before `free_frame()` on the skip paths; R4-23's GPU wait before `RecreateFrame`. |
| **Suspected:** overlap age shorter than before in menus | `TC:624-638` (ef0963c) | At fewer than 8 submits per frame (pause menu, loading), 4 frames is less than the old 32 ticks, so aliases are freed up to 8× sooner. `ResolveOverlap` frees without a download. | Require both `frames_ok` and the tick age. Measure `overlap_frees` in menus. |
| R3-B3 still open | `TC:537-546` | `UnmapMemory` still frees images on guest threads. The commit only counts this (`TEXTURE_FREE_OFF_CP`). | As in R3-B3; measure with `ISS_ASSERT_FREE_ON_CP=1`. |
| A cancelled event-flag waiter can return a later set's bits | `port/src/runtime_tlg.c:303` (c355870/d372769) | The ECANCELED result is the cancel pattern plus bits a later set left, and the test codifies it (`test_sema.c:218`). Back-to-back cancels can count a not-yet-departed waiter twice (`:210`). | Store the cancel pattern in `w->result` and return that. |
| Plain copies and RMWs beside the atomic flags (pre-existing) | `S/video_core/texture_cache/image.h:91-92`, `O/vk_rasterizer.cpp:4605-4608`, `TC:843,846` | `Image(Image&&)` copies `flags` plainly. The `usage`/`binding` bitfields are written by the guest (`RegisterVideoOutSurface`) and by the CP without a common lock, only during `sceVideoOutRegisterBuffers` at startup. | Move under `mutex`, or make them atomic as well. |
| **Suspected:** token writes now take the global map lock | `G/bridge.cpp:290`, `NP:756,923,1063` (c1e64a4) | Each token write does a `host.backing` lookup under the global `RpLock` (R1-13, still open). 100-300 per frame, about 0.02-0.06 ms off the CP; the risk is contention with the CP's own lookups. | Fixed by R1-13; measure `cp_wait_us` before and after. |
| One redundant lock per `found_again` | `O/vk_rasterizer.cpp:1110-1114` | `TouchImageLocked` takes the mutex only to push an id `GetImage` already deduped, about 8 times per frame. | Drop the call. |
| Cache blob semantics (pre-existing) | `VS:350-353,839`; `G/cache_blob.cpp:47-49`; `docs/PERFORMANCE_DIAGNOSTICS.md:268-272` | `static_vector` sizes, stage enums and pNext/pSampleMask pointers are read from disk unchecked. `WriteAtomic` does not flush before the rename, so after a power loss a zeroed blob fails the version check and stays "stale" forever. The doc still says blobs are written in place. | Range-check inside the throwing scope; flush before the rename; drop on a version mismatch too; fix the doc. |

---

## Verified correct

Each area below was traced and found sound. They need no action.

- **Split submit lock (b95cde7).** One thread submits per scheduler at a time:
  - The CP's direct submit runs only after `SyncRecorder` (`O/vk_scheduler.cpp:448,917-930`).
  - The tick is taken under the lock on the unthreaded path (`:972-976`) and in FIFO job order on the recorder (`:565-569,906`).
  - `submitted_tick` is published under `submit_mutex` (`:1079-1090`).
  - `SetSubmitDependency` takes both mutexes with one `std::scoped_lock`.
  - Split is opt-in: the code default is `global`.
- **Bounded tick waits (8f32997).** `WaitTickPublished` (`:880-895`) never returns before the tick is published; it logs after about 1 s and keeps waiting. `WaitFlushedSubmitted`'s early return (`:872-876`) cannot be reached from `Present`.
  - Optional: make that early return an `ASSERT`.
- **Early flip label (60e3f0e).** `ReleaseFlip` frees `prev_index`, not the buffer being presented (`O/videoout_driver.cpp:495`). `Present` reads a host copy recorded before any later guest rendering.
- **Submit interval and occlusion hold (df23df4, 8052afe, e45992b).**
  - `PeriodicSubmit` (`O/vk_rasterizer.cpp:204-227`) only ever adds `Flush` calls, and the hold only skips the periodic one.
  - Waiters flush on their own (`O/vk_scheduler.cpp:717-721,861-864`), so the interval cannot delay a fence, readback or flip, and cannot deadlock.
  - The per-draw `NowNs()` costs about 25-75 µs per frame.
  - A lost query becomes `EmulatedWithBegin` with valid bits, so the guest does not spin.
- **`ISS_VO_WAIT=event` (43f89bf).** It cannot miss a wakeup:
  - The flag store and `seq_cst` fence (`O/liverpool.cpp:3419-3422`) pair with the fence in `WakeVoWait` (`G/shim/video_core/amdgpu/liverpool.h:283-288`).
  - Every label write is followed by `SignalVoLabel`.
  - There is a single waiter.
  - Note: `ISS_VO_WAIT` is not a default anywhere; it is absent from `defaults.env` and the launcher.
- **Lossless SubmitDone (43f89bf).** `exchange` fixes R3-B9.
- **Scoped SRT-watch invalidation (c1e64a4).** It walks every page of the 12 written bytes (`page_manager.cpp:272-278`), so a write that crosses a page boundary invalidates both pages.
- **AscRelease at parse time (6d254a5), unified memory off.** The guest reaches compute results only through readbacks, and those wait on the compute tick (`O/vk_rasterizer.cpp:1003,5731-5733`).
- **Frame CSV (edcfebc, 961a7cd, fc533c0).** The header and each row iterate the same `FrameCsvDetails` list (`S/common/perf_stats.h:683-732,752,797`). Each counter slot has a single writer. The `Perf:` format string has 182 placeholders for 182 arguments.
- **Texture cache (a1458bd, b1bff39, d542a17):**
  - The atomic flag helpers (`S/video_core/texture_cache/image.h:119-148`) re-check in their CAS loops and skip no-op writes. No plain `flags |=` or `&=` remains in compiled code.
  - The queued LRU touches are safe against freed and reused ids (`alive`, and the insertion tick equals `gc_tick`).
  - The lock order is `mutex` → `off_cp_touch_mutex` everywhere.
  - The sampler `ASSERT` (`TC:1904`) is live in release builds but unreachable on any traced path.
  - R4-7 (`TC:398-411`) is lock-free in the fault handler.
- **Pipeline cache (152bc9a, 8f793cd):**
  - The driver-cache envelope checks magic, version, size, checksum, header, vendor, device and UUID before `vkCreatePipelineCache`.
  - The checkpoint writes a temporary file, calls `FlushFileBuffers`, then `MoveFileExW(REPLACE|WRITE_THROUGH)`.
  - "Insert after build" leaves no half-inserted entries: WarmUp is serial, and `CompileSPV` runs under `program_cache_mutex`.
  - `Archive::Advance` is overflow-safe.
- **Guest sync (c355870, d372769).** The fixes for R4-4, R3-B2, R3-B8, R3-B10b and R4-18 close their failure paths:
  - `wait_cond` drops extra recursive levels while it still holds the last one (`port/src/runtime_mutex.c:264-269`).
  - The event-flag handoff runs entirely under the flag's lock.
  - `busy = waiters || first` (`port/src/runtime_tlg.c:163`) prevents a use-after-free on delete.
  - Each waiter is decremented exactly once, so there is no ABA on the waiter count.
  - The generation check (`:177`) closes cancel-then-set.
  - A waiter that times out while being handed bits returns them (`:301`).
  - Nested rwlock reads keep exactly one host read lock (`port/src/runtime_rwlock.c:115-118,170`).
  - Uncontended paths are no slower; a set that satisfies nobody now skips a futex wake.
- **SSE4A inline stubs (a085047).** Bit-exact (item 6). The patcher writes only inside its own slot. R5-6 still applies to both stub types.
- **Launcher precedence for players.** Process environment > the player's changed switches > `defaults.env` > code default:
  - The three `defaults.env` readers agree: Python, PowerShell and bash.
  - `bench.ps1` restores the environment after every run and exits with code 3 on failed runs.
  - The t quantile in `round_analyze` is correct (df = n−1).
  - n < 2 and a control value of 0 fall back to the unpaired test without dividing by zero.

## Status of earlier audit items touched by these commits

| Item | Commit | Status at `d7c30e3` | Remaining gap |
|---|---|---|---|
| R4-1 (sampler/view lifetime) | a1458bd | partial | pool jobs skip the sampler-generation check (item 3); LRU touches graphics-only (item 12) |
| R3-B1 (native fault deadlock) | 43f89bf | partial | DCB reads still through the guest mapping (item 4) |
| R4-5 (swapchain errors) | 152bc9a | partial | `SURFACE_LOST` (item 10) |
| R4-6 (cache corruption) | 152bc9a, 8f793cd | partial | `bad_alloc` path (item 1), truncated SPIR-V (item 2) |
| R2-1 (Linux CP CPU) | 647cadb | partial | no exclusion, non-hybrid fallback, README (item 11) |
| R4-20 (sleep) | c355870 | partial | not visible to freeze reports (item 19) |
| R3-C3 (v2-ignored switches) | 0fe5f49, 2f95d3e | partial | six rows, bench `ini:`, test scan (item 13) |
| R3-C6 (stale `TLG_` twins) | 87daebe | partial | keys outside `defaults.env` (item 18) |
| R3-C7 (`~` mark) | 6c2a858 | fixed, with a new bug | wrong sign when unbalanced (item 7) |
| Round 6 C2 fix 3 (exceptions on cache threads) | 8f793cd | partial | checkpoint thread unguarded (item 8) |
| R3-B3 (unmaps off the CP) | a1458bd | counted only | item 20 |
| R4-23 (RecreateFrame) | — | open | item 20 |
| R1-13 (global map lock) | — | open | now also on token writes (item 20) |
| R4-4, R3-B2, R3-B8, R3-B9, R3-B10b, R4-18, R4-7, R4-13, R4-14, R4-19 | various | fixed | — |

## Per commit

| Commit | Result |
|---|---|
| b95cde7 split submit lock | ok (item 20 present-stream note) |
| 43f89bf CP waits, VO wait, SubmitDone | item 4 |
| 60e3f0e early flip label | ok |
| 647cadb HR-timer window loop, Linux `auto` CPU | items 11, 15 |
| a1458bd atomic flags, LRU touches, sampler destroy | items 3, 12, 20 |
| c355870 guest sync fixes | items 17, 19; item 14 is pre-existing |
| 152bc9a presenter and cache robustness | items 2, 10, 17, 20 |
| 8f32997 bounded waits, flip scheduler | ok |
| 0fe5f49 defaults.env, ignored switches | item 13 |
| 87daebe variant environment | item 18 |
| 6c2a858 paired-CI `~` | item 7 |
| c1e64a4 token-scoped SRT invalidation | ok (item 20, suspected) |
| d372769 event-flag handoff | item 17, item 20 (cancel bits) |
| b1bff39, d542a17 texture-cache follow-ups | ok |
| 8f793cd cache review fixes | items 1, 8, 17 |
| 2f95d3e bench tooling review | items 13, 18 |
| edcfebc, fc533c0 perf log | item 5 |
| 961a7cd frame CSV columns | ok |
| 8052afe, df23df4 submit interval | ok |
| ef0963c overlap age | items 18, 20 |
| e45992b occlusion hold | ok |
| a085047 `ISS_EXTRQ_INLINE=1` | item 6 |
| 6d254a5 AscRelease at parse time | item 16 |
| Merges and docs (f2dcce3, aedd08b, 8b11ffb, d160f23, dd964aa, 59ea414, e684162, 6519380, d7c30e3) | item 5 (docs describe the `Perf:` line) |

## Player issues opened since round 6

Round 6 read every open issue up to #703 on `HayatoG/iss_pc`. Two more were opened on 2026-10-26.

| Issue | What it is | Code finding |
|---|---|---|
| #704 | Feature request: the right stick turns the camera too slowly on an Xbox Series controller, even with dead zone 0 and the game's aim speed at fast. | Not a code bug. `axis()` (`port/src/runtime_pad.c:85-91`) maps the full stick range linearly to 0-255, and the dead zone is the only stick setting the launcher offers (`ISS_STICK_DEADZONE`, `port/launcher/infamous_run.py:818`). A gain or response-curve setting such as `ISS_STICK_GAIN` and `ISS_STICK_CURVE` would cover it. Apply it to the right stick only, after the dead zone, with a clamp to 0-255. |
| #705 | v1.0 (`cde38af`) on an RTX 4060 Ti with renderer v2 and native passes on. No description was given. | The attached log (32 MB, cut by the launcher) shows a clean exit (`PORT_STOP`) after about 37 minutes, with no `Fault:`, `STOP:` or `FREEZE:` lines. Verbose GPU logging (`TLG_GPU_LOG=debug`) was on. The 25,000 `BindBuffers: Clamped size from 2638020712` lines are already rate-limited on `main` (`O/vk_rasterizer.cpp:4311-4319`). Nothing actionable without a description. It is useful field evidence for item 5: 0 `Perf:` lines. |

## Suggested order

1. **Before the next release:** items 1, 2 and 3. Each is a few lines, and each can otherwise turn a working install into one that crashes on every launch or in play.
2. **Diagnostics:** items 5 and 3's `samplers=` log, so the next player reports show the counters these fixes depend on.
3. **Cheap wins:** item 6 (EXTRQ default), item 9 (boot checkpoint, about 1 s per launch), item 15.
4. **Hang and crash hardening:** item 4 (backing-view reads, or the wait pump after an A/B), item 10, item 8, item 16.
5. **Tooling:** item 7 (re-check the 2026-10-26 verdicts), items 13, 17 and 18.
6. **The rest:** items 11, 12, 14, 19 and 20.
