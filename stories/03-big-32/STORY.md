# Story 03 — big32: The Bug That Was Real, and the Stall That Wasn't Its Fault

Repo: `themythofsisyphusjayant/BIG-32`. Local tree: `~/workspace/big32/`.
Linux 6.6.52, rv32, on a from-scratch RISC-V CPU in Verilog. Simulation only, Icarus Verilog 12.0. No silicon.

## The problem

By Step 31, Linux booted to a working shell on big32. Steps 26–31 are proven in the README. But every one of those boots cheated a little. The testbench loaded the kernel image straight into RAM with `+KIMGBIN`. The real path is different. The bootloader is supposed to read the kernel off the virtio disk by itself, then drop to it in S-mode.

Step 34 was that real path. No shortcut. Bootloader loads the 34 MB kernel from the virtio disk. Kernel boots. Prompt.

The bootloader goal is done. The full-boot goal is not. This story is about why.

## Design decisions

Three decisions shaped Step 34.

First, virtio-mmio block, not a custom device. A custom device would need a custom kernel driver. Virtio block is in mainline Linux. The bootloader already spoke virtio. The kernel already had the driver. That was the whole bet: use the standard path and let the kernel do the rest.

Second, checkpoints. Icarus runs at about 0.15 ms per cycle. A full boot needs hundreds of millions of cycles. No single session survives that. The runner banks a checkpoint every 5M cycles, about every 11 minutes. A VM restart costs one partial segment, never the run. This mattered. The re-boot survived 19 VM restarts.

Third, the disk layout. The BIMG header sits in the ext4 1 KiB padding, sector 0. The superblock is still at byte 1024. The ext4 magic still verifies. The kernel image starts at sector 2048. Nothing about the disk was improvised. That matters later, because when the boot stalled, the disk was one of the first suspects. It was innocent.

## Failed attempt one: the boot that died at 0.52 seconds

The first Step 34 boot stalled almost immediately. Kernel time froze at 0.527s. One Oops: `load address misaligned`, in `__riscv_copy_words_unaligned`. No kernel output after that, across 300M cycles.

The cause was a single CSR value. The bootloader set `medeleg = 0xB9FF`. That delegates misaligned traps to S-mode. But this kernel has no S-mode misaligned emulation. There is no `CONFIG_RISCV_M_MODE` in the config. There is no `handle_misaligned_load` in System.map. So the first deliberately misaligned access in `check_unaligned_access` died on the first S-mode misaligned trap. The boot could never pass 0.52s.

The SBI firmware had already solved this in Attempt 12. It uses `medeleg = 0xB9AF`, with bits 4 and 6 cleared, and M-mode misalign_entry emulation. The bootloader had simply not carried that fix over. Its header even claimed the emulation was something "no bootloader payload needs."

The fix was bootloader-only. Change medeleg to 0xB9AF. Port the misalign_entry emulation from the firmware into the bootloader: 666 lines, 86 labels, zero collisions. The bootloader went from 472 lines to 1,161.

The probe passed. 70M cycles. Zero Oops. Zero `!`. Kernel time past 1s. The unaligned check printed its ratio line. The fix was proven.

Then the full re-boot launched. It ran 43 hours and 18 minutes. 178 segments. It never reached the prompt.

## Failed attempt two: the stall after the I/O scheduler

The last kernel line was `[ 39.359726] io scheduler bfq registered`, at 450M cycles. After that, nothing. Not a panic. Not an Oops. The kernel was alive and ticking. Jiffies advanced steadily from 450M to 830M. The program counter samples sat in the idle path. Kernel time reached about 77 seconds. Attempt 17 had passed this same window in about 10 seconds: bfq at 39.36s, serial console at 44.74s, virtio block at 45.56s, root mounted at 49.86s, `/init` running at 50.14s.

The first hypothesis wrote itself. The bootloader does 67,192 virtio transactions to load the kernel. The used index is 16-bit. It wraps at 65,536. The RTL reset does not clear it. So the kernel's first block I/O never completes, and rootwait never fires. Clean story. Wrong story.

It was wrong in a way that took a full day of read-only forensics to prove.

## The investigation: the hypothesis splits in two

On 18 Nov, the 835M checkpoint was read, word by word. No simulation was launched. Nothing was modified. The verdict split the hypothesis in half.

Part (a) was proven. The virtio device really was broken. At 835M cycles, `v_used_idx` was 1,533,989 and still growing, about 541 cycles per chain. The avail index was frozen at 1657, which is exactly 67,193 mod 65,536. The device was stuck mid-copy of the last kernel sector, sector 69239, re-processing it forever. The mechanism was precise: `v_last_avail` and `v_used_idx` are 32-bit counters. `v_avail_idx` is a 16-bit ring index. Once the total chain count passed 65,536, the two could never agree again, so the idle state kept re-entering the read-avail state after every completion. A phantom chain, forever. The bootloader's own comment acknowledged the wrap for its own 16-bit check. It did not consider the device's re-trigger.

Part (b) was refuted. The virtio runaway was not the stall. The kernel never touched the virtio device at all. No Status=0 reset ever reached it. The queue addresses were still the bootloader's. The avail index was never rewritten. There were zero virtio lines in the log. The proof is a single number: after a reset, the used index could never exceed the guest's avail index of 1657. It was 822,344 at 450M. So no reset happened. So the kernel never reached the virtio probe. The virtio device could not stall the CPU anyway. Its interrupt enables were 0 in both M and S mode, so its permanent interrupt was delivered nowhere. Its RAM writes do not arbitrate against CPU reads.

The kernel was stuck somewhere else entirely. The live stack said where. `kernel_init`, PID 1, asleep in `schedule()`, reached through `klist_remove()` inside `jh7110_pll_driver_init`, device-initcall entry [102]. The StarFive JH7110 PLL clock driver. A driver with no device-tree node on big32.

The stack was read live, not guessed. The task struct at physical address 0x82838000, comm `swapper/0`, thread stack pointer 0xc2437de0, mapped through the Sv32 page tables from the satp root. The return chain was unbroken: `klist_remove+0xb0`, `schedule+0x60`, `bus_remove_driver+0x68`, `driver_unregister+0x38`, `__platform_driver_probe+0xc8`, `jh7110_pll_driver_init+0x2c`, `do_one_initcall+0xa0`. Disassembly confirmed `klist_remove+0xb0` is the return address after the `schedule()` call in the wait loop. The waiter had `woken = 0`. It was still waiting.

The driver was doing nothing wrong by its own lights. It registered. It found no device, because big32 has no JH7110 PLL in its device tree. It unregistered. The unregister path went through `klist_remove`. And there it slept.

Attempt 17 passed the same initcall. Same driver, same path, no hang. So the hang is timing or state dependent, not systematic per driver. That is the uncomfortable part. It means the next probe-only driver in the list could hang the same way. The probe will tell me. It has not run yet.

## The one open question

Who holds the extra reference that keeps `klist_remove` waiting? The waiter list is empty at every checkpoint from 100M to 835M. The waiter's own next and prev pointers still point at the head. Its node memory reads as all zeros at 835M. The driver private pointer is NULL. So the refcount could not be read directly.

The leading guesses are unproven: a klist iterator that took a reference and never dropped it, or a timing-dependent race that attempt 17 won and attempt 19 lost. Resolving it needs a live probe with `initcall_debug`, or a checkpoint taken while the PLL initcall is actually running, before the node memory is zeroed. The 835M checkpoint is too late for that read.

This does not block the fix. Disabling the drivers removes the hanging path. But it is honest to say the underlying behaviour is unexplained, and the probe is the test that decides whether the problem was one driver or something systematic in the driver core on this platform.

## The fix, in two parts

I approved it on 18 Nov: "Apply both parts and run the probe."

Part A fixes the real virtio bug, even though it was not the stall. Three changes. The RTL idle trigger compares 16-bit to 16-bit, `v_last_avail[15:0] != v_avail_idx`, so the device processes exactly one phantom chain after the wrap and then goes idle. A Status=0 device reset now also clears `v_avail_idx`, which the hardware reset already did but the device reset did not. The bootloader quiesces the device before handoff: Status=0, then QueueReady=0. All three are complementary. The bootloader quiesce alone stops the runaway at handoff. The RTL changes make the device correct even if a future loader leaves it mid-chain.

Part B fixes the actual stall. The kernel config had ten StarFive JH7110 drivers enabled: six clock drivers, three pinctrl drivers, one reset driver. None of them has a device-tree node on big32. They are dead weight, and each one runs the same register, find-nothing, unregister path that hung at entry [102]. All ten are off now. The live hardware is untouched: SiFive clocks, the PLIC, the 8250 serial.

The rebuild landed the same evening. The kernel was rebuilt with the JH7110 drivers cut: 34,401,148 bytes. The disk image was rebuilt and verified: 35,449,856 bytes, ext4 magic intact. The DTB was rebuilt with `initcall_debug` added to the bootargs, so if the probe hangs again, the log names each initcall and the next hang is pinpointed immediately. The bootloader was rebuilt at 901 words. The simulation was recompiled from the fixed RTL with the new disk size. `boot_check.py` passed.

Two rebuilds were lost to VM restarts along the way. The cross-compiler was wiped twice in one day and restored twice. A missing `bc` broke one build before it started. None of that is in the story's main line. It is just what the work cost.

## What changed the plan

Four things changed.

First, the estimate. The investigation file said the re-test probe to 500M cycles would take about 2 hours. That was wrong. The real pace is about ten and a half minutes per 5M segment. 500M cycles is 18 to 20 hours of wall time. Two hours would only cover about 50M cycles, short of the 450M stall point. I was given the corrected number and asked to choose: proceed or hold. [I still need to decide this — no reply to the proceed-or-hold choice is recorded, and the probe has not launched.]

Second, the test. The probe's pass criteria are written down before it runs. Kernel time past 40 seconds. The serial8250 and ttyS0 lines, entry [124]. Then virtio block, `[vda]`, Legacy PMU, VFS root. The falsifier is written down too. If the log stalls again after bfq with no serial line, Part B did not help, the hang moved to the next probe-only driver, and the root cause is systematic. That is a result, not a failure. It would point at a kernel driver-core investigation.

Third, the load check. `boot_check.py` passed after the rebuild. The longer load check, `bootload_check.py`, was running after that. [I still need to check this — its result is not in the record, and the probe is only ready once it goes green.]

Fourth, the checkpoints. The 835M checkpoint set is preserved as evidence. The old checkpoint sets are not resume points. The attempt-18 set and the attempt-17 sets carry the wrong disk or the wrong state, and resuming one would silently replay a dead run. Do not resume old checkpoints.

## Lessons

Write the falsifier before the run. The virtio hypothesis was plausible, detailed, and wrong about the one thing that mattered. The number that killed it, `v_used_idx` 822,344 against an avail index of 1657, was already in the checkpoint before the investigation began. A hypothesis that cannot name the number that would kill it is a story, not a diagnosis.

A real bug and the actual cause can coexist and be unrelated. The virtio runaway was proven, line by line, from the RTL state. It was also irrelevant to the stall. Fixing it was still correct. Shipping a device that re-processes a sector forever is not acceptable just because the CPU does not notice. But fixing it would not have unblocked the boot. Both parts of the fix were needed, for different reasons. That is why the fix has two parts.

A driver with no hardware can still hang a boot. The JH7110 drivers do nothing on big32. They register, find nothing, and unregister. The hang lives in the unregister path, in generic driver-core code, not in any StarFive logic. Deleting dead config is not cleanup. Here it was the fix.

Attempt 17 passing is evidence, not a contradiction. The same initcall passed once and hung once. Timing-dependent hangs do not care that the code is deterministic. The kernel is deterministic. The state it runs in is not the same every time.

Estimates are claims. The re-boot was estimated at 20 to 30 hours. It took 43 hours and 18 minutes to reach 830M and stall. The probe was estimated at 2 hours. It is 18 to 20. Both estimates came from the same mistake: dividing cycles by a nominal rate instead of measuring the segment pace. Measure the pace first. Then estimate.

## Status

Step 34's bootloader goal is done and verified: the bootloader loads the kernel image from the virtio disk by itself, with no testbench shortcut. The full-boot goal is blocked on the probe. The fix is applied and rebuilt. The probe waits on my call. The one open question, who holds the extra klist reference, is still open.

## Sources

- `~/workspace/big32/LINUX_BOOT.md`, Step 34 section (from line 7286): first boot stall, medeleg diagnosis and fix, probe PASS at 70M, re-boot timeline, stall evidence, investigation summary.
- `~/workspace/big32/work/step34_virtio_reset_investigation.txt` (18 Nov 2026, 436 lines): the split verdict, the live stack, the virtio device state, the RTL reset behaviour, the proposed two-part fix, the open question.
- `~/workspace/big32/work/step34_fix2_backups/`: pre-edit backups of the kernel `.config` (all ten JH7110 drivers `=y`), the DTB (1,486 bytes), `big32_soc2.v`, and `bootloader.s`.
- Live tree check, 21 Nov 2026: the rebuilt kernel `.config` has JH7110 PLL/SYS clocks and SYS/AON pinctrl `not set`; the remaining JH7110 clock, pinctrl, and reset options are absent entirely.
- `~/MEMORY.md`, big32 section, and the 18 Nov 2026 daily log: fix approval, rebuild numbers (kernel 34,401,148 bytes, disk 35,449,856 bytes, DTB 1,502 bytes with `initcall_debug`, bootloader 901 words), `boot_check.py` PASS, the probe-time correction (18–20h, not ~2h), probe awaiting my call.
- Live repo README, `themythofsisyphusjayant/BIG-32` (Contents API, 21 Nov 2026): Phase 5 done, Steps 26–31 proven, Step 34 not yet in the boot-results table.
