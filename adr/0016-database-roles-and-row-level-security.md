# ADR-0016: Database Roles and Row Level Security

**Date:** 2026-08-24
**Status:** Accepted (not yet implemented), planned for `obur-backend` Phase 8

## Context

PDD §17 draws a line this project has otherwise not needed to draw: `can_view` (`app.core.authz`) and `ensure_visible_and_owned` protect the *application's own access path*, every API request passes through them because there is no other way to ask the API for data. They do nothing for someone who reaches the data store directly, and they do nothing for a query that simply forgets to call them. That second failure mode is not hypothetical here: this project has already shipped an existence-leak bug (Phase 4, a stranger `PATCH`ing a private check-in got a 403 instead of a 404, confirming the id belonged to something real, found via adversarial testing) and a stale-visibility bug, both fixed at the application layer after the fact. PDD §17 names PostgreSQL Row Level Security as the concrete mechanism for the second, database-level layer: a policy that still blocks a row even when the query in front of it forgot to filter.

RLS was deliberately deferred out of Phase 7 (Cross-Cutting Guardrails), even though every other cross-cutting mechanism landed there. The reason is a real PostgreSQL constraint, not scheduling convenience: RLS policies do not apply to a table's owner, and the application connects to the database as the same role that owns every table (it ran the migrations that created them). Enabling RLS today would enable a set of policies that silently enforce nothing, which is worse than not having them, because it would be believed. The prerequisite, a database role that is not the table owner, is deployment configuration, and it does not exist until a real deployment target does.

## Decision

### 1. Two database roles

An **owner role**, used only by migrations and the seeder (human/CI, never the running API), and a separate **application role**, which is the API's only connection identity from Phase 8 onward. The application role owns nothing and has no `BYPASSRLS` attribute, so it is never exempt from a policy.

`FORCE ROW LEVEL SECURITY` on the single existing role was considered and rejected: it would filter the seeder's own writes too, and an owner can turn `row_security` off at will, meaning a compromised application connecting as owner could disable its own protection. A real second role is the only version of this that cannot be quietly bypassed from inside the running process.

### 2. RLS enabled on every table, policies mirroring `can_view`

Each policy re-expresses in SQL the same rule `app.core.authz.can_view` already enforces in Python: the owner and an admin may always see a row; otherwise `public` is visible to anyone, `private` to no one else, and `close_friends` only to an entry in `CLOSE_FRIEND`. Enumerated bypass roles/paths: migrations, the seeder, admin moderation tooling, and Phase 14's platform-wide badge `rarity_pct`, the cases the roadmap already named as needing to see across every user's data.

### 3. Per-transaction identity via `SET LOCAL`, not per-connection or per-request

A policy has to know who is asking. The current user's id is set with `SET LOCAL app.current_user_id` at the start of every transaction, not once per pooled connection (a later, unrelated request could reuse it) and not once per request (`app/services/checkin.py` and others already call `session.commit()` mid-request, which ends the transaction the setting was scoped to).

Mechanism: SQLAlchemy's `Session.after_begin` event, which fires again automatically every time a session starts a new transaction, including the autobegin that follows a service's own mid-request commit, exactly the "per-transaction, not per-request" behaviour this needs. One specific implementation detail matters and is easy to get wrong: the handler must call `connection.execute(...)`, not `session.execute(...)`; the latter raises `InvalidRequestError` on SQLAlchemy 2.0.17+, because the session is mid-provisioning when the event fires. `SET LOCAL` does not accept bind parameters, a limitation this codebase already hit and documented in `app/services/venue.py` for `pg_trgm.word_similarity_threshold`; the user id has to be inlined as a validated `UUID`, never through raw request input.

If identity is never set (a bug skips the event, or a connection is used outside a session), every policy must default to returning zero rows, not everything. Fail closed.

### 4. Policy DDL lives in migrations, not `app/`

`CREATE POLICY` and `ALTER TABLE ... ENABLE ROW LEVEL SECURITY` are written via `op.execute()` in Alembic migrations; Alembic's autogenerate has no native representation for policies. This needs an explicit carve-out in `CLAUDE.md`'s "never write raw SQL" rule, parallel to the PostGIS carve-out that already exists there for the same structural reason: some things are only expressible as SQL, and pretending otherwise doesn't remove the need, it just hides it.

### 5. Testing: extend the existing integration approach, not a second toolchain

`pgTAP` (a SQL-level testing framework purpose-built for RLS, common in the Supabase ecosystem) was considered and set aside, not rejected outright, but not adopted now, since this project already runs integration tests against a real database rather than mocks (`docs/testing-strategy.md`, ADR-0001), and a two-person team maintaining two test languages for one feature is a cost with no corresponding benefit yet. The existing `pytest` plus real-Postgres approach is extended instead, held to five criteria that distinguish an RLS test suite that actually proves something from one that is merely green:

- Tests run as the real **application role**, never the owner/superuser role connecting today, otherwise they would pass even if RLS were completely missing, since an owner bypasses every policy by default.
- Both directions are asserted: the "can see" case, and explicitly the "cannot see" case. The negative case is the one it's easy to only implicitly cover, and the one that matters; a policy that's too permissive still passes a suite that only checks what should be visible.
- At least some tests issue a raw query directly as the application role, bypassing the API and `can_view` entirely. A suite that only ever calls the API risks the API's own check rejecting the request first, meaning RLS is never actually exercised and the suite is green without proving anything about the database layer.
- A parity test simulating this project's own past failure: a query that skips calling `can_view` (the exact shape of the existence-leak and stale-visibility bugs already found), asserting RLS blocks it independently anyway. This is the concrete, testable answer to the risk named in Consequences below, that `can_view` and its SQL re-expression can drift.
- A fail-closed test: with identity never set, a protected table returns zero rows, not every row.

## Rationale

Alternatives considered:

1. **`FORCE ROW LEVEL SECURITY` on the single existing role**, instead of a real second role. Rejected in Decision §1: it filters the seeder, and an owner can disable `row_security` from inside a compromised process.
2. **Build RLS in Phase 7, alongside the other cross-cutting mechanisms.** Rejected: its prerequisite (a non-owner role) is deployment configuration that does not exist before a real target is chosen, and building policies against a role split that would have to be rebuilt against the real instance is the same work twice, with the second attempt the one that can surprise silently.
3. **pgTAP for RLS testing.** Set aside in Decision §5, not the wrong tool in general, just a second toolchain this team doesn't need yet given the existing real-database integration approach already covers the same ground.
4. **Testing only through the API.** Rejected: risks a suite that is green because the API layer caught everything first, never actually exercising the policies it exists to verify.

## Consequences

**Positive:**

- Closes the second layer PDD §17 asked for. A query that forgets to call `can_view`, this project's actual, already-demonstrated failure mode, can no longer return a row it shouldn't.
- Every phase from Phase 9 onward is written with RLS already active, rather than retrofitted across everything already built, which is the more expensive order and the one this roadmap was explicitly rewritten to avoid repeating.

**Negative / trade-offs:**

- `can_view` and its SQL re-expression are now two implementations of one rule that can drift from each other, a new, subtler version of the exact failure RLS exists to catch. The parity test in Decision §5 is a mitigation, not a guarantee; it catches drift only in the scenarios it's written to cover.
- RLS is not free: with the policy's own filter column properly indexed, published measurements put the overhead near zero (sub-millisecond); without an index, the same policy forces a sequential scan on every request, which is fine at today's volume and a real bottleneck once it isn't. Every column a policy reads (`owner_id`, `visibility`, and anything `close_friend_of_owner_exists` touches) needs an index, verified with `EXPLAIN ANALYZE` once real policies exist, not assumed.
- A query joining multiple RLS-protected tables picks up one additional predicate per table. Whether the planner pushes these down efficiently needs verifying per query, not assumed from the single-table case.
- A new required carve-out in `CLAUDE.md` for raw SQL in migrations, specific to policy DDL.

## Sources

- [Postgres Row-Level Security in Practice, QueryPlane](https://queryplane.com/blog/postgres-row-level-security-in-practice/), indexed vs. unindexed policy performance
- [Optimizing Postgres RLS, Scott Pierce](https://scottpierce.dev/posts/optimizing-postgres-rls/)
- [Postgres Row-Level Security, Daniel Imfeld](https://imfeld.dev/notes/postgresql_row_level_security), `SET LOCAL` and connection pooling
- [Using Postgres Row-Level Security in Ruby on Rails, pganalyze](https://pganalyze.com/blog/postgres-row-level-security-ruby-rails), transaction-scoped identity pattern
- [SQLAlchemy `after_begin` discussion #10469](https://github.com/sqlalchemy/sqlalchemy/discussions/10469), the `connection.execute()` vs `session.execute()` fix
- [Row-Level Security with SQLAlchemy and Alembic, Adriano Vieira](https://www.adrianovieira.eng.br/en/posts/architecture/row-level-security-sqlachemy-alembic-guide/), `op.execute()` pattern for policy DDL
- [pgTAP RLS testing, Supabase Docs](https://supabase.com/docs/guides/local-development/testing/pgtap-extended)
