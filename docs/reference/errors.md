# Error reference

Every error response carries the same shape:

```json
{
  "error": "invalid_field",
  "detail": "must be one of: allowed, denied, error",
  "field": "outcome",
  "request_id": "req_01JB8Q9F2M"
}
```

| Key | Always present | Notes |
|---|---|---|
| `error` | Yes | Stable machine-readable code. Branch on this |
| `detail` | Yes | Human-readable. Wording may change — do not parse it |
| `field` | No | Present on validation errors |
| `request_id` | Yes | Quote this in support requests |

Branch on `error`, never on `detail` or on the status code alone — several distinct
conditions share a status.

## 400 — Bad request

| `error` | Cause | Resolution |
|---|---|---|
| `invalid_json` | Body is not valid JSON | Check the payload and `Content-Type` |
| `missing_field` | A required field is absent | `field` names it |
| `invalid_field` | A field has an unacceptable value | `detail` states what is accepted |
| `invalid_timestamp` | `occurred_at` is not RFC 3339 | Use `2026-09-27T09:14:02Z` form, with an offset |
| `timestamp_too_old` | `occurred_at` over 365 days ago | Set `"backdated": true`, or import via a migration |
| `timestamp_in_future` | `occurred_at` more than 5 minutes ahead | Check clock sync on the emitting host |
| `batch_too_large` | Over 500 events | Split the batch |
| `cursor_expired` | Cursor older than one hour | Restart pagination |
| `cursor_filter_mismatch` | Filters changed mid-pagination | Keep filters identical across pages |

## 401 — Unauthenticated

| `error` | Cause |
|---|---|
| `missing_credentials` | No `Authorization` header |
| `malformed_credentials` | Header present but not `Bearer <token>` |
| `invalid_token` | Token not recognised |
| `token_revoked` | Token was revoked |
| `token_expired` | Token passed its expiry |

Do not retry a `401`. Nothing about waiting will make the credential valid.

## 403 — Authenticated, but refused

| `error` | Cause | Resolution |
|---|---|---|
| `insufficient_scope` | Token lacks the scope | `detail` names the scope needed. Create a token that has it |
| `wrong_environment` | Test token on production, or the reverse | Match the `tsk_live_` / `tsk_test_` prefix to the host |
| `workspace_suspended` | Workspace suspended for billing or policy | Contact your administrator |

## 404 — Not found

| `error` | Cause |
|---|---|
| `not_found` | No object with that ID in this workspace |

A `404` is also returned for objects that exist in another workspace. This is deliberate —
returning `403` would confirm the ID exists somewhere.

## 405 — Method not allowed

| `error` | Cause |
|---|---|
| `method_not_allowed` | `PATCH` or `DELETE` against an event |

Events are append-only. Append a correction instead — see
[Correcting a mistake](../concepts/events-and-retention.md#correcting-a-mistake).

## 409 — Conflict

| `error` | Cause | Resolution |
|---|---|---|
| `idempotency_key_reuse` | Key reused within 24 hours with a *different* payload | Use a new key, or resend the identical payload |
| `export_in_progress` | An identical export is already running | Poll the existing export |

An idempotency key reused with the *same* payload returns `200` and the original object.
Only a differing payload is a conflict.

## 413 — Payload too large

| `error` | Cause |
|---|---|
| `payload_too_large` | Request body over 5 MB |

Most often an oversized `context` object. Batches of 500 ordinary events stay well inside
the limit.

## 422 — Semantically invalid

| `error` | Cause |
|---|---|
| `correction_target_not_found` | `corrects` names an event that does not exist |
| `correction_of_correction` | Attempt to correct an event that is itself a correction |
| `retention_below_legal_hold` | Retention change would delete events under hold |

## 429 — Rate limited

| `error` | Cause |
|---|---|
| `rate_limited` | Window budget exhausted |

Honour `Retry-After`. Nothing in the request was written. See
[Handle rate limits](../guides/handle-rate-limits.md).

## 5xx — Server side

| Status | `error` | Retry |
|---|---|---|
| `500` | `internal_error` | Yes, with backoff |
| `502` / `504` | `upstream_timeout` | Yes, with backoff |
| `503` | `service_unavailable` | Yes, honour `Retry-After` if present |

Always retry a `5xx` on a write with the same `Idempotency-Key` — the request may have
been applied before the failure.

## Which errors are safe to retry

| Class | Retry | Why |
|---|---|---|
| `400`, `422` | No | Deterministic. Will fail identically |
| `401`, `403` | No | Fix the credential or scope first |
| `404` | No | Unless you are racing object creation |
| `409` | No | Change the key or payload |
| `429` | Yes | After `Retry-After` |
| `5xx` | Yes | With exponential backoff and jitter |

## Related

- [Handle rate limits](../guides/handle-rate-limits.md)
- [Authenticate with the API](../get-started/authentication.md)
