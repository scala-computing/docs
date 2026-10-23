# Documentation project instructions

## About this project

- Public documentation site for the Scala Computing Network Simulation API, built on [Mintlify](https://mintlify.com).
- Pages are Markdown (`.md`) with YAML frontmatter (`title`, `description`).
- Navigation and site configuration live in `docs.json`.
- The site deploys automatically when changes land on `main`.

## Source of truth

The site is versioned: `unreleased/` holds the latest merge and each `v0.1.<build>/` folder a
platform release. Every path listed in a version folder's ledger, `<folder>/.sync-owned-paths`,
is **owned by the `scala-computing/scala` repository** and is rewritten by its customer-docs
publication (`docs/api/src` for the Documentation tab, `docs/models` for the Models group inside
it, `docs/release-notes` for Release Notes). Files under `<folder>/api-reference/` are generated
there from the OpenAPI spec by `docs/scripts/generate-api-reference.py` and carry a
generated-content marker. The publication also owns, in `docs.json`, each version's
`navigation.versions[]` entry, the order of those entries, their `default` and `tag` keys, and
the `redirects` entries for pre-versioning page addresses. Everything else in this repository
(branding, theme, `index.md`, `README.md`, this file, any other redirect, any page in no
ledger) is owned here.

Do not hand-edit an owned page: the next sync overwrites it. Fix the source page or the
OpenAPI spec in the source repository; the sync opens a pull request here on
`sync/from-monorepo` for Unreleased, and a release publication on `sync/release-v<version>`.

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
