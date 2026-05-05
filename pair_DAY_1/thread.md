# Tweet Thread (thread.md)

**Tweet 1/6**
Why does your critic get 97% preference accuracy but produce ZERO lift in rejection sampling?

I just got an excellent explainer that nailed the exact reason this keeps happening. Thread 🧵

**Tweet 2/6**
The core problem is **Margin Distribution Mismatch**.

Training: Your critic learns on high-margin pairs (chosen vs rejected deliberately very different).

Deployment: You generate k candidates from the _same model_ at the _same temperature_. They’re all slight variations of each other → almost no quality difference.

**Tweet 3/6**
Classic example:
Imagine a world-class wine expert (your critic). Give them 8 glasses poured from the _same bottle_. They can’t pick the “best” one meaningfully.

Same thing happens with LLMs. You get lexical diversity, not quality diversity.

**Tweet 4/6**
The fix is simple and cheap:

Run this diagnostic on your candidate scores:

```python
avg_std = mean([std(scores) for scores in candidate_pools])
if avg_std < 0.1 → Low variance pool → Fix generation (temp, models, prompts)
else → Critic is the bottleneck
```
