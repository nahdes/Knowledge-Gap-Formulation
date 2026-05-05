# explainer.md

**How LoRA Actually Modifies the Forward Pass — and Whether System-Prompt
Simulation Is a Valid Proxy for Adapter Inference**

_Pair-day explainer · 10academy TRP1 Week 12_

---

## The Gap This Closes

Your `model_card.md` states that Delta A (+0.263, p<0.0001) was measured using
system-prompt simulation via API rather than real LoRA weight loading. You need to
know one of two things: either the system-prompt result is a valid proxy for what
real adapter inference would produce, or you need a technically grounded correction
to your model card that states the validity boundary precisely.

This explainer answers that question by working through the LoRA forward pass
mechanically, then comparing the three inference modes at the level of what they
actually compute.

---

## How LoRA Modifies the Forward Pass

A transformer layer contains weight matrices — attention projections (Q, K, V, O),
and feed-forward projections (gate, up, down). In full fine-tuning, training updates
every element of every matrix. LoRA does not do this.

Instead, for each target weight matrix W₀ (shape d_out × d_in), LoRA introduces
two small matrices:

- **A** (shape r × d_in) — initialized from a random normal distribution
- **B** (shape d_out × r) — initialized to **zero**

where r is the rank (your run used r=16). The modified weight is:

```
W_effective = W₀ + (alpha/r) * B @ A
```

At the start of training, B is all zeros, so W_effective = W₀ exactly — the adapter
contributes nothing. As training progresses, B learns non-zero values. The term
`(alpha/r) * B @ A` is the adapter's learned correction to the original weight.

For a single input vector x, the output of that layer becomes:

```
h = W₀ @ x  +  (alpha/r) * B @ (A @ x)
    \_______/   \________________________/
    base path        adapter delta
```

The base path is the frozen pretrained computation. The adapter delta is a low-rank
correction — it lives in an r-dimensional subspace of the full d_out-dimensional
output space. With r=16 and d_out=2048 (typical for a 1.7B model), the adapter
controls roughly 0.8% of the output dimensions independently; the rest are
constrained by the low-rank structure.

**The key architectural fact**: the adapter modifies the weight matrix, not the
input. It changes _how the model responds to any input_, not _what input the model
sees_.

---

## Three Inference Modes — What Each Actually Computes

### Mode 1 — Merged adapter inference

Before serving, fuse the adapter into the base weights permanently:

```python
W_merged = W0 + (alpha / r) * (B @ A)
h = W_merged @ x          # single matrix multiply, no overhead
```

This is mathematically identical to unmerged inference. It is preferred in
production because it eliminates the extra computation at serving time. The output
token distribution is **exactly** the LoRA-trained distribution.

### Mode 2 — Unmerged adapter inference (PEFT / `PeftModel.from_pretrained`)

Keep W₀ frozen and compute the delta separately at every forward pass:

```python
h = W0 @ x + (alpha / r) * B @ (A @ x)
```

This is numerically identical to merged inference — the same floating-point
operations, just ordered differently. The maximum absolute difference between merged
and unmerged output on the same input is on the order of floating-point epsilon
(demonstrated below). It produces **exactly** the LoRA-trained distribution.

### Mode 3 — System-prompt simulation

Send a condensed version of the training objective (your style guide, honesty
rules, banned-phrase list) as a system prompt to the base model via API. No adapter
weights are loaded. The forward pass is:

```python
h = W0 @ x_prompted      # x_prompted has different activations due to the prompt
```

The system prompt changes **x** — the input token sequence and its resulting hidden
states — but leaves W₀ unchanged. The adapter delta `(alpha/r) * B @ (A @ x)` is
absent entirely.

---

## Concrete Simulation — What the Numbers Show

```python
import numpy as np
rng = np.random.default_rng(11)

d_in, d_out, r = 64, 64, 8
alpha = 16.0
scale = alpha / r   # = 2.0

W0        = rng.normal(0, 0.02, (d_out, d_in))
A         = rng.normal(0, 0.01, (r, d_in))
B_trained = rng.normal(0, 0.05, (d_out, r))   # non-zero after training
x         = rng.normal(0, 1.0, d_in)

h_merged   = (W0 + scale * B_trained @ A) @ x
h_unmerged = W0 @ x + scale * B_trained @ (A @ x)
h_base     = W0 @ x

# System prompt: same W0, different input activations
x_prompted = x + rng.normal(0, 0.3, d_in)
h_sysprompt = W0 @ x_prompted
```

**Output (first 5 dimensions):**

```
Base (no adapter):   [-0.1137  -0.0629  -0.1058  -0.1683  -0.1917]
Merged adapter:      [-0.1681  -0.0926  -0.1187  -0.1685  -0.1957]
Unmerged adapter:    [-0.1681  -0.0926  -0.1187  -0.1685  -0.1957]
System prompt only:  [-0.0903  -0.0452  -0.1425  -0.1239  -0.2327]
```

**Numerical equivalence check:**

```
Max diff merged vs unmerged:  8.33e-17   ← floating-point noise only
```

**Adapter delta vs system-prompt delta (L2 norm):**

```
Adapter delta L2:      0.1658
System-prompt delta L2: 0.3371
```

Three things are visible immediately:

1. Merged and unmerged are numerically identical to machine precision. There is
   no scenario where choosing between them changes your results. If your
   evaluation had used either mode, it would have produced the same output.

2. The system prompt produces a _larger_ L2 perturbation than the adapter in this
   simulation. That is not a general law — it depends on how much text is in the
   system prompt and how the model routes that information — but it illustrates
   that system-prompt and adapter effects are not calibrated to each other.

3. The system prompt and the adapter are operating on **different parts of the
   computation graph**. The adapter moves the weight matrices; the system prompt
   moves the input activations. They can produce similar _behavioral_ effects on
   downstream tasks without producing similar _distributional_ effects on token
   probabilities.

---

## What This Means for Your Delta A Result

Your Delta A of +0.263 (p<0.0001) was measured by comparing:

- **Baseline**: base model with no system prompt, k=1
- **System-prompt condition**: base model with the condensed style guide as system
  prompt, k=1

This is a valid measurement of what it purports to measure: the effect of injecting
the style guide as a system prompt. The number is real, the p-value is meaningful,
and the result is reproducible.

**What it is not**: a measurement of what the trained LoRA adapter would produce.
The adapter learned `(alpha/r) * B @ A` — a set of weight corrections distributed
across 7 target modules (q_proj, k_proj, v_proj, o_proj, gate_proj, up_proj,
down_proj). The system prompt injected information into the input context. These
are different interventions.

**Whether they are close enough to be treated as equivalent** depends on two things:

**Factor 1 — What the adapter learned to do.** If the adapter primarily learned to
follow style-guide constraints (banned phrases, honesty flags, segment fingerprints),
then a well-constructed system prompt that contains those same constraints should
produce similar behavioral outputs, even though the mechanism differs. If the
adapter learned something the style guide does not explicitly state — implicit
stylistic patterns, subtle phrasing preferences, context-dependent tone adjustments
— the system prompt will miss those.

**Factor 2 — The task's sensitivity to token-distribution differences.** Your
rubric is rule-based on four of five dimensions (banned phrases, ICP fingerprints,
TZ tokens, honesty flags). These dimensions respond to whether specific strings
appear in the output, not to the fine-grained token probability distribution. A
system prompt that successfully suppresses banned phrases will score identically to
an adapter that suppresses banned phrases, because the rubric cannot see the
mechanism. Only the tone-marker dimension (your LLM sub-judge) is sensitive to
subtle distributional differences.

---

## The Validity Boundary — What to Write in Your Model Card

Here is a technically grounded correction that replaces the current limitation
statement:

---

**Revised model card limitation (replace existing text):**

> **Evaluation method**: Delta A (+0.263, p<0.0001) measures the effect of
> system-prompt simulation — injecting the condensed Tenacious style guide v2 as a
> system prompt to the base Qwen3-1.7B model — compared to baseline (no system
> prompt). The trained LoRA adapter weights were not loaded during evaluation.
>
> **Validity as a proxy**: System-prompt simulation and LoRA adapter inference
> operate on different parts of the forward pass. The adapter modifies weight
> matrices W_eff = W₀ + (α/r)·B@A; system-prompt simulation modifies input
> activations while leaving weights unchanged. For rule-based rubric dimensions
> (banned phrases, ICP fingerprints, timezone tokens, honesty flags), the two
> approaches produce equivalent scores when the system prompt successfully encodes
> the same constraints the adapter learned — because these dimensions are sensitive
> to string presence, not token-distribution shape. For the tone-marker dimension
> (LLM sub-judge), fine-grained distributional differences between the two
> approaches may produce divergent scores.
>
> **Conservative interpretation**: Delta A should be read as an upper bound on
> the lift achievable via style-guide injection (prompt or adapter). Whether the
> trained adapter meets, exceeds, or falls below this bound on the tone-marker
> dimension is unknown. Given that 4 of 5 rubric dimensions
> are rule-based, the adapter result is unlikely to differ by more than the
> tone-marker dimension's weight (estimated ≤15% of the overall score). Delta A
> is a valid proxy for the four rule-based dimensions and an unvalidated proxy for
> tone.

---

## The Three-Line Test for Any Future Evaluation

Before reporting a Delta for any critic-gated system, verify three things:

1. **What changed between baseline and treatment?** Weight matrices (adapter),
   input context (system prompt), or decoding strategy (temperature, k)? Name it
   precisely.

2. **Is the rubric sensitive to the mechanism or only to the output?** Rule-based
   rubrics are mechanism-blind — they score strings, not distributions. LLM
   sub-judges are mechanism-sensitive. Know which dimensions fall into which
   category.

3. **Are the two conditions comparable on the dimension that varies?** A
   system-prompt simulation is a valid proxy for adapter inference on
   string-matching dimensions. It is an unvalidated proxy on distributional
   dimensions. State the boundary, don't elide it.

---

## What Was Scoped Out

This explainer covers the LoRA forward pass, the numerical equivalence of merged
and unmerged inference, and the specific validity question for your Delta A result.
It does not cover:

- How LoRA rank r and alpha interact with the adapter's expressive capacity (the
  intrinsic dimension literature — Aghajanyan et al. 2021)
- Why B is initialized to zero (to ensure the adapter starts as an identity
  perturbation — covered in the original LoRA paper, Section 4.1)
- The case where merged adapter inference produces different results from unmerged
  due to quantization (relevant only in 4-bit or 8-bit quantized inference, not
  in your fp16 setup)

---

## Sources

**Hu et al. 2022 — "LoRA: Low-Rank Adaptation of Large Language Models"**
(arXiv 2106.09685). The original paper. Section 4 defines the W_eff = W₀ + BA
formulation, the zero-initialization of B, and the alpha/r scaling. Section 7
contains the analysis of why merged and unmerged inference are equivalent.
Load-bearing for the forward-pass mechanics section of this explainer.

**Meng et al. 2024 — "SimPO: Simple Preference Optimization with a
Reference-Free Reward"** (arXiv 2405.14734). Load-bearing for understanding what
the SimPO training objective causes the adapter to learn: it optimises the
length-normalised reward margin between chosen and rejected outputs. Whether that
learned behavior is recoverable via system-prompt simulation depends on whether
the margin-generating behavior is expressible as explicit instructions — which is
the validity question this explainer addresses.

---

_Explainer by: [Nahom Desalegn S] · answering question filed by yakobd ·
Tenacious-Bench v0.1 · 10academy TRP1 Week 11_
