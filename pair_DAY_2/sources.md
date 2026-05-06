# Sources — Day 1

**Explainer written by:** Nahom  
**Topic:** Retry logic, exponential backoff, jitter, and retry budgets

---

## Canonical sources

**1. Brooker, M. (2015). *Exponential Backoff and Jitter.* AWS Architecture Blog.**  
Primary source for the empirical case that full jitter outperforms fixed-interval and truncated exponential backoff under concurrent load. The post includes simulation results showing retry storm suppression. Cited in the explainer to support the claim that jitter reduced call volume by 75% and client-side errors by 98% vs. fixed-interval retry. Available at the AWS Architecture Blog.

**2. Beyer, B., Jones, C., Petoff, J., & Murphy, R. (Eds.). (2016). *Site Reliability Engineering: How Google Runs Production Systems*, Chapter 21: "Handling Overload." O'Reilly Media.**  
Primary source for retry budgets, load shedding, and the distinction between per-call retry limits and system-level backpressure. The chapter defines the concept of a retry budget as a ceiling that prevents retry amplification from compounding a partial outage into a full one. Cited in the explainer to ground the `MAX_ATTEMPTS` and `BACKOFF_CAP` constants in `email_handler.py`.

---

## Tool / pattern used hands-on

Read `agent/email_handler.py` (the correct retry implementation, lines 121–154) and `agent/calendar_handler.py` (the missing retry, lines 97–105) side by side in the live codebase. The concrete comparison between the two files — one with `MAX_ATTEMPTS`, `BACKOFF_CAP`, jitter, and `Retry-After` parsing; one with a bare `requests.post` and no loop — is the demonstration in the explainer. The idempotency gap in `calendar_handler.py` (no idempotency key on a mutating booking call) was discovered during this read, not before.

---

## Follow-on directions

- OpenAI Cookbook: "How to handle rate limits" — covers token-bucket vs. leaky-bucket rate limiting from the client side, adjacent to the retry budget discussion.
- `tenacity` Python library documentation — the canonical Python retry decorator; useful if the pipeline grows to the point where hand-rolled retry loops in every handler become a maintenance burden.
