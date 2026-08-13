# ADR-0001: Test Database Strategy

**Date:** 2026-08-13  
**Status:** Accepted

## Context

`obur-backend`'s original testing rule stated two things that turned out to
be in tension: "Never call real external services in tests — always mock
(R2, Redis, Clerk, PostGIS)," and separately, "Integration tests cover
critical flows end-to-end (checkin creation, venue search, auth)."

Venue search and the duplicate-venue check depend on real PostGIS spatial
queries (`ST_DWithin` for the 50-meter radius check, see
[obur-pdd.md §13](../pdd/obur-pdd.md#13-venue-and-product-architecture)).
A fully-mocked test only proves a mock returns what it was told to
return — it never verifies the actual SQL/PostGIS query is syntactically
and semantically correct. Something in the test suite needs to exercise
PostGIS for real, without turning every test into a slow, order-dependent,
or environment-fragile thing.

## Decision

Two test tiers, enforced by directory (`tests/unit/` vs
`tests/integration/`):

- **`tests/unit/`** — zero real I/O. Database, Redis, R2, Clerk, and
  Mapbox are all mocked. Fast, hermetic, runs with no infrastructure.
- **`tests/integration/`** — exercises real flows against a real
  database. Third-party services (R2, Clerk, Mapbox) stay mocked even
  here; only the database is real.

The real database is `obur_test`, a second database inside the *same*
docker-compose Postgres+PostGIS container already used for local
development — not a separate service or tool. It's created by
`docker/postgres-init/20_create_test_database.sql` on first container
startup, with the `postgis` extension installed.

Each integration test runs inside a transaction that's rolled back at the
end, so tests never see each other's writes and the database returns to
its prior state regardless of what a test did.

`obur_test`'s schema is kept at the current Alembic head automatically: a
session-scoped, `autouse` fixture in `tests/integration/conftest.py` runs
`alembic upgrade head` against it at the start of every test session.
Nobody manually re-migrates it.

Connection details for both `tests/conftest.py` (unit) and the integration
fixtures are derived from the real `.env` (via `python-dotenv`), not
independently hardcoded — see the production-vs-local-only rule in
[obur-docs/CLAUDE.md](../CLAUDE.md#environment-variables). A local port
override in `.env` (e.g. to dodge a collision with an already-running
Postgres) is honored automatically by tests too.

## Rationale

Alternatives considered and rejected:

1. **Mock PostGIS everywhere, always** (the original rule). Rejected:
   never actually verifies the query is correct, only that the mock
   returns what it was configured to return. The whole point of the
   50-meter duplicate check is the query itself.
2. **`pytest-postgresql`** (a plugin that provisions its own ephemeral
   Postgres for tests). Rejected: redundant — docker-compose already
   provisions Postgres+PostGIS for local dev — and the plugin doesn't
   bundle PostGIS by default anyway.
3. **Run integration tests against the real dev database (`obur`)**.
   Rejected: tests mutate data, so the dev database would accumulate test
   garbage; results would depend on whatever ad hoc data a developer
   happened to have; parallel test runs would collide; CI has no
   persistent dev database to point at in the first place.
4. **Migrate `obur_test` manually, on demand.** Rejected: violates the
   "no two independent copies of the same fact" rule — a forgotten manual
   migration silently lets `obur_test` drift from the real schema. This
   isn't hypothetical: during Phase 0, a `docker compose down -v` reset
   wiped `obur`'s own migration state, and it went unnoticed until
   checked by hand.

## Consequences

**Positive:**
- Integration tests verify actual PostGIS query correctness, not mock
  behavior.
- No new infrastructure dependency — reuses the docker-compose container
  that already exists for local dev.
- Schema drift between `obur_test` and the real migration history is
  structurally prevented, not just discouraged.
- Local port overrides are honored automatically by both the app and the
  test suite, since both read from the same `.env`.
- The dev database (`obur`) is never touched by automated tests.

**Negative / trade-offs:**
- Integration tests require the docker-compose Postgres container to be
  running — they can't run in the total isolation unit tests can.
- Small added latency per test session from running `alembic upgrade
  head` (currently negligible; expected to stay small).
- More test infrastructure to maintain than a single "mock everything"
  approach: `tests/integration/conftest.py`, the `obur_test` init script.
