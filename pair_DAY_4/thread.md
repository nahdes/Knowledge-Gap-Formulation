# Tweet Thread — Day 2

**Topic:** LLM-as-judge biases in single-output scoring
**Platform:** Twitter/X
**Author:** Nahom Desalegn (@nahoma)

---

**Tweet 1 / 6**

You trained a reranking critic. Held-out Δ = +0.0025. Your paired bootstrap CI includes zero. Most people call that a null result and move on.

The more honest call: "under-identified at this margin."

Here's why the distinction matters. 🧵

---

**Tweet 2 / 6**

Two separate things can make your eval margin look like noise:

1. Sampling variance — 52 examples isn't a lot. Bootstrap CIs address this.

2. Judge measurement error — the LLM scoring your outputs has systematic biases. Bootstrap CIs don't touch this one.

When |Δ| is 10⁻³, source 2 dominates.

---

**Tweet 3 / 6**

Single-output absolute scoring vs. pairwise comparison expose you to DIFFERENT biases.

Pairwise → position bias (judges prefer first/last slot). Fix: swap order and average.

Single-output → verbosity bias + style-prior bias. No in-prompt counterfactual to cancel them.

Know your setup.

---

**Tweet 4 / 6**

Verbosity bias in practice: if your reranked outputs are 15% longer (reranking selects "more complete" outputs), a judge that correlates discourse markers with quality inflates their score by ~0.01–0.03.

Against Δ = +0.0025, that's 4–12× your observed signal. The sign isn't identified.

---

**Tweet 5 / 6**

Two cheap checks you can run retroactively:

① Score–length correlation: Spearman(judge_score, token_count) per condition. Asymmetry = verbosity confound.

② Inter-judge agreement: re-score 30 items with a paraphrased rubric. If sign(Δ) flips, your primary result wasn't identified.

Neither requires new API calls on the full set.

---

**Tweet 6 / 6**

"Null result" = the effect isn't there.
"Under-identified" = measurement precision can't confirm or rule it out.

These have different product implications. Null → the method provides no value. Under-identified → the experiment needs better measurement, not a bigger sample.

Know which one you have before you ship the no-deploy memo.

---

*Sources: Zheng et al. 2023 (MT-Bench, arXiv:2306.05685); Wang et al. 2025 (position bias, ACL 2025.ijcnlp-long.18)*
