# ADR-0009: Venue Discovery Enrichment — District, Duplicate Detection, and Verification

**Date:** 2026-08-19
**Status:** Accepted (supersedes [ADR-0002](0002-defer-google-places-verification.md)); not yet implemented — see `docs/roadmap.md` Phase 5 in `obur-backend`

## Context

ADR-0002 deferred any Google Places integration until "a concrete
verification feature is scoped." Two things surfaced that concrete
trigger:

1. Scoping Phase 6 (Badges) exposed a real gap: example badges like
   "Kadıköy explorer" need sub-city geography (district/ilçe) that
   `VENUE.city` alone can't express.
2. A separate discussion about venue-creation accuracy (precise
   coordinates, real addresses instead of a bare pin) converged on using
   Google Places (client-side Autocomplete for search, Geocoding for
   resolving a free-text address to an approximate coordinate) as the
   actual mechanism.

That second discussion also surfaced two things worth deciding
explicitly rather than assuming: how backend and client should split
this work, and how `VENUE`'s existing 50-metre proximity-based duplicate
detection (`ST_DWithin`, from Phase 2) should account for
`google_places_id` now that it may actually get populated, instead of
sitting permanently `NULL` as ADR-0002 left it.

A third, related question surfaced once verification was scoped: how a
venue's details are ever corrected after creation, and what happens to a
venue that closes or needs to be hidden. The PDD's original schema had a
single `status` field (`active | closed | unconfirmed`) with no defined
editing path at all — worth resolving here rather than as a separate
decision, since it's the same underlying concern as verification: how
much to trust venue data, and who's allowed to change it.

## Decision

**Backend scope is deliberately narrow and has zero new external
dependencies:**

- `VENUE.district` (ilçe / sub-city administrative area) — required for
  every venue created from this phase forward, whichever path created
  it (Google Places address components via Autocomplete, or typed by
  hand in the manual-entry form — both required to supply it, not just
  Autocomplete). Nullable only for venues that already existed before
  this phase shipped; no backfill planned. The backend never calls
  Google directly for this — it only stores whatever the client already
  resolved.
- `VENUE.google_places_id` gains a partial unique index
  (`WHERE google_places_id IS NOT NULL`) — two venues can no longer
  carry the same Google Places identity.
- `create_venue`'s duplicate check becomes two layers instead of one:
  1. **Exact `google_places_id` match** — if the incoming
     `google_places_id` matches an existing venue's, that's a certain
     duplicate (Google's own identity says so, not a coordinate
     estimate). Resolves to the existing venue idempotently, the same
     pattern as Phase 4's `follow_user`/`add_close_friend` — no "did you
     mean this one?" prompt, and **not bypassable** via
     `confirm_duplicate`. There's nothing to confirm; it's the same
     business by definition.
  2. **50-metre `ST_DWithin` fallback** — unchanged from Phase 2, for
     everything without a `google_places_id` match: a possible, not
     certain, duplicate. Still surfaces "did you mean this one?" and
     remains overridable via `confirm_duplicate`.

**`VENUE.is_verified` — cosmetic only, never affects ranking, search, or
discovery.** It answers exactly one question: "does this location
definitely exist there?" — not a quality or trust signal about the
venue itself. Set `true` when either:

1. `google_places_id` is set *and* at least `N` independent users
   (distinct `user_id`, soft-deleted and non-`public` check-ins
   excluded) have checked in, or
2. with no `google_places_id`, at least `M` independent users have
   checked in under the same public-only rule *and* an admin has
   separately confirmed it via a new admin-only endpoint. Only
   `public` check-ins count toward `N`/`M` — the same restriction the
   aggregate rating (PDD §8) already applies, and for the same reason:
   a `close_friends`/`private` check-in is real evidence to whoever
   made it, but letting it feed a platform-wide "verified" mark would
   surface a deliberately restricted share's consequence to everyone,
   undercutting the visibility choice it was given. The check-in
   threshold is a filter, not the verification itself — it keeps the
   admin queue limited to venues
   that already show real community traction, instead of every
   manually added venue landing in front of an admin.

Evaluated synchronously, as one extra count query inside check-in
creation — not built as a general-purpose "evaluation engine." That
architecture question (synchronous vs. periodic vs. hybrid) belongs to
Badges (next phase), which has a much wider variety of trigger
conditions to support; venue verification's own trigger is simple
enough not to need that machinery, and building it prematurely here
would risk shaping it around one narrow case.

No notification is sent for verification — decided against deliberately
as unnecessary friction on a purely cosmetic signal.

**Venue status becomes two independent, admin-only booleans — no `status`
enum, and no user-facing edit path at all.** `VENUE.is_active` (default
`true`) and `VENUE.is_suspended` (default `false`) replace what the
original PDD schema had as a single `status` string:

- `is_active = false` — the business itself has closed. Set only by an
  admin, acting on a report (below). The venue stays fully visible,
  shown transparently — grayed out, clearly marked inactive — the same
  "historical data is never deleted" treatment already given to closed
  venues' own check-ins.
- `is_suspended = true` — a separate admin moderation action, unrelated
  to whether the business is open. The venue is hidden entirely: its own
  page returns a generic "not found," never an explanation that it was
  suspended. Check-ins referencing it are untouched wherever else they
  surface (a user's feed, their profile) — only the venue's own page
  becomes inaccessible. This mirrors the "hidden must be indistinguishable
  from nonexistent" treatment already applied to a blocked profile: the
  fact of moderation must not leak to whoever it affects.

**No venue field is ever directly editable by any user, including
whoever added it.** Every correction — wrong address, wrong name,
permanently closed, duplicate — goes through the same reporting
mechanism used for check-ins and profiles, with venue-specific report
reasons. An admin reviewing a venue report can dismiss it, correct the
field directly, set `is_active = false`, or set `is_suspended = true`.
Direct community editing (letting any user, or even just the original
adder, edit a venue's fields) was considered and rejected on a real
precedent: an unrelated app allowed users to directly edit street-animal
location pins on a map, and that capability was abused to relocate or
delete other people's pins maliciously. Unmoderated edits to shared
location data are a standing abuse surface regardless of team size.

**Explicitly out of scope, deferred, not yet scheduled:**

- A backend-proxied Geocoding endpoint (free-text address → approximate
  coordinate, for the map-pin step of manual venue entry). Real new
  scope: a third-party secret, an outbound HTTP dependency, its own
  failure modes and cost monitoring — kept separate on purpose rather
  than bundled into this lower-risk change. Gets its own ADR when
  actually prioritized.
- Client-side Places Autocomplete itself isn't backend work — it
  belongs in `obur-web`/`obur-mobile`, neither of which exists yet
  (client work is intentionally sequenced after backend reaches a
  stable, feature-complete state — a team-capacity decision, not an
  architectural one, so not itself recorded here).

## Rationale

Alternatives considered:

1. **Keep the single 50-metre check, ignore `google_places_id` even
   once populated.** Rejected: coordinate proximity alone produces real
   false positives exactly where they matter most — malls and office
   buildings ("iş hanı") with several genuinely different businesses
   within a few metres of each other, a case the PDD itself already
   flags. A canonical identity signal, when available, is strictly
   better than a distance estimate.
2. **Let `confirm_duplicate` bypass an exact `google_places_id` match
   too**, for consistency with the fuzzy case. Rejected: an exact match
   isn't a judgment call the way "is this the same place?" is at 50
   metres — Google's own identity already answered that question.
   Allowing a bypass would let two rows exist for one verified-identical
   real business, defeating the reason `google_places_id` was made
   unique in the first place.
3. **Bundle the Geocoding proxy and/or client Autocomplete into this
   same piece of work**, since they were discussed together. Rejected:
   they don't share this decision's defining property — zero new
   external dependency — and mixing a no-risk schema/logic change with
   a new third-party secret and outbound HTTP call would force the safe
   part to wait on review/scoping the riskier part for no benefit.

For verification specifically, three designs were weighed against being
both reliable and sustainable for a two-person, bootstrapped team:

4. **Google match alone = verified**, no check-in requirement.
   Considered, not chosen outright: cheap and instant, but trusts a
   single external source with zero corroborating signal from actual
   platform activity, and does nothing for the real, legitimate venues
   Google hasn't indexed.
5. **Admin review for every submitted venue.** Rejected: doesn't scale
   past a handful of venues a week for a two-person team — directly
   violates the "sustainable" requirement, turning every venue addition
   into a manual chore.
6. **Hybrid (chosen): Google match + check-in count where available,
   check-in count + admin review where not.** Two independent signals
   instead of trusting either alone, zero human labor in the common
   (Google-covered) case, and admin effort only spent on the smaller
   pool of venues that already show real community traction — the
   check-in threshold does the filtering work an admin would otherwise
   have to do by hand.

For venue status and editing, two more alternatives were weighed:

7. **Let the original adder (or any user) edit a venue's fields
   directly**, at least for narrow, obviously-safe corrections like a
   typo'd address. Rejected outright on the abuse precedent above — even
   a narrow, well-intentioned direct-edit surface is a standing abuse
   vector, and there's no reliable way to distinguish a good-faith
   correction from a malicious one at write time. Report-then-admin-review
   is strictly slower but puts a human in the loop before anything
   actually changes.
8. **A single `status` enum (`active | closed | suspended`) instead of
   two independent booleans.** Considered, rejected: "the business closed"
   and "an admin suspended this listing" aren't mutually exclusive from
   the system's own perspective, and collapsing "real-world operating
   state" and "moderation action" into one field reproduces the same kind
   of semantic overlap the original `status`/`is_verified` pairing already
   had (see Context) — independent booleans keep those two concerns
   actually independent, including the case where both are true at once.

**Known, not-yet-verified risk, recorded rather than hidden:** a small
business inside a mall or office building that Google hasn't separately
indexed could, in rare cases, have Autocomplete resolve to the parent
building's own `place_id` instead of failing to match at all — which
would make the exact-match layer wrongly auto-merge two different real
businesses. This has not been tested against a real Turkish POI dataset
yet. Before this ships, verify against at least one real mall/iş hanı
listing and at least one small-town address (Google's Turkish
administrative-hierarchy coverage is generally solid in major cities,
less proven elsewhere).

## Consequences

**Positive:**

- Unblocks Phase 6 (Badges) — `district` exists before badge evaluation
  logic needs it.
- Duplicate resolution gets strictly more reliable when a canonical
  identity signal is available, while degrading gracefully (unchanged
  behavior) when it isn't.
- Zero cost or third-party-dependency risk introduced in this phase —
  consistent with ADR-0002's original low-risk framing, just no longer
  deferred indefinitely.
- Backend stays agnostic to *how* `district`/`google_places_id` were
  resolved, so it doesn't need to change again when the client-side
  Autocomplete or backend Geocoding-proxy work eventually lands.
- Verification adds a real trust signal with zero ongoing human labor in
  the common case, and bounds the admin-review case to a naturally small
  queue instead of an unbounded one.
- Venue data quality now has a single, coherent trust model end to end —
  identity (duplicate detection), positive trust (verification), and
  negative/corrective trust (status, editing) all live in one document
  instead of being decided piecemeal across separate discussions.
- No standing abuse surface for venue data: every change to a venue after
  creation passes through a human admin, the same guarantee reporting
  already gives check-ins and profiles.

**Negative / trade-offs:**

- The mall/iş hanı false-merge edge case remains a real, open risk until
  empirically verified — must not ship silently unverified.
- `district` stays `NULL` for every venue added before this phase ships,
  the same "field exists, unpopulated until used" trade-off ADR-0002
  already accepted for `google_places_id` itself — no backfill planned.
- Two duplicate-detection code paths (exact match vs. proximity) instead
  of one add a small amount of conditional complexity to `create_venue`.
- A venue with a `google_places_id` but very low check-in activity (or a
  niche venue awaiting admin review) stays unverified indefinitely —
  acceptable since verification is purely cosmetic and never gates
  discoverability.
- Two additional boolean columns and an admin-facing venue-edit surface
  (edit endpoint, venue-specific report reasons) are new scope beyond
  this ADR's original "zero new external dependency" framing — still
  true in the sense that matters (no third-party dependency), but more
  backend surface than the title alone suggests.
- Every venue correction now depends on admin availability — there's no
  fast path for even an obviously-correct fix (a genuine typo), which is
  an accepted trade-off given the abuse precedent, not a free one.
