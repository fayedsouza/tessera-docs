# Query the audit log

Filter events, page through large result sets, and reconstruct what happened in a given
window.

Requires a token with the `events:read` scope.

## A basic query

`GET /v1/events` returns events newest first:

```bash
curl "https://api.tessera.example/v1/events?limit=25" \
  -H "Authorization: Bearer $TESSERA_TOKEN"
```

```json
{
  "data": [
    { "id": "evt_01JB8Q2X7K", "sequence": 184023, "action": "read",
      "outcome": "allowed", "occurred_at": "2026-09-27T09:14:02Z",
      "principal": { "id": "user_4471", "type": "user" },
      "resource": { "id": "doc_99120", "type": "document" } }
  ],
  "has_more": true,
  "next_cursor": "cur_eyJzZXEiOjE4NDAyM30"
}
```

## Filters

Combine any of these as query parameters. Multiple filters are ANDed.

| Parameter | Example | Matches |
|---|---|---|
| `principal_id` | `user_4471` | Events by one principal |
| `principal_type` | `service` | Events by principals of a type |
| `resource_id` | `doc_99120` | Events against one resource |
| `resource_type` | `document` | Events against resources of a type |
| `action` | `export` | One action. Repeat the parameter for several |
| `outcome` | `denied` | `allowed`, `denied` or `error` |
| `occurred_after` | `2026-09-01T00:00:00Z` | Inclusive lower bound on `occurred_at` |
| `occurred_before` | `2026-10-01T00:00:00Z` | Exclusive upper bound |
| `context.KEY` | `context.session_id=sess_a11b93` | Exact match on a key inside `context` |

Everything a user did to one resource in September:

```bash
curl -G "https://api.tessera.example/v1/events" \
  -H "Authorization: Bearer $TESSERA_TOKEN" \
  --data-urlencode "principal_id=user_4471" \
  --data-urlencode "resource_id=doc_99120" \
  --data-urlencode "occurred_after=2026-09-01T00:00:00Z" \
  --data-urlencode "occurred_before=2026-10-01T00:00:00Z"
```

Every denial across the workspace in the last day — the query most investigations start
with:

```bash
curl -G "https://api.tessera.example/v1/events" \
  -H "Authorization: Bearer $TESSERA_TOKEN" \
  --data-urlencode "outcome=denied" \
  --data-urlencode "occurred_after=2026-09-26T00:00:00Z"
```

!!! note "Filter on `occurred_at`, sort by `sequence`"
    Results are ordered by `sequence` — the order Tessera received them — not by
    `occurred_at`. For live traffic the two agree. For imported or replayed events they
    do not, and a batch imported today appears at the top of the log even if it describes
    last year. Sort client-side if chronological order matters to your reader.

## Pagination

Tessera uses cursor pagination. `limit` defaults to 25 and caps at 100.

When `has_more` is `true`, pass `next_cursor` back as `cursor`:

```bash
curl -G "https://api.tessera.example/v1/events" \
  -H "Authorization: Bearer $TESSERA_TOKEN" \
  --data-urlencode "outcome=denied" \
  --data-urlencode "cursor=cur_eyJzZXEiOjE4NDAyM30"
```

Keep every filter identical between pages. Changing a filter mid-pagination produces a
`400 cursor_filter_mismatch` rather than silently wrong results.

A cursor is valid for one hour. Because the log is append-only, a cursor is stable —
events written after you started paging do not shift the pages you have already seen.

```python
def all_events(session, **filters):
    cursor = None
    while True:
        params = {**filters, "limit": 100}
        if cursor:
            params["cursor"] = cursor
        page = session.get("https://api.tessera.example/v1/events",
                           params=params).json()
        yield from page["data"]
        if not page["has_more"]:
            return
        cursor = page["next_cursor"]
```

## Counting without paging

For a count alone, use `/v1/events/count` with the same filters. It returns a number
without the result bodies and does not consume the query rate budget:

```bash
curl -G "https://api.tessera.example/v1/events/count" \
  -H "Authorization: Bearer $TESSERA_TOKEN" \
  --data-urlencode "outcome=denied" \
  --data-urlencode "occurred_after=2026-09-01T00:00:00Z"
```

```json
{ "count": 1483 }
```

Counts above 100,000 are returned as `{ "count": 100000, "exact": false }`. If you need
an exact figure above that, use an [export](export-for-compliance.md).

## When not to use the query API

Paging a million events over HTTP to build a report is slow and will hit rate limits.
For anything approaching a full-corpus read, create an export instead — it runs
asynchronously and delivers a single archive.

As a rule of thumb: under ten thousand events, query; above that, export.

## Related

- [Export for compliance](export-for-compliance.md)
- [Handle rate limits](handle-rate-limits.md)
- [Error reference](../reference/errors.md)
