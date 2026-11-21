# Story 04 — MOON-OS: an operating system in Qasida

Repo: [themythofsisyphusjayant/MOON-OS](https://github.com/themythofsisyphusjayant/MOON-OS).
Written 21 November 2026. Every number below comes from the repo, its README,
its phase log, or the work notes. Where no source exists, it says so.

## The problem

A language can only prove itself by what it can build. Qasida was designed
for safe, simple, fast tools. The strongest test of a language is whether it
can hold up an operating system: raw memory, interrupts, scheduling, a
filesystem, with no runtime underneath to catch mistakes.

So the OS is the test. It runs in the open.

## The goal

Set 27 October 2026: recreate something bigger than TempleOS, completely in
his own programming language. No Linux anywhere in the stack. Own kernel,
drivers, filesystem, shell, GUI, networking, built on his own CPU (big32)
and in his own language.

Order matters. First make the language capable of it, then build the OS.
That order was set on the same day as the goal.

[NEEDS JAYANT'S WORD] What "bigger than TempleOS" means in concrete terms.
More lines of code? More features? More apps? The goal as written does not
say. And why TempleOS is the benchmark at all — what about it is worth
beating — is not written down anywhere either.

## Design decisions

**Freestanding kernels, proven by transcripts.** Every kernel builds with
`qasida build --target big32 --freestanding` — no libc, own `_start` — and
proves itself the same way: by the exact UART transcript it prints on the
CPU model. Each test asserts an exact transcript. The OS proves itself by
what it prints.

**Layers, bottom to top.** big32 CPU (Python model + Verilog SOC2) → HAL
(trap handler, CSRs, `ecall`, `peek32`/`poke32`) → drivers (UART + timer,
MMIO, interrupts) → allocators → filesystem (VFS in RAM, FS v1, FS v2 +
delete) → processes and preemptive scheduler → networking (loopback, SLIP
over UART) → apps (Finder, Terminal, shell REPL, ANSI GUI windows).

**The language is the foundation.** Qasida's freestanding mode, raw memory
access, bitwise operators, CSR builtins, `ecall`, and the `trap_handler`
hook are what make a kernel in Qasida possible at all. If the language
cannot do one of those things, the OS cannot be built. That is why the
language comes first.

**JalOS stays a prototype.** `jalos.fg` (1,408 lines, ~30 shell commands,
VFS, users, cron, package manager) runs on top of a host OS. Its ideas feed
the kernel. Its code does not become kernel code. That line is written into
the README on purpose, because it is easy to blur.

**Phases, not a big bang.** Phases A–E (27 Oct 2026): bitwise ops, raw
memory, freestanding mode, CSRs + trap handler + `ecall`, model timer,
preemptive scheduler. Phase F (9 checkpoints): bump allocator, free-list
allocator, FS v1, FS v2 + delete, processes, shell REPL, ANSI GUI, drivers,
loopback networking. Then one unified kernel combining all Phase F
subsystems on one memory map — `kernel_unified.fg`, 866 lines, idle task
integrated 13 Nov 2026.

## The daily-ten cadence

On 29 October 2026 the rule became: 10+ OS items AND 10+ library/feature/fix
items per day. Two separate daily targets. Surplus on one track does not
cover a shortfall on the other. Both counts are reported separately.

That rule is the heart of how this repo grew. From 13 to 20 November 2026,
80 of the 120 native OS apps landed — ten a day, each batch pushed the same
day with its test (40 checks, 0 failures, both compilers). The 17–18 Nov
push covered two days' batches (20 apps) in one commit, `3319ee38`.

19 November: both tracks hit 10/10 and both were pushed. OS apps — dd,
nohup, sync, tar, shuf, groups, logname, unlink, renice, vmstat — live as
`c93665f8`, 40/40 on both compilers.

20 November: both tracks hit 10/10 again and both were pushed. OS apps —
dmesg, lspci, lsblk, pstree, lsof, printenv, nproc, hostid, time, size —
live as `3f131344`, 40/40 on both compilers.

The other 40 app sources predate the daily cadence: the core utilities
(`cat`, `cp`, `ls`, `ps`, `grep`, `sort`, `wc`, and the rest).

[NEEDS JAYANT'S WORD] Why ten a day, and why the two tracks are counted
separately instead of as one pool of twenty. The rule exists in the notes.
The reasoning behind the number does not.

## Failed attempts

**The first push died mid-flight.** The initial repo push on 12 November
2026 was killed at 817 blobs by a service restart. The remote stayed at its
old commit — nothing landed. It was relaunched and verified: `2721a87a`,
300 files, read-back matched by blob SHA. Lesson: a push is not done until
the read-back matches. Saying "pushed" after the command returns is not
enough.

**The idle task was missing, and sleep() lied.** The unified kernel carried
a known limitation: no idle task. `sleep()` with no ready task degraded to
an immediate resume instead of blocking. It was fixed 13 November 2026
(Slice 2): the idle task went into slot 1 (state 5), children in slot 2
only, and a new shell command `z` spawns a task that prints its start tick,
sleeps 5 ticks, then prints its wake tick. Verified: wake tick 10, start
tick 4 — 6 ticks for a 5-tick sleep. With the old immediate resume the wake
tick would have been ~4. Full standalone suite after the fix: 118 pass, 0
fail, 9 skip, 509 checks, 42.3 s.

**The lifetime allocator hit a wall.** This is the open failure, and it is
still open. The lifetime-allocator v2 slice — the next kernel slice after
the idle task — was built and then stopped honestly. The adopted kernel
builds to 4,319 words. The Python big32 model has only 4,096 words of
instruction memory. The model wraps the program counter to 0 and the kernel
restarts in a loop. A 3,983-word stub runs the full tour cleanly. Size is
the cause, not the code.

All kernel files were reverted to the verified baseline (866 lines).
Baseline tests re-verified: 0 failures. The slice did not land.

Three routes exist: a slimmer allocator, freeing room elsewhere, or the
SOC2 instruction-memory route. None is chosen yet.

[NEEDS JAYANT'S WORD] Which route. This is Jayant's call, and it is the
one decision blocking the OS chain.

**A wrong docstring, caught by the same work.** While the allocator slice
was being reverted, the unified kernel docstring was found to be wrong: it
said FS control words are `0x8060`/`0x8064`. The code uses `0x8058`/`0x805C`.
Small. Worth recording because docs that disagree with code are how the
next person wastes an afternoon.

**The SOC2 tests cannot run here.** Nine tests need a BIG-32 checkout, the
rx-proof testbenches, and `iverilog` on PATH. `iverilog` is absent from this
environment. The tests print a loud `SKIP SOC2 test ...` and exit 0 — a
skip, never a pass. The desktop kernels (interactive desktop, Finder,
Terminal) are SOC2-only, so they are currently unverified beyond the model.
That is a limit, not a result.

## Lessons

**Count tracks separately, or the easy one eats the hard one.** Ten OS
items and ten library items is not twenty items. If they were one pool, the
fast track would always fill the quota and the slow track would starve.
The separate count is the whole point of the cadence.

**A slice that does not land is still work.** The allocator slice was
reverted, but it found the real limit: 4,096 words. That number is now the
OS chain's next problem, stated exactly. Stopping honestly beats landing a
kernel that loops on restart.

**Verify by read-back, not by exit code.** The 817-blob push failure and
the daily batches both taught the same thing: the command succeeding is
not the proof. The proof is reading the remote back and matching blob SHAs.

**The test is the transcript.** Exact UART transcripts are unforgiving.
They catch regressions a unit test would miss, because the whole kernel has
to print the right thing in the right order. That is why every kernel test
asserts one.

## What changed the plan

**The cadence changed what got built.** Before 29 October, the OS grew in
big slices — phases, a unified kernel, a desktop. After 29 October, it grew
ten small apps a day. Eighty apps in eight days. The repo went from a
kernel with a shell to a kernel with a shell plus 120 Unix-style utilities,
each one verified under both compilers.

**The push grant changed the pace.** On 10 November 2026, the blanket push
grant landed: verified-green work (own test all pass, ASan clean, format
idempotent, full suite pass — or flake-only with a 4× rerun plus docs) can
be pushed freely, without asking per push. The 17–18 Nov batches went live
as `3319ee38` on Jayant's "do whatever is best" delegation under that
grant. Spending money, Kaggle runs, new repos, merges, big32 pushes, and
releases still need his word.

**The size wall changed the next slice.** The plan was: adopt the lifetime
allocator into the unified kernel. The plan now is: decide the route round
4,096 words first. Until that decision lands, the next kernel slice is
blocked — not by effort, by a number.

**The sprint week paused the OS track.** From 21 to 27 November 2026 the
work is a libs-only sprint: ≥20 verified libraries a day, OS track paused,
normal 10+10 resuming 28 November. The OS repo's last code commit is
`3f131344` (20 Nov). The pause is deliberate, not a stall.

## Where it stands — 21 November 2026

Live repo main: `01f820da` (20 Nov README pass). Last code commit:
`3f131344`, the 20 Nov daily-ten batch.

Contents, counted from the tree: 246 `.fg` files, 22,112 lines. 125 kernel
sources (18,088 lines), 120 app sources (2,616 lines), JalOS (1,408 lines),
unified kernel (866 lines). 127 kernel test files plus 8 daily app-batch
tests. Full standalone run, 13 Nov 2026: 118/118 pass, 0 fail, 509 checks,
42.3 s. That run predates the 14–20 Nov app batches; those were each
verified at push time, and a fresh full-suite run after the 20 Nov batch
has not been recorded yet.

Limits, stated plainly: not bootable on real hardware yet. No persistence —
the VFS lives in RAM. One child slot only in the unified kernel. Palette
indices only (0–9). Lifetime allocator v2 blocked on the 4,096-word wall.
Nine SOC2 tests skipped until `iverilog` is restored.

What's next: the allocator decision, then SOC2 re-verification of the
desktop kernels, then Notes as a third desktop app — one verified slice at
a time, toward the TempleOS-sized goal.

## Sources

- Live repo README, `themythofsisyphusjayant/MOON-OS`, read via GitHub
  Contents API 21 Nov 2026 (blob `ee260ccf`); live main `01f820da`, last
  code commit `3f131344`.
- Staging tree `~/workspace/os-repo-staging/` — `README.md`,
  `docs/PROGRESS.md`, `STAGING_REPORT.md` (initial push `2721a87a`,
  300 files; first attempt killed at 817 blobs).
- Work notes: daily-ten 19 Nov (`c93665f8`) and 20 Nov (`3f131344`),
  17–18 Nov batches pushed as `3319ee38`; lifetime-allocator blocker
  (4,319 vs 4,096 words; 3,983-word stub clean); blanket push grant
  10 Nov 2026; daily 10+10 cadence from 29 Oct 2026; libs-only sprint
  week 21–27 Nov 2026, OS track paused, resumes 28 Nov.
- Repo commit list, verified via GitHub API 21 Nov 2026.
