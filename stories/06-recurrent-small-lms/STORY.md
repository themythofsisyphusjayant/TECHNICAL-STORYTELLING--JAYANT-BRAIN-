# Story 06 — The Loop-Depth Sweep: A Negative Result, Reported Honestly

**The question:** do recurrent transformers gain anything from running extra loops at test time?

**The answer:** no. Perplexity falls from 29.38 to 4.41 as loops go from 1 to 4 — the trained depth — then flatlines. Loops 8, 12, and 16 buy nothing, and going past the trained depth makes things slightly worse.

This story is about the sweep that answered the question, and the 22-page report that recorded the answer. The model itself and how it was trained are Story 05. This one is about the measurement.

---

## 1. The problem

The OpenMythos architecture shares one block instead of stacking distinct layers. Prelude layers in, a recurrent block run N times, coda layers out. Same weights, used again and again. That turns parameters into effective depth.

That design carried a big claim. Train with N loops. Then, at test time, run more than N loops and get better results for free. Test-time depth scaling. My roadmap called it the killer feature.

Nobody had tested it. The claim was written down before any model existed to check it against. That is the problem this story starts with: a headline claim with no measurement behind it.

The 44M model changed that. Once it was trained, the claim could finally be put to a test. Either extra loops help, or they do not. Both outcomes are worth reporting. Only one of them is true.

## 2. The design decisions

### The model, briefly

The reference model is the 44M OpenMythos checkpoint from Story 05: 43,974,722 parameters, dim 512, GPT-2 BPE vocabulary of 50,257 tokens. Six prelude layers, a loop block run 4 times, two coda layers, then logits. The loop block uses grouped-query attention and an 8-expert sparse MoE.

The wider design, as documented in the September 2026 plan and the from-scratch implementation, pairs GQA with RoPE attention, SwiGLU feed-forward with an optional sparse MoE, and a shared recurrent stack trained with truncated BPTT. One honest note: for the 44M run itself I used the kyegomez/OpenMythos package, not the from-scratch code. So the design above is the architecture's specification. The trained checkpoint is the package's build of it.

Training: I trained on TinyStories, about 120M tokens, 4,850 steps on a Kaggle T4 in October 2026. Loss fell from 9.52 to 1.53. That run is Story 05. What matters here is what happened after the weights were frozen.

### The sweep design

I designed the sweep to be hard to fool. Four decisions mattered.

**Fresh data, fixed size.** I evaluated on 256 fresh TinyStories sequences of 256 tokens each, pulled at dataset offset 1,500,000. The training split holds about 2.1M stories, so these are almost certainly unseen. Same 256 sequences at every loop count. No cherry-picking.

**Loop counts that bracket the trained depth.** n ∈ {1, 2, 4, 8, 12, 16}. The 4 matches training. 1 and 2 test what happens below it. 8, 12, and 16 test extrapolation above it. If extra loops helped, this grid would show it.

**Uncertainty, quantified twice.** I gave every mean a bootstrap 95% confidence interval — 10,000 percentile resamples over the 256 sequences. And I tested the key comparison, 4 loops vs 8 loops, as a paired per-sequence difference, not just by comparing two means. Two means with overlapping intervals can still differ. A paired test settles it.

**A generation check, kept qualitative.** From a fixed prompt — "Once upon a time there was a little girl named Lily." — I generated continuations at 1, 4, 8, and 16 loops. One prompt, one seed. That is a sanity check on coherence, not a claim about diversity. The report says so.

The script was `eval_loop_sweep.py`. The numbers landed in `loop_sweep_results.json` on 16 October 2026.

### The GQA side-study

I also worked out in the report why grouped-query attention matters, in the reference dimensions: d = 512, 8 query heads, head dim 64, 2 KV heads. Per attention layer, full multi-head attention costs 1,048,576 parameters. GQA costs 655,360. That saves 393,216 per layer, about 3.15M across the 8 unique attention layers — 7.2% of the model. A welcome discount. Not the reason to adopt GQA.

The reason is the KV cache. In fp16, multi-head attention needs 2,048 bytes per token per layer. GQA with 2 KV heads needs 512. A 4× cut, independent of model size. At batch 8 and 2k context, the multi-head KV cache is 403 MB — four times the weights. GQA reclaims roughly 300 MB, down to about 101 MB. At heavy batching, that is the difference between fitting on one GPU or not.

## 3. What the sweep found

| Loops | 1 | 2 | **4 (trained)** | 8 | 12 | 16 |
|---|---|---|---|---|---|---|
| Loss | 3.380 | 2.414 | **1.484** | 1.590 | 1.591 | 1.591 |
| Loss 95% CI | [3.343, 3.419] | [2.384, 2.446] | **[1.454, 1.516]** | [1.559, 1.622] | [1.560, 1.624] | [1.560, 1.623] |
| Perplexity | 29.38 | 11.18 | **4.41** | 4.90 | 4.91 | 4.91 |
| PPL 95% CI | [28.30, 30.52] | [10.84, 11.54] | **[4.28, 4.55]** | [4.75, 5.06] | [4.76, 5.07] | [4.76, 5.07] |

Three regimes are visible.

**Recurrence helps up to the trained depth.** One loop gives perplexity 29.38. Two loops more than halve it to 11.18. Four loops reach 4.41. The recurrent iterations do real, non-redundant work. The architecture is not a gimmick. It works — inside the depth it was trained at.

**Extrapolation buys nothing.** Eight loops is slightly worse than four: 4.41 → 4.90. Twelve and sixteen loops change nothing: 4.91, 4.91. The dynamics do not keep refining. They settle to a fixed point near, but slightly worse than, the trained-depth output.

**The degradation is not noise.** The paired per-sequence difference from 4 to 8 loops is +0.106 loss, 95% CI [+0.101, +0.110]. The bootstrap intervals on the two means do not overlap either: [4.28, 4.55] at 4 loops against [4.75, 5.06] at 8. Small, real, and in the wrong direction.

The generations agree. At 1 loop, the story degrades into repetition. At 4, 8, and 16 loops, all three continuations are fluent TinyStories-style prose. Extra loops do not break generation. They just do not improve it.

The likely mechanism, stated as a hypothesis and not a proof: extra test-time loops push the hidden state off the manifold the model learned to live on during training. The extra computation adds noise, not signal. Then the recurrent dynamics settle, which is why the loss plateaus instead of diverging.

## 4. Failed attempts

The sweep itself is the first failed attempt. My hypothesis was that test-time loops buy depth for free. The measurement says they do not. That is the story's central failure, and it is also its main result. A negative result, measured cleanly, is still a result.

The second failure was the roadmap claim. The onion-ai README had called test-time depth extrapolation the killer feature. The sweep contradicted it. So I corrected the README with a tempering note, and moved the claim to rest on a future run instead of the one that had just finished. The 44M experiment report, published in the onion-ai research directory, states the correction plainly: loop-depth extrapolation fails at 44M, and the paired CI proves the 4→8 degradation is significant.

The third failure was in the report itself — caught before it shipped. I reviewed the merged 38-page report on 19 October and rated it 7/10. Honest and well organized, but not shareable yet. My review found real errors:

- The curriculum root-pacing rationale was backwards. The concave square-root schedule rises fastest early, so it gives *more* hard data early — the opposite of what the text claimed. Hypothesis H2 had to be rewritten to match the math.
- Chapter numbers in the introduction did not match the actual layout.
- Four internal cross-references were broken, carried over from the ten separate papers.
- Citations were wrong in four places: Snell et al. 2024 cited for loop extrapolation it does not cover, a Dehghani misattribution, a Wang year mismatch, and Elbayad and Raposo sitting in the bibliography without ever being cited.
- The KV-cache table mixed decimal and binary units.
- Three overstatements: the model called "inert outside trained depth" when the data shows 4.41 → 4.90, loop scaling called "decorative" on the strength of one 44M run, and a "linear in the squared range" that was actually quadratic.
- The same sweep table was repeated across five chapters.

Three review rounds followed: 7/10 → 8/10 → 8.5/10 → 9/10. I landed every fix in the LaTeX sources and rebuilt. The second round caught a KV-cache arithmetic error (0.83 GB corrected to 0.81 GB) and an approximation in Equation 4.3 that is about 6% off at 8 experts. The NeurIPS rebuild caught a layout bug of its own: an equation parenthetical overprinted the right column on page 4, fixed by moving the caveat into the sentence below.

None of this was wasted work. A report that claims to be honest has to survive being read like an examiner reads it. This one did, three times.

## 5. Lessons

**A claim is not a result.** The killer feature was written into the roadmap before any model existed. It took one trained checkpoint and one afternoon of evaluation to falsify it. The order should have been reversed: measure first, claim second.

**One run can falsify, but it cannot generalize.** Everything here rests on a single seed, a single 44M model, a single dataset, a single checkpoint. The confidence intervals cover sampling noise over sequences. They say nothing about run-to-run seed variance. The report names the second seed as the next experiment. It does not pretend the measurement exists.

**Paired tests beat eyeballing intervals.** The 4→8 degradation is small — 0.106 loss. Two overlapping intervals would have left it ambiguous. The paired per-sequence test, CI [+0.101, +0.110], settled it. Design the comparison before you run it.

**Label what is proposed.** The report grew from one executed study into ten chapters. Only the sweep was measured. The rest — MoE routing dynamics, the curriculum protocol, loop-aware distillation, quantization, mechanistic probes, TinyLoop-Bench — is analytical or proposed. The report says so in a status table up front, and the README repeats it: proposed experiments are labelled as proposed. The only executed study is the loop sweep.

**Review catches what authors cannot see.** The root-pacing error, the citation misattributions, the repeated table — all of them survived my own passes. They did not survive me reading the draft as a skeptic. The 7/10 was the most useful grade the report got.

## 6. What changed the plan

The plan changed in three ways.

**First, the fix moved from inference to training.** If depth generalization is the goal, it probably has to be trained in, not assumed. The proposed fix is a loop curriculum: train with randomly varying loop counts, so the model learns to live at many depths instead of one. The report is explicit that this is untested. The conclusion states that randomized-loop-count training is a proposal, not a finding.

**Second, the extrapolation story moved to the 1B run.** The claim now rests on the future 1B loop-curriculum run in the 1B battle plan, not on the 44M result. That is the honest place for it: a bigger model, trained the right way, is the next place the claim can be tested. Until then, it is a hypothesis with a failed small-scale test behind it.

**Third, the reporting standard changed.** The ten separate papers were merged into one technical report on my call — "Into one" — 38 pages, 13 chapters, the sweep as the centerpiece, the model spec stated once, one unified bibliography of 19 references. Then the single-column report was reskinned to a 22-page two-column NeurIPS format, my pick, and published as a public repo on 19 October 2026 with diagrams, heatmaps, a 3D saturation plot, the sweep curve with bootstrap CIs, full LaTeX sources, and a figure-regeneration script.

One more nuance arrived later, and it belongs here because it keeps the story honest. The 125M MiniMythos run, measured on zero-shot accuracy rather than held-out loss, showed the opposite pattern at small scale: HellaSwag rose from 29.5% to 30.0% going from 8 to 16 loops, with PIQA flat at 50.0%. That result is reported only, not verified, and PIQA at 50.0% is exactly chance on a two-choice task — so "stable" there shows no knowledge to lose, not robustness. But the honest picture is that extrapolation looks scale- and task-dependent, not simply true or false. The 44M loss result stands. It does not settle the question for every scale and every task.

## 7. What is still unknown

- The second seed was never run. Seed variance is still unmeasured. [I still need to write this part: did a second seed ever run, and if so, where are the numbers?]
- The sweep was never re-run on the 133M or 500M checkpoints. The October research note planned those re-runs, with the 500M results pull waiting on a Kaggle token. [I still need to write this part: did the 133M/500M re-runs happen, and what did they show?]
- Loop-curriculum training was never attempted at 44M. The before/after story the note sketched — same model, trained with varying loop counts — does not exist yet. [I still need to write this part: was loop-curriculum training ever tried?]
- The 1B loop-curriculum run, where the extrapolation claim now rests, is a plan. No 1B model has been trained. [I still need to write this part: what is the current status of the 1B run?]

---

## Sources

- Live repo README, `themythofsisyphusjayant/RECURRENT-SMALL-LMS` (verified via GitHub Contents API; README revised November 2026, file audit 21 November 2026). Sweep table, bootstrap CIs, GQA arithmetic, repo layout, report contents.
- `~/workspace/openmythos-kaggle/loop_sweep_results.json` — raw sweep numbers and generations, 16 October 2026.
- `~/workspace/openmythos-kaggle/Loop_Depth_Extrapolation_Note.md` — research note, 17 October 2026: setup, interpretation, limitations, next steps.
- `~/workspace/papers/technical-report-neurips/chapters/ch-loop.tex` — sweep protocol, 10,000 bootstrap resamples, paired test.
- 44M experiment report, `onion-ai/research/44m_experiment_report.md` — matched-parameter baseline, compute caveats, the roadmap correction.
- Memory record, October 2026: 44M training run (4,850 steps, 120,004,608 tokens, loss 9.52 → 1.53); ten-paper program merged into one report; my 7/10 review and the three fix rounds to 9/10; NeurIPS 22-page build; repo publication 19 October 2026; MiniMythos 125M reconciliation note.
