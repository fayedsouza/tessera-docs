# API reference

!!! info "In progress"
    This section will be generated from a hand-authored OpenAPI 3.1 specification
    (`openapi/tessera.yaml`) and rendered here by
    [`neoteroi-mkdocs`](https://www.neoteroi.dev/mkdocs-plugins/) at build time.

    Until the specification lands, the endpoint summary below is maintained by hand. The
    intent is that this page becomes generated output while the conceptual and task
    content around it stays hand-written — generated reference alone is not
    documentation.

## Base URL

```
https://api.tessera.example/v1
```

All requests require an `Authorization: Bearer` header. See
[Authenticate with the API](../get-started/authentication.md).

## Endpoints

### Events

| Method | Path | Scope | Description |
|---|---|---|---|
| `POST` | `/events` | `events:write` | Record one event |
| `POST` | `/events/batch` | `events:write` | Record up to 500 events. Partial acceptance |
| `GET` | `/events` | `events:read` | Query events with filters and cursor pagination |
| `GET` | `/events/count` | `events:read` | Count matching events without bodies |
| `GET` | `/events/{event_id}` | `events:read` | Retrieve one event by ID |

`PATCH` and `DELETE` on `/events/{event_id}` return `405`. Events are append-only.

### Exports

| Method | Path | Scope | Description |
|---|---|---|---|
| `POST` | `/exports` | `exports:create` | Create an export job |
| `GET` | `/exports/{export_id}` | `exports:create` | Poll status; returns download URL when complete |
| `GET` | `/exports` | `exports:create` | List recent exports |
| `POST` | `/exports/schedules` | `exports:create` | Create a recurring export to your own storage |
| `GET` | `/signing-keys/{key_id}` | — | Fetch a public signing key. Unauthenticated |

### Grants

| Method | Path | Scope | Description |
|---|---|---|---|
| `POST` | `/grants` | `grants:admin` | Record a standing permission |
| `GET` | `/grants` | `grants:admin` | List grants, filterable by principal or resource |
| `DELETE` | `/grants/{grant_id}` | `grants:admin` | Revoke a grant. Sets `revoked_at`; does not delete |

### Workspace

| Method | Path | Scope | Description |
|---|---|---|---|
| `GET` | `/workspace` | any | Workspace details and the calling token's scopes |

## Common objects

### Event

| Field | Type | Notes |
|---|---|---|
| `id` | string | Assigned by Tessera |
| `sequence` | integer | Monotonic within the workspace |
| `principal` | object | `id`, `type`, optional `display_name` |
| `resource` | object | `id`, `type`, optional `display_name` |
| `action` | string | Your vocabulary |
| `outcome` | enum | `allowed`, `denied`, `error` |
| `reason` | string | Optional machine-readable code |
| `occurred_at` | timestamp | RFC 3339. Defaults to `recorded_at` |
| `recorded_at` | timestamp | Set by Tessera |
| `context` | object | Free-form. Stored verbatim |
| `corrects` | string | Optional. ID of the event this supersedes |
| `corrected_by` | string | Set by Tessera when a correction arrives |
| `held` | boolean | Under legal hold |
| `written_by` | object | Set by Tessera. `token_id`, `token_name`, `source_ip` |

### Error

See the [error reference](errors.md) for codes and retry guidance.

## Pagination

Cursor-based. `limit` defaults to 25, maximum 100. Responses carry `has_more` and
`next_cursor`. Filters must stay identical across pages.

## Rate limits

See [Handle rate limits](../guides/handle-rate-limits.md).
