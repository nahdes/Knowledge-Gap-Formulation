# Morning Call Summary - Day 1 (Inference-time Mechanics)

**Date:** May 5, 2026  
**Pair:** Nahom Desalegn & Yakob Dereje

## Question Sharpening Summary

In the morning call we reviewed and significantly sharpened **my question** on why high preference accuracy fails to predict rejection-sampling lift.

### Key Improvements Made

- **Before the call**: The question was somewhat general about the divergence between training and deployment metrics.
- **After sharpening**: We made it highly specific and diagnostic by:
  - Naming the exact numbers from my work: **96.9% preference accuracy** on validation pairs vs **ΔA = +0.0025 (p=0.40)** with **17/52 ties** on held-out tasks.
  - Clearly referencing the grounding artifacts (`ablations/ablation_results.json`, `tenacious_bench_v0.1` dataset, etc.).
  - Explicitly stating the core puzzle: understanding the structural relationship between **training-pair margin distribution** and **inference-time candidate pool variance**.
  - Highlighting the decision I need: training-side fixes (more/better pairs, SimPO hyperparameters) vs deployment-side fixes (temperature, multi-model sampling, etc.).

The refined question now perfectly satisfies the Week 12 criteria:

- **Diagnostic**: Points to the exact margin-distribution mismatch mechanism.
- **Grounded**: Directly tied to concrete files and results in `nahdes/tenacious-bench-v0.1`.
- **Generalizable**: Applies to any critic-gated system in FDE work.
- **Resolvable**: Can be closed with mechanism explanation, minimal simulation, and a clear decision rule.

This version gives the explainer a very clear target while remaining focused.

**Call duration:** ~32 minutes  
**Prepared by:** Nahom Desalegn  
**Confirmed by:** Yakob Dereje
