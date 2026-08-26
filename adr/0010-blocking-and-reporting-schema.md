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

### `BLOCK`'s access control was an open question this schema didn't answer

Who may `SELECT` a block row was left undecided when the table shape above
was written. Two shapes are both defensible:

- **A. Either party can see it.** The blocker and the blocked person can
  each confirm a block exists between them. A plain RLS policy
  (`blocker_id = current OR blocked_id = current`) is enough, and every
  enforcement call site can use an ordinary correlated subquery, the same
  shape `close_friends_select` already has (migration `e4f8b21ac930`) and
  `app.core.authz.close_friend_of_owner_exists` already reuses for listing
  queries.
- **B. Only the blocker can see it, period.** The blocked person's own
  database session can never successfully `SELECT` the row under any
  circumstance, including a future bug in application code that queries
  `blocks` directly instead of going through the sanctioned check. Every
  other table's enforcement then has to go through a narrow, purpose-built
  bypass rather than a plain query, since a plain correlated subquery run
  as the blocked person's session would find nothing: RLS would filter
  `blocks` down to rows where they are the blocker, which for the one row
  that matters, they never are.

Chosen: **B**. The cost is real: a `SECURITY DEFINER` bypass function is
now required everywhere blocking needs to be checked, including standing
in for `blocks`' own would-be-natural correlated subquery, and
`docs/roadmap.md`'s original Phase 10 sketch (reuse
`close_friend_of_owner_exists`'s pattern directly) no longer applies as
written, since that pattern depends on a bidirectionally-readable table
the way `close_friends` is and `blocks` now deliberately isn't. The
benefit: "can the blocked person ever see this row" stops being a
property every call site has to get right and becomes a property of the
schema itself, the same reasoning already applied to `private`/
`close_friends` visibility, carried one step further for the one
primitive whose entire purpose is keeping one specific person out.

```sql
CREATE FUNCTION rls_is_blocked_pair(user_a uuid, user_b uuid)
RETURNS boolean
LANGUAGE sql
STABLE
SECURITY DEFINER
SET search_path = public
AS $$
    SELECT EXISTS (
        SELECT 1 FROM blocks
        WHERE (blocker_id = user_a AND blocked_id = user_b)
           OR (blocker_id = user_b AND blocked_id = user_a)
    );
$$;
```

Same shape as `rls_is_admin()` (migration `c1d5a8f042e7`): owned by the
table-owning role, so it bypasses `blocks`' own restrictive `SELECT`
policy internally regardless of which identity calls it, then granted
`EXECUTE` to `obur_app` explicitly. Every blocking check in the system,
RLS policy or application code, goes through this one function rather
than querying `blocks` directly, the same role `rls_current_user_id()`
already plays as the one place `app.current_user_id` is ever read.

```sql
CREATE POLICY blocks_select ON blocks
FOR SELECT
USING (rls_is_admin() OR blocker_id = rls_current_user_id());

CREATE POLICY blocks_insert ON blocks
FOR INSERT
WITH CHECK (rls_is_owner_or_admin(blocker_id));

CREATE POLICY blocks_delete ON blocks
FOR DELETE
USING (rls_is_owner_or_admin(blocker_id));
```

`blocks_delete` is blocker-only, matching "only the blocker can unblock"
exactly, with no gap between the stated product rule and what the
database actually allows. An earlier draft of this section made it
permissive on `blocked_id` too, reasoning from `follows_delete`/
`close_friends_delete` (migration `e4f8b21ac930`) that account-deletion
cascades needed it. That reasoning doesn't hold: PostgreSQL's
referential integrity checks always bypass row security ("Row Security
Policies" in the PostgreSQL manual), specifically so a policy can never
leave a dangling foreign key behind, so `DELETE FROM users` cascading
into `blocks` via `ON DELETE CASCADE` never consults `blocks_delete` at
all, for either party. Verified directly: a blocked person's raw
`DELETE` against the row naming them affects zero rows under this
policy, and a real `DELETE FROM users` for that same account still
removes the row, in the same test. A permissive policy would have bought
nothing for cascade correctness while opening a real gap: it would let
the blocked person delete the row protecting the blocker from them,
through any future code path that issues a raw `DELETE` rather than
going through the sanctioned unblock endpoint. `follows_delete`/
`close_friends_delete`'s own permissiveness is correct on its own
terms, both sides of a follow or a close-friend relationship really can
end it through the application (PDD §11), so their policies have to
allow it; blocking has no equivalent rule, only the blocker ever has a
legitimate reason to delete the row.

**Enforcement extends into RLS on every table blocking has to reach, not
only `app.core.authz.can_view`.** `docs/roadmap.md`'s original sketch
scoped this to `can_view` and its callers, on the same reasoning
`close_friends` visibility used. Reconsidered: `can_view` is exactly the
layer `obur-backend/CLAUDE.md`'s own Security section already warns is
easy to bypass by a query that forgets to call it, the same gap RLS as a
whole exists to close, and blocking is the one social-safety primitive
where that gap is least acceptable. Concretely:

- `rls_can_view_visibility` (migration `b7e4f209ac31`), the function
  `checkins_select`, `lists_select`, `venue_saves_select`, and
  (transitively, through `rls_can_view_checkin`/`rls_can_view_list`)
  `checkin_likes_select`, `list_likes_select`, and `list_items_select`
  all resolve through, gains a blocking guard around its `public`/
  `close_friends` branches only. The owner-or-admin branch stays
  unconditional, both because a block cannot exist between an account and
  itself and because admin moderation access is explicitly untouched by
  blocking (PDD §11). One function edit, six `SELECT` policies corrected
  without touching any of their own already-applied migrations.
- `users_select` (currently `USING (true)`, migration `e4f8b21ac930`)
  gains `rls_is_admin() OR NOT rls_is_blocked_pair(id, rls_current_user_id())`.
  PDD §11 is explicit that "a blocked profile behaves exactly like a
  nonexistent one" to the other party, which a `users` row that stays
  fully visible would not satisfy. This doesn't reopen the
  identity-resolution problem that keeps `users_select` from being
  owner-restricted: `app.core.auth._find_user` resolves the caller's own
  row by `auth_provider_id` before any blocking comparison is relevant,
  since nobody is ever blocked relative to themselves.

  `users_select` itself stays exactly this, symmetric, no exception,
  once `app.services.block.list_blocked_users` needed to show a blocker
  who they'd blocked (usernames and avatars, for an unblock button, the
  same thing every real app's blocked-accounts screen shows). An
  earlier draft of this migration widened `users_select` itself with an
  `rls_am_i_blocking(target_id)` exception; caught before it reached
  `main` for reaching further than intended: `users_select` governs
  every query that touches `users`, not only the blocklist screen, so
  the widened policy let a blocker resolve the blocked person's profile
  anywhere their id surfaced, a third party's checkin's likes, a shared
  follower list, not only the one screen that actually needed it.

  The narrower replacement is `rls_list_blocked_users(p_limit, p_offset)`
  (migration `f1d017015e34`), a table-returning `SECURITY DEFINER`
  function scoped to exactly this one query, joining `blocks` to `users`
  directly rather than going through `users_select` at all. It also
  takes no `blocker_id` parameter: a `SECURITY DEFINER` function is a
  real RLS bypass, so trusting an argument for whose blocklist to
  return would only be as safe as every future caller happening to
  pass their own id correctly. It reads `rls_current_user_id()`
  internally instead, the same way `rls_is_admin()` never takes a
  caller-supplied id either, making the mistake impossible rather than
  merely unlikely.
- `checkin_bookmarks`/`list_bookmarks` need no change: their `SELECT`
  policy is already owner-only regardless of blocking ("nobody, including
  the checkin's owner, can see who bookmarked it"), so there is nothing
  left for a block to additionally hide.
- `follows`/`close_friends` need no RLS change either. PDD §11 makes
  blocking auto-unfollow in both directions at the moment of blocking, an
  active deletion the block-creation service performs, not an ongoing
  visibility rule; `close_friends` then clears for free through its
  existing cascade off `follows`.
- `notifications` gets both a `SELECT` and an `INSERT` guard:
  `notifications_select` adds `AND NOT rls_is_blocked_pair(user_id, actor_id)`,
  and `notifications_insert` adds the same condition to its existing
  `WITH CHECK`. The retroactive purge below already deletes any
  notification that predates a block; this is the forward-looking
  backstop against a notification-service bug that fires across an
  already-active block, the same "backstop, not a re-derivation" role
  `notifications_insert`'s existing `rls_current_user_id() IS NOT NULL`
  check already plays.

**Retroactive purge (likes, bookmarks, notifications, and `follows`, in
both directions, at the moment of blocking) is service-layer work no RLS
policy performs**, per PDD §11. Mentions are named in the same PDD
paragraph but have no table yet; the purge service leaves a documented
gap for Phase 12 to close once `MENTION` ships, rather than guessing at
that table's shape now.

### Reports: two tables, split by concern rather than by target table

```sql
CONTENT_REPORT                            -- interpersonal-safety reports
  id            UUID PK
  reporter_id   UUID FK → USER ON DELETE SET NULL
  target_type   VARCHAR                  -- checkin | user
  target_id     UUID                     -- not a real FK — see Rationale
  reason        VARCHAR                  -- spam | harassment | hate_speech |
                                          -- sensitive_content | violence |
                                          -- fake_account | other
  details       TEXT, nullable           -- required only when reason='other'
  status        VARCHAR DEFAULT 'pending' -- pending | dismissed | actioned
  resolved_by   UUID FK → USER, nullable -- the admin who acted
  resolved_at   TIMESTAMPTZ, nullable
  created_at    TIMESTAMPTZ
                                          -- UNIQUE (reporter_id, target_type,
                                          -- target_id): one open report per
                                          -- person per target
                                          -- CHECK (reason != 'other' OR
                                          -- details IS NOT NULL)

VENUE_REPORT                              -- data-quality reports
  id            UUID PK
  reporter_id   UUID FK → USER ON DELETE SET NULL
  venue_id      UUID FK → VENUE          -- a real foreign key, unlike above
  reason        VARCHAR                  -- wrong_address | wrong_name |
                                          -- permanently_closed | duplicate |
                                          -- other
  details       TEXT, nullable           -- required only when reason='other'
  status        VARCHAR DEFAULT 'pending' -- pending | dismissed | actioned
  resolved_by   UUID FK → USER, nullable
  resolved_at   TIMESTAMPTZ, nullable
  created_at    TIMESTAMPTZ
                                          -- UNIQUE (reporter_id, venue_id)
                                          -- CHECK (reason != 'other' OR
                                          -- details IS NOT NULL)
```

**`details` is optional free text on both tables, required only when
`reason` is `other`.** A fixed reason vocabulary can't cover every real
report, `other` without any explanation is close to useless in the admin
queue, so the one reason that admits "none of the above" is the one
reason that has to come with an explanation. Enforced by a correlated
`CHECK` per table, not left to the request schema alone: the same
"database backstop, not only application validation" standard this
schema already applies everywhere else (e.g. `ck_checkins_visibility_allowed`).

**`sexual_content` is `sensitive_content` instead.** Broader on purpose:
a single narrow category forces a reporter facing something disturbing
but not sexual (graphic injury, self-harm content, animal cruelty) to
either misfile under `sexual_content` or fall back to `other` for
something a specific category should have caught. `sensitive_content`
covers the class these belong to without pretending to enumerate it.

The split follows the one §11 already draws in prose: check-ins and profiles
"carry interpersonal-safety risk," while a venue report "is a data-quality
concern instead." That isn't a cosmetic difference — the two kinds share
almost nothing:

| | `CONTENT_REPORT` | `VENUE_REPORT` |
|---|---|---|
| Reason vocabulary | spam, harassment, hate speech, sensitive content, violence, fake account | wrong address, wrong name, permanently closed, duplicate |
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

**Because `target_id` isn't a real foreign key, checking that it exists
before accepting a report is a service-layer choice, not a database
guarantee, and the service deliberately makes different choices for its
two target types.** Reporting a checkin verifies it first, through the
same `can_view` a like or bookmark already goes through, matching that
precedent. Reporting a user skips the check entirely. `users_select`
carries the blocking guard from earlier in this ADR, so a lookup on the
reported person would go through the same asymmetric policy: if they had
already blocked the reporter, the row would come back invisible and the
report would fail exactly when it matters most. §11 is explicit that a
reported user can always be blocked and reported by the person affected,
independent of and faster than any admin action, so the report path
cannot be allowed to depend on a query the reported person can suppress.

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
- `BLOCK` reuses `FOLLOW`'s shape for storage, so its indexing carries
  over rather than being invented again, even though its access-control
  policy deliberately does not reuse `close_friends_select`'s (see
  above).
- "A blocked profile behaves like a nonexistent one" (PDD §11) becomes a
  property `users_select` itself guarantees, not something every future
  profile-reading call site has to separately remember.

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
- Every read path now carries a bidirectional block check, and enforcing
  it in RLS as well as `can_view` means the check is paid at the database
  layer on `users`, `checkins`, `lists`, `venue_saves`, their likes and
  bookmarks, and `notifications`, not only in application code. That cost
  is real and is the reason `obur-backend`'s roadmap places Phase 10
  before the read-heavy phases (feed, discovery, rankings) rather than
  after them.
- Choosing access-control Option B (above) over Option A means the
  simpler correlated-subquery pattern `close_friends`/`follows` already
  established doesn't extend to `blocks`; every consumer, RLS policy and
  application code alike, depends on one more function
  (`rls_is_blocked_pair`) than it would have under Option A. Accepted
  deliberately: blocking is judged the one relationship where "the
  blocked person's session can never even query the row" is worth that
  extra indirection everywhere it's checked.

**Explicitly unchanged:** admin moderation access is never affected by a
block between two other users — reports must stay reviewable regardless of
who blocked whom (§11). Aggregate ratings are likewise untouched: they are
anonymous, platform-wide numbers with no per-viewer dimension, so a block
has nothing to act on there.
