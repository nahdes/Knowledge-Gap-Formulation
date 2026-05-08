# Grounding Commit — Day 2

**Artifact edited:** `scoring_evaluator.py`
**Location:** `llm_tone_judge()` docstring, line 288
**Type of change:** Documentation — bias-audit section added to the sub-judge function

---

## What was changed

Added a 40-line bias-audit block to the `llm_tone_judge` docstring immediately after the existing offline-mode note. The block:

1. Names the two biases that dominate in single-output absolute scoring (**verbosity bias**, **style-prior bias**) and explains why **position bias** does not apply here but would if a pairwise pass were added.
2. Lists four design choices in the current setup that partially control bias: temperature=0, symmetric session conditions for both ablation conditions, four deterministic rubric dimensions, and explicit 5-point scale anchors in the judge prompt.
3. Lists three things that are **not** controlled: no inter-judge calibration, no score–length diagnostic, no human gold set.
4. States the corrected epistemic status: **"under-identified at this margin"** rather than "null result" — the paired CI addresses sampling variance only, not judge measurement variance.

---

## Why this file, why this location

`scoring_evaluator.py` is the only artifact where the LLM sub-judge is invoked. Any reviewer asking "how do you know the judge is trustworthy?" will read this function first. The docstring is the right place for calibration evidence and known limitations — not a separate memo that can drift out of sync with the code.

The existing module-level docstring documents *that* an LLM sub-judge is used and *that* an offline fallback exists. This grounding commit adds the missing layer: *what systematic errors the sub-judge can introduce and what the current design does or does not do about them.*

---

## Connection to the gap

The gap was: "I cannot name which specific biases a single-output LLM judge is most vulnerable to, nor whether the held-out scoring protocol did anything to detect or mitigate them."

The commit directly answers both halves: verbosity and style-prior dominate (named and explained); the protocol controls temperature and session symmetry but leaves verbosity coupling and inter-judge agreement unverified (stated explicitly).

The framing upgrade — "under-identified" replacing the implicit "null" — also corrects a downstream risk: anyone reading `ablation_results.json` and citing the no-deploy conclusion should understand it rests on sampling evidence, not bias-audited measurement.

---

## Diff summary

```
scoring_evaluator.py  llm_tone_judge() docstring
  + 40 lines: bias-audit block
  - 0 lines removed
  No logic change; scoring behavior is identical.
```
