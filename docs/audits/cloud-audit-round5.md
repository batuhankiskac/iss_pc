# Cloud audit, round 5: packet semantics, memo keys, instruction rewriting and input

This is the fifth code-only pass over `main` at `fac17b4` (2026-10-26). It covers three areas that rounds 1-4 ([`cloud-audit.md`](cloud-audit.md), [`round 2`](cloud-audit-round2.md), [`round 3`](cloud-audit-round3.md), [`round 4`](cloud-audit-round4.md)) did not:

- **P:** what the command processor does with each PM4 packet (`liverpool.cpp`), and the Gnm HLE functions that build packets (`gnmdriver.cpp`).
- **K:** correctness of the memos, caches and fast paths that skip work, i.e. stale or aliased hits.
- **I:** instruction rewriting and emulation (`scripts/isa_rewrite.py`, `src/isa_*.c`), the guest/host ABI bridges, and the game hooks and input code.

Nothing from rounds 1-4 is repeated. The same rules apply:

- Nothing was run or changed.
- Every line number was read from the file.
- Items not traced end to end are marked **suspected**.

Several P items were compared with current upstream shadPS4 `main`. Where upstream has since fixed the same code, the port is running the older upstream version, and that is noted.

Paths are relative to `port/`. `LP` stands for `gpu_v2/gpu/overrides/liverpool.cpp`, and `GD` for `gpu_v2/gpu/overrides/gnmdriver.cpp`.

## Ranking (severity × likelihood)

| # | Finding | Main location | Symptom | Default path? | Fix risk |
|---|---|---|---|---|---|
| 1 | Indexed indirect draws bind an index buffer sized by the *last direct draw's* count, not by `INDEX_BUFFER_SIZE` | `LP:2955-2958`, `vk_rasterizer.cpp:3747` | missing or garbled triangles | yes | low |
| 2 | `sceGnmDrawIndirectMulti` writes a header for 5 body dwords but emits 6 | `GD:886-895` | CP abort (`UNREACHABLE`) on first use | if the game calls it | none |
| 3 | The buffer sync memo keeps returning the old arena after an arena migration | `buffer_cache.cpp:3006-3014,3391-3428` | stale reads (missing barriers), aliased sparse binding | yes (`ISS_RENDER_REUSE=1`), after a migration | low |
| 4 | Scout and native-pass recordings trust SRT snapshots that later GPU or CPU writes can make stale | `vk_pipeline_cache.cpp:1125-1134,2979,3636`, `native_passes.cpp:547-559` | wrong textures, buffers or constants; possible GPU fault | yes | medium |
| 5 | The texture memo records the generations *after* the lookup, so a fault in between is lost | `vk_rasterizer.cpp:4640-4648,4706`, `fast_draw.inl:1356` | a CPU-updated texture freezes | yes | low |
| 6 | Suspected: a relocated follower instruction may be the target of an indirect jump | `scripts/isa_rewrite.py:141-160` | crash or corruption on that code path | yes, on Intel CPUs | low |
| 7 | Suspected: `CLEAR_STATE` zeroes every register, not just the context range | `LP:2192-2196`, `shadps4/video_core/amdgpu/regs.cpp:26-27` | draws see zeroed SH/uconfig state | yes (per-frame init sequence) | low/medium |
| 8 | CP-label bookkeeping is wrong for deferred (unified-memory) fences | `LP:3105-3132,4129-4142,4336-4343` | needless re-uploads (frame time); rare stale upload | yes (`ISS_UNIFIED_MEMORY=1`) | low |
| 9 | `IT_COPY_DATA` is logged and dropped (and aborts on the compute queue) | `LP:3280-3289` | values never written; waits stall or read stale data | yes, when used | low |
| 10 | The permutation memo key hashes only the first 48 dwords of the fetch shader | `vk_pipeline_cache.cpp:3311-3321,3361-3391` | wrong vertex fetch after streaming | yes | low |
| 11 | Suspected: native passes never verify the user-data words a lane inherited | `native_passes_rasterizer.inl:95-100,1691-1700`, `native_passes.cpp:666-668` | one-frame-stale table pointers | yes | low |
| 12 | The compute-ring wrap buffer has no bounds checks | `LP:3550-3581`, `liverpool.h:305` | memory overrun or garbage packets | low | low |
| 13 | HLE edge cases: marker stubs that leave reserved dwords, `count == 0`, a missing CCB check, an unvalidated DingDong offset | `GD:1056,1362,1388,2564,2590` | replayed stale packets; out-of-bounds reads | low/unknown | low |
| 14 | Unchecked register indices and packet sizes | `LP:2198-2325,3798-3805,1640`, `pm4_cmds.h:53-55` | out-of-bounds writes in the CP | malformed streams only | low |
| 15 | Suspected: CE/DE counter handling can livelock the CP | `LP:1671,1852,3426` | CP hang | only with constant command buffers | low |
| 16 | Smaller packet gaps: `CondExec` reads a byte, `MemSemaphore` is not atomic, `ReleaseMem` aborts on two legal settings | `pm4_cmds.h:1073-1087,1296-1312,1335`, `LP:3457` | wrong branch, hang, abort | low | low |
| 17 | The AVX check ignores OSXSAVE/XCR0; AES-NI and PCLMULQDQ are not checked | `src/cpu_check.c:22-24` | `#UD` crash instead of the clean STOP message | rare machines | none |
| 18 | A NaN mouse setting passes `Clamp` and pins the right stick | `gpu/shim/tlg_settings.cpp:24-26`, `src/mouse_stick.h:57-59,77` | camera spins; value saved back to the ini | rare (hand-edited ini) | none |
| 19 | Suspected: contention on the karma `working` flag leaves sticky errors | `src/runtime_karma.c:115,155,224` | karma switcher refuses requests; ledger drifts | unknown | low |
| 20 | Direct mouse aim drops all mouse input when the camera hook is stale or rejects its limits; its timestamps can wrap | `src/runtime_mouse_aim.c:30-34,57-58`, `src/runtime_pad.c:676` | dead mouse | **no**: needs `ISS_DIRECT_MOUSE_AIM=1` | low |
| 21 | Partial re-resolve paths skip `ClampHugeRead`, and the stage-memo verify toggle never exercises them | `stage_bind_memo.inl:262-264,300`, `fast_draw.inl:1011,1060` | huge bindings (VRAM); verify blind spot | latent | low |
| 22 | Suspected: the draw-distance writer writes game data with no writability check | `src/probe.c:748-751` | access violation in a host thread | if those pages are read-only | none |
| 23 | `GetSampler` ignores the border-colour base (upstream bug) | `texture_cache.cpp:1446` | wrong border colour | rare | low |
| 24 | Suspected: rumble is dropped within 100 ms windows and capped at 1 s | `src/rumble_gate.h:29`, `src/runtime_pad.c:1042` | sustained rumble stops or stays stale | if the game sets vibration only on change | low |

Items 1, 3 and 5 are the most likely to show up on the default path as rendering bugs, and each fix is a few lines. Item 2 is a one-character fix.

---

## P. Command processor packets and the Gnm HLE

### 1. Indexed indirect draws bind an index buffer sized by the last direct draw's count

- **Location:**
  - `INDEX_BUFFER_SIZE` writes `regs.num_indices = index_size->num_indices;` (`LP:2955-2958`).
  - Every direct indexed draw also overwrites the same field, for example `DrawIndex2` (`LP:2390`), `DrawIndexOffset2` (`:2549`), `:2618`, and the native-segment commits (`:2016`, `:2091`, `:2367`, `:2601`).
  - The indexed indirect paths (`DrawIndexIndirect`, `…Multi`, `…CountMulti`) reach `BindIndexBuffer`, which sizes the binding as `regs.num_indices * index_size` (`vk_rasterizer.cpp:3747`).
- **Mechanism:** on GCN, `INDEX_BUFFER_SIZE` (the maximum index-buffer size for indirect draws) and the per-draw index count are separate state. Here they share one field. So after `setIndexCount` → a direct draw → an indexed indirect draw, the index buffer is bound and synced with the direct draw's count. Upstream shadPS4 has since split it into a separate `index_buffer_num_indices`, used as the maximum size for indexed indirect draws.
- **Symptom:** `ObtainBuffer` syncs only the short range. Indices past it are stale or out of bounds, so triangles go missing or come out wrong.
- **Fix:** store `INDEX_BUFFER_SIZE` in its own field, written only by that packet, and use it in the indirect paths.
- **How to detect:** log when an indexed indirect draw's maximum `first_index + count` exceeds the bound size. The native indirect path already reads the arguments.

### 2. `sceGnmDrawIndirectMulti` writes the wrong header size

- **Location:** `GD:886-887`: `WriteHeader<PM4ItOpcode::DrawIndirectMulti>(cmdbuf, 4, …)`, followed by six body dwords (`:890-895`, then `cmdbuf += 6`). The sister function `sceGnmDrawIndexIndirectMulti` passes `6` for the same layout (`GD:803-815`).
- **Mechanism:** the CP advances by the header's size, then parses `stride` (`sizeof(DrawIndirectArgs)` = 16) as a packet header. That is a type-0 packet, which reaches `UNREACHABLE` (`LP:1933`).
- **Symptom:** an abort on the first use. The game imports the function; whether it calls it is unknown.
- **Fix:** `WriteHeader<…>(cmdbuf, 6, …)`, as upstream now has.
- **How to detect:** a type-0 abort right after opcode 0x2C.

### 7. Suspected: `CLEAR_STATE` zeroes every register, not just the context range

- **Location:** `LP:2192-2196` calls `regs.SetDefaults()`, which does `std::memset(reg_array.data(), 0, reg_array.size() * sizeof(u32))` and then restores context defaults (`shadps4/video_core/amdgpu/regs.cpp:26-30`). No override replaces `regs.cpp`.
- **Mechanism:** the hardware CLEAR_STATE resets only the context registers (0xA000-0xA3FF) to the clear-state defaults. Here it also zeroes:
  - SH user data and shader pointers;
  - `VGT_PRIMITIVE_TYPE` (0xC242) and `num_instances` (0xC24D).
- **Trigger:** two sources of CLEAR_STATE:
  - the per-frame init sequence that `PerformSubmit` prepends (`GD:2449-2474`);
  - `sceGnmDrawInitToDefaultContextState400`, imported by the game, whose `CtxInitSequence400` starts with CLEAR_STATE (`gnmdriver_init.h:430`). It can be called mid-command-buffer.
- **Symptom:** draws that rely on persistent SH or uconfig state see zeros: a primitive type of 0, or null user-data tables.
- **Fix:** reset only the context range, plus its defaults.
- **How to detect:** count draws that see `primitive_type == 0`, or zero SH user data, right after a ClearState.

### 8. CP-label bookkeeping is wrong for deferred (unified-memory) fences

- **Location:**
  - Compute `ReleaseMem`: after the write has been deferred (`AscRelease` / `DeferUnifiedFence`), `NoteCpWrite(addr, release_mem->Address<const void*>(), …)` (`LP:4129-4142`) records whatever is in memory *now*, i.e. the old value.
  - Graphics EOP, deferred path (`LP:3105-3132`): records no label at all.
  - EOS already records the intended value (`RecordCpLabel(dst, &value, …)`, `LP:3031`).
  - `IsWrittenByCp` (`LP:4336-4343`) compares the recorded value with memory.
- **Mechanism:**
  - With a stale value recorded, a guest-CPU write equal to the old value is taken as "the CP wrote this", so it skips `NextUploadEpoch`: a stale upload, low likelihood.
  - Once the real write lands, the check says "not CP".
  - For deferred EOP, every `WaitRegMem` satisfied through `PendingUnifiedLabel` (`LP:3364-3369`), or after the write lands, counts as a guest-CPU release. Each one calls `NextUploadEpoch()` (`LP:4402-4404`), which forces hot pages to upload again.
- **Effect:** extra uploads per frame on the default path, which is the only item this round with a steady frame-time cost (not measured), plus a rare stale upload.
- **Fix:** call `RecordCpLabel(dst, &value, bytes)` with the intended value when the write is deferred. For clock values, record from inside the deferred write.
- **How to measure:** compare the `EpochCpuWaits` and `EpochCpWaitsSkipped` counters before and after.

### 9. `IT_COPY_DATA` is logged and dropped

- **Location:** `LP:3280-3289` only logs `unhandled IT_COPY_DATA src_sel=…, dst_sel=…`. On the compute queue the opcode is unknown, so it aborts.
- **Effect:** register-to-memory and memory-to-memory copies, and GPU-clock timestamps, are never written. A later `WaitRegMem` or a CPU poll on the destination can stall or read stale data.
- **Likelihood:** R1-22 named this warning as a hot log line, so the game does emit the packet. The warning already prints which `src_sel` / `dst_sel` combinations occur.
- **Fix:** implement Memory / Immediate / GpuClock → Memory / Register, starting with the combinations the log shows.

### 12. The compute-ring wrap buffer has no bounds checks

- **Location:** `LP:3550-3581`. `tmp_packet` holds 1024 dwords (`liverpool.h:305`).
- **Two problems:**
  - **Overflow.** A packet that straddles the end of the ring is copied in with `acb.size_bytes()` and no bound. A packet over 1024 dwords overruns into the next queue's state.
  - **Short continuation.** The reassembly `memcpy(tmp + tmp_dwords, acb.data(), next_dw_off * 4)` (`:3558`) is not checked against `acb.size()`. If the continuation is short, it over-reads, and the re-buffer at `:3565` then overwrites the saved header, so the next submission parses garbage.
- **Likelihood:** low; the rings mostly hold 4-dword indirect-buffer packets.
- **Related (performance only):** `BeforeAscSubmit` stops scanning at a straddled packet (`native_passes.cpp:1826`), so the next part, which starts mid-packet, is never natively recorded.
- **Fix:** check both copies against `Pm4BufferSize` and `acb.size()`, and keep accumulating rather than overwriting.

### 13. HLE edge cases

- **Stub markers leave reserved space unfilled (suspected).** `sceGnmInsertSetColorMarker` (`GD:1362`), `sceGnmInsertThreadTraceMarker` (`:1388`) and `sceGnmDrawOpaqueAuto` (`:1056`) take no arguments and return `ORBIS_OK`. All three are imported. If the guest reserved command-buffer space for them, it keeps either stale packets from an earlier frame, which are replayed, or zeros, which abort as a type-0 packet. Fix: take `(cmdbuf, size)` and fill the space with a NOP of `size` dwords.
- **`sceGnmSubmitAndFlipCommandBuffers` with `count == 0`** reads `dcb_gpu_addrs[count-1]` (`GD:2564`).
- **`sceGnmSubmitCommandBuffersForWorkload`** lacks the `ccb_sizes && !ccb_addrs` check (`GD:2590`), so `ProcessCeUpdate` dereferences null.
- **`sceGnmDingDong`** accepts `next_offs_dw >= ring_size_dw`, so the span runs past the ring.

### 14. Unchecked register indices and packet sizes

- **Register writes.** `SetConfigReg`, `SetContextReg`, `SetShReg` and `SetUconfigReg` (`LP:2198-2325`, `:3805`) index `reg_array[Offset + reg_offset]` with a 16-bit offset and up to 0x3FFF dwords, against `NumRegs = 0xD000`. So `memcmp` and `memcpy` can read and write past the array. Uconfig offsets at 0xD000 and above (performance counters) overflow too.
- **`cs_state`.** The range test uses `<=` against `0x200 + sizeof/4` (`LP:2297-2298`, `:3798-3799`), and `ASSERT(set_size <= sizeof)` ignores the start offset. A write that starts near the end runs past `cs_state` into the next members.
- **Packet length.** `NumWords()` is `(count + 1) & 0x3FFF` (`pm4_cmds.h:53-55`), so a 16 384-dword NOP advances one dword, and its embedded data is parsed as packets. The native scanners use `+ 1` with no mask, so they disagree with the CP.
- **Other reads and writes.**
  - Register-space `WaitRegMem` reads `regs[Reg()]` unchecked.
  - `WriteConstRam` writes `offset + size` into the 48 KB `constants_heap` unchecked (`LP:1640`).
- **Fix:** clamp, or skip and log, in each of these. They only fire on malformed or mis-skipped streams, but a mis-skip elsewhere (such as #2 or #13) turns into memory corruption inside the CP instead of a clean abort.

### 15. Suspected: CE/DE counters can livelock the CP

- **Mechanism, two parts:**
  - Every nested `ProcessGraphics`, i.e. every indirect buffer (`LP:3405`), runs `cblock.Reset()` (`:1852`) while the outer CE task is still live.
  - `WaitOnDeCounterDiff` tests `(de_count - ce_count) >= diff` (`:1671`), which is the inverse of the hardware rule: CE waits while `ce - de >= diff`. Upstream has since rewritten this test.
- **Trigger:** a constant command buffer, with an indirect buffer between sync points. After the reset, de > ce with diff == 1, so CE waits, while DE's `WaitOnCeCounter` loop (`:3426`) keeps resuming CE.
- **Symptom:** a CP hang.
- **Likelihood:** only if the game uses constant command buffers; no evidence was found.
- **Fix:** reset only at the top level, and use `ce - de >= diff`.

### 16. Smaller packet gaps

- **`CondExec`** reads its condition through a `bool*`, i.e. one byte (`pm4_cmds.h:1335`, `LP:3457`). The hardware tests a dword, so 0x100 reads as false. Reading a byte that is not 0 or 1 as `bool` is also undefined behaviour.
- **`MemSemaphore`:** `Decrement` and `Signal` are plain `u64` read-modify-writes (`pm4_cmds.h:1296-1312`). A guest-CPU signal at the same moment can be lost, and a later wait then hangs (suspected).
- **`ReleaseMem::SignalFence`** reaches `UNREACHABLE` for `data_sel = None` (`pm4_cmds.h:1073-1075`) and for `int_sel = IrqOnly` (`:1085-1087`). Both are legal settings; the deferred path accepts them.

---

## K. Memo and cache-key correctness

### 3. The buffer sync memo keeps returning the old arena after an arena migration

- **Location:**
  - `gpu_v2/gpu/overrides/buffer_cache.cpp:3006-3014`. On a hit, the memo returns `{memo->arena, memo->arena->Offset(device_addr)}`. It checks only the address, the size and the CPU versions of the 16 MB regions.
  - Unlike `SmallReadStamp` and the stage-memo stamp, it does not check `arena_binding_generation`.
  - The memo is on by default: `tuning.sync_memo = render_reuse` (`bridge.cpp:548`), and `ISS_RENDER_REUSE=1` (`defaults.env:106`).
- **Mechanism:** `GetArena` migration (`:3391-3428`) points both 4 GB pages at a new arena. It leaves the old `Buffer` alive and sparse-bound to the same memory. Memo entries made before the migration keep handing out the old `VkBuffer` for data whose CPU region versions never change, such as static vertex and index data.
- **Trigger:** a buffer request crosses a 4 GB boundary. The garlic heap at `0x1068000000` runs about 2.5 GB, past `0x1100000000`, so this is plausible.
- **Symptom:**
  - `Runtime::IsBufferAccessed` tracks barriers per buffer handle (`vk_runtime.cpp:871-882`). A GPU write through the new arena followed by a read through the old one gets no barrier, so the read can see stale data.
  - Both arenas are bound to the same memory without `SPARSE_ALIASED` (the flag appears nowhere in the code), which the Vulkan spec leaves undefined.
- **Fix:** store `arena_binding_generation` in `SyncMemo` and compare it, or require `memo->arena == address_space[page]`.
- **How to detect:** count hits where `memo->arena != GetArena(...)`.

### 4. Scout and native-pass recordings trust SRT snapshots that later writes can make stale

- **Location:**
  - The CP adopts the scout's `scouted->flat` snapshot in three places: `TakeScoutedStage` (`vk_pipeline_cache.cpp:2979`), `GetProgram` (`:3636`) and fast draw (`fast_draw.inl:2059,2073`). The snapshot holds descriptors *and* constants loaded through user-data pointers. The CP checks only `pgm_base`, the start bindings and the user-data registers.
  - The scout reads that memory at guest submit time. Its walker (`shadps4/video_core/amdgpu/scout.cpp`) has no case for `WAIT_REG_MEM`, and does not account for shader writes from compute or draws, on either the graphics ring or async compute.
  - For `DMA_DATA` / `WRITE_DATA`, `MayBeWrittenLater` checks only the 4 KB after each first-level user-data pointer (`vk_pipeline_cache.cpp:1125-1134`); nested tables are not checked.
  - Native passes in submit mode (the default) record at submit time. The scan treats DMA and WRITE_DATA only as segment boundaries and tracks no writes (`native_passes.cpp:547-559`). Flat data is copied at record time (`native_passes_rasterizer.inl:1195`).
- **Trigger:** a late-latched table that the CPU fills after submit, behind a label wait, or an SRT table written by compute or DMA and read by a later draw.
- **Symptom:** wrong textures, buffers or constants. A bad buffer or texture descriptor can fault the GPU.
- **Fix:**
  - Stop the scout at `WAIT_REG_MEM` and at dispatches, treating everything after as possibly written.
  - Track nested walker reads.
  - Give native passes the same written-range tracking.
- **How to detect:**
  - `ISS_NATIVE_PASSES_FLAT_CHECK=1` counts `flat_mismatch` (`=2` also rejects), but only for the **first** draw of each segment. Its own comment records a stale "late map structures" case on 2026-10-18.
  - The scout has no verify toggle. Add one that refreshes the flat buffer and compares it on a sample of draws.

### 5. The texture memo records the generations after the lookup

- **Location:** `vk_rasterizer.cpp:4640-4648`: `FindTexture` runs first, and only then are `.generation = texture_cache.Generation()` and `.image_generation = texture_cache.ImageGeneration(image_id)` stored. The native image capture does the same (`:4706`), and so does `FinishFastDraw` for `f.stamp` (`fast_draw.inl:1356`).
- **Mechanism:**
  1. A guest write fault lands between the refresh inside `FindTexture` and the memo store. The fault bumps the image generation, sets `CpuDirty` and calls `UntrackImage` (`texture_cache.cpp:394-399`).
  2. The memo then holds the *bumped* generation.
  3. Later CPU writes no longer fault, because the image is untracked.
  4. Every following memo hit skips `RefreshImage`.

  The render-target memo already does this correctly, reading the generations before the lookup (`vk_rasterizer.cpp:1050-1054`).
- **Symptom:** a texture the CPU updates repeatedly (video, UI atlas, streamed texture) freezes until some unrelated global generation bump.
- **Fix:** read both generations before `FindImage`, and store those.
- **How to detect:** a stress test that writes an image's pages in a loop on one thread while another binds it, checking that every write reaches the GPU.

### 10. The permutation memo key hashes only the first 48 dwords of the fetch shader

- **Location:** `SpecializationKey` hashes the fetch shader's address plus `min(48, room to the end of the page)` dwords of its code (`vk_pipeline_cache.cpp:3311-3321`). `VertexAttributesMatch` (`:3361-3391`) then checks the current vertex-buffer descriptors against the *cached* permutation's parsed fetch data. Upstream's `StageSpecialization::operator==` compares `fetch_shader_data`; this path never does.
- **Trigger:** level streaming loads a different fetch shader at the same address, for the same vertex-shader hash, and the two differ after dword 48. The scout uses the same key (`:1154`, `:1197`).
- **Symptom:** wrong vertex fetch: garbled geometry.
- **Fix:** hash the fetch shader up to its `s_setpc`, or memcmp the stored words on a hit.

### 11. Suspected: native passes never verify the user-data words a lane inherited

- **Location:**
  - `StateKeep` zeroes VS/PS user data (`native_passes_rasterizer.inl:95-100`).
  - The segment check compares only the user words the lane *wrote* (`:1691-1700`), unless `ISS_NATIVE_PASSES_STRICT_UD=1`.
  - `first_user` takes the lane's assumed values (`native_passes.cpp:666-668`), which come from the CP's registers where that command buffer started last frame.
- **Condition:** this is safe only if the engine rewrites every user-data register its shaders read. A comment asserts that (eboot 0x14AD0); nothing enforces it.
- **Symptom:** a one-frame-stale table pointer, giving wrong constants or textures.
- **How to detect:** run with `ISS_NATIVE_PASSES_STRICT_UD=1` and compare the `RejectUser` counts.

### 21. Partial re-resolve paths skip `ClampHugeRead`; the verify toggle never exercises them

- **Location:** `stage_bind_memo.inl:300`, `fast_draw.inl:1011` and `:1060`, and `native_gbuffer_rasterizer.inl:539` use only `ClampRangeSize`. `BindBuffers` also applies `ClampHugeRead` (`vk_rasterizer.cpp:4241`).
- **Effect:**
  - A read-only, unformatted binding over 1 GB whose descriptor moves is bound whole, which brings back the whole-heap residency problem (about 3 GB of VRAM).
  - Above 4 GB, the size is also silently truncated by `ObtainBuffer(VAddr, u32)`.
- **Why it is latent:** the one known 2.5 GB descriptor is formatted, so these paths reject it.
- **Verify gap:** with `ISS_STAGE_MEMO_VERIFY`, the replay returns at `verify_only` (`stage_bind_memo.inl:262-264`). The partial-hit re-resolution therefore never runs under verification, so changed buffers are never checked.
- **Fix:** apply `ClampHugeRead` on these paths, and let verify mode run the re-resolution and compare.

### 23. `GetSampler` ignores the border-colour base

- **Location:** `texture_cache.cpp:1446` keys samplers on `XXH3(S#) + is_depth` only. A custom border colour is read from memory at `ta_bc_base` (`shadps4/video_core/texture_cache/sampler.cpp:30`). The port's sampler memo adds `border_color_base` to *its* key, but the cache underneath still returns the first sampler created for that S#. This bug is inherited from upstream.
- **Symptom:** a wrong border colour when the same S# is used with two different `TA_BC_BASE` values. Rare.
- **Fix:** add `ta_bc_base` (or the border colour itself) to the sampler cache key.

**Hash-only keys.** The texture memo, the sampler memo, `GetSampler`, the permutation memo and all native-pass hashes compare a 64-bit XXH3 without a memcmp. With thousands of live keys the collision odds are around 1e-12 per lookup, which is negligible. The real gaps are inputs left out of the hash (#10, #23), not collisions.

---

## I. Instruction rewriting, ABI bridges, hooks and input

### 6. Suspected: a relocated follower instruction may be the target of an indirect jump

- **Location:** `scripts/isa_rewrite.py:141-160`. A 4-byte site (e.g. EXTRQ) becomes a 5-byte `jmp` that also covers the next instruction, which is copied into the stub (`MODE_JMP_RELOCATE`). The script refuses this only when the next instruction is a target of a *direct* branch in the same segment:
  ```python
  if ins.flow_control in (... BRANCH, CALL) and ins.op0_kind == OpKind.NEAR_BRANCH64:
      targets.add(ins.near_branch_target)
  ...
  if ip not in targets:
      site.mode, site.relocated = MODE_JMP_RELOCATE, relocated
  ```
- **What it misses:**
  - switch jump-table cases (`red_zone.py` resolves those, but this script does not use them);
  - addresses taken through relocations (function pointers, vtables);
  - exception landing pads;
  - branches from other segments.

  Sites are also found by a straight linear decode of the whole executable segment, so data inside a code segment could decode as a false site (lower risk).
- **Trigger:** an indirect jump to `site + 4` lands on the last byte of the rel32 and the NOP padding, and runs garbage bytes.
- **Symptom:** a crash or silent memory corruption, only on that code path.
- **Likelihood:** this is on the default path on Intel CPUs. `docs/isa_rewrite.md` lists 73 relocated 4-byte EXTRQ sites, all in one routine group (0x890a77-0x89475c). Each follower is either safe or not, so it can be checked once, offline.
- **Fix:** add to `targets`:
  - `red_zone.py`'s per-function jump-table targets;
  - every relocation value that points into code;
  - the function starts from `.eh_frame_hdr`;
  - the LSDA landing pads.
- **How to check:** an offline script that intersects the 73 follower addresses with those sets. Any hit becomes an INT3 or plain-JMP site.

### 17. The AVX check ignores OSXSAVE/XCR0; AES-NI and PCLMULQDQ are not checked

- **Location:** `src/cpu_check.c:22-24`: `{"AVX", LEAF1, 1u << 28, 0}` tests only the CPUID bit.
- **Mechanism:** OSXSAVE (CPUID.1:ECX bit 27) and `XGETBV(0) & 6 == 6` are not checked, so an OS or VM with XSAVE disabled passes the check. Examples are Linux booted with `noxsave`, or Windows with `xsavedisable`. The F16C stub then uses `vmovdqu ymm`.
- **Symptom:** an unexplained `#UD` crash instead of the clean STOP message.
- **Related, suspected:** AES-NI and PCLMULQDQ, which Jaguar has, are not checked at all. Sandy Bridge and Ivy Bridge Core i3 parts have AVX but no AES-NI, and those are exactly the CPUs the rewrite targets. Scan the image for AESENC and PCLMULQDQ; if any are present, add them to `fatal_sets`.
- **Fix:** the standard OSXSAVE + XCR0 test.

### 18. A NaN mouse setting passes `Clamp` and pins the right stick

- **Location:** `gpu/shim/tlg_settings.cpp:24-26`: `return std::clamp(v, lo, hi);`. NaN passes through `std::clamp`, and `atof("nan")` produces it. Renderer v2 builds this file too (`gpu_v2/CMakeLists.txt:179`).
- **Mechanism:**
  - With `mouse_smoothing = nan`, `k` is NaN in `mouse_stick_update` (`src/mouse_stick.h:57-59`), so `out` stays NaN for good.
  - `mouse_stick_add` (`:77`) then casts NaN to `uint8_t`, which gives 0 on x86 (undefined behaviour in C), so the right stick is pinned and the camera spins.
  - `Save()` writes the NaN back to the ini.
- **Fix:** reject non-finite values in `Clamp`, and in `mouse_stick_update` reset `out` and `target` if they are not finite.
- **Test:** `MouseStickParams p = {1, NAN, 0, 0, 0, 1}`; after one update, `out` is NaN.

### 19. Suspected: contention on the karma `working` flag leaves sticky errors

- **Location:** `src/runtime_karma.c:224` (the tick holds `working` every frame, including the C++ store mutex inside `iss_karma_binding_poll`), `:155` and `:115`.
- **Mechanism:** a shard or purchase event on another thread that finds the flag held sets `shard_error = 1` or `rollback_error = 1`. These stay set until a reload or a capture, and the transaction is never recorded.
- **Symptom:** the karma switcher refuses requests (failure 6 or 7), and its ledger drifts.
- **Fix:** wait briefly on the flag, or queue the event, instead of poisoning the state.
- **Repro:** hold `working` in a test and call `runtime_karma_shard_event` from a second thread.

### 20. Direct mouse aim drops all input when the camera hook is stale; its timestamps can wrap (opt-in)

- **Scope:** only with `ISS_DIRECT_MOUSE_AIM=1` (`src/probe.c:994-995`); off by default.
- **Dead mouse:** once the hook is attached, `runtime_mouse_aim_motion` returns 1 even when the camera hook has not run for 250 ms: `if (!active || !last || now - last > 250000u) {…; return 1;}` (`src/runtime_mouse_aim.c:34`). The pad side (`src/runtime_pad.c:676`) then skips the mouse-to-stick fallback. The same happens when `iss_mouse_apply` rejects limits outside ±3.2 (`src/iss_mouse_aim.h:37`), although the comment there says the live context uses [-10, 10]. Fix: return 0 (use the stick fallback) when the camera is stale, and fall back after N rejections.
- **Timestamp wrap:** `now` is read before the other thread's stamp (`:30-34`, `:57-58`). If that stamp is newer, `now - last` wraps to a huge value and is treated as stale, so motion is cleared or a frame of motion is dropped and follow is released. Fix: use a signed difference. It can be reproduced with `ISS_MOUSE_AIM_TEST_CLOCK`.

### 22. Suspected: the draw-distance writer writes game data with no writability check

- **Location:** `src/probe.c:748-751` checks only `mapped(…)`. The writer thread (`tlg_view.cpp:36-42`, `:67`) writes every 100 ms. `install_iss_mouse_aim`, by contrast, checks that its data address is in a writable segment.
- **Risk:** `kViewLodTable` (0xda08c0) and `kViewBands` sit near the vtables (0xd795d0). If those pages are read-only, the first change of LOD or clip distance in the menu ends the process with an access violation in a host thread.
- **Fix:** require a segment with `flags & 2`, as the mouse installer does.

### 24. Suspected: rumble is dropped within 100 ms windows and capped at 1 s

- **Location:** `src/rumble_gate.h:29` drops a change that arrives within 100 ms of the last one until the game calls again. `src/runtime_pad.c:1042` asks SDL for 1000 ms on every command.
- **Mechanism:** PS4 vibration lasts until it is changed. If the game sets vibration only when it changes, sustained rumble stops after 1 s, or stays at a stale strength.
- **Fix:** keep the dropped value and send it from a timer, or renew it from the pad-read path.

---

## Checked, no issue found

- **Gnm packet sizes:** these match the bytes they write:
  - draws: `DrawIndex`, `DrawIndexAuto`, `DrawIndexOffset`, `DrawIndirect`, `DrawIndexIndirect`, `DrawIndexIndirectMulti`, `DrawIndexIndirectCountMulti`;
  - dispatches: `DispatchDirect`, `DispatchIndirect`, `DispatchIndirectOnMec`;
  - `ComputeWaitOnAddress`, `InsertWaitFlipDone`, and the Push/Pop/Set/DingDong markers;
  - every `Set*Shader` / `Update*Shader`;
  - the init sequences, padded to exactly 0x100 dwords.
- **CP packet details:**
  - `WriteData` (`count - 2`) and the `Set*Reg` payload size (`count - 1`) are right;
  - the DMA and draw-batch advances are right;
  - the DingDong ring split and `asc_next_offs_dw` resets are right;
  - `asc_queues` never reallocates, so the CP's `queue&` references stay valid;
  - `IsAlwaysSatisfied` math is right;
  - deferred EOS records the intended label value.
- **Memos:**
  - The stage memo memcmps V#/T#/S#, bindings, `ta_bc_base` and clip enable. Its stamp is taken before and compared after the lookup, and `LOOSE` is safe because every `bind_epoch` bump also bumps another stamp field.
  - The pipeline key memo compares the full key.
  - The permutation-memo seqlock is correctly ordered.
  - The shader disk cache compares the whole `Profile` plus version bits.
  - Speculative translation memcmps the runtime info.
  - The render-target and depth memos read generations first.
  - `ExpandedMask` uses hash plus memcmp.
  - The native compute key includes start offsets and the program address.
  - The small read memo is off by default and fully stamped.
- **Instruction rewriting and emulation:**
  - EXTRQ/INSERTQ: immediate and register control, length 0 = 64, index + length > 64 truncated, the high qword zeroed, and the inline sequence equivalent to the C path.
  - LZCNT, TZCNT, ANDN, BEXTR, BLSI, BLSMSK, BLSR, MOVBE: flags, partial-register writes and operand order.
  - F16C rounding, denormals and NaN payloads.
  - Stubs save flags first, step over the red zone, keep 16-byte alignment, encode prefixes correctly and never relocate RIP-relative instructions.
  - The int3 path advances RIP on both OSes, and the Linux handler sets XSTATE_BV.
- **ABI bridges:**
  - the karma, HUD and mouse bridges save all GPRs, flags, 16 YMM registers and FXSAVE state, and reserve shadow space;
  - guest function-pointer types are `sysv_abi`;
  - no exported function is variadic;
  - `tlg_setjmp` / `tlg_longjmp` save xmm6-15, MXCSR and the x87 control word.
- **Input:**
  - pad sampling is under one mutex;
  - the motor order is correct;
  - hooks are installed before game threads start and are guarded by byte signatures.

## Suggested order

1. **One-line fixes with clear wins:**
   - #2: header size `6`;
   - #1: a separate `INDEX_BUFFER_SIZE` field;
   - #5: read the generations before the lookup;
   - #3: compare `arena_binding_generation` in the sync memo.
2. **Offline check:** #6, intersecting the 73 relocated followers with jump-table, relocation and landing-pad targets. If none hit, the item closes with no code change.
3. **Measure, then fix:**
   - #8: compare the `EpochCpuWaits` counters, then record the intended label values;
   - #9: read the `COPY_DATA` warning's `src_sel` / `dst_sel` combinations, then implement those;
   - #4: run `ISS_NATIVE_PASSES_FLAT_CHECK=1` on a long route, and add a scout verify.
4. **Then:**
   - #7 (context-only CLEAR_STATE);
   - #10 (fetch-shader key);
   - the bounds checks: #12, #13, #14;
   - the small items: #16 to #24.
