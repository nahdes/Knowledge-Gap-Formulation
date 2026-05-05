# Sources - Day 1

## Canonical Papers

1. **Stiennon et al. (2020)** — _Learning to summarize with human feedback_  
   NeurIPS 2020 | https://arxiv.org/abs/2009.01325  
   Load-bearing for the Best-of-N / rejection sampling pipeline and the relationship between reward models and downstream task improvement.

2. **Lambert et al. (2024)** — _RewardBench: Evaluating Reward Models for Language Modeling_  
   arXiv:2403.13787 | https://arxiv.org/abs/2403.13787  
   Load-bearing for understanding why high preference accuracy frequently fails to predict downstream performance.

## Tool / Hands-on Experiment

- **Python simulation** using NumPy (seed=42) to demonstrate the margin distribution mismatch (12.5× difference between training margins and deployment pool margins).  
  Code was run and verified during explainer writing.

## Additional References (Context)

- Original LoRA explainer (reciprocal) used Hu et al. (2021) as primary source.

**All claims in the explainer trace back to these sources or the verified simulation.**
