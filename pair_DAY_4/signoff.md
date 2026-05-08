# Signoff — Day 2 | Nahom's Gap

**Asker:** Nahom Desalegn (nahoma@10academy.org)
**Researcher:** Melkam Beyene
**Gap:** Systematic biases of an LLM sub-judge in single-output absolute scoring; whether the held-out null result is identified given Δ A = +0.0025

---

## Closure judgment: CLOSED

---

## What changed

Before this explainer I could name three bias types (verbosity, position, self-enhancement) as a list but could not reason about which ones matter in a *single-output* setup versus pairwise, nor could I articulate why the size of Δ A relative to plausible judge drift changes the epistemic status of the result.

Melkam's explainer gave me two things I was missing:

**1. The two-uncertainty split.**
Paired bootstrap CIs measure *sampling* variance — what happens if you re-drew 52 tasks from the same distribution. They say nothing about *judge measurement* variance — what happens if you paraphrased the rubric, changed the judge model, or shifted temperature. When |Δ A| is on the order of 10⁻³ and published judge fragility estimates put systematic bias at ±0.5–1%, the second source dominates. The CI I reported is valid conditional on the scoring rule being unbiased, but I had no calibration evidence that it was.

**2. The under-identified framing.**
"Null result" implies the effect isn't there. "Under-identified at this margin" is the correct claim: measurement precision cannot confirm or rule out a small positive effect. These have different product implications — null means the critic provides no value; under-identified means the experiment was underpowered on the measurement axis, not just the sample-size axis.

**Concrete update:** The grounding commit adds a bias-audit section to `llm_tone_judge`'s docstring naming which biases are controlled (four deterministic dimensions, temperature=0, sealed slice) and which are not (verbosity/style confound in the tone-marker sub-judge, no inter-judge calibration, no score–length diagnostic). The module docstring now flags the result as "under-identified at this margin" rather than implying a clean null.

**What the gap does not change:** The deploy=no recommendation stands. Under-identified is still insufficient evidence to deploy. The framing upgrade matters for intellectual honesty and for anyone who reads the artifact and wonders whether a small positive Δ is real.
