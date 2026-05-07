# Tweet Thread — Day 1

## Thread: DPO "has no reward model" — that's a myth, and here's the proof

---

**Tweet 1**
"DPO doesn't have reward model overoptimization because there's no reward model."

This is one of the most common wrong intuitions in post-training. DPO absolutely has a reward model. It's the policy itself. 🧵

---

**Tweet 2**
Rafailov et al. 2023 showed the implicit reward in DPO is:

`r*(x,y) = β · log(π_θ(y|x) / π_ref(y|x))`

The policy ratio IS the reward. β controls the KL brake.

When `rewards/margins` climbs during training, the policy is hill-climbing this proxy — not your eval metric.

---

**Tweet 3**
The overoptimization fingerprint is specific: **mean score up, pass rate down**.

- Both metrics falling → noise overfit or OOD shift
- Both metrics rising → genuine generalization
- Mean up, pass down → proxy misalignment

The policy learned to score high on soft rubric dims at the cost of hard-fail constraints.

Why? Because the preference pairs were labeled on soft dims. Hard-policy failures had weak margin signal. DPO only optimizes what the gradient sees.

---

**Tweet 4**
Gao et al. 2023 measured this in classic RLHF: past a KL threshold, RM scores keep rising while gold scores fall.

Same pattern. Same cause. Different plumbing.

In DPO: small β = weak KL penalty = policy drifts past the threshold where proxy and true objective agree.

---

**Tweet 5**
Three dials, in order of ease:

1. **Increase β** — stronger KL anchor, less proxy drift. Try 2–4× your current value and compare pass_rate checkpoints.
2. **Stop on pass_rate, not mean_score** — mean score looks like it's still improving past the real optimum.
3. **IPO (Azar et al. 2023)** — replaces logistic loss with squared loss that saturates; prevents margin from growing unbounded.

The model card "Limitations" section can now name the mechanism, not just the symptom.

---

**Tweet 6**
Full explainer: [blog link]

Papers worth your time:
- Rafailov 2023 (DPO) → arxiv.org/abs/2305.18290
- Gao 2023 (overoptimization scaling laws) → arxiv.org/abs/2210.10760
- Azar 2023 (IPO) → arxiv.org/abs/2310.12036
