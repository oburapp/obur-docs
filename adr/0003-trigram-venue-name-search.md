# ADR-0003: Trigram-Based Venue Name Search

**Date:** 2026-08-13
**Status:** Accepted

## Context

Phase 2 (Venues & Products) originally implemented venue name search with
PostgreSQL's built-in full-text search, using a `VENUE.search_vector`
column generated as `to_tsvector('turkish', name)`, queried with
`websearch_to_tsquery('turkish', ...)`.

This was built with Turkey/Istanbul-first in mind, but Obur is planning
global expansion, and even within Turkey many venues carry non-Turkish
names (international chains, transliterated names). A search
configuration hardcoded to `'turkish'` has two compounding problems:

1. **Wrong language for non-Turkish text.** PostgreSQL's `'turkish'`
   config applies Turkish-specific stemming and stopwords. Run against a
   non-Turkish venue name or a non-Turkish query, stemming is either
   useless or actively misleading.
2. **No typo tolerance, no partial-word matching by default.**
   `to_tsquery`/`websearch_to_tsquery` match whole tokens (or explicit
   prefix syntax); a search box needs to work well for partial names and
   misspellings, which linguistic FTS doesn't provide.

Reviewing the requirement more precisely: venue names are proper nouns
(`VENUE.name` is a single field, not translated per locale — see the
ER diagram's "Translation tables over embedded strings" design decision,
which deliberately does *not* apply to venue names). Proper nouns aren't
meaningfully stemmed by any language's grammar rules, so the one thing
language-specific FTS is uniquely good at contributes little here, while
its language-specific weakness (wrong-language stemming) is a real cost
for a product that must support "any venue name, any query language,
partial input, typos" as first-class scenarios, not edge cases.

Also worth recording: the original `search_vector` column, as shipped,
had no GIN index backing it — every FTS query was doing a sequential
scan. This didn't surface as a bug report because Phase 2 had no real
traffic yet, but it's a concrete reminder that this path was never
load-bearing in practice.

## Decision

Replace `VENUE.search_vector` (generated `tsvector`, Turkish-only) with a
`pg_trgm` trigram GIN index directly on `VENUE.name`. Venue search uses
PostgreSQL's `word_similarity()` (`<%` operator), which finds the
best-matching substring of `name` against the query — this single
mechanism is language-agnostic, tolerates typos, and works for both short
partial queries and longer ones, without needing to know or guess what
language a given venue's name or a given user's query is in.

**Diacritics are folded before comparison.** Verified directly against a
real database: a plain trigram comparison scores a Turkish name typed
without its diacritics (e.g. `"doner"` vs. `"Döner"`) poorly —
`word_similarity('doner', 'Kadıköy'de En İyi Döner')` returns `0.33`
instead of `1.0` — because trigram matching is character-exact and `ö`
and `o` share no trigrams. Since typing Turkish text without diacritics
is common (non-Turkish keyboards, typing shortcuts), the index is built
on `immutable_unaccent(name)` instead of raw `name`, and queries apply
the same normalization to the search term. This is verified to fully
close the gap: the same case scores `1.0` once both sides are unaccented.

`unaccent()` (a separate PostgreSQL contrib extension from `pg_trgm`) is
marked `STABLE`, not `IMMUTABLE` — confirmed by attempting to index it
directly, which fails with `functions in index expression must be marked
IMMUTABLE`. The standard, documented workaround is a thin SQL wrapper
function, `immutable_unaccent(text)`, that calls `unaccent()` with an
explicit dictionary name and is itself marked `IMMUTABLE STRICT
PARALLEL SAFE`. Verified via `EXPLAIN` that the resulting GIN index
(`immutable_unaccent(name) gin_trgm_ops`) is actually used by the query
planner (`Bitmap Index Scan`), not silently falling back to a sequential
scan.

The similarity threshold is an explicit, named constant in application
code (`app.core.search.MIN_NAME_SIMILARITY`), set per-query via
`SET LOCAL pg_trgm.word_similarity_threshold` rather than left at
PostgreSQL's session-default GUC value — so the actual matching
sensitivity is visible in the codebase, not an implicit server default
someone would have to go look up.

`pg_trgm`, `unaccent`, and `immutable_unaccent()` are all created inside
the Alembic migration itself via `CREATE EXTENSION IF NOT EXISTS` /
`CREATE OR REPLACE FUNCTION` (not only via a docker-init script, unlike
`postgis`) — none of this has side effects beyond what those statements
declare, so making the migration fully self-sufficient is strictly
simpler and works uniformly across local dev, CI, and any future
non-docker-managed database, without relying on a bootstrap script that
only runs once per fresh volume.

## Rationale

Alternatives considered:

1. **Keep `'turkish'` FTS as-is.** Rejected: doesn't meet the stated
   requirement — degrades for non-Turkish names/queries, which are a
   real, not hypothetical, part of Obur's target content and audience.
2. **Switch FTS to PostgreSQL's language-agnostic `'simple'` config.**
   Rejected: removes the language mismatch problem, but still has zero
   typo tolerance and no partial/substring matching — doesn't solve the
   other stated requirements (incomplete queries, misspellings).
3. **Run both `tsvector` FTS and `pg_trgm`, combined or as a fallback
   chain.** Rejected for now: `word_similarity` alone already covers
   every scenario in the requirement (long, short, any language,
   partial, typos); maintaining two parallel search mechanisms — deciding
   which runs first, how to merge or rank combined results, testing both
   paths — adds real ongoing complexity for marginal quality gain on a
   proper-noun-heavy dataset. Revisit if venue names start carrying
   substantial free-text content (they don't today) or if the product
   later needs full-text search over long-form content elsewhere (e.g.
   check-in notes), where linguistic FTS genuinely earns its keep.

## Consequences

**Positive:**

- One search mechanism, not two — less code, less to test, no ambiguity
  about which path a given query takes.
- Works uniformly regardless of venue name language or query language,
  directly supporting Obur's global-expansion plans.
- Typo and partial-query tolerance, which the previous FTS-based search
  never had.
- The matching threshold is an explicit, named constant in
  `app.core.search`, not an implicit PostgreSQL session default.
- The migration is self-contained — no separate manual or docker-init
  step is required for the extension to be usable.

**Negative / trade-offs:**

- No linguistic stemming at all (e.g. Turkish "dönerci" vs. "döner" is no
  longer matched via grammatical stemming, only via raw trigram character
  overlap, which is weaker but still meaningfully non-zero for related
  words).
- `pg_trgm` GIN indexes are typically larger than the table they index;
  not a concern at Obur's current or near-term scale.
- Two extra pieces of database-level machinery beyond `pg_trgm` itself: a
  second extension (`unaccent`) and a hand-written `IMMUTABLE` wrapper
  function around it. Both are standard, well-documented PostgreSQL
  patterns, but a future contributor touching venue search needs to know
  they exist and why (this ADR is that record).
- If venue names ever grow substantial free-text content, this decision
  should be revisited (see Rationale, alternative 3).
