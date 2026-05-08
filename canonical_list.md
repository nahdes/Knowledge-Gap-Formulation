# Canonical List — Week 12 Cohort Contribution

**Contributor:** Nahom Desalegn (nahoma@10academy.org)  
**Program:** 10 Academy TRP1, Week 12

> Annotated papers, tools, and patterns worth every Forward-Deployed Engineer reading. Each entry names what it is, why it matters in FDE work, and what specifically to read.

---

## Papers

### Evaluation and Benchmarking

**Lambert et al. (2024). RewardBench: Evaluating Reward Models for Language Modeling.**  
arXiv:2403.13787 — https://arxiv.org/abs/2403.13787  
*Why it matters:* If you use a trained reward model or critic as an evaluation gate (Best-of-N, rejection sampling, preference pair labeling), this paper explains why high reward model accuracy frequently fails to predict downstream task improvement. The gap between preference accuracy and task lift is not a fluke — it is a systematic property of distribution mismatch between training pairs and deployment candidates. Directly applicable to any FDE who reports "96% reward model accuracy" as a quality signal.  
*What to read:* Sections 3–5 (benchmark construction and results); Figure 2 (preference accuracy vs. task improvement scatter).

---

**Zheng, Lianmin et al. (2023). Judging LLM-as-a-Judge with MT-Bench and Chatbot Arena.**  
NeurIPS Datasets and Benchmarks, 2023 — arXiv:2306.05685 — https://arxiv.org/abs/2306.05685  
*Why it matters:* The canonical taxonomy of LLM judge failure modes. Names three bias types — position bias, verbosity bias, self-enhancement bias — and explains which setups each affects. The single-output vs. pairwise distinction is the load-bearing detail: position bias only activates in pairwise setups (requires an in-prompt reference); verbosity and style-prior biases dominate in absolute scoring. Every FDE building an eval pipeline with an LLM judge needs this taxonomy before they trust a margin.  
*What to read:* Sections 3–4 (bias taxonomy and mitigation proposals); Table 3 (inter-judge agreement results).

---

**Wang et al. (2025). Judging the Judges: A Systematic Study of Position Bias in LLM-as-a-Judge.**  
IJCNLP 2025 (long paper) — ACL Anthology: https://aclanthology.org/2025.ijcnlp-long.18/  
*Why it matters:* Empirical follow-up to MT-Bench that quantifies how much position bias varies by judge model family and task type. Key FDE insight: bias magnitude is not fixed — a Sonnet-class judge on outputs from a Qwen-family model introduces style-prior risk that a same-family judge does not. You cannot assume your judge is calibrated across all output distributions.  
*What to read:* Sections 4–5 (results across model families); Section 6 (swap-averaging as mitigation).

---

**Berg-Kirkpatrick, Burkett & Klein (2012). An Empirical Investigation of Statistical Significance in NLP.**  
EMNLP 2012  
*Why it matters:* The authoritative empirical case for paired bootstrap in NLP evaluation. Shows that for same-test-set comparisons, unpaired methods are consistently anti-conservative: they overstate variance and bias practitioners toward keeping the status quo even when a new method genuinely wins. When you run the same eval set across multiple methods, paired bootstrap is correct by construction — the covariance term is real and dropping it is an error, not a conservative choice.  
*What to read:* Section 3 (paired vs. two-sample bootstrap derivation); Table 2 (divergence rates in practice).

---

### Training and Post-Training

**Hu et al. (2021). LoRA: Low-Rank Adaptation of Large Language Models.**  
arXiv:2106.09685 — https://arxiv.org/abs/2106.09685  
*Why it matters:* The primary source. Most FDEs use LoRA via a wrapper (Unsloth, PEFT, trl) without reading the paper, which means they inherit defaults without understanding the tradeoffs. Section 7 on rank sensitivity is the part most people skip — it is the part that explains when low rank fails and how to diagnose it.  
*What to read:* Section 4 (why low-rank works — intrinsic dimensionality argument); Section 7 (ablations on rank and target modules); Table 6 (module contribution breakdown).

---

**Rafailov et al. (2023). Direct Preference Optimization: Your Language Model is Secretly a Reward Model.**  
NeurIPS 2023 — arXiv:2305.18290 — https://arxiv.org/abs/2305.18290  
*Why it matters:* DPO is widely used but its implicit reward model (`r* = β·log(π_θ/π_ref)`) is rarely internalized. If you do not know what DPO is optimizing, you cannot diagnose overoptimization, choose β, or interpret the mean-up/pass-down training pattern. Section 4 is the derivation that makes everything else legible.  
*What to read:* Section 4 (implicit reward reparameterization); Section 6 (β sensitivity analysis).

---

**Gao et al. (2023). Scaling Laws for Reward Model Overoptimization.**  
ICML 2023 — arXiv:2210.10760 — https://arxiv.org/abs/2210.10760  
*Why it matters:* Empirical characterization of what happens when you optimize a proxy reward too hard. The proxy/true-objective divergence appears as a function of KL budget — the "RM score up, gold score down" pattern is predictable, not mysterious. Directly applicable to any FDE whose agent evaluation uses a trained judge or reward model as the optimization target.  
*What to read:* Figure 1 and Figure 3 (proxy vs. gold score divergence); Section 4 (KL scaling).

---

**Azar et al. (2023). A General Theoretical Paradigm to Understand Learning from Human Feedback.**  
arXiv:2310.12036 — https://arxiv.org/abs/2310.12036  
*Why it matters:* The IPO paper. Proves that DPO's logistic loss has no regularization against unbounded margin growth — the gradient does not saturate — which is why β tuning is load-bearing rather than cosmetic. The squared-loss fix (IPO) directly addresses the overoptimization failure mode without requiring a separate reward model.  
*What to read:* Section 3 (proof that DPO loss does not saturate); Theorem 1 (motivation for squared loss); Section 5 (IPO vs. DPO empirical comparison).

---

**Stiennon et al. (2020). Learning to Summarize with Human Feedback.**  
NeurIPS 2020 — arXiv:2009.01325 — https://arxiv.org/abs/2009.01325  
*Why it matters:* The foundational treatment of Best-of-N / rejection sampling with a learned reward model. If you are using a critic as a rejection-sampling gate, this paper is where the pipeline pattern originates. The gap between reward model accuracy and downstream task quality first appears clearly here.  
*What to read:* Section 3 (reward model training and Best-of-N); Section 5 (human evaluation results vs. reward model scores).

---

### Production Patterns

**Brooker, M. (2015). Exponential Backoff and Jitter.**  
AWS Architecture Blog — https://aws.amazon.com/blogs/architecture/exponential-backoff-and-jitter/  
*Why it matters:* The empirical case for full jitter over fixed-interval and truncated exponential backoff. Includes simulation results: full jitter reduced retry call volume by 75% and client-side errors by 98% compared to fixed-interval retry. The FDE takeaway is that retry storm suppression is not a theoretical concern — it is measurable and the fix is four lines of code.  
*What to read:* The whole post (short); the simulation graph comparing retry strategies.

---

**Beyer, B., Jones, C., Petoff, J., & Murphy, R. (Eds.). (2016). Site Reliability Engineering. O'Reilly Media. Chapter 21: Handling Overload.**  
*Why it matters:* The authoritative treatment of retry budgets, load shedding, and the distinction between per-call retry limits and system-level backpressure. The retry budget concept — a ceiling on the total fraction of requests that can be retries at any given time — prevents retry amplification from turning a partial outage into a full one. Essential before instrumenting any agent that calls external APIs under bursty load.  
*What to read:* Chapter 21 (Handling Overload) — specifically the retry budget and load shedding sections.

---

## Tools

### `trl` DPOTrainer — β sensitivity sweep
Run a β sweep [0.01, 0.1, 0.5] on a small preference dataset. Observe the mean score / pass_rate divergence: at low β, pass_rate peaks at epoch 1 and then declines while mean score continues rising; at β=0.5, both metrics track together. This is the fastest way to observe overoptimization firsthand before deploying a fine-tuned model.

### `scipy.stats.bootstrap` — paired vs. unpaired
Run both paired and unpaired bootstrap on any benchmark with shared test examples. Compute the CI width ratio. At ρ=0.85 you will see a 2.6× inflation from unpaired — concrete evidence for why method choice matters for deploy decisions, not just academic neatness.

### `scipy.stats.spearmanr` — score–length verbosity diagnostic
For any LLM-judged eval, run `Spearman(judge_score, token_count)` per condition. Asymmetric ρ across conditions is direct evidence of verbosity coupling — the judge is partially scoring output length, not just output quality. This check is retroactive: you can run it on existing traces without new API calls.

### NumPy candidate pool diagnostic
Compute `std(candidate_scores)` across your rejection-sampling pool before interpreting preference accuracy as a quality signal. If `std < 0.1`, the bottleneck is generation diversity, not model or critic quality. This one number prevents misattributing a null lift result to critic failure.

---

## Patterns

### Margin distribution audit before deploying a critic
Before attributing near-zero rejection-sampling lift to critic quality, measure the score variance in the deployment candidate pool. High preference accuracy on training pairs does not imply the critic can discriminate between same-model fixed-temperature candidates. If `std(candidate_scores) < 0.1`, the fix is candidate pool diversification, not critic retraining.

### Idempotency classification before writing retry logic
Before adding retry to any API call, classify the call: is it idempotent (read or write with idempotency key), or non-idempotent (mutating write without a deduplication mechanism)? The retry strategy is different in each case. Copy-pasting a working retry from `email_handler.py` to `calendar_handler.py` introduces a booking duplication bug if `calendar_handler.py` lacks an idempotency key.

### Two-uncertainty split for eval margins
For any eval margin near the boundary of a deployment decision, separate the two sources of uncertainty: sampling variance (bootstrap CI) and measurement variance (judge fragility). A paired bootstrap CI answers only the first question. If the margin is on the order of judge fragility (±0.5–1% for LLM judges), label the result "under-identified at this margin," not "null." The distinction has different product implications.

### Rank concentration for small preference datasets
For preference fine-tuning with fewer than ~1,000 pairs, do not spread LoRA rank uniformly across all projection modules. Compute `n / num_modules` to estimate examples-per-module: if it is below ~100, rank is too thin for nuanced rubric distinctions. Concentrate rank on `v_proj` (attention value retrieval) and `down_proj` (MLP decision logic) — the two pathways most directly involved in rubric scoring — at higher r.
