# ADR-0005: Check-in Fields Are Editable, Rated Products Are Not

**Date:** 2026-08-18
**Status:** Accepted

> **2026-08-18 update:** `is_public` (referenced below as one of the
> editable fields) was replaced by the three-tier `visibility` field in
> ADR-0006. This ADR's own decision — which fields are editable at all —
> is unaffected; only that one field's name changed.

## Context

docs/roadmap.md's Phase 3 (obur-backend) says a user may edit their own
check-in, without specifying what "edit" actually covers. A check-in has
two kinds of data: its own fields (note, photo, visit date, visibility,
venue-level ratings) and the set of products it rates, each with its own
rating (`CHECKIN_PRODUCT`).

The open question: once a check-in is saved, can the user add or remove
a rated product, or change an individual product's rating — or is only
the check-in's own fields editable?

## Decision

`PATCH` on a check-in updates only its own fields: `note`, `photo_url`,
`is_public`, `visited_at`, and the venue-level `rating_service` /
`rating_ambiance` / `rating_value`. The set of rated products, and each
product's `rating`, is fixed once the check-in is created. Correcting a
wrong product or rating means deleting the check-in (soft-delete, same
as any other deletion — see the "Historical data is never deleted"
design decision) and creating a new one with the correct data.

## Rationale

Real-world precedent was checked before deciding, not just reasoned
from first principles:

- **Untappd** (the closest comparable product — check in an item, rate
  it, earn badges from the aggregate) allows editing a check-in's own
  fields (rating, tasting notes, photo, serving style, location) but
  never which beer was checked in — its check-ins are single-item, so
  the "add/remove an item" question doesn't arise for them at all.
  Notably, Untappd's own documentation states that editing a check-in
  may change what badges it qualifies for *going forward*, but **badges
  already earned are never removed** — the same principle already
  applied to Obur's check-in soft-delete.
- **MyFitnessPal** (multi-item per entry, like Obur) allows freely
  adding/removing food items from a saved meal log. This works for them
  because a personal food diary has no public aggregate or badge riding
  on it — editing it corrupts nothing anyone else depends on. Obur's
  check-ins are the opposite: public, aggregated, and badge-generating,
  much closer to Untappd's stakes than MyFitnessPal's.

Alternatives considered:

1. **Allow adding/removing rated products.** Rejected: the PDD's own
   check-in flow (§10) is a single guided sequence — venue, then
   products, then ratings, then save — with no described re-entry
   point. The practical need ("I forgot an item") is already served by
   creating a new check-in. The complexity this would add is real: a
   product added after the fact would need its own creation timestamp
   distinct from the check-in's, which future badge logic (e.g. "first
   to try this product on the platform," PDD §13) would have to account
   for separately.
2. **Allow editing an individual product's existing rating in place.**
   Considered more seriously — this looked like a low-risk "fix a typo"
   case, since it doesn't change which products are rated. Rejected
   anyway: it adds a real chunk of validation surface (which product,
   does it belong to this check-in, is the new rating in range) for a
   need that's already fully covered, with zero new code, by deleting
   and recreating the check-in. The delete-and-recreate path is also
   more consistent with "historical data is never silently rewritten" —
   the wrong rating stops counting (excluded via `deleted_at`) and the
   corrected one starts fresh, rather than a row mutating in place after
   it may already have been read into some derived computation.

## Consequences

**Positive:**

- `PATCH`'s validation surface stays small and unambiguous — it only
  ever touches scalar fields on `CHECKIN` itself.
- No new question for future badge/aggregate logic (Phase 5) about
  "when was this specific product rating actually added" versus "when
  was the check-in created."
- The correction path (delete, recreate) already exists with no
  additional code.

**Negative / trade-offs:**

- Fixing one wrong rating among several costs more user effort
  (re-entering the whole check-in) than an in-place fix would. Judged
  an acceptable trade for the simplicity and integrity gained — revisit
  if this turns out to be a frequent, frustrating flow in practice.
