# Story 01 — Provenance-Aware Belief Aggregation

**Project:** [PROVENANCE-AWARE-BELIEF-AGGREGATION](https://github.com/themythofsisyphusjayant/PROVENANCE-AWARE-BELIEF-AGGREGATION) · **Manuscript:** [paper/main.pdf](https://github.com/themythofsisyphusjayant/PROVENANCE-AWARE-BELIEF-AGGREGATION/blob/main/paper/main.pdf) (17 pages) · **Plain-language version:** [EXPLAINED_FOR_BEGINNERS.md](https://github.com/themythofsisyphusjayant/PROVENANCE-AWARE-BELIEF-AGGREGATION/blob/main/EXPLAINED_FOR_BEGINNERS.md)

**Status as of 21 Nov 2026.** Manuscript complete, not peer-reviewed, not published. Preprint (Zenodo) still open. Target: AAMAS 2027 workshop, Hanoi, 3–4 May 2027; recommended workshop deadline 24 January 2027.

---

## 0. Where it started

> **[I still need to write this part — what the existence value paradox actually was]** — I started this work from the "existence value paradox" and ended somewhere completely different. No source I could find — not the paper, not the repo, not the notes — says what that paradox was. This section is mine to fill in: what the paradox was, and what made me chase it.

What the sources do show is where the chase landed: a question about what a message means, not who to trust.

---

## 1. The problem

Multi-agent systems aggregate messages by weighting them on source reliability. The standard move is a weighted vote: `ŷ = sign(Σ w_s X_s)`, nonnegative weights, more reliable sources count for more.

That answers the wrong question when sources report through different conventions — mappings from internal readings to messages that can invert the signal or transform it depending on context.

The paper opens with one sentence: *"The bridge is safe."* Heard from a structural engineer, it licenses crossing. Heard from a five-year-old, it licenses nothing. Heard from an anonymous account, it licenses suspicion. The message is identical; the decision it supports is not.

Take a source that is perfectly reliable — its internal reading always matches the truth — but which reports through a private convention: it flips its reading, or XORs it with an observable context bit. Its messages are uninterpretable without the convention, and no nonnegative weight, however carefully estimated, repairs the aggregation. Reliability and convention are orthogonal. A source can be maximally trustworthy and still unreadable.

That is the problem, named in the paper as **convention heterogeneity**. It is distinct from the reliability heterogeneity the literature had studied. The decision-theoretic value of provenance is the change in decision value from knowing who sent the message: with the source known, the receiver can decode through that source's convention; without it, the message may be uninterpretable.

---

## 2. Design decisions

**Model the convention, not just the reliability.** I formalised the problem as a generative latent-variable model in which source identity is part of the process that generates each message — with involutive conventions, a stability parameter ρ, and reliability kept strictly separate from convention.

**Eight conventions, all involutions.** I used eight conventions in the benchmark (identity, NOT, XOR with context bits and their combinations), all satisfying f(f(x, a), a) = x. The aggregator relies on that property to recover the underlying reading from an observed message.

**A Bayesian aggregator that decodes first.** Instead of weighting messages directly, I built an aggregator that treats each source's reporting convention as a latent variable and decodes it jointly with the underlying reading. It enumerates a fixed grid over convention stability (ρ) and reliability (r) in full — no sampling, no randomness in the aggregator itself.

**Prove the limits, don't just beat baselines.** I proved three theorems, not one: nonnegative-weight insufficiency (no nonnegative weighting can recover the truth under convention heterogeneity), scalar insufficiency under context-dependent conventions (a single scalar per source cannot capture a convention that conditions on context), plus equal-reliability optimum and posterior consistency (the aggregator converges on the observational-equivalence class of the truth as evidence accumulates).

**Test it like you don't believe it.** I ran ten lettered experiments (A–I plus an LLM communication benchmark and a real-human-judgment CIFAR-10H study), headline tables banked over 190 seeds (A: 100; B/C/E: 30 each), 95% confidence intervals throughout. Reproducibility built in from the start: seeded RNGs (`SafeRNG`), smoke tests, a brute-force check of the posterior-predictive fix, one-command regeneration with `bash reproduce.sh`.

---

## 3. Failed attempts

### The theorem that was too strong

The original Theorem 2 was stated too strongly. A skeptical audit caught it: only the repaired, narrower version is proved. On 21 Oct 2026 I ordered "Fix everything", and Theorem 2 was restated (log-odds form ρ > 0; decision form ρ > 1/2 + dominance) with a new BSC-channel proof, propagated through the abstract, introduction, discussion and conclusion.

The lesson was not "we made a mistake". The lesson was that the audit was the work — the paper got *more* trustworthy by having its central claim narrowed.

### The audit found wrong numbers everywhere — including in the audit

The repair pass corrected numbers the paper had printed wrong:

- Posterior averaging was worth **0.6 points, not 2.6**.
- The CtxLogReg sweep was corrected (K=2: 0.534 → K=50: 0.713; the printed "0.63" was the K=10 value).
- TransformerMeta was **not** capacity-matched (104,834 vs 25,667 parameters) — its result is a deficit, not a tie.
- BetaBinomial was renamed ContextMatchFreq, because it was context-matching, not a beta prior.
- The EMDecoder gap decomposes as 4.7 points rule + 1.9 fit quality + 0.6 averaging.
- Convention recovery: grid-MAP 0.640 vs EM 0.393.
- The yin2018 reference was the wrong paper entirely; replaced with the real ICML 2018 paper (third author Ramchandran). All 19 references were verified against publisher records — and the audit itself had two page numbers wrong (Patrini, Kim–Ghahramani).

A genuine iterated Dawid–Skene EM had to be implemented (DS_ITERS=100, tolerance 1e-6 — the cap matters: the 5-judgment case needs 92 iterations).

### The first validation run was thrown away

PVI-REAL Study A (virtual RCT, the paper's own machinery) had its first run **discarded**: a support-order bug in the new harness, caught by the negative control. The rerun was clean. The control did its job — that is what controls are for.

### ChaosNLI was excluded, definitively

The release has **no annotator IDs at all** — verified on every item, with the official link dead and the mirror data genuine. A dataset without annotator IDs cannot test per-source behaviour. It was cut, not worked around.

### Calibration was the bottleneck, not accuracy

On real CrowdTruth medical judgments, the context decoder beat the best scalar on accuracy (+2.2 points) but **lost on log loss at every shrinkage level**. Both weighted methods were miscalibrated raw (ECE ~0.145). One temperature fitted on support-only flipped the log-loss result — and the per-family split showed all of the gain was in the treat family (+4.3), with the cause family null (−1.0). The fix became a design change: calibration built into the aggregator, cross-fitted, and the RelDir replication confirmed it (log loss +0.101, accuracy +3.4, both families gaining).

### The adversarial suite priced the assumptions

- Out-of-set convention (c=8): simpler methods beat PVI, though PVI still beat scalar.
- Drift between support and query: PVI slips behind on log loss at K=40 — stale-support overconfidence.
- Mimic-then-flip: **every** support-based method collapses to chance (log loss 1.3–2.9), oracle 0.089. Stationarity is a priced assumption.
- Reliability-only context: majority vote is near-oracle; context methods add variance.
- CIFAR-10H: majority vote is already near ceiling and source modelling gains nothing. A boundary condition, stated as one.

### The honest losses, kept in the paper

- The best learned calibrator **beats Bayes** in Experiment B (MetaAttn+Dec, +0.0649).
- The Bayesian posterior grows **overconfident** as conventions stabilise — reported as a limitation, not hidden.
- At low convention stability (ρ = 0.5), a population-average mapping **beats** per-source Bayesian decoding (+0.0196 at K=10). Decode only when conventions are stable enough; at ρ ≥ 0.6 decoding wins decisively.

---

## 4. Lessons

**Reliability is not convention. Weighting evidence is not decoding evidence.** The one-line version of the whole paper. A weight answers "how much do I trust this source". A convention answers "what did this source mean". When the second question is unanswered, the first one cannot save you.

**Scope the claim to what you proved.** The theorems are proved inside the model's assumptions only. Real-world universality is NOT proved — experiments support it, CIFAR-10H is a sanity check, the LLM section is suggestive only. The paper says so, in print.

**Audit your own numbers, then audit the audit.** The repair pass found errors in the paper and in the audit that found them. Nineteen references, checked against publisher records. Every descriptive statistic re-verified from raw data.

**A negative control that catches a bug is a success, not a delay.** Study A's first run was discarded because the control caught the harness bug. That is the system working.

**Calibration is a first-class problem, not a post-processing step.** The real-data validation turned on it: accuracy said "we win", log loss said "you're miscalibrated", and the fix had to go inside the aggregator.

**Report the losses.** The learned calibrator beating Bayes, the overconfidence at high stability, the ρ = 0.5 boundary, the CIFAR ceiling — all in the paper. A result you hide is a result someone else finds.

---

## 5. What changed the plan

| When | What changed | Why |
|---|---|---|
| Start | I began this work from the "existence value paradox" | [I still need to write this — see §0] |
| During the build | The paper became two impossibility theorems + a constructive aggregator, not a better weighting scheme | The problem turned out to be interpretation, not trust |
| 21 Oct 2026 | Skeptical audit → "Fix everything" repair pass | Theorem 2 too strong; numbers wrong across the paper; references wrong |
| 21 Oct 2026 | PVI-REAL validation programme ("Perform every experiment") | The synthetic benchmark was not enough evidence — controlled panels, real CrowdTruth judgments, adversarial failure map added as §6.13; paper grew 16 → 17 pages |
| 21 Oct 2026 | Calibration built into the aggregator (Experiment Pack II) | Study B showed calibration, not accuracy, was the bottleneck |
| 8 Oct 2026 | AAMAS 2027 main track closed → workshop route | Deadline missed; workshop paper (Hanoi, 3–4 May 2027) is the route, recommended deadline 24 Jan 2027 |
| Still open | Zenodo preprint upload; LLM panel re-run (needs OpenRouter credit — zero balance); human RCT (needs participants, consent, payment) | Blocked on my sign-in/approval and on spend approval, not on the science |

The paper that exists now is not the paper I started. I started from a paradox about value, and it became a proof that trust weights cannot decode conventions, survived an audit that narrowed its central theorem and corrected its numbers, and then grew a validation programme that tested it against real human judgments and adversarial attacks. Each change made the claims smaller and the evidence stronger.

---

## Sources

- Paper repo: [PROVENANCE-AWARE-BELIEF-AGGREGATION](https://github.com/themythofsisyphusjayant/PROVENANCE-AWARE-BELIEF-AGGREGATION) — README, `paper/sections/`, `code/`, `results/`, `figures/`, `reproduce.sh`
- Manuscript of record: `PVI_Paper_Jayant_Dubey_REVISED.pdf`, 17 pages (21 Oct 2026)
- Beginner explainer: [EXPLAINED_FOR_BEGINNERS.md](https://github.com/themythofsisyphusjayant/PROVENANCE-AWARE-BELIEF-AGGREGATION/blob/main/EXPLAINED_FOR_BEGINNERS.md)
- Validation programme: `PVI_REAL_Results.pdf`, `PVI_REAL_Deep_Analysis.pdf`, `PVI_REAL_Experiment_Pack.pdf`
- Headline numbers: Experiment A (ρ = 1.0) 0.7576 ± 0.0021; B 0.7590 ± 0.0044; C (K = 50) 0.8204 ± 0.0029; E (12 agents) 0.9489 ± 0.0114 — 190 seeds
