# question.md

**Pair-day question · Nahom Desalegn · 10academy TRP1 Week 11**

---

## The Question

When a preference-tuned critic is deployed as a rejection-sampling gate — scoring k candidates
and selecting the highest-reward output — under what conditions does the critic's
in-distribution preference accuracy predict held-out task lift, and when does it not?

Specifically: my Tenacious-Bench v0.1 SimPO critic achieved **96.9% preference accuracy**
on 31 validation pairs but produced **ΔA = +0.0025 (95% CI [−0.019, +0.023], p = 0.40)**
across 52 sealed held-out tasks with **17/52 ties**. The training metric and the deployment
metric diverged completely. I need to understand the mechanism that causes this divergence
so I can determine whether the fix is a training-side change (more pairs, different β/γ,
different pair construction) or a deployment-side change (higher temperature, multi-model
candidate pool, family mixing) — because the answer changes how I build every future
critic-gated evaluation system in FDE work.

---

## Grounding Artifact

`ablations/ablation_results.json`, `training_data/train_pairs.jsonl`, and
`tenacious_bench_v0.1/held_out/held_out_scores.jsonl` from
[nahdes/tenacious-bench-v0.1](https://huggingface.co/datasets/nahdes/tenacious-bench-v0.1).

The **17/52 tie rate** is the specific observation that the question must explain. The
question is not answered by "get more data" — it requires naming the structural relationship
between preference-pair margin distribution at training time and candidate-pool variance at
inference time.

---

## Why It Generalises

Every FDE engagement that uses a preference-tuned critic as a quality gate — email scoring,
code review, proposal ranking, meeting summary grading — faces the same architecture. The
training metric (preference accuracy on held-out pairs) is always available and cheap to
compute. The deployment metric (task-level lift under rejection sampling) is expensive and
slow. If preference accuracy is not a reliable proxy for deployment lift, every FDE team
building critic-gated systems is optimising the wrong objective. Closing this gap changes
the evaluation protocol for every critic deployment, not just Tenacious-Bench.

---

## Resolvability Scope

A 600–1,000 word explainer can close this by:

1. Naming the margin-distribution mismatch mechanism precisely
2. Showing with a minimal worked example what happens when training-pair margin and
   inference-pool variance are mismatched
3. Giving a two-item decision rule for whether the fix is training-side or deployment-side

It does **not** require covering all of SimPO theory, all of RLHF, or all of rejection
sampling literature.
