# Portfolio Update — Week 12 Grounding Commits

**Author:** Nahom Desalegn (nahoma@10academy.org)  
**Prepared for:** FDE hiring review  
**Week:** TRP1 Week 12, May 5–9, 2026

> **Note:** This document covers four of five grounding commits (Days 1–4). The Day 5 commit will be added before the 21hr UTC submission deadline on May 9.

---

## Summary

Week 12 produced four concrete edits to existing portfolio artifacts in `nahdes/tenacious-bench-v0.1` and the sales agent pipeline. Each edit traces directly to a gap that was named in a morning call, researched and explained by a partner, and paid back into the artifact the same day. The edits are not cosmetic documentation improvements — they correct epistemic overreach, name previously unnamed tradeoffs, and add diagnostic tools that a reviewer can run against the live artifacts.

The cumulative effect: any FDE hiring manager reading the portfolio can now see not only what was built but why the key decisions were made, where the current limitations sit, and what evidence base underlies the evaluation claims.

---

## Commit 1 — Model card evaluation methodology (Day 1)

**Artifact:** `nahdes/tenacious-bench-v0.1/model_card.md` — Evaluation Methodology section  
**Change:** Replaced "Further investigation is needed" with a mechanism-driven analysis of the preference accuracy vs. rejection sampling lift divergence.

**Before:** The model card reported 96.9% preference accuracy and ΔA = +0.0025 without connecting them. A reviewer would reasonably conclude that a high-accuracy critic that produced near-zero lift represented a training failure or an underperforming model.

**After:** The model card now explains the root cause — margin distribution mismatch between high-gap training pairs and near-zero-variance deployment candidates — names the diagnostic (`std(candidate_scores) < 0.1` → generation diversity bottleneck, not critic failure), and correctly attributes the null lift to the deployment setup rather than the model. It also adds `diagnose_candidate_pool()` as a reusable pre-deployment check.

**Why this matters to a hiring manager:** A Forward-Deployed Engineer who reports an evaluation result without understanding the mechanism is a liability in a client presentation. This edit shows the ability to distinguish "model did not work" from "model was applied to the wrong distribution" — a distinction that routinely determines whether a production AI system gets retrained or redeployed.

---

## Commit 2 — Agent KNOWLEDGE.md compose stage (Day 2)

**Artifact:** `KNOWLEDGE.md` — §Compose stage, text-completion vs. function-calling rationale  
**Change:** Added a defended paragraph explaining the choice not to use function-calling in `compose_email`, with a mechanistic account of why the choice is correct.

**Before:** The compose stage in `KNOWLEDGE.md` noted that `compose_email` does not pass a `tools` parameter but gave no reason beyond "the output format is fixed." A reviewer could not assess whether this was a deliberate engineering decision or an oversight.

**After:** The paragraph now explains the causal mechanism: tool schemas are serialized into the context window and attend into all generated tokens during prefill, including tokens before the tool name. Every unused schema consumes context budget and adds inference latency with no benefit when only one output shape is ever needed. The note also names the correct trigger for switching to function-calling: if the compose stage is extended to select between multiple output actions, the switch is warranted at that point, and description strings should be written to the action boundary rather than the implementation detail.

**Why this matters to a hiring manager:** Agent architecture decisions that are described as "we did it this way" rather than "we did it this way because X, and would switch when Y" do not demonstrate engineering depth. This edit shows the ability to defend a design choice at the mechanism level — the kind of explanation that holds up in an FDE engagement when a client's senior engineer asks why.

---

## Commit 3 — Model card LoRA configuration (Day 3)

**Artifacts:** `model_card.md` — §Training Configuration; `methodology_rationale.md` — §Adapter Architecture  
**Commit:** b7c14d5  
**Change:** Added a defended section on the LoRA configuration (r=16, alpha=32, seven modules) that names the tradeoff, acknowledges the limitation, and connects it to the held-out null result.

**Before:** Both files were silent on adapter architecture. The configuration was the Unsloth default with no justification. A reviewer running `model.print_trainable_parameters()` would see the numbers but have no basis for evaluating whether they were appropriate.

**After:** The model card now explains what each module class contributes (attention projections: which patterns to attend to; MLP projections: decision logic), quantifies the supervision budget (~44 training examples per module at n=306, r=16, 7 modules), and names the more defensible configuration for this dataset size (v_proj + down_proj, r=32). Critically, it connects the configuration to the null result: the 96.9% validation accuracy is consistent with shortcut learning under gradient diffusion — the model learned to distinguish training pairs correctly but lacked the focused capacity to generalize to near-identical held-out outputs.

**Why this matters to a hiring manager:** Inherited defaults without ablation is the most common silent failure mode in rapid-iteration ML projects. This edit demonstrates the ability to audit a shipped configuration against first principles, acknowledge a limitation honestly, and frame it as a named engineering tradeoff rather than hiding it. That skill is directly relevant in client-facing FDE work where the practitioner is accountable for explaining model behavior.

---

## Commit 4 — `scoring_evaluator.py` bias audit (Day 4)

**Artifact:** `scoring_evaluator.py` — `llm_tone_judge()` docstring, line 288  
**Change:** Added a 40-line bias-audit block naming the biases the current setup does and does not control, and correcting the epistemic status of the held-out null result.

**Before:** The docstring documented that an LLM sub-judge is used for `voice_adherence` scoring and that an offline fallback exists. It gave no calibration evidence and no account of what systematic errors the sub-judge could introduce. Any reviewer asking "how do you know the judge is trustworthy?" had no answer in the artifact.

**After:** The docstring now names the two dominant biases in single-output absolute scoring (verbosity bias, style-prior bias), explains why position bias does not apply in this setup, lists four design choices that partially control bias (temperature=0, symmetric session conditions, four deterministic rubric dimensions, 5-point scale anchors), and flags three things that are not controlled (no inter-judge calibration, no score–length diagnostic, no human gold set). The module-level evaluation claim is corrected from an implicit null result to "under-identified at this margin": the paired bootstrap CI addresses sampling variance only; at |Δ A| = 0.0025, judge measurement variance dominates and the sign of the effect is not identified.

**Why this matters to a hiring manager:** Reporting a null result without auditing the measurement instrument is a common and consequential error in production ML evaluation. An FDE who can distinguish "the effect is not there" from "the measurement is not precise enough to confirm or rule out an effect at this scale" is equipped to give clients technically defensible conclusions — and to know when a result needs better measurement rather than a bigger sample.

---

## Day 5 Commit — [To be added May 9, 2026]

**Artifact:** [TBD after morning call]  
**Change:** [To be added after evening call and signoff]

---

## Collective Effect

Read together, the four commits (five after Day 5) transform the portfolio from a set of shipped systems with implicit design decisions into a set of shipped systems with explicit, defensible engineering rationales. The key shift is not in what was built — the code and models are unchanged — but in the epistemic honesty of how the decisions are documented:

- Evaluation results are now reported with the right uncertainty framing (under-identified vs. null)
- Architecture choices are now defended at the mechanism level, not just described
- Known limitations are named with their causes and their consequences for downstream claims
- Diagnostics are embedded in the artifacts so a reviewer can verify the claims, not just read them

This is the portfolio posture expected of an FDE: systems you can ship, explain, and defend.
