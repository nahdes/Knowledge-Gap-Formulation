# Tweet Thread — Day 1

**Topic:** Retry logic, backoff, jitter, and retry budgets  
**Written by:** Nahom

---

**Tweet 1**
Most API failures in agent pipelines are silent permanent failures masquerading as transient ones. Your Cal.com booking call timed out — was the booking created or not? Your answer to that question determines whether retrying is safe or whether you're about to double-book a prospect. A thread on retry logic, backoff, and why each handler needs a different strategy. 🧵

---

**Tweet 2**
The first classification: transient vs. permanent failure.

Retry on: 429, 5xx, connection reset, read timeout (with caveats).  
Raise immediately on: any other 4xx — the server understood you and said no.

```python
if resp.status_code == 429 or 500 <= resp.status_code < 600:
    # retryable
    ...
raise PermanentError(f"HTTP {resp.status_code}")
```

Retrying a 400 wastes quota and delays the failure you need to surface.

---

**Tweet 3**
Exponential backoff with jitter prevents retry storms.

If 100 clients all hit a 503 and retry after exactly 1s, they hit the overloaded server again simultaneously. Jitter desynchronizes them.

```python
delay = BASE * (2 ** (attempt - 1))  # exponential
delay = min(delay, CAP)              # ceiling
delay += random.uniform(0, 0.5)      # jitter
```

AWS measured this in 2015: full jitter cut retry call volume by 75% vs. fixed-interval retry.

---

**Tweet 4**
The ceiling (`min(delay, CAP)`) is your retry budget enforcer.

Without it, exponential backoff eventually sleeps for hours. With `CAP = 15s` and `MAX_ATTEMPTS = 4`, your worst-case blocked time is ~45s — a number you can actually reason about when setting timeouts on the upstream span.

If the server gives you a `Retry-After` header, use that instead of your computed delay. The provider knows its own rate-limit window; you don't.

---

**Tweet 5**
The part everyone misses: not all calls can be retried the same way.

- Email send (Resend): safe to retry with an idempotency key — provider deduplicates.
- Careers page scrape: idempotent read, safe to retry on timeout.
- Cal.com booking: **NOT safe to retry naively**. A timeout after the server accepted the POST may mean the booking was created. Retry = duplicate meeting.

Fix: pass an idempotency key on the booking request. Check before retry if you can't.

---

**Tweet 6**
The gap I closed today: I had retry logic in `email_handler.py` but not in `calendar_handler.py`. Reading both files side by side made the asymmetry visible — and revealed that the missing piece in the calendar handler isn't just a retry loop, it's an idempotency key that makes retrying safe in the first place.

Full explainer → [blog post URL]
