# ADR-0002: Defer Google Places Verification

**Date:** 2026-08-13
**Status:** Accepted

## Context

`VENUE.google_places_id` has existed in the PDD's data model since the
original schema (`obur-pdd.md` §7) as an "optional reference," but the PDD
never specified what populates it or what consumes it. During Phase 2
(Venues & Products) implementation in `obur-backend`, the field was
carried into the `Venue` model as-is — nullable, client-suppliable string
— with no integration behind it: no Google Places API client, no API key
configuration, nothing in the venue-creation flow actually sets it.

This became a real question during implementation review. The PDD's
documented venue-creation flow (§11): "Search box → coordinate-based
nearby venue suggestions → not found: add a new venue (name + map pin +
address note)" — doesn't involve a Google Places search/autocomplete step
at all; a user just drops a pin and types a name. Obur's own map
rendering (Mapbox) and duplicate-venue detection (`ST_DWithin` on
`lat`/`lng`) already work entirely off the venue's own coordinates and
its own database identity (`Venue.id`), so `google_places_id` isn't
needed for anything currently built or planned in the near term.

The one coherent future use case identified: verifying that a
user-submitted pin actually corresponds to a real business (as opposed
to, say, a pin dropped in the middle of a park), using Google's Places
API as an external source of truth. This is consistent with the PDD's
broader stance that Obur intentionally leaves the "verification layer"
(operational data like hours and phone numbers) to Google Maps rather
than maintaining it itself (`obur-pdd.md` §2–3).

## Decision

Leave `google_places_id` in the `Venue` model and API schema as an
optional (`nullable`), client-suppliable field with no integration behind
it. No Google Places API client, key, or verification logic is built in
Phase 2 — the field stays `NULL` for every venue until a concrete
verification feature is scoped. This is a deferred decision, not a
rejection: revisit when a specific feature needs it (e.g. verifying a
submitted pin against a real business, or a "View on Google Maps"
link-out on the venue detail page).

## Rationale

Alternatives considered:

1. **Remove the field entirely until a real integration exists.**
   Rejected for now: leaving it nullable costs nothing (no migration
   complexity, no validation burden), and removing it would just mean
   re-adding it later via a new migration once verification is actually
   scoped. The PDD's own schema already reserves it.
2. **Build the Google Places verification flow now, alongside venue
   creation.** Rejected: not part of the PDD's documented venue-creation
   flow; adds a paid third-party dependency and API key management before
   there's evidence it's needed; Phase 2's scope is the core venue/product
   data model, not third-party verification.
3. **Justify keeping the field via Obur's own map rendering or
   duplicate-venue detection.** Rejected as a rationale — both already
   work entirely off `lat`/`lng` (Mapbox rendering, `ST_DWithin` duplicate
   detection) and `Venue.id` (Obur's own database identity), so this
   wasn't a valid reason to keep the field, just an initial mix-up
   clarified during review.

Google Places API pricing was researched to confirm deferring is
low-risk, not just convenient: as of March 2025, Google replaced its
shared $200/month credit with per-SKU free monthly thresholds (10,000
free calls/month for Essentials-tier SKUs, 5,000 for Pro, 1,000 for
Enterprise — each independent, resetting monthly, no rollover). At Obur's
expected bootstrap-stage venue-creation volume, a verification check per
new venue would very likely stay within the free tier indefinitely, so
cost isn't a reason to build this earlier than it's needed.

## Consequences

**Positive:**

- Zero cost or complexity added in Phase 2 — no premature third-party
  dependency, no API key to manage yet.
- The field remains in the schema, so a future verification feature won't
  need a new migration just to add it.
- Documents, for future contributors, exactly why the field exists but is
  unused — avoids it being rediscovered and re-litigated from scratch.

**Negative / trade-offs:**

- Every venue's `google_places_id` is `NULL` indefinitely until this is
  revisited — no way yet to verify pin accuracy or link out to Google
  Maps.
- The field sits unused in the schema, which could look like dead code to
  a new engineer without this ADR.
