# Story 07 — The ASIC puzzle

Jane Street published a chip layout and a waveform. No netlist. No documentation. My task: recover the netlist from the geometry, work out what the chip does, and find the input that makes it succeed.

The files were `puzzle.gds` (a SkyWater 130 nm layout) and `example_inputs.vcd` (a waveform recorded from the real chip). That was everything.

## The problem

A layout file is geometry, not logic. It holds polygons and vias. It does not say which polygons are wires, which are gates, or what the circuit computes.

I had to reverse-engineer the chip from the geometry alone. Then I had to simulate the recovered circuit. Then I had to find the input that unlocks it.

There was also a warmup. The puzzle shipped with one: a small design, `warmup/`, with source Verilog at one end and a finished layout at the other. It was the test bench for everything that followed.

## Design decisions

My first decision: reproduce, don't copy. Shubham's public REJS writeup was the method guide. The key was not to be taken from it. I had to rebuild the solve from the layout and verify it by simulation. If the answer matched the writeup, that would be independent confirmation, not a source.

My second decision: try every approach, and cross-check them. I said "try all of them." I ran four approaches in parallel:

1. Scripted GDS extraction — parse the polygons, union them into nets, emit a netlist — then a Z3 solve.
2. VCD protocol inference — read the example waveform alone and work out the protocol.
3. KLayout LVS extraction — use KLayout's own built-in extractor, plus a transistor-level spot-check of standard cells.
4. Brute force plus structural tracing — simulate, and trace backwards from the `success` output.

My third decision: prove the toolchain on the warmup before touching the puzzle. The warmup's answer was known from its source: its output `S` is 1 exactly when `A + B == 496`. Any extractor or simulator that could not reproduce that was not ready for the real chip.

## What the waveform said

The VCD work finished first. The example file held two full runs, not one — the writeup had missed that.

Each run had the same shape: `rst_n` low for exactly 3 rising clock edges, then 1 idle cycle, then 121 bits clocked in on pin `I` while `enable` was high, then `enable` fell and the output streamed. The clock was 10 ns.

Both example keys were wrong keys. Each streamed `TRY AGAIN` on the 8-bit output bus, one byte per clock, and `success` never went high. The output was byte-identical for both wrong keys. That meant there was no oracle gradient: a wrong key gave no hint about how close it was. Guessing was useless.

Two things the VCD could not settle: the key length was about 121 bits (or 120 plus a framing bit), and bit order — MSB or LSB first — was not determinable from the waveform. The netlist work had to settle both.

One more detail from the waveform: both example keys had popcount exactly 38 out of 121. Random would average about 60.5. That is roughly 4σ low. It suggests Jane Street's example generator was zero-biased. A hint about the examples, not about the true key.

## What the chip turned out to be

My scripted extraction found 728 functional standard cells: 636 combinational gates and 92 flip-flops. 12,615 merged metal polygons. 33,323 vias. 13 top-level pads: `clk`, `rst_n`, `enable`, `I` in; `success` and `O[7:0]` out.

The chip is a serial-key lock with a talking output. After reset it shifts in a 121-bit key, one bit per clock, on pin `I` while `enable` is high. Then `enable` drops and the chip judges the key.

Right key: `success` goes high and stays high — it is a flip-flop output with feedback holding it — and `O[7:0]` streams one ASCII character per clock.

Wrong key: `success` stays low and the output streams `TRY AGAIN`.

Two easter eggs: 121 zeros streams `EMPTY SKY`. 121 ones streams `BIG BANG`. Both with `success` low.

## Failed attempts

**Brute force was impossible.** I ruled out the easy idea first, with the structural trace. There is no 121-bit shift register in the chip — only 92 flip-flops exist. What is there is an 11-bit shift window fed by `I`, feeding a nonlinear state machine. A GF(2) affinity test failed, so the machine is not linear and no linear-algebra shortcut exists. Brute force over 2^121 was out of the question. The key had to come from a solver, not from trying inputs.

**The first Z3 candidate failed in simulation.** The solver returned a key. The simulator said it did not work. Differential checking — Z3 model versus simulator, step by step — traced the divergence to a single cell type, `o2bb2a`. The Python cell model had `X = (A1_N | A2_N) & (B1 | B2)`. The PDK functional model says the `_N` pins feed the NAND directly, not inverted first: `X = !(A1_N & A2_N) & (B1 | B2)`. The Z3 model had it right. The simulator had it wrong. I fixed one line in `gsim.py`, and the divergence report went to zero over steps 0–122. The solver's key then verified: `success` = 1, flag out. I checked every other `_N` cell against its PDK primitives after that and confirmed them correct.

**The LVS extraction fragmented.** I ran KLayout's extractor without intra-layer `connect()` calls, and it split the warmup into 1,642 nets with singleton nets — lone pins that should have been joined. In KLayout 0.30's Python binding, `connect(a, b)` is inter-layer; the single-argument `connect(layer)` is the intra-layer merge, and it is required. With intra-layer connects on every layer, the warmup extraction became an exact structural match: 79 cells, 84/84 net signatures. I also tried via flattening along the way — 1,099 down to 230 top instances — and it changed nothing once the intra-layer connects were in place.

**The net count looked wrong.** My scripted extraction found 2,626 polygon components. The writeup said 741 nets. That looked like a conflict until I counted it properly: only 739 of the components touch an instance pin, plus 2 power nets, which is exactly 741. The other 1,887 components are floating geometry — mostly single isolated shapes inside cells, plus routing stubs — connected to no pin, affecting no logic. The merged-polygon count (12,615) and via count (33,323) matched the writeup exactly, so the geometry extraction agreed completely.

**One net has no driver.** Two gate inputs connect to a routed net with no driver anywhere in the layout. It is a genuine floating net, not an extraction miss — all 728 cells, all 2,767 signal pins, and all 13 pads are accounted for, and no signal is shorted to power. The simulator defaults it to 0. I ran the sensitivity check anyway: forcing it to 1 returns the same key, still unique, and the key still asserts `success`. The answer does not depend on the assumption.

**The submission was not possible.** I asked to submit the solution. Jane Street's post said submissions closed 4 September 2026, and its rules asked solvers not to feed the puzzle files into an AI tool or use AI to generate writeups. I was told both facts plainly. Nothing was submitted to Jane Street or anywhere else. My solution was agent-built and would not have qualified as a contest entry regardless.

## How it ended

I unrolled 141 cycles in the Z3 solve: 3 reset, 1 gap, 121 symbolic shift bits, 16 stream cycles, constraining `success = 1` at cycle 125. It returned SAT in 0.1 seconds. A blocking clause excluding exactly that key returned UNSAT. The key is provably unique — no second 121-bit input asserts `success`.

The key, first bit first on `I`:

```
0000000101010000100000000000010101010000000000001010000001000001000000100000101000010000000100000010000010010001010000000
```

I fed it through the concrete simulator. It asserts `success` from cycle 125 onward and streams the flag, one character per clock:

```
(* TWO STARS *)
```

The golden check held: I drove the extracted netlist with the exact per-edge inputs from `example_inputs.vcd`, and it matched 312/312 edges with 0 mismatches, both runs streaming `TRY AGAIN` exactly as recorded from the real chip.

All four approaches agreed. The key is character-for-character identical to the one in the REJS writeup — from a separate extraction and separate simulators, not from copying. Single-bit flips at positions 0, 60, and 120 all fail with `TRY AGAIN`. The key is exact, not a near-miss.

## What changed the plan

My plan was to solve and submit. The deadline and the rules closed that. What happened instead: publish after the close, which Jane Street's post explicitly allows.

I created the repo myself — `jayantt1dubeyy/JANE-`, public, 25 October 2026 — because the stored GitHub credential returned 403 on repo creation. The upload path had its own failure first: an empty repo rejects git-blob creation with 409, so I had to initialize the repo with one file through the Contents API before the Git Data API path worked. I uploaded the full package as 251 files, commit `aeb2a52`.

I renamed the repo myself later that day, from `JANE-` to `asic-puzzle-2026-solution`. The repo now lives at `themythofsisyphusjayant/ASIC-PUZZLE-2026-SOLUTION`. I rewrote the README with badges, the layout image, and three Mermaid diagrams, and I added a 3D viewer from the real LVS placements — all 728 cells at their true coordinates, with a playable animation of the winning key streaming in and latching `success`.

One piece is still open. The 3D viewer was built to be served from GitHub Pages, which needs a one-click enable in the repo's Settings. [I still need to write this part — whether Pages was ever enabled. The Pages link returned 404 when checked on 21 November 2026 and was removed from the README.]

## Lessons

- Prove the toolchain on a known answer before trusting it on an unknown one. The warmup caught nothing by luck — it was the reason the puzzle extraction could be trusted.
- A solver's answer is a claim, not a result, until the concrete simulator agrees. The first Z3 candidate failed. The bug was in the simulator, not the solver, and only step-by-step differential checking found which side was wrong.
- Cross-checks are worth the extra work. Four independent methods, four chances to be wrong in different ways. They agreed.
- Count what matters. 2,626 components versus 741 nets looked like a failure of the extraction. It was a difference in what was counted.
- A wrong key gives no gradient. When the only feedback is `TRY AGAIN`, stop guessing inputs and solve the circuit instead.
- Read the rules before planning the ending. The submission was impossible from the start — closed 4 September, and AI-assisted entries excluded. The plan changed to publishing, which was allowed.

## Still unknown

- The LVS cross-check counts 729 top nets against the 741 figure. Unresolved, most likely a counting difference. It does not touch the key.
- The transistor spot-check was partial: geometry confirmed (row height, diffusion strips, transistor counts), but NAND-versus-NOR series/parallel topology was not resolved geometrically.
- [I still need to write this part — what drew me to this puzzle in the first place. The sources record the ask ("Can you do it for me", 25 October 2026) but not the reason.]

## Sources

- Live repo README — `themythofsisyphusjayant/ASIC-PUZZLE-2026-SOLUTION` (read 21 Nov 2026)
- `solution/SOLUTION.md`, `solution/verification_log.txt` — the solve record and golden checks
- `solution/brute-structural/BRUTE_STRUCTURAL.md` — the brute-force/structural analysis
- `solution/lvs/LVS_REPORT.md` — the KLayout LVS extraction report
- Daily memory notes, 25–27 Oct 2026
