# Documentation project instructions

## About this project

- Public documentation site for the Scala Computing Network Simulation API, built on [Mintlify](https://mintlify.com).
- Pages are Markdown (`.md`) with YAML frontmatter (`title`, `description`).
- Navigation and site configuration live in `docs.json`.
- The site deploys automatically when changes land on `main`.

## Source of truth

Pages listed in `.sync-owned-paths` are **owned by the `scala-computing/scala` repository**
and are rewritten by its customer-docs sync (`docs/api/src` for the Documentation tab,
`docs/models` for the Models tab). Files under `api-reference/` are generated there from the
OpenAPI spec by `docs/scripts/generate-api-reference.py` and carry a generated-content marker.
The sync also owns `navigation.tabs` in `docs.json`. Everything else in this repository
(branding, theme, `README.md`, this file, any page not in `.sync-owned-paths`) is owned here.

Do not hand-edit an owned page: the next sync overwrites it. Fix the source page or the
OpenAPI spec in the source repository; the sync opens a pull request here on
`sync/from-monorepo` for review.

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
