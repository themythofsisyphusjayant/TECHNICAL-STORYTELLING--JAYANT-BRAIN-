# Story 05 — OpenMythos: the model that had to earn its depth

Repo: [themythofsisyphusjayant/OPENMYTHOS-RLM](https://github.com/themythofsisyphusjayant/OPENMYTHOS-RLM)

Status: my 44M pilot finished. The 1B run has not started. Training is stopped.

## 1. The problem

A language model gets smarter two ways. More parameters. Or more compute per token.

More parameters costs memory and money. More compute per token costs time, but the weights stay small.

I bet on the second way. One block, reused across iterations. The same weights, run again and again, so the model thinks longer instead of growing wider.

The research question, as the README states it: can a model that reuses one block across iterations buy effective depth more cheaply than a wider model with the same parameter count?

That is the whole story. Everything else is testing whether the bet holds.

## 2. Design decisions

The architecture is Prelude → Recurrent Block × N → Coda. The block's weights are shared across iterations.

- **Prelude:** token embeddings. The from-scratch code uses byte-level embeddings, vocab 260. No tokenizer training needed.
- **Recurrent block (shared):** grouped-query attention plus a SwiGLU feed-forward, or a sparse MoE. Pre-norm residuals. RoPE position embeddings.
- **Coda:** final RMSNorm plus the LM head. Optionally tied to the embeddings.

Two consequences of the design are handled explicitly, not left to luck.

First, truncated backpropagation through time. Gradients flow through at most `grad_recurrence_steps` iterations. State is detached between chunks. Without this, training through 16 iterations would blow up memory and gradients.

Second, per-iteration KV caches. Each iteration attends over a different input distribution, so generation keeps one cache per iteration depth.

I worked in a deliberate order: prove the pipeline on CPU before spending GPU time. A 722K-parameter recurrent LM trained end to end on CPU — data, train, eval perplexity, checkpoint, generate — on the Gutenberg corpus. That pilot exists in `experiments/pilot_cpu.py`. Only after it worked did I send the real run to a GPU.

One decision to be honest about: I did not use the from-scratch code in this repo for the 44M run. I used the `kyegomez/OpenMythos` package (`pip open-mythos`). The report lives in this repo because this is the project's research home. [I still need to write this part — why I chose the packaged implementation over the from-scratch code for the 44M run. The README records the fact, not the reason.]

## 3. The 44M run: it worked, and it changed the question

Stage 0. I ran the 44M model on TinyStories, on a Kaggle T4. My account is `arshhdubeyy`.

The config: dim 512, 6 prelude layers plus 2 coda layers, 4 loops, 8 experts MoE, GQA, GPT-2 BPE tokenizer. 43,974,722 parameters as printed by the run.

The run finished 16 October 2026. 4,850 optimizer steps. 120,004,608 tokens. About 152 minutes. Throughput steady near 13.1k tokens per second. Loss fell from 9.5242 at step 50 to 1.5263 at step 4,850. Learning rate warmed up to 3.00e-04 and cosine-decayed to almost zero. Gradient clipping at 1.0. The recurrent block was contractive — spectral radius 0.3679, under 1 — so the loop was stable by construction, not by hope.

The samples tell the story better than the loss curve. At step 500, broken sentences. At step 4,500, a coherent short-story picnic scene. The model went fluent. That matters, because fluency is the bar everything else in this story gets measured against.

But the headline hypothesis did not survive the run. Test-time depth extrapolation — run the trained model at more loops than it trained on, and watch it get smarter — did not help. Evaluating at depth 8 was significantly worse than the trained depth 4. Past the trained depth, extra loops saturate. They do not compound.

The full loop-depth numbers, confidence intervals, and limitations are in Story 06. The short version: recurrence does real work at the trained depth, and nothing in this run supports extrapolating past it.

A second data point exists, and I report it here — I did not reproduce it. MiniMythos 125M: HellaSwag 29.5% at 8 loops, 30.0% at 16 loops. PIQA flat at 50.0% — chance level on a two-choice task, which shows no PIQA knowledge rather than robustness. The numbers come from a July 2026 one-page summary with no seeds, no confidence intervals, and no eval code. Treat them as reported. The two results do not contradict each other, but they narrow the claim: depth extrapolation currently looks scale- and task-dependent, not settled either way.

## 4. Failed attempts

### Mythos-T4-Max: the 1B-scale model that never got to train

Separate from the repo's 1B config, I built a second 1B track from scratch inside a Kaggle notebook, following a build-prompt PDF. 1,001,900,049 parameters measured on GPU. Dim 896. 720 routed experts plus 12 shared, top-16 per token. Loops 8–12 in training, extrapolate-test at 18. GPT-2 BPE. Data: FineWeb-Edu at 87.5%, CodeAlpaca plus TinyStories at 12.5%. Token budget: a 200M-token cap.

It failed three times in one day, 29 October. I diagnosed each failure from the Kaggle log.

**v1** errored about 33 seconds in, before any training. The Phase 0 gate called `torch.cuda.get_device_properties(0).total`. That attribute does not exist. The correct one is `total_memory`. A one-line bug. I fixed it and re-pushed.

**v2** passed the parameter gate — 1,001,900,049 params, one over the analytic count, the injection-gate scalar — then failed the headroom gate. Peak 15.19 GB on a 15 GB card. Only 0.45 GB headroom. And the gate loss was 418, not the ~10.8 you expect from a random model over a 50,257 vocab. The cause: `nn.Embedding`'s default N(0,1) init with a tied output head, giving logits with standard deviation around 30. I fixed it with embedding init std 0.02 and micro-batch 2, and re-pushed.

**v3** passed both gates. Parameter count right. Gate loss 10.9988, sane. Peak 10.67 GB, headroom 4.96 GB. Data pipeline ready. Optimizer ready. Training started. Then CUDA out-of-memory at about 289 seconds, in `cross_entropy` over the full 50,257 vocab: tried to allocate 198 MiB, only 98 MiB free, 14.26 GiB already allocated. The gate number lied. Optimizer states and accumulated gradients are allocated after the gate, so training VRAM is well above the gate number. Throughput was also fatal: roughly one optimizer step per 209 seconds, mostly data streaming and the per-expert MoE Python loop. The 200M-token cap was not feasible at that rate in one Kaggle session.

My call was the quick fix, not the rewrite: micro-batch 1, grad-accum 32, expandable CUDA segments. **v4** ran on that fix.

The resume saga is where it stalled for good. v4's run ended. I pushed a resume from the step-400 checkpoint. It resumed at step 400, then OOMed. I pushed the fix again. That third resume run was cancelled at step 650 — loss 5.7574, 10.6M tokens. The last saved checkpoint is step 600: 9.8M tokens, loss 6.0205, 6,061,346,503 bytes, verified by download and hash.

Getting that checkpoint back onto Kaggle became its own failure chain. A single-file upload died past 60 minutes. A parts-in-one-dataset upload died the same way. Splitting into six ~1 GB parts did not help: the batch upload script was killed twice by runtime restarts, and the per-part uploads hit 403s and a proxy disconnect — part 00 failed on 2 November after about 67 minutes. All six local parts are intact. No training has started from step 600.

The decision is still open, and it is mine: keep pursuing step 600, or fall back to the already-ready step-400 dataset and lose 200 steps — about 5–6 hours of GPU training, loss back from 6.02 to 6.29. No retry has been recorded since 2 November. No upload or training starts without my word.

### The decoding sweep: no sampling trick can fake fluency

On 21 November, I got a clean answer to a different question. I tested a small Qasida TinyStories checkpoint — 12,608 parameters — to see whether decoding could reach fluency without more training.

First, I corrected the baseline. The README's 385.06 perplexity and the stopped run's 518.88 were measured on earlier checkpoints. They are stale. The current checkpoint measures 158.15 perplexity on held-out windows. I verified a numpy replica of the forward pass two independent ways, including reproducing the README's published init loss exactly, before any sweep ran.

Then the sweep: 39 samples across 15 decoding settings, 3 prompts, 4 seeds. Greedy decoding collapsed to `<unk>` on all 30 generated words. Temperature 0.5 to 1.2, top-k 5 to 40, top-p 0.85 to 0.95 — every sample was word salad. No fluent sentence anywhere.

The reason is structural, not a tuning problem. `<unk>` is 14.8% of all tokens — the single most frequent token id, rarer than any real word. The model's argmax is `<unk>` in essentially every context, so greedy collapses immediately. The rest of the distribution has the right story vocabulary but no syntax. Higher temperature only trades `<unk>`-collapse for random-word salad.

A bigger small model did not save the idea either. The Qasida 153,744-parameter run — 9,000 steps, perplexity 651 — was still not fluent.

The verdict: decoding cannot reach fluency. Fluency needs scale. The reference point is the 44M run itself: fluent at step 4,850, loss 1.5263, 119.2M tokens, 152.2 minutes on a Kaggle T4. What the small-model path needs is a 44M-class model, about 120M tokens, and about 2.5 T4-hours on Kaggle. That needs my fresh Kaggle token and explicit per-run authorization. It is not feasible on this VM: 1.5 shared CPUs, no GPU, and the scalar forward takes about 5 minutes per window.

### The stop

On 8 November, I said: "Stop the training nd completely focus on the programming language."

I killed the TinyStories trainer and verified it stopped. The final checkpoint survived at window 315, epoch 0 — training could resume later from there, but it is not running. The watcher cron was disabled. Focus shifted completely to Qasida self-hosting.

Training stays stopped on my word. The decoding sweep ran no training, used no Kaggle token, and pushed nothing.

## 5. Lessons

**Measure the checkpoint you have, not the one you remember.** The README's perplexity was stale by more than a factor of two. The current checkpoint was far better than the published number — and still not fluent. Both facts mattered, and only a fresh measurement found either.

**A gate that runs before training cannot see training memory.** The v3 headroom gate passed with 4.96 GB to spare. Training OOMed anyway. Optimizer states and accumulated gradients arrive after the gate. A gate that does not include them is a hope, not a check.

**Throughput is a feasibility result, not a detail.** One optimizer step per 209 seconds made the 200M-token cap impossible in one Kaggle session, before any loss curve existed. Speed has to be measured in the first minutes, not discovered at the end.

**Decoding is not a substitute for training.** When the distribution itself lacks sentence structure, no sampling scheme can extract fluent sentences from it. The sweep proved a negative cleanly, and a clean negative is worth having: it closed a line of work instead of letting it linger.

**State efficiency claims in compute, not parameters.** At depth 4, the loop block runs four times per token. The 44M model costs roughly four times the loop-block compute of its own 44M unlooped baseline. Any efficiency claim has to be stated in tokens and GPU-hours.

**Single-seed results are provisional.** The 44M run is one seed. A second seed is the main open gap. The loop-depth verdict, the loss numbers, all of it rests on one run.

## 6. What changed the plan

The first target was 3B. On 16 October, I pivoted to 1B: beat the open 1B class at any cost. [I still need to write this part — what made 1B the right target instead of 3B. The pivot and the quote are on record, the reasoning behind it is not.]

The targets, from the landscape survey in `research/3b_landscape.md`: TinyLlama-1.1B (3T tokens), OLMo-2-1B (4T), Llama-3.2-1B (9T), Qwen2.5-1.5B (18T), SmolLM2-1.7B (11T). The evals to win: MMLU, GSM8K, MATH, ARC-Challenge, HellaSwag, HumanEval/MBPP. Same harness, same few-shot setups as the reported numbers. No cherry-picking.

The plan is written in `research/1b_battle_plan.md`, and it is honest about what it is. Three stages:

- **Stage 0:** the 44M pipeline validation on a Kaggle T4. 120M tokens. Free. Done.
- **Stage 1:** a 1B proof run — beat TinyLlama and DCLM-1B, demo loop extrapolation. 100B+ tokens. About 1–2K H100-hours. Needs a cluster or a grant.
- **Stage 2:** the full 1B — staged data plus the post-training ladder. 2–4T tokens. About 10–20K H100-hours. Needs a partnership.

Stage 1 is the milestone that earns Stage 2. A 1B proof run cannot happen on the Kaggle free tier — 100B tokens is months on a T4.

The 1B config exists and is verified: `configs/mythos_1b.py`, 1,056,094,338 parameters, forward pass smoke-tested on CPU. Dim 2048, GQA 16/4, 64 experts with 8 active, 64K custom BPE, 4K context, 16 max loops, loop-depth curriculum — 2 loops early, 4 through the middle, anneal at 8, eval sweep at 4, 8, 16.

Two caveats, stated plainly in the README. The config imports a model module (`open_mythos.main`) that is not in this repo, and the loop-curriculum training code it refers to is not here either. The config is verified for its parameter count, not as a runnable training setup. And the extrapolation claim the roadmap rests on is unproven: the 44M run contradicts it on TinyStories loss, and the 125M report supports it only weakly on HellaSwag.

The architecture for the 1B run borrows two stability imports, documented in `patches/` against upstream: QK-Norm, because a looped block re-processes tokens up to 16 times and attention-logit blowup across iterations is the number-one instability risk; and a sliding-window interleave, so the run does not burn FLOPs on full attention times 16 iterations.

The data plan is SmolLM3-shaped: mostly web early, more code and math later, reasoning and instruction data in the anneal. The recipe copies the winners: AdamW (0.9, 0.95), warmup-stable-decay, peak LR around 4e-4, batches near 2M tokens per step, bf16 plus FSDP. The post-training ladder — SFT, then DPO/APO, then RLVR/GRPO on verifiable math and code rewards — is budgeted as part of the run, not an afterthought. That is where the MATH points live.

What the 44M run changed, concretely: the loop-depth curriculum is now the experiment, not the assumption. Fixed-depth training at 44M says extrapolation hurts. Whether training across depths changes that is an open hypothesis. The 1B loop-curriculum run is the experiment that would decide it, and it has not been run.

## 7. What is still unknown

- Whether the loop-depth curriculum changes the extrapolation verdict. Open hypothesis, not a finding.
- Whether a second seed confirms the 44M loop-depth result. Single seed so far.
- Whether MiniMythos 125M reproduces at all. Reported, not verified.
- The step-600 versus step-400 decision for Mythos-T4-Max. Pending my word since 2 November.
- [I still need to write this part — which 1B track is the plan now: the Mythos-T4-Max notebook model, or the repo's `mythos_1b.py` config? Two 1B tracks exist. No source says which one continues.]
- [I still need to write this part — why I used the packaged `kyegomez/OpenMythos` implementation for the 44M run instead of the from-scratch repo code.]
- [I still need to write this part — what made 1B the right target instead of 3B.]

## Sources

- Live repo README, `research/1b_battle_plan.md`, `research/3b_landscape.md`, `configs/` — themythofsisyphusjayant/OPENMYTHOS-RLM, read 21 Nov 2026.
- Decoding sweep summary — `~/workspace/gpt2eval/decode-20261121-61008/SUMMARY.md` (21 Nov 2026).
- 44M run record, Mythos-T4-Max v1–v4 and resume saga, 8 Nov stop — `~/MEMORY.md` and `~/memory/2026-10-16.md`, `2026-10-29.md`, `2026-10-30.md`, `2026-11-01.md`, `2026-11-02.md`, `2026-11-08.md`, `2026-11-14.md`, `2026-11-21.md`.
- My Kaggle account is `arshhdubeyy`. I used tokens transiently via environment only, never stored; future Kaggle work needs me to re-supply a fresh token plus give explicit per-run authorization.
