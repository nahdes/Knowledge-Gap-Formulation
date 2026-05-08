# Question — Day 2 | Evaluation and Statistics

**Author:** Nahom (nahoma@10academy.org)
**Topic:** Evaluation and statistics
**Subtopic:** Systematic biases of an LLM sub-judge in a single-output rubric scoring setup; detection and mitigation

---

## The Question

In `scoring_evaluator.py` I use an LLM sub-judge (OpenRouter Sonnet 4.6 on the
sealed held-out slice) to score the `voice_adherence` dimension — specifically,
tone-marker compliance that the deterministic heuristic cannot catch. The docstring
documents the offline fallback (`TENACIOUS_BENCH_OFFLINE=1`) but says nothing about
what systematic errors the LLM sub-judge introduces or whether the rubric design
does anything to bound them.

The problem is concrete and quantitative: the held-out mean scores are
0.6618 (baseline) vs 0.6643 (critic_rs) — a Δ A of +0.0025. A sub-judge bias
of even ±0.5% in the `voice_adherence` sub-score, applied asymmetrically across
the two conditions, is large enough to flip the sign of Δ A and change the
deploy recommendation. I cannot name which specific biases a single-output
(not pairwise) LLM judge is most vulnerable to, nor whether the held-out
scoring protocol did anything to detect or mitigate them.

**My specific gap:**

> Which systematic biases does an LLM sub-judge introduce in a single-output
> absolute-score setting — as opposed to a pairwise comparison setting — and
> given that my held-out margin (Δ A = +0.0025) is narrower than any plausible
> judge bias, what design changes to `scoring_evaluator.py` would detect or
> bound those biases? Did the current setup control for any of them, or did it
> leave the null result vulnerable to a systematic scoring artifact?

---

## Connection to existing artifact

The gap sits in two places in my shipped work:

1. **`scoring_evaluator.py` — LLM sub-judge invocation (lines ~8–13, docstring)**
   The docstring documents *that* an LLM sub-judge is used for tone-marker scoring
   and *that* an offline fallback exists, but provides no calibration evidence,
   no bias audit, and no rationale for why Sonnet 4.6 is a trustworthy scorer
   for the `voice_adherence` dimension specifically. Any reviewer asking "how do
   you know the judge is consistent?" has no answer in the artifact.

2. **`ablations/ablation_results.json` — held-out mean scores and Δ A**
   The no-deploy conclusion rests on `mean: 0.66182` vs `mean: 0.66428`, a
   difference of 0.00246 with `ci_lo: -0.0186`. If the LLM sub-judge has a
   systematic verbosity or instruction-following bias that inflated or deflated
   scores for the critic_rs condition specifically (critic_rs outputs are
   reranked by a separate adapter, so their surface form distribution differs
   from baseline), the paired CI captures sampling variance but not judge
   variance. The claim "does not clear baseline at 95% CI" is only valid if
   the judge is unbiased or if its bias is symmetric across conditions.

**Artifact pointer:** `scoring_evaluator.py` (docstring + sub-judge call);
`ablation_results.json` (`delta_A_critic_rs_minus_baseline`, n=52, ties=17).
The grounding commit will add a bias-audit section to `scoring_evaluator.py`'s
docstring — naming the three main LLM judge biases, stating which design choices
in the current rubric mitigate them, and flagging what remains uncontrolled.

---

## Why it generalizes

Any FDE who uses an LLM as a sub-judge — to score generation quality, rank
candidate outputs, or filter a preference dataset — will face the same question:
is the judge measuring the target construct or is it measuring its own biases
(response length, position in a list, self-consistency with its own training
distribution)? The distinction between single-output absolute scoring and pairwise
comparison scoring is not commonly taught but changes which biases dominate.
This applies directly to LMSYS Chatbot Arena, MT-Bench, custom rubric evaluators,
and any preference-pair pipeline that uses an LLM to label chosen vs. rejected
outputs — including the `build_preference_pairs.py` that generated the 306 pairs
used to train the critic.

---

## Why it is resolvable in one explainer

The question is bounded to one contrast (single-output vs. pairwise) and three
bias types. A 600–1,000 word blog can:

- Name and explain the three main LLM-as-judge biases: verbosity bias (longer
  outputs score higher regardless of quality), position bias (first or last
  option preferred in pairwise setups), and self-enhancement bias (outputs
  stylistically similar to the judge's own training distribution score higher)
- Explain why single-output absolute scoring is more vulnerable to verbosity
  and self-enhancement bias than pairwise comparison, and why pairwise comparison
  introduces position bias that absolute scoring does not
- Show the concrete risk in the held-out setup: Δ A = +0.0025 with ties=17/52;
  a verbosity bias of +0.01 on critic_rs outputs (which are reranked, not
  randomly sampled) fully explains the observed lift without any true quality
  difference
- Name two detection methods (inter-judge agreement on a calibration split;
  score-vs-length correlation test) and two mitigation methods (rubric anchoring
  with concrete examples; swapping output order in pairwise mode) and state which
  ones the current `scoring_evaluator.py` implements

That is a complete and answerable explainer. It does not require covering all
of the LLM-as-judge literature.

---

## Self-check against the four rubric properties

| Property | Self-assessment |
|---|---|
| **Diagnostic** | Names a specific mechanism — single-output absolute scoring bias — not the broad topic "LLM judge reliability." The question asks which of the three known biases (verbosity, position, self-enhancement) dominate in a *single-output* setup, and whether the +0.0025 Δ A is within the range a verbosity bias alone could produce. |
| **Grounded** | Tied to `scoring_evaluator.py` (sub-judge call for `voice_adherence`) and `ablation_results.json` (`delta_A`, n=52, ties=17, mean difference=0.00246). The grounding commit is identified: a bias-audit paragraph in `scoring_evaluator.py` docstring. |
| **Generalizable** | Single-output vs. pairwise judge bias distinction applies to any eval pipeline using LLM scoring — rubric evaluators, preference pair builders, RLHF reward model validation. Not Tenacious-specific. |
| **Resolvable** | Three bias types + single-output vs. pairwise contrast + two detection + two mitigation methods + one worked example on Δ A = fits cleanly in 700–900 words. Not a literature survey; a specific mechanistic answer. |
