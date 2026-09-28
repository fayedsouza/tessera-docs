# The access model

Tessera describes access with four objects: **principals**, **resources**, **grants** and
**events**. Getting the mapping right between your system and these four is most of the
integration work, and it is worth doing deliberately rather than discovering it in
production.

## The four objects

``` mermaid
graph LR
  P[Principal] -->|acts on| R[Resource]
  P --> E[Event]
  R --> E
  G[Grant] -.->|may explain| E
```

| Object | Answers |
|---|---|
| **Principal** | Who acted |
| **Resource** | What was accessed |
| **Event** | A single recorded access |
| **Grant** | A standing permission that may explain the access |

A **grant** says a principal *may* do something. An **event** says a principal *did*
something. Tessera stores both, but they answer different questions, and conflating them
is the most common modelling mistake.

## Principals

A principal is whoever or whatever performed the access. Every principal has an `id` that
is stable in *your* system, a `type`, and optionally a human-readable `display_name`.

| Type | Use for |
|---|---|
| `user` | A human with an account |
| `service` | A machine identity — a job, a daemon, another service |
| `api_key` | Access attributable only to a credential, not to a person |
| `anonymous` | Unauthenticated access you still want recorded |

### Choosing a principal ID

Use an identifier that will not be reassigned. Email addresses look convenient and are a
poor choice: people change names, and addresses get recycled to new joiners. If your
directory issues an immutable user ID, use that and put the email in `display_name`.

`display_name` is denormalised on purpose. It is captured as it was at the time of the
event, so a record from 2023 still shows the name the person used then. This is what an
auditor expects — the log should say what was true when it happened.

## Resources

A resource is whatever was accessed. As with principals, the `id` must be stable in your
system.

Resource `type` is a free-text label you define. Keep the vocabulary small and agree it
across teams before you start emitting: a log containing `document`, `Document`, `doc`
and `file` for the same concept is difficult to query and embarrassing to hand to an
auditor. Record the agreed list somewhere your teams will find it.

!!! tip "Model the thing being protected"
    Where a resource is nested — a field within a record within a dataset — record the
    level at which access is actually controlled. If permissions are set on the dataset,
    the dataset is the resource, and the record ID belongs in `context`.

## Actions and outcomes

`action` is a verb describing what was attempted: `read`, `write`, `delete`, `export`,
`login`, `approve`. Again, agree the vocabulary up front.

`outcome` is constrained to three values:

| Outcome | Meaning |
|---|---|
| `allowed` | The access was permitted and happened |
| `denied` | The access was refused |
| `error` | The attempt failed for a reason other than permission |

Record denials. An access log containing only successes cannot answer the question
auditors most often ask, which is whether anyone tried and was stopped.

## Grants

A grant records a standing permission — that a principal holds some level of access to a
resource, from a point in time, possibly until another.

Grants are optional. Tessera does not enforce them and will happily record an `allowed`
event for a principal with no matching grant; that divergence is itself a finding worth
surfacing. Teams that record grants do so to answer a second question: not "who accessed
this?" but "who *could have*?"

A grant is a mutable object with an immutable history. Revoking a grant sets its
`revoked_at` rather than removing the row, so a point-in-time reconstruction stays
possible.

## Events

An event is one recorded access. Events are append-only: once written, an event cannot be
edited or deleted by anyone, including a workspace administrator.

This is a deliberate constraint and it has a consequence worth stating plainly — if you
write bad data, you cannot clean it up. You can only append a correcting event that
references the original. Test your event schema against the sandbox before pointing
production traffic at it.

See [Events and retention](events-and-retention.md) for how corrections work and how long
records are kept.

## Worked example

A finance analyst opens a board pack. Your application has already decided to allow it.
The event you record:

```json
{
  "principal": { "id": "user_4471", "type": "user",
                 "display_name": "j.okafor@acme.example" },
  "resource":  { "id": "doc_99120", "type": "document",
                 "display_name": "Q3 Board Pack" },
  "action": "read",
  "outcome": "allowed",
  "occurred_at": "2026-09-27T09:14:02Z",
  "context": { "ip": "203.0.113.42", "session_id": "sess_a11b93" }
}
```

The same analyst then tries to export it, and your application refuses:

```json
{
  "principal": { "id": "user_4471", "type": "user",
                 "display_name": "j.okafor@acme.example" },
  "resource":  { "id": "doc_99120", "type": "document",
                 "display_name": "Q3 Board Pack" },
  "action": "export",
  "outcome": "denied",
  "reason": "policy.export_requires_approval",
  "occurred_at": "2026-09-27T09:16:48Z",
  "context": { "ip": "203.0.113.42", "session_id": "sess_a11b93" }
}
```

Two events, same session, different outcomes. Together they tell a story that either one
alone would not.

## Related

- [Events and retention](events-and-retention.md)
- [Record an access event](../guides/record-an-access-event.md)
- [Glossary](../reference/glossary.md)
