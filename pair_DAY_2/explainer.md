# Explainer: Retry Logic, Backoff, Jitter, and Retry Budgets

**Written for:** [Partner name]  
**Question:** What distinguishes a transient from a permanent API failure, how does exponential backoff with jitter prevent retry storms, what is a retry budget and why do you need a ceiling, and would the same retry strategy apply to all three affected files?

---

## The Question in Context

Our conversion engine calls external APIs at every stage of the pipeline. A Cal.com booking call, a Resend email send, a Playwright careers-page scrape — each can fail. The system currently handles these failures differently across files: `email_handler.py` has a full retry loop; `calendar_handler.py` and the Playwright scraper in `enrichment.py` do not. One timeout in either of those means a permanent, silent failure. Understanding *why* the retry strategy in `email_handler.py` is correct — and why you cannot copy it verbatim to the other two files — is the gap this explainer closes.

---

## The Load-Bearing Mechanism

**Transient vs. permanent failure** is the first classification every retry decision depends on. A permanent failure is one where retrying cannot help: a 400 Bad Request means your payload is malformed; a 422 means the server understood you and rejected the data; a 404 means the resource does not exist. Retrying these wastes time and quota. A transient failure is one where the server is temporarily unavailable or overloaded and a later attempt may succeed: a 429 Too Many Requests, a 500/503 from a momentarily unhealthy instance, a connection reset, or a read timeout where the server accepted the request but the network dropped the response.

The rule: **retry on `429`, `5xx`, and transport exceptions (timeout, connection reset). Raise immediately on any other `4xx`.**

`email_handler.py:144` implements this exactly:
```python
if resp.status_code == 429 or 500 <= resp.status_code < 600:
    # retryable — sleep and loop
    ...
# Permanent 4xx — don't retry.
raise EmailSendError(...)
```

**Exponential backoff with jitter** is what you do during the sleep between retries. Naive fixed-interval retry causes a retry storm: if 100 clients all hit a 503 at the same moment, they all retry one second later, hitting the server again simultaneously. Exponential backoff grows the wait time geometrically — attempt 1 waits 1s, attempt 2 waits 2s, attempt 3 waits 4s — so the load spreads over time. Jitter adds randomness on top: `delay += random.uniform(0, 0.5)`. This desynchronizes clients that entered the retry loop at the same moment. AWS's 2015 analysis of their own retry storms showed that jitter reduced call volume by 75% and client-side errors by 98% versus fixed-interval retry (see sources).

`email_handler.py:72-74`:
```python
delay = BACKOFF_BASE * (2 ** (attempt - 1))
delay = min(delay, BACKOFF_CAP)
delay += random.uniform(0, 0.5)
```

The `min(..., BACKOFF_CAP)` is the **retry budget ceiling**. Without it, exponential growth eventually produces delays measured in hours — a thread sitting on `time.sleep(3600)` while the rest of the pipeline stalls. The ceiling trades theoretical optimal spread for practical throughput. `BACKOFF_CAP = 15.0` in `email_handler.py` means the worst-case per-attempt sleep is ~15.5s. Combined with `MAX_ATTEMPTS = 4`, the worst-case total blocked time before the function raises is ~45s — a number a caller can reason about and a timeout you can set on the surrounding Langfuse span.

**Retry-After**: when the server tells you exactly how long to wait, use that instead. `email_handler.py:65-70` reads the `Retry-After` header from Resend 429 responses and prefers it over the computed backoff. This is correct: the provider knows its own rate-limit window; your computed delay may be shorter or longer than what it actually needs.

---

## Why Each File Needs a Different Approach

This is where the single biggest mistake in copy-paste retry logic lives.

**`calendar_handler.py` — booking is not idempotent.** `book_slot` posts to Cal.com's bookings endpoint. If the HTTP call succeeds on the Cal.com server but the response is dropped (network partition, read timeout), you cannot distinguish "the booking was created and I lost the response" from "the booking was never created." A naive retry creates a duplicate booking. The fix has two parts: (1) add a retry loop only on *connection errors*, not on *read timeouts after the request was sent*; and (2) pass an idempotency key in the request so Cal.com deduplicates. `email_handler.py` already does this correctly for Resend via `Idempotency-Key`. The same pattern applied to `calendar_handler.py:97` would pass a `uid` derived from `name + email + start` so a second POST for the same slot returns the existing booking rather than creating a new one.

**`enrichment.py` — the Playwright scraper is a read.** Scraping a careers page is idempotent — re-running it has no side effects. Retrying is safe. But the current failure path wraps the exception into a `status='error'` dict and returns it, which means the hiring brief continues with `total_roles=0, confidence='low'`. A single Playwright retry on `TimeoutError` or a navigation exception would recover most transient cases (flaky JS rendering, slow initial load). The retry budget here is different: one retry is probably enough; the Playwright timeout itself is already 20s, so two attempts total = 40s, which fits within a reasonable enrichment timeout budget.

---

## Adjacent Concepts

**Idempotency keys** are what make retry safe for mutating operations. They are not a retry-specific concept — they belong on every API call that creates or modifies state, regardless of retry. The absence of an idempotency key on `book_slot` is the actual bug; the absence of retry is a second, separate problem.

**Circuit breakers** are the layer above retry budgets. Where a retry budget limits *per-call* attempts, a circuit breaker tracks the *overall* failure rate of a dependency and stops sending requests entirely when the rate exceeds a threshold. For our pipeline — low-volume, not customer-facing in real-time — retry budgets are sufficient. Circuit breakers become necessary when call volume is high enough that retries themselves become a significant fraction of the load on the failing service.

---

## Pointers

**Canonical sources:**
- Brooker, M. (2015). *Exponential Backoff and Jitter*. AWS Architecture Blog. Demonstrates empirically that full jitter outperforms fixed and truncated exponential backoff under concurrent load.
- Beyer, B. et al. (2016). *Site Reliability Engineering*, Chapter 21: "Handling Overload." O'Reilly. Covers retry budgets, load shedding, and the interaction between client retry behavior and server-side backpressure.

**Tool used:** Read `agent/email_handler.py` and `agent/calendar_handler.py` directly, comparing the retry loop in the former against the bare `requests.post` in the latter. The concrete difference is visible at `email_handler.py:121-154` vs `calendar_handler.py:97-105`.

**Follow-on direction:** The next question worth asking is how to handle the partial-success case in `book_slot_and_sync`: Cal.com succeeds but HubSpot write fails. That is not a retry problem — it is a compensation/saga problem. The comment at `calendar_handler.py:172` already flags it: "the exception propagates so the caller can compensate." What that compensation looks like is the adjacent open question.
