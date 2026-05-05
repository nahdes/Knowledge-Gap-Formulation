# TRP1 Week 12 — Day 1: Inference-time Mechanics

**Topic:** Inference-time mechanics  
**Date:** May 4–5, 2026  
**My Role:** Asker + Explainer  
**Partner:** Yakob Dereje

---

## Executive Summary

On Day 1 I worked both sides of the pair:

- **As Explainer**: Delivered a detailed technical explainer to Yakob on **LoRA inference mechanics** (merged vs unmerged vs system-prompt simulation).
- **As Asker**: Received a high-quality explainer from Yakob on why high preference accuracy often fails to translate into rejection-sampling lift.

Both explainers directly closed genuine gaps in our Week 11 portfolio work.

---

## 1. Explainer I Wrote (for Yakob Dereje)

**Question Received:** [`pair_DAY_1/question.md`](pair_DAY_1/question.md)

**My Explainer:** [`pair_DAY_1/explainer_lora_inference.md`](pair_DAY_1/explainer_lora_inference.md)

### Key Insights Delivered

- How LoRA matrices **A** and **B** modify the transformer forward pass: `W_eff = W₀ + (α/r)·B@A`
- Merged and unmerged inference are **numerically identical** (difference at floating-point epsilon).
- System-prompt simulation modifies **input activations**, while LoRA modifies **weights** — fundamentally different mechanisms.
- Provided a precise validity boundary and ready-to-use revised paragraph for his `model_card.md`.

---

## 2. Explainer I Received (from Yakob Dereje)

**My Question:** [`pair_DAY_1/question_general.md`](pair_DAY_1/question_general.md)

**Explainer Received:** [`pair_DAY_1/explainer.md`](pair_DAY_1/explainer.md)

### Key Insights Received

- The core reason high preference accuracy fails to predict rejection-sampling lift: **Margin Distribution Mismatch**.
- Training pairs have large, deliberate quality margins; same-temperature same-model candidates have very low variance.
- Practical diagnostic: Compute the standard deviation of critic scores across the candidate pool (`score_std < 0.1` → generation problem).
- Clear decision rule: Whether to fix the **generator** (diversity) or the **critic** (training).

This directly improves how I design and evaluate trained judges/critics in agent systems.

---

## Grounding Commits to My Portfolio

**File:** [`pair_DAY_1/grounding_commit.md`](pair_DAY_1/grounding_commit.md)

**Changes Made:**

- Updated the **Limitations** section in `The_Sales_Agent_Evaluation_Bench/model_card.md` with technically accurate language about system-prompt vs real LoRA inference (from the explainer I wrote).
- Improved evaluation methodology documentation and validity boundaries for my critic/rejection sampling setup (informed by Yakob’s explainer).

---

## Daily Deliverables

Located in [`pair_DAY_1/`](pair_DAY_1/):

- `question.md` 
- `explainer.md` 
- `morning_call_summary.md`
- `evening_call_summary.md`
- `signoff.md`
- `grounding_commit.md`
- `sources.md`
- `thread.md` — Tweet thread (ready for publication)

---

## Public Artifacts (to be published)

- **Blog Post 1:** [Link to be added]
- **Blog Post 2:** [Link to be added]
- **Tweet Thread:** [Link to be added]

---

## What I Learned

- Deep understanding of LoRA forward-pass mechanics and when system prompts are valid proxies.
- The structural **margin distribution mismatch** that explains the common failure mode of rejection sampling systems.
- A reusable diagnostic and decision framework for any critic-gated system (extremely generalizable for future FDE work).

---

## Repository Structure

```bash
TRP1-Week12-Day1/
├── pair_DAY_1/
│   ├── question.md
│   ├── question_general.md
│   ├── explainer_lora_inference.md
│   ├── explainer.md
│   ├── grounding_commit.md
│   └── ...
├── README.md
├── synthesis.md
├── canonical_list.md
└── portfolio_update.md
```
