# Documentation project instructions

## About this project

- Public documentation site for the Scala Computing Network Simulation API, built on [Mintlify](https://mintlify.com).
- Pages are Markdown (`.md`) with YAML frontmatter (`title`, `description`).
- Navigation and site configuration live in `docs.json`.
- The site deploys automatically when changes land on `main`.

## Source of truth

The site is versioned. Each `v0.1.<build>/` folder holds one platform release. An
`unreleased/` folder holds the latest merge while the source repository shows Unreleased (its
`DOCS_SHOW_UNRELEASED` setting); hiding it again has the sync propose the folder's removal
here. Each version folder's ledger, `<folder>/.sync-owned-paths`, and every path it lists are
**owned by the `scala-computing/scala` repository** and written by its customer-docs
publication (`docs/api/src` for the Documentation tab, `docs/models` for the Models group
inside it, `docs/release-notes` for Release Notes). Pages under `<folder>/api-reference/` are
generated there from the OpenAPI spec by `docs/scripts/generate-api-reference.py` and carry a
generated-content marker; the `openapi.json` beside them is the public API description itself.
The publication also owns, in `docs.json`, each version's `navigation.versions[]` entry, the
order of those entries, their `default` and `tag` keys, and the `redirects` entries for
pre-versioning page addresses. Everything else in this repository (branding, theme,
`index.md`, `README.md`, this file, any other redirect, any file in no ledger) is owned here.

Do not hand-edit an owned file, ledgers included. No automatic run restores a release's page:
a hand edit stays until the release engineer next publishes that release, and a ledger that no
longer parses stops every later publication. Route the correction to the source repository
instead. While Unreleased is shown, a merged fix reaches `unreleased/` through the sync's pull
request on `sync/from-monorepo`. A released version changes only when the release engineer
publishes it again, which opens a pull request here on `sync/release-v<version>`; a
republication from a later merge carries documentation only.

## Markdown constraints

Mintlify compiles `.md` through MDX, so plain-HTML habits break the build:

- Use `{/* ... */}` for comments. HTML comments (`<!-- ... -->`) fail to compile and the
  page renders an error component instead of its body.
- Do not use bare `{` or `}` in prose. Inside backticks they are safe, so
  `` `/api/v1/simulations/{id}` `` is fine.
- Attribute syntax such as `[text](url){target="_blank"}` is not supported.
- A page can compile and still be unreachable: a filename matching Mintlify's auto-ignore
  list (`README.md`, `LICENSE.md`, `CHANGELOG.md`, `CONTRIBUTING.md`) is skipped. The API
  changelog is named `api-changelog.md` for this reason.

Verify with `mint dev` before pushing. A page returning HTTP 200 is not proof it rendered —
check that body text is actually present.

## Style preferences

- Active voice, second person ("you").
- One idea per sentence.
- Sentence case for headings.
- Bold for UI elements: Click **Settings**.
- Code formatting for file names, commands, paths, and code references.

## Content boundaries

- This repository is public-facing. Do not document internal engineering material,
  infrastructure details, or unreleased features.
- Do not invent endpoints, fields, or behavior. Derive every API detail from the OpenAPI
  spec or existing source documentation.
