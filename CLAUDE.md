# Obur — Development Guide

This document defines the shared development standards across all Obur repositories.
All contributors must read this before starting work.

---

## Repositories

| Repository | Technology | Purpose |
|------------|------------|---------|
| [obur-backend](https://github.com/oburapp/obur-backend) | FastAPI + PostgreSQL | REST API |
| [obur-web](https://github.com/oburapp/obur-web) | Next.js | Web client |
| [obur-mobile](https://github.com/oburapp/obur-mobile) | Flutter | iOS & Android |
| [obur-docs](https://github.com/oburapp/obur-docs) | Markdown | Documentation |

---

## Git Strategy

### Branching

Always create a new branch for each feature, fix, or chore. Never commit directly to `main`.
Branch creation is always done by human contributors. Claude may suggest a branch name when appropriate, but never creates branches itself.

Branch naming convention:

```
feature/checkin-flow
fix/venue-duplicate-detection
chore/update-dependencies
docs/adr-venue-identity
refactor/checkin-product-separation
```

### Commit Messages

Follow the Conventional Commits format. Every commit must do one thing only.

```
feat: add checkin flow step 3
fix: resolve venue duplicate on 50m radius
chore: update fastapi to 0.140
docs: add system context diagram
refactor: extract checkin service from router
test: add venue search unit tests
```

Rules:

- Use the imperative mood: "add" not "added", "fix" not "fixed"
- No period at the end
- Max 72 characters in the subject line
- If more context is needed, add a body after a blank line

### Pull Requests

- Every change goes through a PR, no direct pushes to `main`
- PR title follows the same Conventional Commits format
- PR description must explain what changed and why
- At least one review required before merge
- Squash merge preferred to keep history clean

### Claude's Role in Git

Claude assists with development but must never appear in the repository.
Commits, PRs, and branch creation are always done by human contributors.
Claude may suggest commit messages and branch names when appropriate,
but the human always types and executes the command.

---

## Environment Variables

- Never hardcode credentials, URLs, or secrets in source code
- Every repository has a `.env.example` file listing all required variables without values
- `.env` is always in `.gitignore` and never committed
- When adding a new environment variable, update `.env.example` immediately
- Never log or print environment variable values

### Two kinds of configuration value

Every config value falls into one of two categories, and they're held to
different standards:

**Reaches production** — anything a real user's device (web browser, iOS,
Android) will actually hit: API URLs, DB/cache connection strings, ports,
credentials, third-party endpoints. **No hardcoded fallback, ever.** If
it's missing or misconfigured, the app must fail immediately and loudly at
startup — never silently resolve to a plausible-looking value that hides
the real problem a few steps downstream. This must work correctly across
every device the product ships to, not just the machine it was written on.

**Local-only** — dev tooling: docker-compose, test fixtures, local
defaults. A default is fine here for convenience, but two rules still
apply:
1. If a value already lives somewhere (e.g. `.env`), every other place
   that needs it must *read from there*, not hold its own independent
   copy — two copies of the same fact will eventually drift out of sync.
2. The default must stay portable across different developers' machines.
   A fixed local port, for example, can collide with something already
   running on someone else's setup — make it overridable, don't assume a
   value will always be free just because it is on your machine.

When in doubt: if you can imagine this code running against a real user in
production, treat it as the first category, not the second.

---

## Task Runner

Every repository that has commands worth wrapping (lint, format, test,
migrate, run the dev server, ...) has its own `justfile` at its root,
using [just](https://github.com/casey/just). Run `just --list` (or bare
`just`) in a repo to see what's available there — the exact recipes vary
per repo (a Next.js app's recipes look different from a FastAPI one's),
but the tool and the discovery command (`just --list`) are the same
everywhere.

- Recipes wrap existing commands (`uv run pytest`, `docker compose up`,
  ...) — they don't replace the standards that govern *what* those
  commands do (testing strategy, code style, etc.), just give a short,
  consistent name for running them.
- A `check` recipe (lint + typecheck + test) is the standard name for "run
  everything CI will run" — add it to every repo's justfile as those
  checks come online.
- Don't add a justfile to a repository until it has real commands to
  wrap — an empty or placeholder justfile isn't worth the file.
- `just` is a project dependency like any other, even though it's a
  system-level install rather than something `uv`/`npm` can pin
  (`winget install --id Casey.Just -e` on Windows). Every justfile pins
  the floor with `set minimum-version := "X.Y.Z"` at the top, the same way
  `pyproject.toml` pins `requires-python` — a version older than that
  should fail loudly instead of silently behaving differently.
- Justfiles are living documents: when a new recurring command shows up
  (a new check, a new one-off script someone keeps re-typing), add it as
  a recipe in the same PR — don't let the justfile drift stale behind
  what people actually run.

---

## Documentation Standards

### ADR (Architecture Decision Records)

Every significant architectural decision must be documented as an ADR in `obur-docs/adr/`.
An ADR is written at the moment the decision is made, not before.
File naming: `0001-single-direction-follow.md`

Template:

```markdown
# ADR-XXXX: Title

**Date:** YYYY-MM-DD  
**Status:** Accepted | Deprecated | Superseded by ADR-XXXX

## Context
Why did we need to make this decision?

## Decision
What did we decide to do?

## Rationale
Why did we choose this option over the alternatives?

## Consequences
What are the positive and negative outcomes of this decision?
```

### README

Every repository must have a `README.md` that covers:

- What this service does
- How to set up the local environment
- How to run the project
- How to run tests
- Environment variables reference (link to `.env.example`)

### Changelog

Every repository maintains its own `CHANGELOG.md`.
Format follows [Keep a Changelog](https://keepachangelog.com/en/1.0.0/).

Rules:

- Never paste raw commit messages or a `git log` dump — every entry is
  written for a human reading the file, not derived mechanically from history
- One line per notable change: what changed, and why it matters if that's
  not obvious from the description alone
- Group entries only under the categories that have something in them —
  `Added`, `Changed`, `Deprecated`, `Removed`, `Fixed`, `Security`. Never
  add an empty category.
- New entries go under `## [Unreleased]` as they land, in the same PR as
  the change itself — not batched later from memory
- `obur-backend`, `obur-web`, and `obur-mobile` follow [Semantic
  Versioning](https://semver.org/); on release, rename `[Unreleased]` to
  `[MAJOR.MINOR.PATCH] - YYYY-MM-DD` and start a fresh empty `[Unreleased]`
  above it
- `obur-docs` has no software releases to version — entries accumulate
  under `[Unreleased]`; a dated section is only cut for a major documentation
  milestone (e.g. PDD v2.0)
- Versions are listed newest-first (reverse chronological)

### Versioning and Releases

One version number, three places it has to agree:

| Where | What it's for |
|-------|----------------|
| The package manifest (`pyproject.toml` for `obur-backend`, `package.json` for `obur-web`, `pubspec.yaml` for `obur-mobile`) | The code's own declared version |
| `CHANGELOG.md`'s `## [X.Y.Z] - YYYY-MM-DD` heading | Human-readable summary of what's in that version |
| A git tag, `vX.Y.Z` | A permanent pointer to the exact commit that version was cut from |

**What MAJOR.MINOR.PATCH means:**

- **PATCH** (`0.1.0` → `0.1.1`) — bug fix, no behavior contract changed
- **MINOR** (`0.1.1` → `0.2.0`) — new capability, backward compatible
- **MAJOR** (`0.x` → `1.0.0`, or later `1.x` → `2.0.0`) — breaks existing
  clients (changed response shape, removed endpoint, new required field)

While the version stays `0.x.y`, the public API isn't considered stable
yet — that's expected pre-launch. Bump to `1.0.0` once real users (via
`obur-web`/`obur-mobile` in production) actually depend on the contract
not breaking.

**Bump at a meaningful milestone or PR, never per-commit.** A version
number describes a working, usable state of the software — most
individual commits in a PR aren't independently that (e.g. "add
docker-compose" isn't a release on its own). Tag the PR/milestone as a
whole, not each commit inside it.

**Release flow**, once the feature PR lands on `main`. Branch protection
applies to this too — there is no direct-to-`main` shortcut, not even for
a one-line CHANGELOG edit:

1. New branch off the updated `main`, e.g. `chore/release-X.Y.Z`
2. One commit: rename `CHANGELOG.md`'s `## [Unreleased]` to
   `## [X.Y.Z] - YYYY-MM-DD` (today's date), add a fresh empty
   `## [Unreleased]` above it, and set the manifest's version to match —
   `chore: release X.Y.Z`
3. Push, open a PR, get it reviewed and squash-merged into `main` like any
   other change
4. Pull the updated `main` locally and tag the new commit: `git tag vX.Y.Z`
5. Push the tag: `git push origin vX.Y.Z`

Tagging, like committing, is run by a human — Claude may prepare the
CHANGELOG/manifest edit and suggest the commands, never runs `git tag` or
`git push` itself.

---

## Code Quality

### General Rules (all repositories)

- No magic numbers or strings — define as named constants
- No `TODO` comments — either fix it now or open a GitHub issue
- No commented-out code in commits
- Every function does one thing (Single Responsibility Principle)
- Keep functions short — if it needs scrolling, consider splitting
- File and function names in English, code comments in English

### Testing

- Critical business logic must have unit tests
- Tests must be independent — no shared state between tests
- Never call real external APIs in tests — always mock
- Test names must describe what they test:

```
# correct
test_checkin_rating_returns_aggregate_score_above_threshold()

# wrong
test_checkin()
```

### Security

- Never log sensitive data: passwords, tokens, API keys, user PII
- Always validate user input
- Never expose internal error details to the client
- Rate limiting must be active on all public endpoints
- Never commit `.env` files or any file containing secrets

---

## Deployment

Each repository contains its own deployment documentation in `docs/deployment.md`.
For system-wide incident response procedures, see `obur-docs/runbooks/incident-response.md`.

---

## Communication

- GitHub Issues for bugs, features, and technical debt
- PR comments for code-level discussions
- Keep discussions close to the code — avoid resolving technical decisions outside GitHub

---

*This document is maintained in `obur-docs`. Any changes must go through a PR.*