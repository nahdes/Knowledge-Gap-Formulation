# Sources — Day 2

**Gap researched:** LLM-as-judge systematic biases in single-output absolute scoring (Nahom's question, researched by Melkam Beyene)

---

## Paper 1

**Zheng, Lianmin et al. "Judging LLM-as-a-Judge with MT-Bench and Chatbot Arena."**
NeurIPS Datasets and Benchmarks Track, 2023.
arXiv: https://arxiv.org/abs/2306.05685

Introduces scalable LLM-as-judge evaluation and names the three canonical failure modes: position bias, verbosity bias, and self-enhancement bias. Proposes swap-based mitigation for pairwise position bias. Directly grounds the bias taxonomy in `peer_explainer.md` §3 and the design-choice analysis in the `scoring_evaluator.py` bias-audit docstring.

---

## Paper 2

**Wang et al. "Judging the Judges: A Systematic Study of Position Bias in LLM-as-a-Judge."**
IJCNLP 2025 (long paper).
ACL Anthology: https://aclanthology.org/2025.ijcnlp-long.18/

Systematic empirical study of position bias across multiple judge models and task types. Confirms swap-averaging as the primary mitigation for pairwise setups, and shows that bias magnitude varies substantially by model family — directly relevant to the claim that a Sonnet-class judge on a Qwen-generated output set introduces style-prior risk (`peer_explainer.md` §3.2).

---

## Tool

**Python: `scipy.stats.spearmanr` — retroactive score–length correlation check**

```python
import json, pathlib, scipy.stats as st

traces = [json.loads(l) for l in
          pathlib.Path("ablations/held_out_traces.jsonl").read_text().splitlines()]

for condition in ("baseline", "critic_rs"):
    rows = [r for r in traces if r["condition"] == condition]
    lengths = [len(r["output"].split()) for r in rows]
    va_scores = [r["breakdown"]["voice_adherence"]["score"] for r in rows]
    rho, p = st.spearmanr(lengths, va_scores)
    print(f"{condition}: Spearman(voice_adherence, length) = {rho:.3f}  p={p:.3f}")
```

Run against `ablations/held_out_traces.jsonl`. A significantly positive ρ in `critic_rs` but not `baseline` would confirm asymmetric verbosity coupling — the specific threat identified in §3.1 of `peer_explainer.md`. If ρ is similar across conditions, verbosity bias is symmetric and cannot explain the sign of Δ A either way.
