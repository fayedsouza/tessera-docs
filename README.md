# Tessera Access Events API — documentation

**Live site: <https://fayedsouza.github.io/tessera-docs/>**

A documentation set built docs-as-code: Markdown in Git, structured with a deliberate
information architecture, built with MkDocs Material, and deployed to GitHub Pages by
GitHub Actions on every merge to `main`.

> **This is a portfolio project.** Tessera is a fictional product. The API does not
> exist and the endpoints do not resolve. The documentation is written as though it
> were real, because a doc set hedged on every page would not demonstrate anything.
> Everything here — the structure, the content, the pipeline, the style guide — is my
> own work.

I am a technical writer with sixteen years in enterprise software, most of it in DITA/XML
in a component content management system. This repository exists because the thing I had
not done was work in a Git-based pipeline, and I would rather close that gap with an
artefact than a claim.

---

## The decisions, and why

### Information architecture

The structure is a Diátaxis-style separation, which is the same discipline as DITA's
topic types in a different notation:

| Directory | DITA equivalent | Answers |
|---|---|---|
| `docs/get-started/` | — (a curated path) | "I have never used this. Where do I start?" |
| `docs/concepts/` | Concept | "What *is* this, and why does it work this way?" |
| `docs/guides/` | Task | "How do I do the specific thing I came here for?" |
| `docs/reference/` | Reference | "What is the exact value of this field?" |

The separation is the point, not the folder names. A reader arrives in one of four
states, and content that mixes two of them serves neither: a task guide that stops to
explain the retention model interrupts someone who already knows it, and the explanation
then gets copied into the next guide that needs it. Each page here links to the concept
it depends on rather than restating it.

**What this costs.** More navigation, and more discipline in review. A contributor's
instinct is to add the explanation inline where the reader hits it. `CONTRIBUTING.md`
carries an explicit test per directory because that drift is constant and only catchable
at review.

**Where I bent it.** `get-started/` is a curated sequence, not a fifth content type. It
duplicates a little of what the concepts and guides cover, deliberately — a first-time
reader needs one ordered path more than they need to be told to read four sections.

### Reuse

Shared text lives in `includes/` and is transcluded with `pymdownx.snippets`. This
is the nearest Markdown equivalent of a DITA conref, and it is meaningfully weaker: no
keyref indirection, no conditional profiling, no filtering by audience or product
variant.

So the rule I applied is narrow — reuse only text that **must** stay byte-identical
everywhere, like the conventions block. Text that merely happens to be similar today
stays duplicated. In a CCMS I would reuse far more aggressively; in Markdown, a snippet
that grows a conditional has told you it should have been two pages.

### Style, enforced rather than described

`STYLE.md` is deliberately short. A style guide nobody finishes changes nothing, so it
carries the decisions that actually come up in review and defers everything else to
MSTP. The terminology table exists because inconsistent vocabulary is the defect that
survives longest — nobody files a bug for it, and it makes the whole set feel unreliable.

### The pipeline

`.github/workflows/deploy.yml` builds on every push and pull request, and deploys only
from `main`.

The build runs `mkdocs build --strict`, which turns warnings into errors. A broken
internal link or a missing snippet fails CI rather than shipping quietly. Combined with
MkDocs' `validation:` settings in `mkdocs.yml`, this covers the failure mode that
actually bites a docs site — not a crash, but a page that silently goes stale and a link
that quietly points nowhere.

Pull requests build without deploying. The build is the review gate; the merge is the
publish.

---

## What I would do differently at scale

This is a fifteen-page set maintained by one person. Most of these decisions would not
survive fifty pages and eight contributors.

- **Versioning.** There is none. A product with supported releases needs
  [mike](https://github.com/jimporter/mike) for versioned deployments, and a policy on
  how long old versions stay published. I left it out rather than build a version
  switcher with one version in it.
- **Reuse would hit its ceiling.** Snippets do not do conditional profiling. Documenting
  Cloud and On-Premise variants of one product from a single source — which is what I did
  at Oracle in DITA — is not comfortably achievable this way. At that point the answer is
  either a real CCMS or a static-site generator with proper content models, not more
  snippets.
- **Link checking is internal only.** `--strict` catches internal links. External links
  rot silently and need a scheduled `lychee` run, not a build-time check that would make
  CI fail because someone else's site is down.
- **Reference should be generated, not hand-maintained.** `docs/reference/api.md` is
  currently written by hand, which guarantees it will drift from the implementation. It
  should be generated from an OpenAPI specification at build time, with the conceptual
  and task content around it staying hand-written. That is the next piece of work — see
  below.
- **No content ownership metadata.** At scale each page needs an owner and a review date,
  surfaced in a report, or nothing gets re-reviewed. Front matter plus a small script.
- **Editorial review does not scale by good intentions.** At Oracle I founded the
  documentation QA and peer-review programme; here `CONTRIBUTING.md` is doing that job on
  its own, which works for one contributor and would not for eight.

## Next

- Hand-author an OpenAPI 3.1 specification for the endpoints in
  [`docs/reference/api.md`](docs/reference/api.md), validate it, and generate the
  reference section from it at build time.
- Add a retrieval chatbot over this published corpus, and write up how I evaluated its
  groundedness.

## Running it locally

```bash
python -m venv .venv
source .venv/bin/activate          # Windows: .venv\Scripts\activate
pip install -r requirements.txt
mkdocs serve
```

See [CONTRIBUTING.md](CONTRIBUTING.md) for the full workflow and
[STYLE.md](STYLE.md) for the editorial rules.

## Contents

| File | What it is |
|---|---|
| `docs/` | The documentation source |
| `mkdocs.yml` | Site configuration, navigation, Markdown extensions |
| `.github/workflows/deploy.yml` | Build on every push, deploy from `main` |
| `CONTRIBUTING.md` | How to change the docs, and where content goes |
| `STYLE.md` | Editorial standard |

---

**Faye D'Souza** — Senior Technical Writer, Bengaluru
[LinkedIn](https://www.linkedin.com/in/fayedsouza-techwriter/)

Published work at Oracle Financial Services is on the
[Oracle Help Center](https://docs.oracle.com/en/industries/financial-services/ofs-analytical-applications/accounting-standards-banking-cloud-service/).
