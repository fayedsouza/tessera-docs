# Contributing

How to change this documentation. The process is deliberately close to a software
change, because that is the point of docs-as-code — documentation moves through the same
review, the same CI and the same history as the product it describes.

## Set up locally

```bash
git clone https://github.com/fayedsouza/tessera-docs.git
cd tessera-docs

python -m venv .venv
source .venv/bin/activate          # Windows: .venv\Scripts\activate

pip install -r requirements.txt
mkdocs serve
```

The site is then at `http://127.0.0.1:8000` and rebuilds as you save.

## Make a change

1. Branch from `main`. Name it for the change: `fix-batch-207-example`,
   `add-legal-hold-concept`.
2. Edit the Markdown under `docs/`.
3. Run `mkdocs build --strict` before pushing. This is what CI runs, and `--strict`
   fails the build on a broken internal link or a missing snippet — catching it locally
   is faster than catching it in review.
4. Open a pull request. CI builds every pull request; a red build blocks merge.
5. On merge to `main`, the site deploys to GitHub Pages automatically.

## Where content goes

The structure separates three kinds of content, and the separation is load-bearing —
see the [architecture note](README.md#information-architecture).

| Directory | Holds | Test |
|---|---|---|
| `docs/get-started/` | The path a new reader takes, in order | Would a first-time reader follow these in sequence? |
| `docs/concepts/` | Background read once | Does it explain *what* or *why*, with no steps? |
| `docs/guides/` | Task instructions | Does it start with an imperative verb and produce a result? |
| `docs/reference/` | Look-up material | Would anyone read it start to finish? If yes, it is not reference |

A page that fails its directory's test belongs elsewhere. Mixing a concept into a task
guide is the most common drift, and it makes both harder to maintain — the concept gets
duplicated into the next guide that needs it.

## Reusing content

Shared text lives in `includes/`, outside `docs/` so MkDocs does not publish the
fragments as pages of their own, and is included with the snippets syntax:

```markdown
--8<-- "conventions.md"
```

Reuse anything that must stay identical in more than one place. Do not reuse text that
merely happens to be similar today — a snippet that needs a conditional is a snippet
that should have been two pages.

## Style

See [STYLE.md](STYLE.md). It is short; read it once before your first contribution.

The rules most often missed in review:

- British English, `-ise` spellings.
- Second person, present tense, active voice.
- Every page ends with a **Related** section.
- No "simply", "just", "easy", "obviously".

## Review

Every pull request needs one approval. Reviewers check, in this order:

1. **Is it true?** Technical accuracy first. A well-written wrong instruction is worse
   than a clumsy right one.
2. **Is it in the right place?** Against the directory tests above.
3. **Does it follow the style guide?**
4. **Does it read well?**

Reviewers comment on structure before prose. Rewriting sentences on a page that is in
the wrong section wastes both people's time.
