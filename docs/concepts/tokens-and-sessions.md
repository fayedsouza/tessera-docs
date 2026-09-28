# Tokens and sessions

Two identifiers travel with most events and are easy to confuse. A **token** is the
credential a caller presents to Tessera. A **session** is an identifier from *your*
system that groups related activity. Tessera treats them very differently.

## Tokens are Tessera's; sessions are yours

The token authenticates the service writing to Tessera. It says nothing about who the
end user was — a single `billing-service` token may write events for thousands of
different principals.

The session ID is opaque to Tessera. It is stored in `context.session_id`, indexed for
querying, and otherwise never interpreted.

This separation matters when you are reading a log back. A run of events sharing a token
tells you which service emitted them. A run sharing a session tells you which sitting of
which user produced them. Investigations almost always want the second.

## What Tessera records about the writer

Every event carries the identity of the credential that wrote it, whether or not you
supply anything:

```json
{
  "id": "evt_01JB8Q2X7K",
  "written_by": {
    "token_id": "tok_2f81ba",
    "token_name": "billing-service",
    "source_ip": "198.51.100.7"
  }
}
```

`written_by` is set by Tessera and cannot be supplied or overridden by the caller. It is
the answer to "which of our systems claimed this happened?", and it is the first thing to
check when a log contains something implausible.

## Session IDs and personal data

A session ID should be a random opaque value. Do not derive it from anything meaningful.

Session IDs are retained as long as the events carrying them, which may be seven years.
Anything you encode into one — a user's email, an account number, a device fingerprint —
inherits that retention and lands inside every export you hand to a third party. Teams
that encoded an identifier into the session for convenience have had to explain it during
a data-protection review.

The same applies to `context` generally. It is a free-form object and Tessera will store
what you send, including things you did not intend to keep for seven years.

!!! warning "Tessera does not filter `context`"
    There is no server-side redaction. Whatever you put in `context` is stored verbatim,
    returned in queries, and included in exports. Decide what belongs there before you
    start emitting, because you cannot remove it afterwards — see
    [Events and retention](events-and-retention.md).

## Correlating across services

Where a single user action crosses several services, propagate one identifier so the
events can be stitched back together. Either reuse the session ID, or add your own
correlation field:

```json
{
  "context": {
    "session_id": "sess_a11b93",
    "trace_id": "01JB8Q2X7KQ4M9V"
  }
}
```

`trace_id` is not a field Tessera defines — it is just a key inside `context`. Any key
in `context` can be filtered on at query time, so your own conventions work without
Tessera needing to know about them.

Agree the key name across teams before you start. Half your services writing `trace_id`
and half writing `traceId` produces a log that cannot be correlated, and no amount of
querying will fix it after the fact.

## Related

- [Authenticate with the API](../get-started/authentication.md)
- [Query the audit log](../guides/query-the-audit-log.md)
