# ADR-0006: Three-Tier Visibility, Close Friends, and Bookmarks as a Private Signal

**Date:** 2026-08-18
**Status:** Accepted (supersedes [ADR-0004](0004-checkin-visibility-single-toggle.md))

## Context

Phase 4 (Social Graph & Engagement) needed a real answer to a question
ADR-0004 had deliberately left open: is public/private the whole story
for who can see a check-in, or a list, or a saved venue?

The team's own instinct going in was to add a "followers-only" tier
alongside public/private, mirroring Instagram-style private accounts.
That idea didn't survive scrutiny: Obur's follow model is one-directional
and requires no approval (see the PDD's §11 Social Graph and its
one-directional-follow rationale) — anyone can follow anyone, so a
"followers-only" tier would let in literally anyone who chooses to
follow. It isn't a privacy boundary at all under an open-follow model,
just a follow-gate with no real access control behind it.

Letterboxd was checked as the closest real-world comparable (taste-based
social curation, one-directional follow, no DMs — the same shape of
product Obur is building toward). Letterboxd's actual privacy model for a
diary entry or review is exactly three options: Public / **Close
Friends** / You — where Close Friends is a manually curated allow-list,
independent of the open follow graph, that the account holder builds by
hand from among the people who already follow them. That directly
resolves the "anyone can follow" problem: close-friends status is an
explicit, deliberate grant, not a side effect of an open follow action.

Two adjacent questions came up in the same design conversation and are
recorded here because they were decided together, as one coherent model:
how a "bookmark" (save-for-later) should relate to a "like" (a visible
social signal), and whether close-friends status should be automatic or
curated.

## Decision

**Three-tier visibility, shared identically across `CHECKIN`, `LIST`,
and `VENUE_SAVE`:**

- `public` — anyone can see it
- `close_friends` — only the owner's curated close-friends list can see
  it (see below)
- `private` — only the owner (or an admin) can see it

One shared authorization function (`app.core.authz.can_view`) enforces
this identically for all three resource types — not a bespoke privacy
concept per resource. `CHECKIN` and `LIST` default to `public` (unchanged
from ADR-0004's public-by-default rationale); `VENUE_SAVE` defaults to
`private` — saving a venue ("been here" / "want to go" / "favorite") is a
personal tracking action first, and the owner opts into sharing it, not
the other way around.

**Close friends is a manually curated subset of followers, not an
automatic tier.** A user builds their own close-friends list by hand,
adding people from among those who already follow them
(`app.models.close_friend.CloseFriend`). It is *not* symmetric: A adding
B as a close friend says nothing about whether B has done the same for
A. If A stops following B (or B removes A as a follower), A's
close-friend status is revoked automatically — a close friend can never
outlive the follow relationship it was built on (see the composite
foreign key to `FOLLOW`, `ON DELETE CASCADE`, in
`app/models/close_friend.py`).

**Bookmarks are a separate concept from likes, and are always private.**
A like (`CHECKIN_LIKE` / `LIST_LIKE`) is a visible social signal — a
count anyone can see, tied to the same visibility as the content it's
on. A bookmark (`CHECKIN_BOOKMARK` / `LIST_BOOKMARK`) is a personal
save-for-later note; nobody but the bookmarker themselves can ever see
their own bookmark list, and there is deliberately no "count how many
bookmarked this" anywhere in the system. Separate tables per target type
(not a shared polymorphic table) for real foreign-key integrity, matching
the same pattern already used for `CHECKIN_PRODUCT`.

Liking or bookmarking something is only possible if the actor can already
*see* it — both actions resolve the target through its own visibility
check first. A private check-in can't be liked or bookmarked by anyone
but its owner, for the same reason it can't be viewed.

## Rationale

Alternatives considered:

1. **A "followers-only" visibility tier**, the initial instinct. Rejected
   for the reason above: with a one-directional, no-approval follow
   model, "followers-only" has no real access-control meaning — anyone
   can grant themselves access by following. A close-friends tier, by
   contrast, requires an explicit, separate act of curation by the
   content owner.
2. **Close-friends status granted automatically** (e.g. mutual follows,
   or some engagement heuristic). Rejected: automatic membership
   reintroduces exactly the "anyone can get in" problem a manual
   allow-list exists to prevent, and it's not how the real-world
   comparable (Letterboxd) does it either.
3. **A single `LIKE`/`BOOKMARK` table with a "type" flag**, instead of
   splitting into a visible-signal concept and a private concept.
   Rejected: conflating them would either make bookmarks a visible
   signal (contradicting the explicit product requirement that saving
   something for later is private) or make likes invisible (defeating
   their purpose as social proof). They answer different questions and
   were kept as different concepts with different tables, not a shared
   table with a mode switch.

## Consequences

**Positive:**

- One shared visibility model and one shared authorization function
  across three resource types, instead of three bespoke privacy
  concepts — less code, and no risk of the rules drifting apart between
  `CHECKIN` and `LIST`.
- Close friends is real access control, not a follow-gate with no
  teeth — verified against a real comparable product's design, not
  reasoned from first principles alone.
- A user can safely use bookmarks as a private "read later" list without
  ever worrying it's visible to anyone, including the people whose
  content they bookmarked.

**Negative / trade-offs:**

- Close friends requires the owner to do manual curation work (no
  automatic suggestion or heuristic yet) — acceptable for MVP; a future
  phase could suggest close-friend candidates from engagement signals if
  the manual step turns out to be a real adoption barrier.
- `VENUE_SAVE` defaulting to `private` (unlike `CHECKIN`/`LIST`
  defaulting to `public`) is an inconsistency a new engineer could trip
  over if they assume one blanket default across the whole visibility
  system — mitigated by `app/models/venue_save.py`'s module docstring
  stating the reason directly at the point of definition.

## Superseded ADR

This decision replaces [ADR-0004](0004-checkin-visibility-single-toggle.md)'s
single `is_public` boolean with the three-tier `visibility` field
described above. ADR-0004's actual point — that feed visibility and
aggregate-rating inclusion are the same one decision, not two
independent toggles — still holds; only the number of tiers changed.
