# TINY8 — an 8-bit CPU, built from zero

## The problem

Textbook CPUs are either toy diagrams or full cores too big to hold in your head. I wanted the middle ground. A real CPU. Simulated, not manufactured. Small enough that one person can understand every line of it.

I had just solved the Jane Street ASIC puzzle. I wanted to build something like it from zero, more powerful and useful. I picked: build a tiny CPU.

## The first version

v1 was an 8-bit accumulator CPU in Verilog. 16-bit instruction words. Harvard memory: 256 words of ROM, 256 bytes of RAM. One flag, Z. Two memory-mapped ports: read `0xFE` for input, write `0xFF` for output. Every instruction takes two clocks: FETCH, then EXECUTE. 16 opcodes.

I also wrote a two-pass assembler and a Python model of the same CPU. That gave me two independent checks. If the Verilog and the Python model agree, a bug in one would likely show up in the other.

The first program sums 1 to 10. Both checks agreed: 55, halted, 86 instructions, 173 cycles. One command reruns everything: `./run.sh`.

## Design decisions that held through six versions

**Reuse the reserved field.** The instruction word had a `[11:8]` field that v1 did not use. Every upgrade after v1 put new instructions there. NOP is extension 0, so v1 programs assemble identically and run unchanged on every later version.

**Old programs must keep passing, with identical step counts.** That was the rule for every upgrade. Each version's testbench re-ran all earlier programs. If a step count moved, something was wrong.

**The stack grows down from the top of RAM.** SP resets to `0xFE`. Program data stays below `0xF0`, so deep calls never collide with it. Never RET without a CALL.

**Flags change only where they should.** Z is set by ALU and load results only. STA, jumps, CALL, RET, NOP leave it alone. That is what makes ADD followed by JZ or JNZ loops work.

## Making it play something

I asked for something dope. I picked: make tiny8 actually play something.

The guess-the-number game runs on the v1 ISA with no new instructions. 26 words. The CPU decides every hint. Out port 1 means too low, 2 means too high, 255 means win. The driver is only the console. It feeds one guess per CPU run, because in_port holds one constant per run and out_port holds only the last byte.

The game had a real limit. Guesses only worked from 1 to 100. Too-high versus too-low was decided by whether the mod-256 difference was 128 or more, tested with seven SHRs. That test is only valid in that range.

## The power chain: v2 to v6

I kept asking for more power. Each version removed one ceiling and hit the next.

**v2: jumps back, calls, a stack.** JNZ, CALL, RET went into the extension field. Fib(10) = 55. A nested CALL/RET demo: 7 + 5 + 100 = 112, stack depth 2, SP back to `0xFE`.

**v3: a second register.** B, plus TAB, TBA, ADDB, SUBB, PUSH, POP. LDI still loads A only, so B survives across it. That is what makes B useful as a counter or a saved value. Multiply demo: 6 × 7 = 42, with B as the loop counter. A subroutine saves and restores A with PUSH/POP: out 42, RAM `0x11` = 7.

**v4: a carry flag, real comparisons, hardware multiply.** C is set by ADD/ADDI/ADDB carry and SUB/SUBB/CMP borrow, and left alone by everything else. So JC and JNC can branch on unsigned comparisons. MUL does 15 × 15 = 225 in one instruction.

v4 also fixed the game's range limit. CMP plus JC plays the full 0 to 255 range. The old 1..100 game is unchanged by default; `--full-range` turns the new one on.

**v5: 64KB of RAM.** B became the page register. LDAX and STAX use the effective address (B << 8) | operand. All the old zero-page instructions work exactly as before. Page demo: RAM[16] = 42, RAM[288] = 99, RAM[767] = 7, out 99.

One edge stayed: RAM[254] and RAM[255] are shadowed by the ports in zero-page mode. They are only reachable through paged access with B = 0.

**v6: the last big ceiling.** A 16-bit program counter and a 64K-word ROM, so programs are no longer capped at 256 words. JMPF and JZF jump to (B << 8) | operand. CALL and RET now push and pop a 2-byte return address, high byte first. Old programs still work, because their return addresses are under 256 and the stack still balances. The assembler gained ORG, so code can be placed at a far address.

A far call needs no new opcode. Push the return address by hand, set B to the page, JMPF, and let RET pop it back. Far demo: a call to address 256, out 7.

## Failed attempts

**The far_cond bug.** The first version of the conditional far-jump demo failed in both models. It set Z, then cleared it with LDI 1 right before the jump. The fix was at the program level, not the CPU: set B first, then LDI 0. It was not a CPU bug. Both models failing the same way is what made that clear.

**The push that could not happen.** I staged the full package for GitHub, but there was no repo and the stored token could not create one. The push sat blocked on one thing: I had to create an empty public repo myself. I created `jayantt1dubeyy/tiny8` on 25 October. Then v6 went up: 41 files, verified live.

**No gate count.** The roadmap says synthesis with Yosys, then a TinyTapeout tile. Yosys is not installed on the machine, so no gate count is claimed. The TinyTapeout wrapper compiles, but it is not a submission. A real submission needs my own TinyTapeout account, a confirmed shuttle slot, and SRAM macros for the 64KB RAM and ROM arrays. None of that has happened.

## Lessons

Two independent models agreeing is the whole point of the verification. The Python model and the Verilog simulation checked each other on every program, every version.

Backward compatibility is a design constraint, not a courtesy. Keeping every old program passing with identical step counts caught regressions before they became mysteries.

A reserved field is a plan. Leaving `[11:8]` unused in v1 is what made v2 through v5 possible without breaking anything.

Know where the encoding ends. All 16 extension slots are now used. Further instructions would need a prefix scheme or a wider instruction word. That is the honest next ceiling, along with interrupts and a timer, or a display and keypad port.

## What changed the plan

I asked to make it "basic mobile level powerful." Before building, that got clarified: I picked "just keep pushing CPU power" — not a tiny-phone system with a screen and apps, and not a mobile-web version. All the upgrades were CPU power only.

Then I hit the real wall. I was told plainly that tiny8 cannot run a modern OS or browser. 8-bit, 64KB of RAM, no screen, no network, no graphics hardware. A browser needs hundreds of megabytes. I was offered three lanes: a tiny OS for tiny8, a tiny text browser for tiny8, or a new, bigger CPU. I picked the bigger CPU.

That became a separate project: big32, a 32-bit RISC-V core aimed at one day running Linux. Not a tiny8 upgrade. tiny8 stays what it is: a finished, verified, small CPU that one person can understand end to end.

## What is still open

- Synthesis with Yosys: gate count and area estimate on SkyWater 130nm. Not done. [NEEDS JAYANT'S WORD: do you still want to pursue a TinyTapeout submission, or is tiny8 finished as a learning build?]
- Interrupts, a timer, or a second display/keypad port. Not built.
- The encoding is full. Any new instruction needs a prefix scheme or a wider word.

---

*Sources: TINY8 repo README (themythofsisyphusjayant/TINY8, v6), local tree `~/workspace/tiny8-push/`, daily memory notes 25–26 Oct 2026. All numbers above are from the README's verified section: Python model and Icarus Verilog simulation agreeing on every program.*
