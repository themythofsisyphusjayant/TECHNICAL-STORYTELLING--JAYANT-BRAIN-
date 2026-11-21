# Story 02 — Qasida: a programming language built problem-first

**Status as of 21 Nov 2026.** Repo: `themythofsisyphusjayant/RED-MOON-`. Local tree: `~/workspace/forge-v2/`. Version 2.3.0, released 16 Nov 2026. Every number below comes from the repo, the local tree, or the project record. Where the record ends, the text says so.

## 1. The problem

Most languages force a trade at the boundary between tools and machine learning. You write the fast part in a systems language and the model part in Python. Then you glue them together across a process boundary. The glue is where the bugs, the latency, and the deployment pain live.

Qasida's bet is that one language can cover both sides of that boundary. Checked arithmetic and real tensors in the same program. One native binary. No runtime to install.

That bet was not the starting point. The starting point was a list of problems.

On 26 Oct 2026, the work stopped and restarted. My instruction that day was: do not stop until this is a complete production language like Rust or Go, restart everything if work stops, and first check what people are actually building right now — Reddit and everywhere else, plus papers — and take what is worth taking from them.

So v0.1 was archived untouched as reference, and v2 started clean. Every v2 design choice had to be justified by either a named problem in v0.1's `PROBLEMS.md`, or a finding from the research pass. The research report was 66 KB, with sources. It is in `research_notes/forge-new-languages-research-20261026-0521/report.md`.

`PROBLEMS.md` names nine problems:

1. **Memory unsafety** (C, C++). Buffer overflows, use-after-free, double-free. Rust fixes this with a borrow checker that is hard to learn. Go fixes it with a garbage collector and gives up low-level control.
2. **Null, the billion-dollar mistake.** A variable that "might be nothing" crashes far from the code that created it.
3. **Type errors found at runtime, not at compile time** (Python, JavaScript). Implicit coercion makes it worse: `"5" + 3` is `"53"`.
4. **Silent arithmetic** (C, JavaScript). Overflow wraps silently or is undefined behaviour. Division by zero kills the process with a signal and no readable message.
5. **Errors that are easy to ignore or hard to see** (C, Go, Java, C++). Error codes get ignored. Go's `if err != nil` is verbose. Exceptions are invisible control flow.
6. **The complexity tax** (Rust, C++). Powerful, but beginners spend more time fighting the language than solving the problem.
7. **The speed-versus-simplicity split** (Python vs C). The easy language and the fast language are two different languages, so developers maintain two codebases.
8. **Build and deployment friction** (C/C++, Python, JavaScript). Headers, build files, virtual environments, dependency trees.
9. **Shared mutable state and concurrency bugs.** Races that appear only under load and cannot be reproduced.

The research pass added the finding that now locks the whole design: no single language has replaced Rust, Go, or C. The ones gaining real production use have a narrow beachhead, not a promise to be good at everything. Zig has the C toolchain and cross-compilation. Mojo has AI kernels. Gleam has BEAM services. Odin has games. Hare has Unix tools.

Qasida's beachhead is safe, simple, fast tools, CLIs, and small ML programs. That is the whole positioning. It is Go's niche with Rust's lessons applied: static types, no null, immutable by default, checked arithmetic, and eventually automatic memory management — instead of a borrow checker. That is a trade, stated in the document itself: easier to learn and faster to write, at the cost of a small runtime and no bare-metal, allocation-free guarantee.

What Qasida is not: a Rust or Go replacement, a kernel language, a browser language, or an ML ecosystem rival. The scope is deliberate.

[I still need to write this part — why I started building a language in the first place, in my own words] — No source in the repo or the record says why I started building a language in the first place. The 26 Oct instruction and the research are recorded. The personal reason is not. That gap is left here rather than filled in.

## 2. Design decisions

Four rules carry the design. They are in the README, and they have not changed since the restart:

- **Safe by default.** Checked integer arithmetic and bounds-checked indexing. A failure aborts with a readable message instead of wrapping or crashing. Fast mode is an explicit opt-in for hot code already validated in safe mode.
- **No null, no coercion.** There is no null value and no implicit conversion. Casts are explicit (`x as float`). If a program compiles, its types mean what they say.
- **Immutable by default.** Bindings are immutable unless declared `mut`. Function parameters are immutable. Mutation is visible in the source, not hidden in a call.
- **Small to learn, complete to use.** One syntax for tools and ML: structs, arrays, and string tools next to tensors, autograd, and a transformer stack — all in the standard library, not an add-on.

Two decisions inside those rules matter more than the rest, because they are the ones that cost the most later.

First, the safety model is Go-like, not Rust-like. No borrow checker. The bet is that checked arithmetic, bounds checks, immutable-by-default bindings, and a future garbage collector buy most of Rust's safety at a fraction of its learning cost. The price is honest: Qasida will never be right for a kernel or a no-allocation embedded loop. That is Rust and C territory, and the document says so.

Second, the toolchain comes first, not last. The research found that Gleam is production-viable despite a small community because it shipped a formatter, a package manager, a test runner, and an LSP early. I copied that lesson. By 26 Oct, the same day as the restart, the tree already had `qasida pkg` (init/add/build/run/test/list, vendored dependencies), `qasida fmt` (idempotent), `qasida lsp`, `qasida repl`, `qasida test`, and a VS Code extension. The registry, the package manager, and the editor support were not deferred to "later". Later never comes for those things — Carbon, four years in, was still designing safety; Vale and Roc were still pre-1.0 after eight-plus years. That is the failure mode the restart was built to avoid.

The language compiles through a conventional pipeline — lexer, parser, type checker, C codegen, `cc` — with four backends: native C, big32 RISC-V assembly, WASM for the browser playground, and CUDA emit. The compiler itself is written in Qasida. That is the self-hosting story, and it is where the project nearly broke its own rule about verification.

## 3. Self-hosting: the compiler that compiles itself

On 11 Nov 2026 the hosted compiler — the compiler written in Qasida — became the default. The Python implementation stays as a fallback (`forge.py --python`, or `QASIDA_COMPILER=python`) and as the reference for parity checks.

The rule that governs this is simple: the two compilers are kept in lockstep. A hosted emit that disagrees with the Python emit is a bug, not a difference of opinion. Every release fix is verified against the full bar: bootstrap 7/7 byte-identical, corpus 99/99, regression 62/62, fuzz in both modes, parity checks, and the full suite passing under both compilers.

Self-hosting changed what a compiler change costs. Every fix after 11 Nov had to be implemented twice — once in the Python compiler, once in the hosted Qasida sources (`examples/selfhost_*.fg`) — and verified under both. That doubled the work on Fix 1 and Fix 2. It also caught real bugs: a hosted emit that disagrees with Python is a bug, and the bar treats it that way.

The rename happened the same week. On 7 Nov 2026, I renamed "Jal Pari" to Qasida — Arabic for ode or poem, the same word as "Kavya" or poetry. 227 files were renamed in the tree, zero old-name mentions left in any text file, verified by grep. The dedication file and a personal file were removed from the tree and backed up elsewhere. The technical names did not change: `forge.py`, `forge_lang`, `.fg` files, and the repo name RED-MOON- all stayed, so existing programs keep working.

## 4. Compile speed: Steps 2–6, and the cache that lies

Version 2.2.0, released 12 Nov 2026, was the compiler speed program. Six steps, each verified to the full bar:

- Step 2: a real hash map with integer node-ID keys in inference.
- Step 3: no-copy structs in codegen.
- Step 4: hash-indexed codegen tables, plus a stack-overflow fix — the codegen state moved into one heap array.
- Step 5: each import lexed and parsed exactly once.
- Step 6: incremental compilation, Slice A — a whole-program emitted-C cache.

The measured result at release: cold-compile speed versus 2.1.0 improved 6.77x on a medium program, 12.28x on the 5,254-line checker, and 16.42x on the 6,898-line codegen. Warm re-compiles became effectively instant — about 0.1 seconds — because the Step 6 cache stores emitted C under `/tmp/qasida_cache/`, keyed on program and dependency hashes.

Then the cache bit. The Step-6 emitted-C cache can serve stale C across compiler rebuilds. The fix is blunt: clear `/tmp/qasida_cache/` after every rebuild. That gotcha is now written into the README's known limits and into the pipeline rules, because it was hit more than once.

And then the 2.2.0 numbers themselves were reversed. On 20 Nov 2026, a full re-test (run dir `blackmoon-iso/quant/run-20261120-24320/`, cache cleared first, hosted binary verified as 2.3.0) found the hosted codegen-wrap compile had regressed 34%: 26.637 seconds at release, about 35.8 seconds on re-test, consistent across four repetitions. The medium program and the checker held. Worse, the Python baseline the 2.2.0 speedups were measured against had gone stale — Python is now faster than the hosted binary on the two large emits (codegen-wrap 6.96 seconds Python versus 35.8 hosted; check-wrap 2.0 versus 6.24). The recorded 6.77x / 12.28x / 16.42x hosted advantage no longer holds, and the README no longer quotes it as current. The regression is still under investigation as of 21 Nov. It was not fixed by claiming it.

That is the pattern of this project: a number is published, the number is re-measured, and when the re-measurement disagrees, the re-measurement wins — even when it makes the release look worse.

## 5. The quant-latency phase: three fixes, one falsified idea, one stale note

The quant-latency phase asked a narrow question: how close can safe-mode Qasida get to C on the kind of code a trading system runs? Order-book updates, tick processing, Black–Scholes, Monte Carlo, matrix multiply, allocation.

The baseline, measured 12 Nov (median microseconds per operation, Qasida safe versus C `-O3`): order-book 1.59x, tick 3.45x, Black–Scholes a tie, Monte Carlo a tie, matmul 2.0x, array push 1.14x. Two gaps mattered. Tick was 3.45x slower. Matmul was 2x slower. Everything else was a tie or noise.

### Fix 1: one checked add

On 21 Nov at about 00:20 IST, I said: "Make our programming language as fast as c++." Then: "Do it."

The diagnosis came first, and it was short. C++ is approximately C within about 2% on every benchmark. There is no separate C++ gap; fast mode already ties or beats C++-native everywhere. The safe-mode gaps had proven causes: tick at 3.53x was ONE `forge_add` site — the `up` accumulator — proven by ablation in both directions, swapping that one call moves the whole gap. Matmul at 2.04x was buffer provenance; the checks cost approximately zero, confirmed a third time. Order-book at 1.11x was noise, with no check gap left.

Fix 1 was overflow narrowing for loop-bounded accumulators. In a counted `while` loop with trip bound T, an accumulator incremented by exactly one non-nested `acc = acc + 1` satisfies `acc ≤ E_hi + T`. If `E_hi + T ≤ INT64_MAX`, the `+1` drops to a plain `+`. The proof lives in `loopproof.py`, with a hosted mirror in `selfhost_loopproof.fg`. The gates are narrow on purpose: counted While only, increment of exactly one, no continue, no spawn, no defer, entry known and not stale, accumulator distinct from the counter, loop-top interval capped at 2^40. No codegen change.

It landed 21 Nov at about 02:40 IST as commit `4ec9a8fe`. Headline, median of five runs: tick safe 8,576 → 2,348 microseconds — 0.97x versus C `-O3` (2,414) and 0.97x versus C++ `-O3` (2,431). The 3.53x gap is closed, and the hot loop keeps zero overflow checks. Everything else held: order-book 1.07x, Black–Scholes 0.97x, Monte Carlo 0.83x, matmul 1.94x.

The gates were all green: bootstrap 7/7 byte-identical, fuzz 102/102 hosted and 102/102 Python, parity 112/112, corpus 99/99, regression 62/62, AddressSanitizer clean. The full suite had six failures, all pre-existing on 2.3.0. The safety evidence was checked both ways: checks still fire where the proof does not hold — entry at INT64_MAX−7 traps on the eighth increment, an unbounded loop traps on the second.

Fix 1 also exposed a latent bug as a side effect. The coarse R3 guard (`n*n > len`) wrongly trapped the correct program `beamsearch` — true max index 7 against length 8, but 4·4 = 16 > 8. The guard now uses the tight bound `ihi*n + khi >= len` when both highs are known. Sixteen new fuzz cases were added.

Fix 1 had a longer history than that one night. Its design was done 12 Nov (753 lines, 9 proof shapes kept, 12 cut with reasons), and its implementation ran in three phases: B1 bounds-check hoisting, B2 range analysis, B3 overflow narrowing. Two design questions came to me on 12 Nov. The first — a trapping program now traps before the loop instead of mid-loop, same message, same exit code, less stdout before the abort — I answered the same day: "Yes, trap before the loop." The second is still open (see section 7).

### Fix 2: the idea that measured 0.0x, and the pragma that worked

Fix 2 is the matmul story, and it is mostly a story about a wrong idea.

The first Fix 2 idea was to hoist `.data` into locals, or restrict after the last escaping call. On 15 Nov, a 21-row probe matrix showed it moves matmul by 0.0x at every flag combination — `-O2`, `-O3`, `-O3 -march=native`. GCC already LICM-hoists the field loads. The design document records the falsification, and the tree was left untouched.

That matters because on 21 Nov at about 03:35 IST, a Fix 2 coordinator was briefed to implement the `.data` hoist — the already-falsified design. The error was caught when the coordinator read `DESIGN.md`, and corrected by interrupt before anything was implemented. The revised Fix 2 — (a) extend overflow narrowing to builder-loop bounds and increments, plus (b) re-place loop guards before the repetition loop, same check, same trap, moved never deleted — I had already picked it on 15 Nov; it shipped in commit `9ea8e190`. It unlocked about 68k → 13k microseconds at `-O3 -march=native`. It did not help `-O2`. That was the honest limit recorded in the 2.3.0 changelog: safe-mode matmul stays around 90–95k at `-O2`, scalar, because the blocker is buffer provenance — push-built arrays — a redesign, not a tuning fix.

The redesign proposal, delivered 21 Nov, offered three options: A, `#pragma GCC ivdep` on compiler-proven non-aliasing nests; B, inline array growth, a big redesign for the same result; D, do nothing at `-O2` and leave the gap at about 1.9x. Two more options, a single arena and hoist/restrict, were already falsified. A fresh finding made option A real: `#pragma GCC ivdep` before the j-loop alone vectorizes the push-built nest at `-O2`, about 35k microseconds, with no restrict needed. The soundness gate was the real work: the compiler must prove the arrays in the nest do not share storage. Array assignment copies the struct including the data pointer, so `let b = a` aliases. Slices, parameters, and call returns alias too. And ivdep never goes before the k reduction loop.

I picked option A. The implementation landed 21 Nov as commit `d6984c46`: ivdep emitted before a `while` loop only when every array in the nest is provably a distinct local allocation, under gates IV-G1..G11 and provenance rules F-P1..F-P6. Matmul safe `-O2`: 68,490 → 16,615 microseconds per 20 repetitions. Interleaved minimum 15,046 versus C `-O3` at 33,095 (0.45x) and C `-O3 -march=native` at 15,086 (1.00x). Safe mode now ties or beats C on every quant benchmark: tick 0.99x, order-book 1.06x, Black–Scholes 0.92x, Black–Scholes via call 0.94x, Monte Carlo 0.75x, matmul 0.44x.

The gates: bootstrap 7/7 byte-identical, parity 124/124, corpus 99/99, regression 62/62, fuzz 122/122 in both modes (102 existing plus 20 new ivdep cases — 6 positives emitting exactly one pragma, 14 aliasing negatives emitting zero), suite 62/69 in both modes with all 7 failures pre-existing, AddressSanitizer 122/122 clean, formatter idempotent.

### Fix 3: the flag, and a stale note corrected

Fix 3 is `-march=native` as the safe-mode default, with `--portable` reproducing the old `-O2` behaviour exactly. I approved it 13 Nov at about 10:06 IST: "Yes, turn on -march=native in safe mode." It was implemented 15 Nov at about 19:30 IST and shipped in 2.3.0.

The honest result: Fix 3 moved safe mode by about 0x. Safe-mode matmul did not land at 13k. Verified: safe native (`-O2 -march=native`) runs about 90–95k microseconds per 20 repetitions, scalar, zero vector registers; the same emit at `-O3 -march=native` runs about 17.7k with 12 vector registers. `-O2` does not vectorize the safe emit even with `-march=native`. The blocker is buffer provenance, not a flag. That is why Fix 3 alone was never the answer, and why Fix 2 had to be a redesign.

One correction belongs here. The 21 Nov C++ diagnosis note listed Fix 3 as a policy call still needing my word. That note was stale. Fix 3 had already been approved, implemented, and shipped six days earlier. The diagnosis itself was right about everything else — the C++ numbers, the tick cause, the matmul cause. The Fix 3 line in it was wrong, and the release record is what stands.

Together, the three fixes shipped in 2.3.0, released 16 Nov 2026 at about 15:13 IST: main `22bd83ec`, tag `v2.3.0`, GitHub Release id 446838858, PyPI `qasida` 2.3.0 live, fresh-venv install verified. The release also carried `sin`, `cos`, and `tan` builtins lowering directly to libm — the trig demo is byte-identical to its C reference — and 40 new standard-library modules added 13–16 Nov, all verified under both compilers. The first push attempt died mid-upload on a VM restart with nothing landed. The retry landed clean.

## 6. The library program: 500 items, and one week of sprinting

A language is not a compiler. It is a standard library. Qasida's library program is a 500-item roadmap — Mojo, Python, Rust, and Go equivalents, built in Qasida itself. The recount on 11 Nov 2026 put it at 71 complete, 62 partial, 367 to build.

The daily cadence was ten OS items and ten library items a day, reported separately, because a surplus on one track does not cover a shortfall on the other. On 21 Nov at about 08:41 IST, I changed the target: "We need to push this overall to 70% this week." The honest answer was that 70% overall in a week is not reachable — it would need about 350 complete items, 279 more in seven days, plus an ecosystem that needs thousands of packages and real users. The options were laid out: sprint both tracks at about 15 a day for roughly 35% overall, sprint libraries only at about 20 a day for roughly 40%, or keep the normal pace for about 28%. I picked libraries only, 21–27 Nov, at least 20 verified items a day, OS track paused. The proven pace was 10 a day. Twenty was unproven. The cron body says so, and the counts are reported honestly.

Day 1, 21 Nov, landed five batches back to back, not one. The first batch took HAVE from 71 to 91 (commit `66a9b7a8`). The second took it to 111 (commit `71a12454`). The third took it to 131 (commit `0922955a`). The fourth took it to 150 (commit `bc4655cd`) — twelve of its twenty items were modules already in the tree that the catalog still listed as gap or partial, verified and flipped, plus a held `icalparse.fg` that extends an existing item without flipping a count. The fifth took it to 170 (commit `94f2552e`), with PARTIAL down to 35 and GAP down to 295.

Each batch runs the same verification: 80 checks across both compilers, plus an independent replica. The fifth batch caught its own near-miss honestly: a CSRF token alphabet with 31 characters, not 32, fixed to use 31; and item #448, reverted to GAP because the candidate module does not auto-escape by default — the first candidate for the next batch, as a wrapper, not as a quiet pass.

The same day also closed two of the README's honest gaps in pure Qasida: a baseline JPEG decoder (30 checks, 0 failures against PIL, maximum per-channel difference 5) and a PNG decoder v2 with a full DEFLATE inflate (51 checks, 0 failures, byte-exact against PIL on 20 fixtures). The public registry went live the same night (commit `5c41d5ab`, 81 files, 30 packages on GitHub Pages), and the playground's fetch list was fixed from 53 to 74 compiler files, then verified end to end in a real browser — all four examples ran. Those three landings, plus the two decoders, were pushed selectively, not by full-tree push, because the full-tree push would have published unverified work sitting in the same tree. That discipline — selective pushes with an explicit allowlist, verified by Contents-API read-back — came from the push failures of 2, 3, and 5 Nov, when GitHub returned 500s and a 422 "input too large" on single giant tree posts. The fix was an incremental tree build, deepest first, at most 190 entries per tree.

Two things in the library program are still open, and both are my calls.

The first is item #42, error wrapping with context. It blocked the production pipeline from 11 Nov: its shape — new syntax, a helper function, or a Result method — is a semantic decision, and the pipeline treats semantic decisions as a hard stop. Batch 2 landed a helper-function slice (`errwrap.fg`, verified 4/4 under both compilers) without my pick, because the batch brief did not carry the #42 block. On 21 Nov at about 11:28 IST I picked: "Keep the helper-function slice as #42." The new-syntax and Result-method variants stay open as possible future extensions. They are not the #42 answer. The pipeline block is cleared, and the next queue item is #44.

The second is the toggles question, from Fix 1's design, section 9, question 2: the check on/off measurement toggles — hidden `--no-bounds-checks` and `--no-overflow-checks` — should they stay hidden and measurement-only, or become user-facing later? The recommendation is hidden for now. It has been open since 12 Nov. It was not covered by my 18 Nov "do whatever is best", and it is still open as of 21 Nov.

## 7. Failed attempts

Not everything in this story worked. The failures, in one place:

- **The `.data` hoist.** Falsified by measurement, 0.0x at every flag combination, 15 Nov. Then almost re-implemented on 21 Nov because a briefing did not carry the falsification. The design document caught it. Briefings now have to carry the falsification, not just the task.
- **Fix 3's expected 13k.** The flag was expected to deliver the vectorization win. It delivered about 0x in safe mode, because `-O2` does not vectorize the safe emit. The blocker was buffer provenance, and only Fix 2's redesign addressed it.
- **The 2.2.0 speedup claims.** Published, then reversed by the 20 Nov re-test. The hosted compiler regressed 34% on the largest file, and Python is now faster than hosted on large emits. The README stopped quoting the old numbers.
- **Full-tree pushes.** Three failures in early November — GitHub 500s, then a 422 on a 934-entry tree post. The fix was incremental tree building, and later, selective pushes with an explicit allowlist.
- **The formatter eating comments.** `forge.py fmt` strips file-leading `//` comments. It deleted a 15-line header from `zlib.fg` during the PNG run. The file was restored byte-identical from backup and re-verified. The formatter was not run on the new files. It is a pre-existing behaviour, not a one-off.
- **Fluent text generation.** The TinyStories word model (12,608 parameters) does not produce fluent sentences under any decoding setting. A 39-sample sweep across 15 settings found only word salad; greedy decoding collapses to `<unk>`. The current checkpoint measures perplexity 158.15 — the README's 385.06 was measured on an earlier checkpoint and is stale. Fluency needs a 44M-class model, about 120M tokens, and about 2.5 T4-hours on Kaggle. That needs a fresh token and an explicit per-run authorization. Training stays stopped on my 8 Nov word.
- **The 174 stale files.** RED-MOON- carries 174 remote-only stale files — the old top-level stdlib layout, a dedication file, an old binary, a personal file. Deleting them needs a separate deletion-aware push decision. It has not been made.

## 8. Lessons

The lessons are the same ones the record keeps re-teaching:

- **Measure first, and measure both directions.** The tick gap was one call site, proven by ablation in both directions. The matmul hoist was 0.0x, proven by a 21-row probe matrix. Without the probe, the hoist would have been implemented.
- **A falsified idea must travel with the task.** The hoist was falsified in a design document and nearly implemented anyway, because the briefing did not carry the falsification. A result that lives in only one document does not exist yet.
- **Re-measure published numbers.** The 2.2.0 speedups, the 385.06 perplexity, the tick 3.49x chart — all three were superseded by later measurement. The project re-measures, and the re-measurement wins.
- **Verification is a bar, not a feeling.** Bootstrap 7/7 byte-identical, corpus 99/99, regression 62/62, fuzz and parity in both modes, the full suite under both compilers, AddressSanitizer clean, formatter idempotent. A fix is not done when it compiles. It is done when the bar is green and the push is verified by read-back.
- **Do not push unverified work by accident.** Full-tree pushes publish everything in the tree. When unverified compiler edits, an unreviewed README makeover, and a probe file sit in the same tree as verified work, the push is selective or it does not happen.
- **Some decisions are mine alone.** Semantic decisions — #42's shape, the toggles, language-default copy-on-write, the 174 stale files — are hard stops. A general "do whatever is best" does not cover them, and the record shows that boundary being held three times in one week.

## 9. What changed the plan

Three things changed the plan, and all three are dated.

First, the problem-first pivot on 26 Oct. Feature-building stopped. Nine problems were named, the research was done, v0.1 was archived, and v2 started clean. Everything since — the beachhead, the four rules, the toolchain-first order — follows from that day.

Second, self-hosting on 11 Nov. The hosted compiler became the default, and every compiler fix after that had to be built twice and verified under both compilers. It doubled the work and it caught real bugs. It is also why the 20 Nov compile regression matters: the compiler you ship is the compiler you compile with.

Third, the sprint week. On 21 Nov, the 70% ask was answered honestly — not reachable — and replaced with a reachable target: libraries only, 20 verified items a day, HAVE 71 to about 211, overall about 40%. Five batches landed on day one, HAVE 71 to 170. The OS track is paused until 28 Nov, when the normal 10-and-10 cadence resumes. The target is still unproven at 20 a day, and the counts are reported as landed, not as promised.

## 10. Open questions

- **The toggles.** Hidden and measurement-only, or user-facing later? Open since 12 Nov. Recommended hidden.
- **The 174 stale files.** Delete them in a deletion-aware push, or carry them? Needs my word.
- **The compile regression.** Hosted codegen-wrap up 34% since release, Python now faster on large emits. Under investigation, not yet fixed.
- **The lifetime-allocator slice.** The kernel builds to 4,319 words; the Python big32 model has 4,096 words of instruction memory. It has not landed. It is a Qasida-adjacent limit, recorded in the README's known limits.
- **Fluent text generation.** Decoding cannot reach it. Training needs a GPU run, a fresh Kaggle token, and my explicit per-run word.
- [I still need to decide — what the next release is.] 2.3.0 shipped 16 Nov. Nothing in the record says when the sprint's work gets a version number, or what that version is. The release rule is that releases happen only at authorized milestones, and no milestone after 2.3.0 has been authorized.
- [I still need to decide — when Qasida is "done".] No source defines the end state. The Top-500 has a finish line — 500 items — but the language itself has no stated definition of done, and the 70% conversation showed that "overall" is a number that needs a definition before it can be a target.

## Sources

- Repo README, `themythofsisyphusjayant/RED-MOON-` — live copy read via the GitHub Contents API, byte-identical to `~/workspace/forge-v2/README.md` as of 21 Nov 2026.
- `~/workspace/forge-v2/CHANGELOG.md` — 2.2.0 (Steps 2–6) and 2.3.0 (quant-latency phase, trig builtins, 40 modules) sections, and the Unreleased section (registry, JPEG, PNG v2).
- `~/workspace/forge-v2/RESTART.md` — the 26 Oct restart, the research findings, and the v2 build log.
- `~/workspace/forge/PROBLEMS.md` — the nine problems (v0.1 design brief; v2 inherits it by the restart rule).
- `~/workspace/forge-automation/STATE.md` and `PIPELINE.md` — the production cadence, the #42 block and its 21 Nov resolution, the push history.
- Project memory, 12–21 Nov 2026 — Fix 1 design and implementation, Fix 2 proposal and ivdep landing, Fix 3 approval and implementation, the 20 Nov speed re-test, the C++ diagnosis, the sprint-week decision and batches 1–5, the toggles question.
