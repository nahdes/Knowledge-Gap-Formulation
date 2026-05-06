# Morning Call Summary — Day 1

**Pair:** Nahom & Mikias Dagem  
**Written by:** Nahom (confirmed by Mikias)

---

Nahom's original draft asked broadly "how does function-calling work at the token level" without specifying what part of the computation he was confused about. Mikias pushed back: "are you asking about how the model encodes tool schemas, how it decides between tools, or how it generates the arguments?" The question was narrowed to a single causal-ordering claim — does the description string influence the tool-name token's probability distribution, or is the description consumed only during argument generation? That reframe made the question answerable in one explainer.

Mikias's original draft asked "what is retry logic and how do I implement it" — broad enough to require a textbook chapter. Nahom pressed: "what specifically breaks in our pipeline when a retry is missing? Name a file and a failure mode." Mikias landed on the three-file framing (Cal.com booking, Resend email, Playwright scraper) and the sub-question of whether the same strategy applies to all three. The addition of "would each file need the same approach, and why?" was the sharpening move — it forced the explainer to address idempotency, not just backoff math.

Both questions were finalized by end of call.
