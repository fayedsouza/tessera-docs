# Handle rate limits

Read this before you write a bulk import. Rate limiting is the most common cause of a
partial import that nobody notices until an auditor asks why March is missing.

## The limits

Limits are per workspace, not per token. Adding tokens does not add capacity.

| Endpoint group | Limit |
|---|---|
| `POST /v1/events` | 1,000 requests per minute |
| `POST /v1/events/batch` | 60 requests per minute, 500 events per request |
| `GET` on events | 300 requests per minute |
| `POST /v1/exports` | 10 per hour |

Batch is the reason to prefer it for volume: 60 × 500 is 30,000 events a minute against
1,000 for single writes.

## Reading the headers

Every response carries the current budget:

```
RateLimit-Limit: 1000
RateLimit-Remaining: 847
RateLimit-Reset: 34
```

`RateLimit-Reset` is seconds until the window refills. Watch `RateLimit-Remaining` and
slow down as it falls, rather than waiting to be rejected — a client that only reacts to
`429` spends its time oscillating between full speed and blocked.

## When you are limited

Tessera returns `429 rate_limited` with a `Retry-After` header in seconds:

```
HTTP/1.1 429 Too Many Requests
Retry-After: 12
```

```json
{
  "error": "rate_limited",
  "detail": "Workspace write limit exceeded",
  "retry_after": 12
}
```

Honour `Retry-After`. It is authoritative, and retrying sooner extends the window rather
than shortening it.

!!! danger "A 429 means nothing was written"
    Unlike a `207`, a `429` rejects the entire request — no events from that batch were
    stored. Retry the whole payload, with the same `Idempotency-Key`, so that a request
    which actually did land before the timeout is not written twice.

## Retry correctly

Exponential backoff with jitter, capped, and bounded by an attempt limit:

```python
import random, time, requests

def post_with_retry(session, url, payload, idem_key, max_attempts=6):
    for attempt in range(max_attempts):
        r = session.post(url, json=payload,
                         headers={"Idempotency-Key": idem_key})

        if r.status_code == 429:
            wait = int(r.headers.get("Retry-After", 2 ** attempt))
        elif 500 <= r.status_code < 600:
            wait = min(2 ** attempt, 60)
        else:
            return r                      # 2xx, 207 and 4xx all return here

        time.sleep(wait + random.uniform(0, 1))   # jitter

    raise RuntimeError(f"gave up after {max_attempts} attempts: {idem_key}")
```

Three things this gets right and most hand-rolled loops do not:

1. **The same idempotency key on every attempt.** A new key per attempt turns a retry
   into a duplicate.
2. **Jitter.** Without it, a fleet of workers that hit the limit together will retry
   together and hit it again.
3. **4xx is not retried.** A `400` will be a `400` forever. Retrying it wastes the budget
   that your valid requests need.

## Bulk imports

For a one-off import of historical data:

- Use `/v1/events/batch` with 500 events per request.
- Run one worker, not several. The limit is per workspace, so parallel workers compete
  for the same budget and mostly generate `429`s.
- Derive idempotency keys from your source data — `import-<run-id>-<chunk>` — so a
  resumed run cannot duplicate a chunk that already landed.
- Record the last successfully written chunk somewhere durable. An import that dies at
  chunk 4,000 of 9,000 should resume, not restart.
- Reconcile at the end: compare your source row count against
  `GET /v1/events/count` with a filter matching the import.

That last step is the one teams skip. A partial import looks exactly like a complete one
until somebody counts.

!!! tip "Ask for a temporary increase"
    For a large one-off migration, support can raise limits for a defined window. This
    is usually faster and safer than engineering around the standard budget.

## Related

- [Record an access event](record-an-access-event.md)
- [Error reference](../reference/errors.md)
