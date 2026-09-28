# Style guide

The editorial rules this documentation follows. Short on purpose — a style guide nobody
reads changes nothing.

Base standard: **Microsoft Writing Style Guide**, with the exceptions below. Where this
file is silent, MSTP applies.

## Voice

**Second person, present tense, active voice.**

> Send a `POST` to `/v1/events`.

Not "The user should send…", not "A `POST` will be sent…".

**Address the reader directly. Describe the product in the third person.** The reader is
"you"; Tessera is "Tessera", not "we".

**British English**, `-ise` spellings. `organisation`, `serialised`, `behaviour`.

## Tell readers what will go wrong

State the failure mode where the reader will hit it, not in a troubleshooting appendix.
If a step commonly fails, say so in the step.

> A batch is **not** atomic. Tessera writes the events it can and reports the ones it
> cannot.

This is the single largest difference between documentation that gets used and
documentation that gets abandoned.

## Admonitions

Four types, used sparingly. More than two on a page means the page needs restructuring.

| Type | Use for |
|---|---|
| `!!! note` | Something true that is easy to miss |
| `!!! tip` | A better way, not required |
| `!!! warning` | The reader may lose data or time |
| `!!! danger` | The reader may lose data irrecoverably |

Never use an admonition for content that belongs in the body text.

## Headings

Sentence case. `## Record an event`, not `## Record An Event`.

Task headings begin with an imperative verb. Concept headings are noun phrases.

No heading levels below `###`. If you need `####`, the page is doing too much.

## Code samples

- Every sample is runnable after substitution. No pseudocode presented as code.
- Placeholders are `UPPER_SNAKE_CASE` and named in the surrounding text.
- `curl` for HTTP; Python for anything needing logic.
- Show real response bodies, abridged to the fields under discussion. Never invent a
  field that does not exist.
- Comment the non-obvious line, not every line.

## Tables

For enumerable facts — parameters, error codes, status values. Not for prose.

Every table has a header row. Cells are fragments, not sentences, and carry no trailing
full stop.

## Links

Descriptive text, never "click here" or a bare URL.

Every page ends with a **Related** section linking the concept it depends on and the task
it leads to. A page with nothing to link to is probably in the wrong place.

## Terminology

One term per concept, always. The [glossary](docs/reference/glossary.md) is the source of
truth; add to it before introducing a new term.

| Use | Not |
|---|---|
| event | log entry, record, audit line |
| principal | actor, subject, user (unless specifically `type: user`) |
| token | key, credential, secret |
| workspace | account, tenant, org |

## What not to write

- **No filler openers.** Not "In this guide, we will explore…". Start with the task.
- **No "simply", "just", "easy", "obviously".** If it were, the page would not exist.
- **No future tense for product behaviour.** "Tessera returns", not "Tessera will
  return".
- **No undated forward promises.** "Coming soon" with no date is noise. Either give the
  release or say nothing.
