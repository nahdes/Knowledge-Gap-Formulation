# TRP1 Week 12 — Knowledge Gap Formulation

**Program:** 10 Academy TRP1  
**Week:** 12  
**Dates:** May 5–9, 2026  
**Participant:** Nahom Desalegn

---

## Overview

Week 12 uses a structured peer-teaching format. Each day, two participants take turns as **Asker** and **Explainer**. The Asker formulates a precise knowledge gap tied to their actual project work; the Explainer closes it with a focused, mechanism-driven explainer. Every session ends with a grounding commit: a concrete edit to a real portfolio artifact that pays the insight back into the work.

---

## Day 1 — Inference-time Mechanics

**Date:** May 5, 2026  
**Partner:** Yakob Dereje  
**Status:** Complete ✅

### Nahom as Asker

**Question:** Why did the SimPO critic (96.9% preference accuracy on 31 validation pairs) produce near-zero lift — ΔA = +0.0025, p=0.40, with 17/52 ties — when used as a rejection-sampling gate on sealed held-out tasks?

**Gap closed:** The root cause is **margin distribution mismatch**. Training pairs have large, deliberate quality gaps by construction; same-model fixed-temperature inference candidates have near-zero score variance. The critic is world-class at distinguishing bottles of different vintages; it cannot rank eight glasses poured from the same bottle. Diagnostic: if `std(candidate_scores) < 0.1`, the bottleneck is generation diversity, not critic quality.

**Grounding commit:** Replaced the vague "further investigation needed" paragraph in `tenacious-bench-v0.1/model_card.md` with a mechanism-driven evaluation methodology note, including the `diagnose_candidate_pool()` diagnostic and a deployment-side bottleneck classification.

### Nahom as Explainer

**Question received from Yakob:** How does LoRA actually modify the transformer forward pass, and is system-prompt simulation a valid proxy for real adapter inference?

**Key points delivered:**

- The adapter modifies weight matrices: `W_eff = W₀ + (α/r)·B@A`; system-prompt simulation modifies input activations — fundamentally different parts of the computation graph.
- Merged and unmerged adapter inference are numerically identical (difference ≤ floating-point epsilon).
- For rule-based rubric dimensions (string-matching), system-prompt simulation is a valid proxy. For distributional dimensions (LLM sub-judge), it is unvalidated.
- Provided a ready-to-use revised limitation paragraph for Yakob's `model_card.md`.

**Artifacts:** [`pair_DAY_1/`](pair_DAY_1/)

---

## Day 2 — Agent and Tool-Use Internals

**Date:** May 6, 2026  
**Partner:** Mikias Dagem  
**Status:** Complete ✅

### Nahom as Asker

**Question:** At the token level, when a model using function-calling selects a tool, is the description string part of the selection computation — or is it processed only after the tool-name token is already committed?

**Gap closed:** Tool schemas are serialized into the context window during prefill. Because the transformer computes attention across the full context before generating any token, description strings are causally upstream of the tool-name logit distribution — they shape which tool gets selected, not just how arguments are filled. Some inference engines apply a token mask at the name position (constrained decoding), which changes when description quality matters most.

**Grounding commit:** Added a defended paragraph to `KNOWLEDGE.md §Compose` explaining the text-completion vs. function-calling trade-off mechanistically — including why adding unused tool schemas has a real context/latency cost.

### Nahom as Explainer

**Question received from Mikias:** What distinguishes a transient from a permanent API failure, how does exponential backoff with jitter prevent retry storms, what is a retry budget, and would the same retry strategy apply to all three affected files (`email_handler.py`, `calendar_handler.py`, `enrichment.py`)?

**Key points delivered:**

- Retry on `429`, `5xx`, transport exceptions. Raise immediately on any other `4xx`.
- Exponential backoff + jitter desynchronizes concurrent retriers; AWS measured 75% reduction in retry call volume vs. fixed-interval.
- `min(delay, CAP)` is the ceiling that keeps worst-case blocked time predictable.
- The three files need _different_ strategies: email (safe with idempotency key), scraper (safe, idempotent read), booking (unsafe to retry naively — missing idempotency key is the actual bug).

**Artifacts:** [`pair_DAY_2/`](pair_DAY_2/)

---

## Day 3 — Training and Post-Training Mechanics

**Date:** May 7, 2026  
**Partner:** Gemechis Worku  
**Status:** Complete ✅

### Nahom as Asker

**Question:** In `train_critic.py` the LoRA adapter uses `lora_r=16`, `lora_alpha=32`, targeting all seven projection modules — inherited from Unsloth defaults. What does each module actually contribute to a preference-discrimination task, how does rank bound what the adapter can learn, and is r=16 across all seven modules a defensible choice for a 306-pair critic or a copy-paste default that could explain the null held-out result?

**Gap closed:** Attention projections (Q, K, V, O) learn *which patterns to attend to* — surface features like value-proposition presence or generic language. MLP projections (gate, up, down) learn *decision logic* — how patterns map to rubric criteria. Both are needed, but spreading r=16 across seven modules gives each roughly 44 training examples of supervision — too few to learn nuanced rubric distinctions beyond shortcuts. The defensible configuration for 306 pairs is to concentrate rank on `v_proj + down_proj` at r=32; r=16 across all seven is ~3.5× over-parameterized relative to dataset size.

**Grounding commit:** `b7c14d5` — added a defended LoRA-configuration paragraph to `model_card.md` §Training Configuration and `methodology_rationale.md` §Adapter Architecture, replacing "defaults were used for reproducibility" with a named tradeoff and an explicit acknowledgment that the null held-out result is consistent with shortcut learning under gradient diffusion.

### Nahom as Explainer

**Question received from Gemechis:** What plays the role of reward model overoptimization in DPO — which has no separate reward model — and why does it produce the specific pattern of mean score rising while pass rate falls?

**Key points delivered:**

- DPO does have an implicit reward model: `r*(x,y) = β·log(π_θ(y|x)/π_ref(y|x))`. The policy ratio is the reward; β is the KL anchor.
- The mean-up / pass-down fingerprint is the specific signature of soft-dimension proxy misalignment: preference pairs were labeled on continuous rubric dims, hard-fail categories had weak margin signal, so DPO optimized the wrong thing.
- Gao et al. (2023) showed the same proxy/gold divergence in classic RLHF as a function of KL budget — same pattern, different plumbing.
- Three corrective dials: (1) increase β by 2–4×, (2) use pass_rate as the checkpoint criterion instead of mean score, (3) IPO (Azar et al. 2023) replaces logistic loss with squared loss that saturates, preventing unbounded margin growth.
- Provided two ready-to-paste model card paragraphs for §Summary and §Bias, Risks, and Limitations.

**Artifacts:** [`pair_DAY_3/`](pair_DAY_3/)

---

## Day 4 — Evaluation and Statistics

**Date:** May 8, 2026  
**Partner:** Melkam Beyene  
**Status:** Complete ✅

### Nahom as Asker

**Question:** In `scoring_evaluator.py` the `llm_tone_judge()` function uses an LLM sub-judge (OpenRouter Sonnet 4.6) for the `voice_adherence` dimension. The held-out Δ A = +0.0025 is narrower than any plausible systematic judge bias. Which specific biases dominate in a single-output absolute-score setup — as opposed to pairwise comparison — and did the current protocol control for any of them, or is the null result vulnerable to a scoring artifact?

**Gap closed:** Verbosity bias and style-prior bias dominate in single-output scoring; position bias does not apply (no reference output to anchor position). Melkam introduced the **two-uncertainty split**: paired bootstrap CIs measure sampling variance only — they say nothing about judge measurement variance, which at ±0.5–1% dwarfs a Δ of 10⁻³. The correct framing is **"under-identified at this margin"**, not "null result" — the experiment was underpowered on the measurement axis, not just sample size.

**Grounding commit:** Added a 40-line bias-audit block to `llm_tone_judge()` docstring (`scoring_evaluator.py` line 288): names the two dominant biases, lists four design choices that partially control bias (temperature=0, symmetric conditions, deterministic rubric dimensions, 5-point scale anchors), flags three uncontrolled risks (no inter-judge calibration, no score–length diagnostic, no human gold set), and corrects the epistemic status to "under-identified."

### Nahom as Explainer

**Question received from Melkam:** In a Sales-Evaluation-Bench where the same test examples score multiple methods, which CI method is correct — regular bootstrap or paired bootstrap? If you pick the wrong one, how much does it change the uncertainty range, and could it flip the deploy verdict?

**Key points delivered:**

- Paired bootstrap resamples task pairs `(a_i, b_i)` intact; regular bootstrap resamples each condition independently, dropping the covariance term from `Var(A−B) = Var(A) + Var(B) − 2·Cov(A,B)`.
- CI half-width ratio: `HW_unpaired / HW_paired = √(1 / (1 − ρ))` — at ρ=0.85 (typical for same-task-set benchmarks), unpaired is **2.6× wider** than it should be.
- Concrete deploy-flip: with n=52 and Δ=+0.035, paired CI is [+0.005, +0.065] (deploy); unpaired CI is [−0.042, +0.112] (don't deploy). Same data, wrong method, wrong decision.
- Rule: use paired bootstrap whenever methods share test examples. Unpaired is only correct for disjoint test sets.

**Artifacts:** [`pair_DAY_4/`](pair_DAY_4/)

---

## Day 5 — [Topic TBD]

**Date:** May 9, 2026  
**Partner:** TBD  
**Status:** In progress 🔄

*To be completed before the 21hr UTC submission deadline.*

---

## Repository Structure

```
Knowledge-Gap-Formulation/
├── README.md
├── synthesis.md              ← 1,500-word week synthesis (10 gaps, reading list, tool list)
├── canonical_list.md         ← Annotated papers, tools, and patterns for FDE cohort canon
├── portfolio_update.md       ← One-page portfolio improvement summary for hiring review
├── pair_DAY_1/
│   ├── question.md               ← Nahom's question to Yakob
│   ├── explainer.md              ← Nahom's LoRA explainer for Yakob
│   ├── morning_call_summary.md
│   ├── evening_call_summary.md
│   ├── signoff.md
│   ├── grounding_commit.md
│   ├── sources.md
│   └── thread.md
├── pair_DAY_2/
│   ├── question.md               ← Nahom's question to Mikias
│   ├── explainer.md              ← Nahom's retry-logic explainer for Mikias
│   ├── morning_call_summary.md
│   ├── evening_call_summary.md
│   ├── signoff.md
│   ├── grounding_commit.md
│   ├── sources.md
│   └── thread.md
├── pair_DAY_3/
│   ├── question.md               ← Nahom's LoRA configuration question to Gemechis
│   ├── explainer.md              ← Nahom's DPO overoptimization explainer for Gemechis
│   ├── morning_call_summary.md
│   ├── evening_call_summary.md
│   ├── signoff.md
│   ├── grounding_commit.md
│   ├── sources.md
│   └── thread.md
├── pair_DAY_4/
│   ├── question.md               ← Nahom's LLM-judge bias question to Melkam
│   ├── explainer.md              ← Nahom's paired bootstrap explainer for Melkam
│   ├── morning_call_summary.md
│   ├── evening_call_summary.md
│   ├── signoff.md
│   ├── grounding_commit.md
│   ├── sources.md
│   └── thread.md
└── pair_DAY_5/                   ← To be added May 9, 2026
    └── ...
```

---

## Final Submission Documents

| File | Description |
|---|---|
| [`synthesis.md`](synthesis.md) | 1,500-word week synthesis: 10 gaps closed, most surprising insight, canonical reading list and tool list |
| [`canonical_list.md`](canonical_list.md) | Annotated cohort canon: 11 papers, 4 tools, 4 FDE patterns |
| [`portfolio_update.md`](portfolio_update.md) | One-page summary of how grounding commits improve Weeks 10–11 portfolio, written for FDE hiring review |

---

## Public Artifacts

- **Tweet Thread (Day 1):** https://x.com/DesalegnNa91842/status/2051733293607879042?s=20
- **Tweet Thread (Day 2):** https://x.com/DesalegnNa91842/status/2052082732239339628?s=20ss
- **Tweet Thread (Day 3):** [Link to be added]
- **Tweet Thread (Day 4):** [Link to be added]
- **Tweet Thread (Day 5):** [Link to be added]
- **Blog Post (Day 1):** [Link to be added]
- **Blog Post (Day 2):** [Link to be added]
- **Blog Post (Day 3):** [Link to be added]
- **Blog Post (Day 4):** [Link to be added]
- **Blog Post (Day 5):** [Link to be added]
