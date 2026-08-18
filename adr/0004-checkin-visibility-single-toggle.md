# ADR-0004: Single Visibility Toggle for Check-ins

**Date:** 2026-08-18
**Status:** Superseded by [ADR-0006](0006-three-tier-visibility-and-close-friends.md)

> **2026-08-18 update:** Phase 4 replaced the boolean `is_public` this
> ADR established with a three-tier `visibility` field
> (`public`/`close_friends`/`private`), shared across `CHECKIN`, `LIST`,
> and `VENUE_SAVE` — see ADR-0006. This ADR's actual point — that feed
> visibility and aggregate-rating inclusion are one decision, not two
> independent toggles — still holds; only the number of tiers changed.
> Kept here for the historical record of why the second toggle was
> dropped.

## Context

An earlier draft of the check-in "Share" step (§10 of the PDD) had two
toggles: "public" (visible in followers' feeds) and a second
"contribute to statistics" toggle, described as counting toward the
venue/product aggregate rating "even when off."

That second toggle's own description is self-contradictory: if it
counts toward the aggregate regardless of its own state, it has no
observable effect, and a user-facing control with no effect is worse
than no control at all — it implies a choice that isn't real.

## Decision

Drop the second toggle. `CHECKIN.is_public` is the only visibility
control, and it now does both jobs: a public check-in is visible in
feeds *and* counts toward the venue's/product's aggregate rating; a
private check-in is visible only to its owner (or an admin) and does
not count toward the aggregate.

## Rationale

Alternatives considered:

1. **Fix the toggle's description instead of removing it** (e.g. make
   it genuinely independent of `is_public`, so a private check-in could
   still opt into contributing to the aggregate). Rejected: adds a
   second, rarely-used control for a niche case, without a clearly
   identified user need driving it. If evidence later emerges that
   privacy-conscious users are a large enough share of check-ins to
   meaningfully bias the aggregate, this can be revisited then, with a
   real design informed by real usage instead of a speculative toggle.
2. **Keep both toggles as originally drafted.** Rejected outright — the
   as-written behavior ("counts even when off") is not a real toggle,
   just a confusing UI element.

## Consequences

**Positive:**

- One control, one clear meaning — simpler for users and for the data
  model (no extra `CHECKIN` column).
- No ambiguity about what a private check-in does or doesn't affect.

**Negative / trade-offs:**

- A user who wants their check-in kept private but still wants it to
  inform the venue's aggregate rating has no way to do that. Not
  supported until (and unless) real usage shows this matters — see
  Rationale, alternative 1.
