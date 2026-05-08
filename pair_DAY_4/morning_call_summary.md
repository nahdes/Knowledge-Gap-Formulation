# Morning Call Summary — Day 2

**Pair:** Nahom Desalegn + Melkam Beyene
**Topic:** Evaluation and statistics

---

Nahom's original question asked broadly about "LLM-as-judge biases" without specifying which mattered most in a single-output versus pairwise context. The morning call sharpened it to a precise contrast: which of the three canonical biases (verbosity, position, self-enhancement) dominate when a judge receives one output at a time with no in-prompt reference, and whether the specific Δ A = +0.0025 margin in the held-out ablation falls within the plausible range those biases could produce unilaterally.

Melkam's question started as "paired vs. regular bootstrap — which is correct?" The call sharpened it to the decision-impact framing: at what true Δ magnitude does CI method choice flip the deploy verdict, and what is the mathematical reason (the dropped covariance term) rather than just the prescription.

Both questions were made more grounded by anchoring to specific numbers in existing artifacts — Δ A = +0.0025, n=52, ties=17 for Nahom; σ=0.20, ρ≈0.85, n=52 for Melkam — so the explainers would produce concrete estimates rather than qualitative warnings.
