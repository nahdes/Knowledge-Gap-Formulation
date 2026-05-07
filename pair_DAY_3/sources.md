# Sources — Day 1 Explainer

## Canonical papers

1. **Rafailov, R. et al. (2023).** Direct Preference Optimization: Your Language Model is Secretly a Reward Model. *NeurIPS 2023.* https://arxiv.org/abs/2305.18290  
   — Primary source for the implicit reward derivation (`r* = β·log(π_θ/π_ref)`) and the reparameterization proof. Section 4 is the load-bearing part.

2. **Gao, L. et al. (2023).** Scaling Laws for Reward Model Overoptimization. *ICML 2023.* https://arxiv.org/abs/2210.10760  
   — Empirical characterization of the proxy/true-objective divergence as a function of KL budget. Figure 1 and Figure 3 show the "RM score up, gold score down" pattern directly.

3. **Azar, M. et al. (2023).** A General Theoretical Paradigm to Understand Learning from Human Feedback. https://arxiv.org/abs/2310.12036  
   — IPO paper. Section 3 proves DPO's logistic loss has no regularization against unbounded margin growth; Theorem 1 motivates the squared-loss fix.

## Tool used

**`trl` DPOTrainer** (HuggingFace Transformers Reinforcement Learning library).  
Ran a toy experiment varying `beta` across [0.01, 0.1, 0.5] on a small preference dataset. Observed that at β=0.01, dev-set pass_rate peaked at epoch 1 and then declined while mean score continued rising through epoch 3 — the divergence appears first at low β. At β=0.5, both metrics tracked together until a joint plateau.
