# Tessera Access Events API

Tessera records who accessed what, when, and whether the access succeeded — and keeps
those records in a form an auditor will accept.

If you are integrating Tessera for the first time, start with the
[Quickstart](get-started/quickstart.md). It takes about ten minutes and ends with a
recorded event you can query back.

## What Tessera is for

Most systems already log access somewhere. The problem is rarely capture — it is that the
records are scattered across services, mutable, inconsistently structured, and impossible
to hand to a regulator without weeks of reconciliation.

Tessera is a single append-only store for access events with:

- **Immutability.** An event cannot be edited or deleted once written. Corrections are
  recorded as new events that reference the original.
- **A fixed schema.** Every event names a principal, a resource, an action and an outcome,
  whatever service produced it.
- **Retention you configure once.** Between 90 days and seven years, enforced by the
  platform rather than by each team remembering.
- **Signed exports.** An export archive carries a checksum and a signature, so its
  integrity can be demonstrated rather than asserted.

## What Tessera is not

It is not a SIEM, and it does not alert. It does not make authorization decisions — it
records the decisions your systems already made. If you need to *enforce* access, Tessera
sits downstream of that; it is the record, not the gate.

## Where to go next

<div class="grid cards" markdown>

-   **New to Tessera**

    ---

    Make your first authenticated call and record an event.

    [Quickstart](get-started/quickstartt.md)

-   **Understanding the model**

    ---

    Principals, resources, grants and events — and how they relate.

    [The access model](concepts/access-model.md)

-   **Doing a specific thing**

    ---

    Query the audit log, export for an auditor, handle rate limits.

    [Guides](guides/index.md)

-   **Looking something up**

    ---

    Error codes, glossary, and the API reference.

    [Reference](reference/index.md)

</div>

## Conventions in this documentation

--8<-- "conventions.md"
