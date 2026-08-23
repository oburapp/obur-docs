# ADR-0012: Migrations Are Self-Contained; Reference Data Belongs to a Seeder

**Date:** 2026-08-24
**Status:** Accepted

## Context

Implementing [ADR-0011](0011-drop-product-layer-four-venue-criteria.md)
removed `app/seeds/global_product_types.py`. That immediately broke the entire
Alembic environment — not one migration, all of it, since Alembic imports
every revision file to build its history:

```
61f7b67600da_seed_venue_categories_and_global_.py:16
  from app.seeds.global_product_types import GLOBAL_PRODUCT_TYPES
  → ModuleNotFoundError
```

`alembic heads`, `alembic upgrade`, and the integration-test fixture that
migrates `obur_test` all stopped working at once. That migration imported
five things from `app/`: the seed lists, the deterministic-id helpers, the
locale tables, and `SUPPORTED_LOCALES`.

The interesting question was not how to unblock it but why it happened at
all, and where else the same shape is waiting.

**The root cause is a lifetime mismatch.** `app/` describes what the system
is *now* — mutable by design, changed by every refactor. A migration
describes a transition that *already happened* on a date — frozen by
definition. Importing one into the other couples a historical record to a
moving target, and the break is a matter of when, not whether. Reusing
`VENUE_CATEGORIES` felt like DRY; across a time boundary, DRY is the defect,
not the virtue.

**The existing rule didn't help, because it guarded the wrong surface.**
`obur-backend/CLAUDE.md` already said applied migrations must never be
modified. That protects a file's *contents*. Nobody ever edited this
migration — it broke anyway, through its *dependencies*.

**No test could have caught it earlier, either.** The import worked the day
it was written. A latent coupling isn't observable; only its eventual
breakage is.

**A second instance was already queued.** Phase 6 of `obur-backend`'s roadmap
grows the venue-category catalog from 9 entries toward ~40, and with the
product layer gone `VENUE.category_id` is the platform's only classification
dimension, read by three separate consumers. But the seed migration has
already run. Editing `app/seeds/venue_categories.py` would change nothing in
any database — a silent no-op — and the alternatives were re-running an
applied migration (impossible) or editing one (forbidden). Same failure,
different quarter, and that one is about data rather than imports.

**A measurement worth recording:** nothing in `app/` reads `app/seeds/` at
runtime. It is migration input that happens to live under `app/`, which is
exactly what made importing it look ordinary.

## Decision

### 1. A migration imports nothing from `app/`

Migrations carry their own literal values and their own lightweight table
definitions (`sa.table()` / `sa.column()`). The migration that broke already
did this correctly for schema; it only took the shortcut for data.

`migrations/env.py` is a deliberate exception and is not a migration: it is
Alembic's configuration, and autogenerate needs live `Base.metadata` to diff
against.

### 2. Reference data belongs to an idempotent seeder, not to a migration

Three kinds of data, three mechanisms:

| Data | Mechanism | Why |
|---|---|---|
| Schema | Migration | A transition — inherently historical |
| One-off data fix (backfill a column) | Migration, self-contained | An event that happened once |
| **Reference / catalog data** (venue categories, translations, badge definitions) | **Idempotent seeder** | Describes what the catalog should contain *now*, and is expected to grow |

`app/seeds/` stays where it is and becomes legitimate: it is the seeder's
source of truth, and it is current-state data. `app/seeds/runner.py` upserts
every row keyed on its deterministic slug id, so running it against an
already-seeded database is a no-op in effect.

Rows are never deleted by the seeder. A category dropped from the seed files
stays in the database, because venues may still reference it
(`VENUE.category_id` is `NOT NULL`). Retiring a category is a deliberate
migration, not a side effect of editing a list.

**The seeder is never part of normal application startup.** It runs as an
explicit step — `just seed` locally, the integration-test session fixture,
and a deploy step beside `alembic upgrade head` from Phase 8 onward. Running
it from the app's own lifespan would race across instances and would require
the running app to hold write access it otherwise doesn't need.

The already-applied seed migration (`61f7b67600da`) becomes a documented
no-op. Its revision identifiers are kept so the chain stays intact for
databases that have recorded it as applied; those keep their rows, and the
seeder upserts the same slugs over them.

### 3. The rule is enforced by a test, not by discipline

`tests/unit/test_migration_isolation.py` scans `migrations/versions/*.py` for
imports from `app/` and fails with the offending files named. This is the
part that makes the decision durable: it catches the coupling in the pull
request that introduces it, rather than months later when something unrelated
moves.

The test also asserts that it found migration files at all, so a wrong path
can't quietly make it vacuous.

## Rationale

This is a well-travelled problem, and the external precedent is unusually
direct.

**Django documents this exact failure**, including its exact trigger:

> "If you import models directly rather than using the historical models,
> your migrations may work initially but will fail in the future when you try
> to rerun old migrations (commonly, when you set up a new installation and
> run through all the migrations to set up the database)."

That last parenthetical is what happened here: the integration suite runs
`alembic upgrade head` against a test database every session. Django cared
enough to build a first-class API for it — `RunPython` hands the migration
`apps.get_model()`, the *historical* model, precisely so live model code is
never reachable from a migration.

**EF Core went further and renamed the feature** to stop people using it the
way we did:

> "Populating the database using the HasData method used to be referred to as
> 'data seeding'. This naming sets incorrect expectations, as the feature has
> a number of limitations and is only appropriate for specific types of data.
> That is why we decided to rename it to 'model managed data'. UseSeeding and
> UseAsyncSeeding methods should be used for general purpose data seeding."

Its stated limit is the line this ADR draws: migration-managed data suits
"static data that's not expected to change outside of migrations… for
example ZIP codes." Venue categories are the opposite — Phase 6 exists to
grow them.

EF Core's warning about *where* a seeder runs is what settled the deploy-step
decision rather than an application-startup hook:

> "The seeding code should not be part of the normal app execution as this
> can cause concurrency issues when multiple instances are running and would
> also require the app having permission to modify the database schema."

**Alembic's own cookbook** points the same way more mildly, noting that a
data migration "may also need a separate ORM model to handle intermediate
state of the database," and using `sa.table()`/`sa.column()` constructs in
its data examples rather than application models.

Alternatives considered:

1. **Keep `app/seeds/global_product_types.py` as dead code** so the import
   resolves. Rejected: it preserves the coupling and only defers the break,
   while requiring four unrelated cleanups to be reverted and leaving a
   module in `app/` that nothing reads.
2. **Inline the seed literals into the migration, freezing a copy.** This was
   the first proposal and is *correct for one-off data fixes* — but wrong
   here. Applied to a catalog it produces the same list duplicated across
   every migration that touches it, and still leaves Phase 6 with no way to
   grow the catalog. EF Core's "static data… for example ZIP codes" limit is
   the precise reason.
3. **Squash all migrations into a fresh initial revision.** Technically
   available pre-deployment, but it discards real history to solve a coupling
   problem that would simply reappear, and it would need redoing after Phase
   8 ships anything.

## Consequences

**Positive:**

- The failure class is closed rather than patched: the rule is checked
  mechanically, in the PR that would introduce it.
- Phase 6's catalog growth has a defined path — edit the seed files, run the
  seeder. No new migration, no silent no-op.
- `app/seeds/` gains an honest purpose and a real runtime consumer.
- Schema and reference data are now set up the same way everywhere: `just
  setup-db` chains migrate and seed, and the integration fixture does the
  same two steps, so a fresh database is never half-prepared.

**Negative / trade-offs:**

- Preparing a database is two steps rather than one. Forgetting the seeder
  leaves `venue_categories` empty, and since `VENUE.category_id` is `NOT
  NULL`, venue creation fails. Mitigated by chaining both in `just setup-db`
  and in the test fixture, and this must be carried into the Phase 8 deploy
  step — a deploy that migrates without seeding is a broken deploy.
- Editing an applied migration was required to unblock this, which the repo's
  own rule otherwise forbids. Done deliberately and with approval, and
  limited to removing imports and turning the body into a documented no-op —
  no already-applied effect was rewritten.
- The seeder can drift from what a long-lived database actually contains if
  someone changes rows by hand. That is the standing trade-off of upsert
  seeding and is accepted; the seeder is authoritative, manual edits are not.
