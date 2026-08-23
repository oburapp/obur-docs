# ADR-0010: Blocking and Reporting Schema

**Date:** 2026-08-23
**Status:** Accepted; not yet implemented — see `docs/roadmap.md` Phase 10 in `obur-backend`

## Context

Blocking and reporting are both P0 in the PDD's feature catalog (§6), and
both are described at length in §11 Social Graph — blocking's absolute,
silent, bidirectional semantics, and reporting's three target types with
their human-reviewed queue. Neither is optional polish: Apple's App Store
Review Guidelines §1.2 requires a mechanism to report objectionable content
*and* a mechanism to block abusive users for any app carrying user-generated
content, and Google Play's UGC policy asks for materially the same. Without
both, the app does not clear store review.

Despite that, neither has a table anywhere. §7's data model lists 28 tables
and includes neither `BLOCK` nor any report table; the entity-relationship
diagram doesn't either. The behavior is fully specified in prose and the
schema it implies was never written down — which surfaced while rewriting
`obur-backend`'s roadmap to cover the PDD in full, when Phase 10 turned out
to have nothing to implement against.

`MUTE` is the exception among the three social-safety primitives: it was
added to §7 and the ER diagram when it was decided, and needs no new
decision here. This ADR covers only the two that are missing.

The real design question is reporting's target-type model. Obur already
contains both possible answers to "one row type pointing at several kinds of
target," each with a stated reason:

- [ADR-0006](0006-three-tier-visibility-and-close-friends.md) split likes and
  bookmarks into `CHECKIN_LIKE`/`LIST_LIKE` and
  `CHECKIN_BOOKMARK`/`LIST_BOOKMARK` — separate tables per target type,
  explicitly for real foreign-key integrity.
- [ADR-0008](0008-synchronous-in-app-notifications.md) made `NOTIFICATION` a
  single table whose `target_type`/`target_id` is deliberately *not* a
  foreign key, because a notification is a transient record whose own
  correctness doesn't depend on its target still existing.

The distinction those two ADRs draw between them is the one that decides
this: **does this row's correctness depend on its target still existing?**

## Decision

### `BLOCK`

```sql
BLOCK
  blocker_id    UUID FK → USER ON DELETE CASCADE
  blocked_id    UUID FK → USER ON DELETE CASCADE
  created_at    TIMESTAMPTZ
  PRIMARY KEY (blocker_id, blocked_id)
                                          -- CHECK (blocker_id != blocked_id)
                                          -- INDEX on blocked_id, for the
                                          -- reverse-direction lookup
```

Deliberately the same shape as `FOLLOW`: a composite primary key over two
user references, with a `CHECK` rejecting self-reference. Two things follow
from storing direction rather than a symmetric pair:

- **Only the blocker can unblock.** §11 makes unblocking a real,
  always-available action — by the person who blocked. A symmetric
  representation would lose who initiated, and with it the ability to
  enforce that.
- **Blocking stays silent.** The blocked person is never told, so nothing
  may ever expose which direction a block runs.

**Enforcement is bidirectional even though storage is directional.** Every
query that filters on blocking asks whether a row exists in *either*
direction — the reverse-direction lookup is what the index on `blocked_id`
serves, the same role `FOLLOW`'s index on `following_id` already plays.

`ON DELETE CASCADE` on both sides: unlike a report, a block is a live
relationship with no archival value once either party's account is gone.

### Reports: two tables, split by concern rather than by target table

```sql
CONTENT_REPORT                            -- interpersonal-safety reports
  id            UUID PK
  reporter_id   UUID FK → USER ON DELETE SET NULL
  target_type   VARCHAR                  -- checkin | user
  target_id     UUID                     -- not a real FK — see Rationale
  reason        VARCHAR                  -- spam | harassment | hate_speech |
                                          -- sexual_content | violence |
                                          -- fake_account | other
  status        VARCHAR DEFAULT 'pending' -- pending | dismissed | actioned
  resolved_by   UUID FK → USER, nullable -- the admin who acted
  resolved_at   TIMESTAMPTZ, nullable
  created_at    TIMESTAMPTZ
                                          -- UNIQUE (reporter_id, target_type,
                                          -- target_id): one open report per
                                          -- person per target

VENUE_REPORT                              -- data-quality reports
  id            UUID PK
  reporter_id   UUID FK → USER ON DELETE SET NULL
  venue_id      UUID FK → VENUE          -- a real foreign key, unlike above
  reason        VARCHAR                  -- wrong_address | wrong_name |
                                          -- permanently_closed | duplicate |
                                          -- other
  status        VARCHAR DEFAULT 'pending' -- pending | dismissed | actioned
  resolved_by   UUID FK → USER, nullable
  resolved_at   TIMESTAMPTZ, nullable
  created_at    TIMESTAMPTZ
                                          -- UNIQUE (reporter_id, venue_id)
```

The split follows the one §11 already draws in prose: check-ins and profiles
"carry interpersonal-safety risk," while a venue report "is a data-quality
concern instead." That isn't a cosmetic difference — the two kinds share
almost nothing:

| | `CONTENT_REPORT` | `VENUE_REPORT` |
|---|---|---|
| Reason vocabulary | spam, harassment, hate speech, sexual content, violence, fake account | wrong address, wrong name, permanently closed, duplicate |
| Admin resolutions | dismiss, remove content, suspend the account (`USER.status`) | dismiss, correct the field directly, `is_active = false`, `is_suspended = true` |
| Urgency | time-sensitive — an unreviewed harassment report is real, ongoing harm | none — a wrong address can wait |
| Target lifetime | can be permanently purged | never deleted |

**`VENUE_REPORT.venue_id` is a real foreign key; `CONTENT_REPORT.target_id`
is not.** This is the ADR-0006-versus-ADR-0008 test applied honestly to each
case rather than picking one pattern for both:

- A `VENUE` row is never deleted. A closed business sets `is_active = false`
  and a suspended one sets `is_suspended = true`; account deletion sets
  `added_by` to `NULL` rather than removing the venue
  ([ADR-0009](0009-venue-discovery-enrichment.md), PDD §7). The target's
  existence is guaranteed, so the foreign key costs nothing and buys real
  integrity.
- A `CHECKIN` can be permanently purged by an admin, and every check-in
  belonging to a deleted account is purged with it. A `USER` row is likewise
  destroyed outright by account deletion — the PDD's one deliberate
  exception to "historical data is never deleted." A real foreign key would
  force a choice between cascading the report away (destroying the audit
  trail of *why* the content was removed) and blocking the purge itself
  (breaking a store-compliance guarantee). Neither is acceptable.

**Both tables keep `reporter_id ... ON DELETE SET NULL`.** A report is not
personal content the way a check-in or a list is — it is a moderation
record, closer in kind to a `VENUE` than to a `CHECKIN`. It survives the
reporter's account deletion with the reporter's identity dropped, matching
`VENUE.added_by`'s existing treatment.

**Reporting stays human-reviewed with no automatic threshold-based hiding**,
per §11. `status` exists to drive an admin queue, never to accumulate toward
an automatic action. `UNIQUE (reporter_id, ...)` stops one person filing the
same report repeatedly; PDD §17 additionally places report submission in the
strict rate-limit tier, since unlimited reporting is itself an abuse vector.

## Rationale

Alternatives considered for the report model:

1. **One polymorphic `REPORT` table** with `target_type` in
   (`checkin`, `user`, `venue`), following `NOTIFICATION`'s pattern
   throughout. Rejected — though not unreasonably: it would force the two
   disjoint reason vocabularies into one column, expressible at the database
   level only through a `CHECK` constraint correlating `target_type` with
   `reason` (a pattern this schema does use elsewhere, e.g.
   `ck_checkins_visibility_allowed`). The deciding cost is that it also
   gives up `VENUE_REPORT`'s perfectly safe foreign key for nothing —
   accepting the weakest integrity guarantee of the three targets across all
   three. It would also put a time-sensitive safety queue and a
   zero-urgency data-quality queue in one table, when §11 already treats
   their urgency as different in kind.
2. **Three tables, one per target** (`CHECKIN_REPORT`, `USER_REPORT`,
   `VENUE_REPORT`), following ADR-0006's like/bookmark split literally.
   Rejected: it buys nothing over two, because neither of the two content
   targets can hold a real foreign key anyway — the only reason ADR-0006
   split its tables. It would duplicate an identical reason vocabulary,
   an identical admin workflow, and an identical queue query across two
   tables that differ in nothing but which id column they carry.
3. **Reports as a `status` field on the reported row itself** (e.g.
   `CHECKIN.report_count`). Rejected outright: it can't record who reported,
   why, or what an admin decided, and it silently reintroduces the
   threshold-counting model §11 explicitly rejected.

For `BLOCK`, a symmetric representation (either two mirrored rows, or a
single row with the pair stored in a canonical order) was considered and
rejected: enforcement is bidirectional, but *authority* is not — only the
blocker may unblock, and the blocked person must never learn a block
exists. Both require knowing the direction.

## Consequences

**Positive:**

- Phase 10 has a schema to build against, and the P0 store-review
  requirement stops resting on prose alone.
- `VENUE_REPORT` keeps real referential integrity, so a venue moderation
  queue can never show a report pointing at nothing.
- Moderation history survives both the purge of the reported content and
  the deletion of the reporter's account — an admin can always answer "why
  was this removed," which is exactly what an audit trail is for.
- `BLOCK` reuses `FOLLOW`'s shape, so its indexing and its
  correlated-subquery filter pattern
  (`app.core.authz.close_friend_of_owner_exists`) carry over rather than
  being invented again.

**Negative / trade-offs:**

- Two report tables means the admin queue is two queries unioned in the
  application layer rather than one — accepted deliberately, since the two
  kinds are reviewed by different criteria and resolved by different
  actions anyway, and their urgency differs enough that an admin may well
  want them as separate views.
- `CONTENT_REPORT.target_id` has no referential integrity, so a report can
  outlive its target and the application must handle a dangling target
  gracefully in the admin queue. This is the same trade-off `NOTIFICATION`
  already accepted, made deliberately here rather than inherited by
  default.
- Blocking's retroactive cleanup — purging likes, bookmarks, notifications,
  and mentions between the two people in both directions (§11) — is
  service-layer work that no schema constraint can express. Only the
  auto-revocation of close-friend status is free, via `CLOSE_FRIEND`'s
  existing cascade off `FOLLOW`.
- Every read path now carries a bidirectional block check. That cost is
  real and is the reason `obur-backend`'s roadmap places Phase 10 before
  the read-heavy phases (feed, discovery, rankings) rather than after them.

**Explicitly unchanged:** admin moderation access is never affected by a
block between two other users — reports must stay reviewable regardless of
who blocked whom (§11). Aggregate ratings are likewise untouched: they are
anonymous, platform-wide numbers with no per-viewer dimension, so a block
has nothing to act on there.
