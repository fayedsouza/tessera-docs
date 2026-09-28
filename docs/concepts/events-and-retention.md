# Events and retention

Tessera's central promise is that a recorded event does not change. This page explains
what that guarantee actually covers, how to correct a mistake without breaking it, and
how long records live.

## Append-only, and what that means

Once Tessera accepts an event it assigns a `sequence` number and writes it. From that
moment:

- `PATCH` and `DELETE` against an event return `405 method_not_allowed`.
- No console action, support request or administrator role can alter the stored fields.
- The `sequence` is monotonic within a workspace and never reissued, so a gap in the
  sequence is evidence of nothing being hidden — there is nowhere to hide it.

The guarantee covers the event's own fields. It does not cover the world outside: if a
`display_name` was wrong at capture, the log faithfully preserves the wrong name. That is
the intended behaviour. The log records what your system believed at the time.

## Correcting a mistake

You cannot edit. You append a correction that points at the original.

```bash
curl -X POST https://api.tessera.example/v1/events \
  -H "Authorization: Bearer $TESSERA_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "corrects": "evt_01JB8Q2X7K",
    "correction_reason": "principal_misattributed",
    "principal": { "id": "user_5522", "type": "user",
                   "display_name": "a.mensah@acme.example" },
    "resource":  { "id": "doc_99120", "type": "document",
                   "display_name": "Q3 Board Pack" },
    "action": "read",
    "outcome": "allowed",
    "occurred_at": "2026-09-27T09:14:02Z"
  }'
```

Both events remain in the log. Queries return the original with a `corrected_by` pointer
and the correction with a `corrects` pointer, so a reader always sees that a revision
happened and can read both versions.

!!! warning "Corrections are visible, not silent"
    A correction does not suppress the original. If your compliance posture requires that
    an erroneous record never be surfaced at all, Tessera is the wrong store — and you
    should expect any auditor to ask why records disappear.

`correction_reason` accepts: `principal_misattributed`,
`resource_misattributed`, `outcome_incorrect`, `duplicate`, `test_data`, `other`. Where
`other` is used, `correction_note` becomes required.

## Duplicates and idempotency

Retries are normal, and a retry that succeeds after a timeout would otherwise write the
event twice. Send an `Idempotency-Key` header with a value unique to the logical event:

```bash
curl -X POST https://api.tessera.example/v1/events \
  -H "Authorization: Bearer $TESSERA_TOKEN" \
  -H "Idempotency-Key: billing-svc-2026-09-27-a11b93-0014" \
  -H "Content-Type: application/json" \
  -d '{ ... }'
```

Tessera remembers idempotency keys for 24 hours. A repeated key within that window
returns the original event and its original `sequence`, and writes nothing new. After 24
hours the key is forgotten and the same request would create a second event, so build
your retry budget inside that window.

## Retention

Retention is set per workspace, between 90 days and 2,555 days (seven years). It applies
from `recorded_at`, not `occurred_at` — a backdated event is kept for the full period
from when Tessera received it.

| Setting | Typical use |
|---|---|
| 90 days | Sandbox, non-regulated internal tooling |
| 365 days | General operational audit |
| 2,555 days | Financial services, regulated record-keeping |

Raising retention applies to events already stored. **Lowering it is irreversible** and
schedules deletion of everything already past the new horizon; the console requires a
typed confirmation and records the change as an event in its own right.

### Legal hold

A legal hold suspends deletion for matching events regardless of the retention setting.
Holds are defined by a query — a principal, a resource, a date range — and events
matching the query are exempt from expiry until the hold is lifted.

Events under hold are flagged in query responses with `"held": true`, so a reader can see
that the log's contents are being preserved for a reason beyond routine retention.

## Expiry is deletion, not archival

When an event passes its retention horizon Tessera deletes it. There is no cold tier and
no recovery. If you need records beyond your retention period, create a scheduled
[export](../guides/export-for-compliance.md) and store the archive yourself.

Teams are caught out by this more often than by anything else in the platform. Decide
your retention and your export schedule at integration time, not at year seven.

## Related

- [The access model](access-model.md)
- [Export for compliance](../guides/export-for-compliance.md)
- [Error reference](../reference/errors.md)
