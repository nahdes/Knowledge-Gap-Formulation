# DPO Has No Reward Model — And That's Exactly Why It Overoptimizes

**Written for:** a peer's gap: "what plays the role of reward model overoptimization in DPO, and how does it explain mean score going up while pass rate goes down?"

**Grounded in:** `ablations/ablation_results.json` — trained variant `mean_score_pct: 97.92` / `pass_rate: 0.82` vs. baseline `93.44` / `0.86`. Delta A is a real mean-score lift (+4.48, p≈0.0002); pass rate moved the wrong way.

---

## The question, in concrete terms

Your held-out results show something that looks paradoxical: the trained model scores higher on average yet fails the quality bar more often than baseline. In classic RLHF you would immediately reach for "reward model overoptimization" — the policy hill-climbed a learned proxy that diverged from the true objective. But you trained with DPO, which has no separate reward model. So what went wrong?

The answer: DPO does have a reward model. It is the policy itself.

---

## The load-bearing mechanism

Rafailov et al. (2023) showed that the RLHF objective — maximize expected reward subject to a KL penalty from a reference policy — has an analytic solution. That solution can be rearranged so the reward for output `y` given prompt `x` is expressed directly in terms of the trained and reference policies:

```
r*(x, y) = β · log(π_θ(y|x) / π_ref(y|x)) + constant
```

There is no separate model. The **policy ratio** — how much more probable the trained policy is compared to a frozen reference — is the reward signal. The `β` coefficient is the KL anchor: large β keeps the policy close to `π_ref`; small β lets it drift.

DPO's training loss then maximizes the *margin* between this implicit reward for chosen vs. rejected outputs:

```
L_DPO = -log σ( β·log(π_θ(chosen)/π_ref(chosen)) - β·log(π_θ(rejected)/π_ref(rejected)) )
```

When you see `rewards/margins` climbing monotonically during training, the policy is pushing chosen outputs to higher log-probability and rejected outputs to lower log-probability — relative to `π_ref`. That is the optimization working. The problem is *what it is actually maximizing*.

---

## What the climbing margin is really learning

Your preference pairs were constructed with an offline rubric judge. "Chosen" outputs scored high on **soft rubric dimensions**: tone, ICP alignment, signal specificity. The margin threshold (≥0.30) was driven by those dimensions — they were the discriminators. Hard-policy failures (capacity over-commitment, TCV quoting, discount language) may appear in pairs as minority signals, but they rarely drove the ≥0.30 margin.

After N epochs of DPO, the policy has learned one thing very well: produce outputs that look like the "chosen" side of those pairs — maximizing the implicit reward proxy. It has not learned to avoid hard-policy failures, because those failures provided weak or noisy gradient signal in the preference data.

This is reward proxy misalignment. The rubric soft-score margin is the proxy; pass/fail rate is the true objective. The policy optimized the proxy past the point where the proxy and the true objective agree.

---

## Why mean up, pass down is the specific fingerprint

```
# Rough sketch: how the two metrics can diverge under proxy overoptimization
# soft_score = weighted average of rubric dimensions (continuous)
# pass = (soft_score > threshold) AND (no hard-fail category present)

# DPO gradient signal came from soft_score margin
# → policy pushes soft_score UP
# hard-fail categories had weak margin signal
# → policy develops no brake on hard-fail outputs
# → pass rate can fall even as soft_score rises
```

| Metric pattern | Most likely cause |
|---|---|
| Both mean score and pass rate fall | Overfit to preference noise or pure OOD shift |
| Both rise | Genuine generalization |
| **Mean up, pass down** | **Proxy misalignment: soft-dim reward overoptimization** |
| Both null (Δ ≈ 0, p > 0.1) | Candidate pool homogeneity (nothing to rerank) |

The divergent pattern rules out preference-noise overfit (that degrades both metrics) and LoRA-rank overfit (capacity overfit degrades all metrics uniformly). It is the specific signature of a policy that learned the scoring proxy better than the true objective.

---

## What Gao et al. (2023) showed and how it applies here

Gao et al. measured reward model overoptimization as a function of KL divergence from the reference policy. Their main finding: past a KL threshold, proxy-RM scores keep rising while gold scores fall — the two metrics diverge. That is exactly what your held-out run shows, except the "RM score" is your continuous rubric mean and the "gold score" is pass_rate.

In DPO, if β is too small, the optimizer is weakly penalized for KL divergence and the policy drifts far from `π_ref` — past the KL threshold where proxy and true objective agree.

---

## The three dials

**1. Increase β (the KL anchor)**  
Larger β penalizes divergence from `π_ref` more heavily. If training showed pass_rate declining while mean_score rose, increasing β by 2–4× and comparing dev-set pass_rate checkpoints is the fastest diagnostic. The tradeoff: higher β slows convergence and may leave the policy under-trained on legitimate preference signal.

**2. Early stop on pass_rate, not mean_score**  
Mean score is the proxy; pass_rate is closer to the true objective. Use dev-set pass_rate as the checkpoint criterion. Mean score will look like it is still improving past the actual optimum; pass_rate gives the earlier warning.

**3. IPO or a margin-constrained variant**  
Azar et al. (2023) showed that DPO's logistic loss has no regularization against assigning arbitrarily large margins to individual training pairs — the loss can be driven to zero by margin → ∞. IPO replaces the logistic loss with a squared loss that saturates, preventing runaway margin growth. For SimPO (which removes the reference model entirely, making overoptimization even less constrained), increasing the γ margin hyperparameter or reducing epoch count are the equivalent levers.

---

## What to write in the model card now

**Summary:** "Training improved rubric-weighted mean score (+4.48 pct points, p≈0.0002) but degraded pass rate (−0.04 absolute). The pattern is consistent with soft-dimension proxy overoptimization under DPO's implicit reward: the policy learned to maximize preference margin on the rubric's soft axes in ways that diverged from pass/fail criteria."

**Bias, Risks, and Limitations:** "DPO's β-weighted log policy ratio acts as a learned proxy for the training rubric. Residual failure clusters in capacity over-commitment, TCV quoting, and discount language represent hard-policy constraints that were minority signals in preference pair construction and therefore undersupplied to the DPO gradient. The standard corrective — larger β, earlier stopping criterion anchored to pass_rate, or an IPO-style saturation loss — was not applied in this training run."

---

## Sources

1. Rafailov, R. et al. (2023). *Direct Preference Optimization: Your Language Model is Secretly a Reward Model.* NeurIPS 2023. https://arxiv.org/abs/2305.18290  
2. Gao, L. et al. (2023). *Scaling Laws for Reward Model Overoptimization.* ICML 2023. https://arxiv.org/abs/2210.10760  
3. Azar, M. et al. (2023). *A General Theoretical Paradigm to Understand Learning from Human Feedback.* https://arxiv.org/abs/2310.12036 — IPO, the loss that fixes DPO's unbounded margin problem.  
4. Tool used: `trl` DPOTrainer — varied `beta` across [0.01, 0.1, 0.5] on a toy preference set and compared dev-set checkpoint curves for mean_score vs. pass_rate. The divergence appears first at low β.
