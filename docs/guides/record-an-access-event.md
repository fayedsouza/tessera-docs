# Record an access event

Write a single event, or a batch of events, to the audit log.

This guide assumes you have a token with the `events:write` scope. If you do not, see
[Create an API token](../get-started/authentication.md#create-an-api-token).

## Record one event

Send a `POST` to `/v1/events` with the four required fields — `principal`, `resource`,
`action` and `outcome`:

```bash
curl -X POST https://api.tessera.example/v1/events \
  -H "Authorization: Bearer $TESSERA_TOKEN" \
  -H "Idempotency-Key: $YOUR_UNIQUE_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "principal": { "id": "user_4471", "type": "user" },
    "resource":  { "id": "doc_99120", "type": "document" },
    "action": "read",
    "outcome": "allowed",
    "occurred_at": "2026-09-27T09:14:02Z"
  }'
```

A `201` response carries the assigned `id` and `sequence`. Log the `id` on your side if
you may later need to correct the event.

Always send `Idempotency-Key`. Without it, a retry after a network timeout writes a
duplicate, and duplicates cannot be deleted — only corrected. See
[Duplicates and idempotency](../concepts/events-and-retention.md#duplicates-and-idempotency).

## Record a batch

For bulk import or for buffering, `POST` up to 500 events to `/v1/events/batch`:

```bash
curl -X POST https://api.tessera.example/v1/events/batch \
  -H "Authorization: Bearer $TESSERA_TOKEN" \
  -H "Idempotency-Key: import-2026-09-27-chunk-014" \
  -H "Content-Type: application/json" \
  -d '{
    "events": [
      { "principal": { "id": "user_4471", "type": "user" },
        "resource":  { "id": "doc_99120", "type": "document" },
        "action": "read", "outcome": "allowed",
        "occurred_at": "2026-09-27T09:14:02Z" },
      { "principal": { "id": "user_4471", "type": "user" },
        "resource":  { "id": "doc_99120", "type": "document" },
        "action": "export", "outcome": "denied",
        "occurred_at": "2026-09-27T09:16:48Z" }
    ]
  }'
```

### Batches are partially accepted

A batch is **not** atomic. Tessera writes the events it can and reports the ones it
cannot, returning `207 multi_status` when the two are mixed:

```json
{
  "accepted": 1,
  "rejected": 1,
  "results": [
    { "index": 0, "status": "accepted", "id": "evt_01JB8Q2X7K", "sequence": 184023 },
    { "index": 1, "status": "rejected", "error": "invalid_field",
      "field": "outcome", "detail": "must be one of: allowed, denied, error" }
  ]
}
```

Match rejections back to your source data by `index`, which is the position in the array
you sent. Do not assume a `207` means retry the whole batch — resending accepted events
under a new idempotency key writes them twice.

!!! tip "Handle 207 before you go to production"
    The commonest bulk-import bug is treating anything that is not `200` as a total
    failure and replaying the batch. Write the `207` branch first.

## Required and optional fields

| Field | Required | Notes |
|---|---|---|
| `principal.id` | Yes | Stable identifier in your system |
| `principal.type` | Yes | `user`, `service`, `api_key` or `anonymous` |
| `principal.display_name` | No | Captured as at the time of the event |
| `resource.id` | Yes | Stable identifier in your system |
| `resource.type` | Yes | Your own vocabulary — agree it across teams |
| `action` | Yes | Your own vocabulary |
| `outcome` | Yes | `allowed`, `denied` or `error` |
| `occurred_at` | No | RFC 3339. Defaults to time of receipt |
| `reason` | No | Machine-readable code, useful on `denied` |
| `context` | No | Free-form object. Stored verbatim — see the warning below |

!!! warning "`context` is stored verbatim and kept for your full retention period"
    Tessera performs no redaction. Do not put personal data in `context` unless you
    intend to keep it for as long as the workspace retains events, and to include it in
    every export. See [Tokens and sessions](../concepts/tokens-and-sessions.md#session-ids-and-personal-data).

## Backdating events

Set `occurred_at` to the real time of the access. Tessera accepts timestamps up to 30
days in the past without comment and up to 365 days in the past with
`"backdated": true` set explicitly. Beyond 365 days the request is rejected.

Retention is still counted from `recorded_at`, so an imported historical event is kept
for the full period from the day you import it, not from the day it happened.

## Verify what you wrote

Read the event back by ID, or query the last few events for the principal:

```bash
curl "https://api.tessera.example/v1/events?principal_id=user_4471&limit=5" \
  -H "Authorization: Bearer $TESSERA_TOKEN"
```

## Related

- [Query the audit log](query-the-audit-log.md)
- [Handle rate limits](handle-rate-limits.md)
- [Events and retention](../concepts/events-and-retention.md)
