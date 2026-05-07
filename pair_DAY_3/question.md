# Question — Day 1 | Training and Post-Training Mechanics

**Author:** Nahom (nahoma@10academy.org)
**Topic:** Training and post-training mechanics
**Subtopic:** What LoRA actually adapts; why low rank works at all; what changes at higher rank

---

## The Question

In `train_critic.py` I configured the LoRA adapter with `lora_r=16`,
`lora_alpha=32`, and targeted all seven projection modules:
`q_proj`, `k_proj`, `v_proj`, `o_proj`, `gate_proj`, `up_proj`, `down_proj`.
I chose these settings because they were the Unsloth defaults — I cannot
defend them if pressed.

Specifically, I do not know what each module class contributes to the
critic's task of distinguishing a rubric-passing B2B sales email from a
rubric-failing one, nor can I explain why r=16 is the right rank for a
306-pair domain-specific preference task rather than r=4 or r=64. If r=16
is too low to represent the rubric-dimension distinctions the critic needs to
learn, or if targeting all seven modules spreads the adapter budget across
weight matrices that carry no signal for this task, that is a mechanistic
cause of the null held-out result I have not yet named.

**My specific gap:**

> What does LoRA's low-rank decomposition actually adapt inside a transformer
> layer — which weight matrices carry the domain-relevant signal for a
> preference task, and how does the rank hyperparameter bound what the adapter
> can and cannot learn? And given those mechanics, is r=16 targeting all seven
> projection modules a defensible choice for a 306-pair critic, or is it
> an under-specified default that could explain why the adapter reached 96.9%
> validation accuracy without transferring to held-out reranking?

---

## Connection to existing artifact

The gap sits in two places in my shipped work:

1. **`train_critic.py` lines 54–58** — the `lora_modules` list and
   `lora_r=16` / `lora_alpha=32` are hardcoded with no inline justification.
   The script's docstring documents reproducibility but says nothing about
   why these modules or this rank. Any reviewer running `model.print_trainable_parameters()`
   will ask: why all seven? why not attention-only? why not r=4?

2. **`methodology_rationale.md` §"Why SimPO over DPO and ORPO"** — the
   rationale defends the *loss function* in detail (Meng 2024, β/γ choice,
   leakage policy) but is completely silent on the *adapter architecture*.
   The model card (not yet written) will need a defended LoRA configuration
   section. Without understanding the mechanism, I cannot write that section
   honestly.

**Artifact pointer:** `train_critic.py` L47–72 (DEFAULTS block); the shipped
adapter at `training/adapter_seed11/` (r=16, alpha=32, all 7 modules, ~70 MB
unmerged). The grounding commit will add a one-paragraph LoRA-configuration
justification to `model_card.md` that names which modules carry the relevant
signal and why r=16 is sufficient (or acknowledges it may not be).

---

## Why it generalizes

Any FDE who trains a LoRA adapter for a domain-specific task — a preference
critic, an instruction-tuned scorer, a retrieval reranker — faces the same
configuration question: which layers to target, and what rank. The default
answer ("target everything at r=16") is widely used but rarely defended. A
practitioner who understands what each projection module computes, and what
the rank hyperparameter bounds, can make that choice deliberately rather than
copying a template. This applies to every LoRA run in the FDE toolkit, not
only SimPO critics for B2B sales.

---

## Why it is resolvable in one explainer

The question is bounded to two decisions: (1) which weight matrices to target
and why, and (2) what rank controls and how to choose it for a small-data
task. A 600–1,000 word blog can:

- Name what each of the 7 projection modules computes in a transformer layer
  and which of them are most likely to carry task-relevant signal for a
  preference task
- Explain why low-rank decomposition works at all (the intrinsic-dimensionality
  argument) and what increasing rank buys — and costs
- Show a concrete comparison: trainable parameter counts at r=4, r=16, r=64
  across attention-only vs. all-seven targeting, and what that implies for a
  306-pair training set
- Connect the configuration back to the 96.9% val acc / null held-out result
  as a case study

That is a complete and answerable explainer. It does not require covering all
of LoRA's claims or the full PEFT literature.

---

## Self-check against the four rubric properties

| Property | Self-assessment |
|---|---|
| **Diagnostic** | Names two specific decisions (which modules, what rank) in `train_critic.py` L54–58 that I cannot defend mechanistically. Not "how does LoRA work" — asks what each module contributes to *this* task and whether r=16 was the right choice. |
| **Grounded** | Tied to `train_critic.py` L54–58 (DEFAULTS block) and `training/adapter_seed11/` (the shipped artifact). The grounding commit is identified: a defended LoRA-config paragraph in `model_card.md`. |
| **Generalizable** | Module targeting and rank selection recur in every FDE LoRA run — domain scorers, rerankers, instruction-tuned critics. Not Tenacious-specific. |
| **Resolvable** | Module function taxonomy + rank-as-intrinsic-dimensionality argument + parameter-count table = fits cleanly in 600–1,000 words. Not a textbook; not a Wikipedia lookup. |
