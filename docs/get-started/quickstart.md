# Quickstart

Record your first access event and read it back. This takes about ten minutes and needs
nothing installed beyond `curl`.

By the end you will have authenticated against the API, written one event, and retrieved
it by ID.

## Before you begin

You need:

- A Tessera workspace. If you do not have one, your workspace administrator can invite
  you, or you can create a sandbox workspace in the console.
- An API token with the `events:write` and `events:read` scopes. See
  [Create an API token](authentication.md#create-an-api-token) if you do not have one.

Store the token in an environment variable so it does not end up in your shell history:

```bash
export TESSERA_TOKEN="tsk_live_REPLACE_ME"
```

!!! warning "Tokens are shown once"
    Tessera displays a token's secret only at the moment you create it. If you lose it,
    revoke the token and create a new one — there is no way to retrieve the original
    value.

## Step 1 — Confirm your token works

Call the workspace endpoint. It requires authentication but changes nothing, which makes
it a safe first request.

```bash
curl https://api.tessera.example/v1/workspace \
  -H "Authorization: Bearer $TESSERA_TOKEN"
```

A working token returns your workspace and the scopes attached to the token:

```json
{
  "id": "ws_8f2a91c4",
  "name": "Acme Production",
  "region": "eu-west",
  "retention_days": 2555,
  "token_scopes": ["events:write", "events:read"]
}
```

If you get a `401`, the token is wrong, revoked or expired. If you get a `403`, the token
is valid but lacks the scope for this endpoint. See
[Error reference](../reference/errors.md) for the distinction.

## Step 2 — Record an event

An event describes one access attempt. Four fields are required: the principal who acted,
the resource they acted on, the action, and the outcome.

```bash
curl -X POST https://api.tessera.example/v1/events \
  -H "Authorization: Bearer $TESSERA_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "principal": {
      "id": "user_4471",
      "type": "user",
      "display_name": "j.okafor@acme.example"
    },
    "resource": {
      "id": "doc_99120",
      "type": "document",
      "display_name": "Q3 Board Pack"
    },
    "action": "read",
    "outcome": "allowed",
    "occurred_at": "2026-09-27T09:14:02Z",
    "context": {
      "ip": "203.0.113.42",
      "session_id": "sess_a11b93"
    }
  }'
```

Tessera responds with the stored event, including the ID it assigned and the sequence
number that fixes this event's position in the log:

```json
{
  "id": "evt_01JB8Q2X7K",
  "sequence": 184023,
  "recorded_at": "2026-09-27T09:14:02.881Z",
  "occurred_at": "2026-09-27T09:14:02Z",
  "outcome": "allowed"
}
```

!!! note "`occurred_at` and `recorded_at` are different"
    `occurred_at` is when the access happened in your system. `recorded_at` is when
    Tessera received it. They diverge when you batch or replay events, and auditors
    generally care about the first. If you omit `occurred_at`, Tessera sets it equal to
    `recorded_at`.

## Step 3 — Read the event back

Retrieve it by the ID from the previous response:

```bash
curl https://api.tessera.example/v1/events/evt_01JB8Q2X7K \
  -H "Authorization: Bearer $TESSERA_TOKEN"
```

The full stored event comes back, including the fields Tessera added:

```json
{
  "id": "evt_01JB8Q2X7K",
  "sequence": 184023,
  "principal": {
    "id": "user_4471",
    "type": "user",
    "display_name": "j.okafor@acme.example"
  },
  "resource": {
    "id": "doc_99120",
    "type": "document",
    "display_name": "Q3 Board Pack"
  },
  "action": "read",
  "outcome": "allowed",
  "occurred_at": "2026-09-27T09:14:02Z",
  "recorded_at": "2026-09-27T09:14:02.881Z",
  "context": {
    "ip": "203.0.113.42",
    "session_id": "sess_a11b93"
  },
  "workspace_id": "ws_8f2a91c4"
}
```

Try editing it — send a `PATCH` or a `DELETE` to the same URL. Both return `405`. That is
the point of the store: see [Events and retention](../concepts/events-and-retention.md)
for why, and for how corrections are handled instead.

## What you have now

One event in an append-only log, retrievable by ID, that no one can quietly alter.

## Where to go next

- [The access model](../concepts/access-model.md) — what principals, resources and grants
  actually mean, and how to map your own system onto them.
- [Query the audit log](../guides/query-the-audit-log.md) — filtering and pagination once
  you have more than one event.
- [Handle rate limits](../guides/handle-rate-limits.md) — read this before you write a
  bulk import.
