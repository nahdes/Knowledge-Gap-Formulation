# Evening Call Summary — Day 2

**Pair:** Nahom Desalegn + Melkam Beyene
**Topic:** Evaluation and statistics

---

Melkam's feedback on Nahom's explainer (paired vs. regular bootstrap): the worked example with 4 tasks showing σ_d=0 was clear and the CI width ratio table was the most useful element. She asked for the n=52 realistic-variance section to lead with the actual Week 11 σ=0.20 and ρ=0.85 numbers earlier so the reader doesn't have to hold abstract parameters in mind — this was incorporated.

Nahom's feedback on Melkam's explainer (LLM-as-judge single-output biases): the two-uncertainty split in §1 (sampling variance vs. judge measurement variance) was the key contribution and should have appeared earlier in the explainer rather than buried after the bias taxonomy. Melkam also noted the detection methods in §5 needed a distinction between checks runnable retroactively on existing outputs (score–length correlation) versus checks requiring a new scoring run (inter-judge calibration) — the distinction was clarified in the final version.

Both agreed the framing upgrade — "under-identified at this margin" rather than "null result" — is the most generalizable takeaway from Day 2 and should appear in both the thread and the grounding commit, not just the explainer.
