# Week 12 Synthesis — Knowledge Gap Formulation

**Participant:** Nahom Desalegn (nahoma@10academy.org)  
**Program:** 10 Academy TRP1, Week 12  
**Dates:** May 5–9, 2026

> **Note:** Day 5 is in progress (May 9). The Day 5 entries below are placeholders to be completed before the 21hr UTC submission deadline.

---

## What the week demanded

Weeks 0–11 produced working systems. Week 12 demanded I account for them. The mechanism — one named gap, one researched gap, one published explainer, one grounding commit, every day — turns out to be a brutally efficient audit. You cannot fake a sharpened question: the four-property rubric (diagnostic, grounded, generalizable, resolvable) forces you to locate the specific artifact where your language papers over a mechanism you do not actually understand. The pairing structure adds accountability. A question you cannot defend to your partner in a morning call is a question you were not ready to ask. By the end of the week I had committed five concrete edits to shipped portfolio work, each traceable to something I now understand mechanistically that I only understood superficially before.

---

## Five Gaps I Named

### Day 1 — SimPO preference accuracy vs. rejection sampling lift (with Yakob Dereje)

**Gap:** The SimPO critic achieved 96.9% preference accuracy on held-out validation pairs but produced near-zero lift — ΔA = +0.0025, p=0.40, 17/52 ties — when used as a rejection-sampling gate. I could describe the numbers but could not explain the mechanism.

**What closed it:** Yakob's explainer identified the root cause as margin distribution mismatch. Training pairs are constructed with large, deliberate quality gaps; deployment candidates generated from the same model at fixed temperature have near-zero score variance. The critic can distinguish excellent from poor, but it cannot rank eight outputs that are nearly identical in quality. The right diagnostic is `std(candidate_scores)`: if it falls below ~0.1, the bottleneck is generation diversity, not critic quality. The 96.9% preference accuracy is not wrong — it correctly describes the critic's ability on its training distribution. It just does not predict rejection-sampling lift on a different distribution.

### Day 2 — Tool description influence on tool-name token selection (with Mikias Dagem)

**Gap:** I used text-completion in `compose_email` and function-calling elsewhere in the agent pipeline, but I could not give a mechanistic account of why the choice mattered at the token level. I knew descriptions went into context; I did not know *when* they influenced generation.

**What closed it:** Tool schemas are serialized into the context window before prefill completes. Because the transformer computes attention across the full context before committing any token, description strings are causally upstream of the tool-name logit — they shape which tool is selected, not merely how arguments are filled. Some inference engines apply constrained decoding at the name position, which shifts when description precision matters most, but the fundamental causal path runs through prefill. This made the existing choice in `compose_email` (no `tools` parameter) more defensible than I knew — every unused schema consumes context tokens and attends into generation — and gave me a principled account of when to switch.

### Day 3 — LoRA rank allocation across modules (with Gemechis Worku)

**Gap:** `train_critic.py` uses r=16, alpha=32, targeting all seven projection modules — the Unsloth default. I could not say whether this was appropriate for a 306-pair preference task, or explain what each module class actually contributes to preference discrimination.

**What closed it:** Attention projections (Q, K, V, O) learn which patterns to attend to — surface features like value-proposition density or generic language markers. MLP projections (gate, up, down) learn decision logic — how attended patterns map to rubric criterion scores. Spreading r=16 across seven modules gives each roughly 44 training examples of gradient signal: enough to learn shortcuts, not enough to learn nuanced rubric distinctions. For 306 pairs, the defensible configuration concentrates rank on `v_proj + down_proj` at r=32. The null held-out result (ΔA = +0.0025) is consistent with shortcut learning under gradient diffusion — the model learned to distinguish training pairs correctly but did not generalize to near-identical deployment outputs.

### Day 4 — LLM-as-judge biases in single-output absolute scoring (with Melkam Beyene)

**Gap:** `scoring_evaluator.py` uses an LLM sub-judge (OpenRouter Sonnet 4.6) for the `voice_adherence` dimension. The held-out Δ A = +0.0025 is narrower than any plausible systematic judge bias. I could list three bias types (verbosity, position, self-enhancement) but could not reason about which dominate in a single-output setup or articulate why the size of Δ A changes the epistemic status of the result.

**What closed it:** Melkam's explainer introduced two contributions. First, the bias taxonomy for single-output scoring: verbosity bias and style-prior bias dominate (position bias requires a pairwise in-prompt reference to activate). A verbosity bias of +0.01 on reranked outputs — plausible given that reranking selects "more complete" candidates — fully explains the observed lift without any true quality difference. Second, the two-uncertainty split: paired bootstrap CIs measure sampling variance only. They say nothing about judge measurement variance — what happens if you paraphrase the rubric, change the judge model, or shift temperature. When |Δ A| is on the order of 10⁻³ and published judge fragility estimates put systematic bias at ±0.5–1%, the second source dominates. The correct framing is "under-identified at this margin," not "null result."

### Day 5 — [To be completed May 9, 2026]

**Gap:** [To be added after morning call]  
**What closed it:** [To be added after evening call and signoff]

---

## Five Gaps I Researched

### Day 1 — LoRA forward pass mechanics (for Yakob Dereje)

Yakob's question: how does LoRA actually modify the transformer forward pass, and is system-prompt simulation a valid proxy for real adapter inference? The core of the explainer was the weight-space decomposition: `W_eff = W₀ + (α/r)·B@A`. The adapter modifies the weight matrix; system-prompt simulation prepends activations — they operate on different parts of the computation graph. Merged and unmerged inference are numerically identical (difference ≤ floating-point epsilon). The validity boundary: for rule-based rubric dimensions (string matching), system-prompt simulation is a valid proxy; for distributional dimensions scored by an LLM sub-judge, it is unvalidated and the claim in a model card should reflect that.

### Day 2 — Retry logic, exponential backoff, and idempotency (for Mikias Dagem)

Mikias asked about transient vs. permanent API failure, exponential backoff with jitter, retry budgets, and whether the same strategy applies across `email_handler.py`, `calendar_handler.py`, and `enrichment.py`. The critical insight was the idempotency asymmetry: email and scraper reads are safe to retry; the booking call in `calendar_handler.py` is not, because a missing idempotency key on a mutating endpoint means a retry creates a duplicate booking. The retry strategy cannot be copy-pasted across files. The AWS empirical result (full jitter reduced retry call volume by 75% vs. fixed-interval) grounds the recommendation as engineering evidence, not just convention.

### Day 3 — DPO reward model overoptimization (for Gemechis Worku)

Gemechis asked about the analog of reward model overoptimization in DPO and why it produces the specific pattern of mean score rising while pass rate falls. The explainer grounded this in the implicit reward derivation: `r*(x,y) = β·log(π_θ(y|x)/π_ref(y|x))`. The policy ratio is the reward; β is the KL anchor. The mean-up / pass-down fingerprint is the signature of soft-dimension proxy misalignment — preference pairs were labeled on continuous rubric dimensions, hard-fail categories had weak margin signal, so DPO optimized the wrong objective. The corrective dials are increasing β, using pass_rate as the checkpoint criterion, and IPO (squared loss that saturates to prevent unbounded margin growth).

### Day 4 — Paired vs. regular bootstrap CIs (for Melkam Beyene)

Melkam asked which CI method is correct for a benchmark where the same test examples score multiple methods, and whether picking the wrong one could flip the deploy verdict. The load-bearing mechanism is the dropped covariance term: `Var(A−B) = Var(A) + Var(B) − 2·Cov(A,B)`. Regular bootstrap computes only `Var(A) + Var(B)` — it inflates variance by the correlation between conditions. At ρ=0.85, unpaired CIs are 2.6× wider than they should be. The deploy-flip example made it concrete: with n=52 and Δ=+0.035, paired CI is [+0.005, +0.065] (deploy); unpaired is [−0.042, +0.112] (don't deploy). Same data, wrong method, wrong product decision.

### Day 5 — [To be completed May 9, 2026]

**Question received from:** [Partner TBD]  
**Key points delivered:** [To be added]

---

## Most Surprising Insight

The most structurally important insight of the week came from Day 4's two-uncertainty split. I had been implicitly treating a paired bootstrap CI as a complete uncertainty statement. Melkam's explainer showed it is half of one: it measures sampling variance (what if you re-drew 52 tasks?) but says nothing about judge measurement variance (what if you paraphrase the rubric, swap the judge model, or shift temperature?). When the margin is 10⁻³ and published LLM judge fragility puts systematic bias at ±0.5–1%, the second term dominates the first. The "null result" I had been reporting for the critic is not wrong — but "under-identified at this margin" is the correct epistemic claim. This distinction matters everywhere a small Δ sits inside a CI: before calling something a null result, the question is whether the measurement instrument is precise enough to identify an effect at that scale.

---

## Canonical Reading List

1. **Lambert et al. (2024)** — *RewardBench* (arXiv:2403.13787) — why high preference accuracy frequently fails to predict downstream task performance; essential for anyone using a reward model or critic as an evaluation gate.
2. **Stiennon et al. (2020)** — *Learning to summarize with human feedback* (arXiv:2009.01325) — foundational treatment of Best-of-N / rejection sampling and reward model-based selection.
3. **Hu et al. (2021)** — *LoRA: Low-Rank Adaptation of Large Language Models* (arXiv:2106.09685) — the primary source; Section 4 on why low-rank works and Section 7 on rank sensitivity are the load-bearing parts for FDE configuration decisions.
4. **Rafailov et al. (2023)** — *Direct Preference Optimization* (arXiv:2305.18290) — the implicit reward derivation (Section 4); essential for understanding what DPO is actually optimizing and where overoptimization comes from.
5. **Gao et al. (2023)** — *Scaling Laws for Reward Model Overoptimization* (arXiv:2210.10760) — empirical characterization of the proxy/true-objective divergence as a function of KL budget.
6. **Azar et al. (2023)** — *A General Theoretical Paradigm to Understand Learning from Human Feedback* (arXiv:2310.12036) — IPO paper; proves DPO's logistic loss has no saturation and motivates the squared-loss fix.
7. **Zheng et al. (2023)** — *Judging LLM-as-a-Judge with MT-Bench and Chatbot Arena* (arXiv:2306.05685) — canonical taxonomy of LLM judge failure modes (position, verbosity, self-enhancement bias).
8. **Wang et al. (2025)** — *Judging the Judges: A Systematic Study of Position Bias in LLM-as-a-Judge* (ACL 2025, IJCNLP) — empirical study of bias magnitude variation across judge models and task types.
9. **Brooker (2015)** — *Exponential Backoff and Jitter* (AWS Architecture Blog) — empirical case for full jitter; simulation results showing 75% retry storm reduction.
10. **Beyer et al. (2016)** — *Site Reliability Engineering*, Chapter 21 — retry budgets, load shedding, backpressure; the authoritative FDE reference for production API reliability.
11. **Berg-Kirkpatrick, Burkett & Klein (2012)** — *An Empirical Investigation of Statistical Significance in NLP* (EMNLP 2012) — paired vs. two-sample bootstrap for same-test-set NLP comparisons; foundational for anyone reporting CIs on benchmark results.

---

## Tool List

| Tool | Day | What it demonstrated |
|---|---|---|
| NumPy simulation (seed=42) | Day 1 | Margin distribution mismatch: 12.5× gap between training margins and deployment pool margins |
| `email_handler.py` / `calendar_handler.py` side-by-side read | Day 2 | Idempotency asymmetry discovered live — the missing retry in `calendar_handler.py` is a bug, not a design choice |
| `trl` DPOTrainer (β sweep: 0.01, 0.1, 0.5) | Day 3 | Divergence between mean score and pass_rate appears first at low β; both metrics track together at β=0.5 |
| `scipy.stats.bootstrap` (paired vs. unpaired) | Day 4 | CI width ratio reproduced on synthetic n=52 benchmark; 2.6× inflation confirmed at ρ=0.85 |
| `scipy.stats.spearmanr` score–length correlation | Day 4 | Retroactive verbosity diagnostic against `ablations/held_out_traces.jsonl` |
| [Day 5 tool] | Day 5 | [To be added] |
