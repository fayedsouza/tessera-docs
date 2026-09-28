# Glossary

Terms as Tessera uses them. Several have looser meanings elsewhere; the definitions here
are the ones the API enforces.

### Action

The verb in an event — what the principal attempted. Free text, defined by you. Agree the
vocabulary across teams before emitting, since a log mixing `read`, `Read` and `view` for
one concept cannot be queried reliably.

### Append-only

The property that a written event cannot be modified or removed. Corrections are appended
as new events referencing the original. See
[Events and retention](../concepts/events-and-retention.md).

### Backdating

Recording an event with an `occurred_at` in the past. Permitted up to 30 days without
comment, to 365 days with `"backdated": true`, and not at all beyond that.

### Correction

An event that supersedes an earlier one via the `corrects` field. Both remain in the log;
neither is hidden. A correction cannot itself be corrected.

### Cursor

An opaque token for the next page of a query. Valid for one hour, and tied to the exact
filter set it was issued against.

### Event

One recorded access: a principal, a resource, an action, an outcome, and when it
happened. The central object of the API.

### Export

An asynchronous job producing a downloadable archive of events matching a filter, with a
manifest carrying a checksum and signature.

### Grant

A record that a principal holds standing permission over a resource. Tessera stores
grants but does not enforce them — a grant says what *may* happen, an event says what
did.

### Idempotency key

A caller-supplied header making a write safe to retry. Remembered for 24 hours; a repeat
within the window returns the original event rather than creating a second.

### Legal hold

A query-defined exemption from retention expiry. Events matching a hold are preserved
regardless of the workspace retention setting, and are flagged `"held": true`.

### Manifest

The metadata block on a completed export: SHA-256 checksum, signature, signing key ID and
sequence range. What an auditor verifies against.

### `occurred_at`

When the access happened in your system. Supplied by you. What auditors care about.

### Outcome

Whether the access succeeded. Exactly one of `allowed`, `denied`, `error`. Constrained by
the API, unlike `action` and resource `type`.

### Principal

Whoever performed the access — a `user`, `service`, `api_key` or `anonymous`. Identified
by an ID stable in your system.

### `recorded_at`

When Tessera received the event. Set by Tessera. Retention is counted from this, not from
`occurred_at`.

### Resource

Whatever was accessed. Model at the level where permission is actually controlled; finer
detail belongs in `context`.

### Retention

How long events are kept, per workspace, from 90 to 2,555 days. Expiry is deletion, not
archival. Lowering retention is irreversible.

### Scope

A permission attached to an API token: `events:write`, `events:read`, `exports:create`,
`grants:admin`. Fixed at creation.

### Sequence

A monotonic integer fixing an event's position in the workspace log. Never reissued.
Query results are ordered by it.

### Session ID

A caller-supplied opaque value in `context.session_id`, grouping related activity.
Meaningless to Tessera, indexed for querying, and retained as long as the events carrying
it — so it should encode nothing personal.

### Workspace

The isolation boundary. Tokens, events, retention, grants and sequence numbering all
belong to exactly one workspace.

### `written_by`

The credential that wrote an event, recorded by Tessera. Cannot be supplied or overridden
by the caller.
