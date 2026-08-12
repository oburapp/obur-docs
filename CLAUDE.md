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