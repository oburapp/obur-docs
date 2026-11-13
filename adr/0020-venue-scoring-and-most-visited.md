# ADR-0020: Venue Scoring, Score Caching, and Most-Visited Places

**Date:** 2026-11-13
**Status:** Accepted; not yet implemented, see `docs/roadmap.md` Phase 13 in `obur-backend`

## Context

PDD §8 fixes the venue score's procedure: pool the four criteria from
`public` check-ins, apply a volume floor, compute a 95% confidence lower
bound on the mean, and place it in a 9-tier band table, showing only the
label. PDD §13 adds a "personalized history" on the profile: the user's
highest-rated venue per category, computed automatically.

Working through it before building surfaced four gaps:

- The floor is "10 ratings" over a pool where each check-in contributes
  four ratings, so 10 ratings is about three visits, while the PDD's own
  justification argues three visits is too few. Treating a visit's four
  ratings as four independent observations also overstates the sample
  size fourfold and narrows the interval, the very overclaiming the lower
  bound exists to prevent.
- The four criteria are shown individually, but how they are computed is
  unstated.
- Computing the score on every read is cheap for one venue page but not
  for ranking many venues, which Phase 16's Discover will do.
- An automatically computed "highest-rated per category" shown on a
  profile would reveal private check-ins indirectly, and overlaps the
  user-chosen favorites `VENUE_SAVE` already provides.

## Decision

### The sample unit is the check-in

- Each check-in contributes **one** observation: the mean of its four
  ratings. The four criteria stay equally weighted, and every check-in
  still weighs the same.
- **The floor is 10 check-ins.** Below it, the label is *New / Low Data*.
- Above it: `lower bound = x̄ − t(0.95, n−1) × s / √n` over the per-check-in
  means, clipped to [1.0, 4.0], placed in PDD §8's 9-tier table unchanged.

### What counts

- `public`, non-deleted check-ins.
- **A frozen account's check-ins count; a suspended account's do not.**
  Freezing is the user's own temporary choice; suspension is a moderation
  outcome, and content behind it shouldn't shape a public statistic.
- Blocking has no effect: the score is an anonymous platform statistic
  with no per-viewer dimension (ADR-0010).

### The four criteria

Each criterion goes through **the same procedure**: per-check-in values,
the same floor, the same lower bound, the same 9 tiers. The API returns a
tier key per criterion (or none below the floor), and the client draws a
compact indicator from it. The headline and the four indicators reach the
floor at the same moment, since every check-in rates all four.

### The API shows tiers, never numbers

Responses carry the headline tier key, the four criteria tier keys, and
the number of check-ins behind them. Never the mean, the bound, or any raw
figure (PDD §8). Tier keys are stable identifiers; the client localises
them, as with problem types (ADR-0015).

### t critical values without SciPy

One-sided 95% critical values for degrees of freedom 1 to 100 are a
constant table in code, generated once from a reference implementation
and checked in tests against published values. Above 100 the normal value
(1.645) is used; the difference at 100 degrees of freedom is under 1%,
well inside the band widths.

### Scores are cached in Redis, guaranteed fresh by four layers

1. **One central invalidation point.** A database session listener sees
   every change that can move a score, however it was made: a check-in
   created, its ratings or visibility edited, soft- or hard-deleted, and a
   user's status changing (which affects every venue they checked into).
   No service calls invalidation by hand, so no new code path can forget
   to.
2. **Invalidation runs after commit**, never before: invalidating first
   lets a concurrent request recompute from the old data and cache it
   again.
3. **A short TTL** (a named constant, 10 minutes to start) bounds how long
   any missed case could live, and satisfies the rule that nothing is
   cached without expiry.
4. **A test per event** proves the score changes after each one, and a
   test reads Redis directly to prove the key is actually gone.

If Redis is unavailable, the score is computed directly from the database
rather than failing the request.

### "Personalized history" is replaced by most-visited places

- The automatic "highest-rated venue per category" is **removed**. A
  user's own choice of standout venues is what favorites (`VENUE_SAVE` of
  type `favorite`) already are.
- In its place: **the user's five most-visited venues**, one list across
  all categories, counted from non-deleted check-ins, ties broken by the
  most recent visit.
- The list has its own **visibility setting** on the user, with the same
  three tiers as everything else, **defaulting to private**.
- **Counts depend on who is looking.** The owner's own view counts every
  check-in, private ones included. Anyone else's view counts only the
  check-ins they could see themselves: `public`, plus `close_friends` if
  they are on the owner's close friends list. A private check-in is never
  revealed indirectly, the rule every other derived feature already
  follows (venue score, venue verification, hashtag pages, a venue's
  representative photo).

## Rationale

**Per-check-in observations.** The confidence bound is only honest if `n`
counts independent observations. A visit's four ratings are not
independent: one good evening lifts all four. Counting check-ins restores
the interval the PDD intended and makes the floor mean what its own
justification says.

**Same procedure for the criteria.** A plain mean would let two or three
check-ins claim "excellent food" while the headline still says *New / Low
Data*, the overclaiming the whole procedure exists to stop.

**Redis with guarantees, not compute-on-read.** Speed on the venue page
and, later, cheap ranking in Discover matter for the experience. The usual
weakness of a cache, an invalidation someone forgets, is answered
structurally: one listener instead of many call sites, after-commit
timing, a TTL as a backstop, and tests per event.

**Most-visited over highest-rated.** A computed "best" competes with the
favorites a user deliberately chooses; how often someone returns is a
fact about them that favorites can't express, and it keeps the sense of
progress the PDD was after.

**Per-viewer counting.** A user marks each check-in private one by one.
Letting a list reveal them in aggregate would undo those choices, and
would need a warning to be honest. Counting per viewer needs none.

### Alternatives considered

- **Pooling four ratings per check-in, as first written.** Rejected for
  the independence problem above.
- **Plain means for the criteria.** Rejected: overclaims at low volume.
- **Compute on every read.** Simplest and never stale, but costly for
  Discover's ranking. **A stored running aggregate on `venues`** is fast
  but shares the cache's invalidation risk with no TTL to bound it.
- **Keeping the automatic "best per category".** Rejected as above.
- **User-chosen "best per category" showcase.** Rejected: a second
  feature doing what favorites already do.
- **Counting private check-ins for every viewer once the list is
  public.** Rejected: an indirect leak needing a warning to be honest.
- **SciPy for t quantiles.** Rejected: a heavy dependency for a constant
  table.

## Consequences

- PDD §8 changes: the floor is 10 check-ins and the observation is a
  check-in's mean. PDD §6 and §13 change: personalized history becomes
  most-visited places.
- `users` gains a visibility column for the most-visited list.
- Every change that can affect a score must go through the session the
  listener watches. A raw bulk statement that bypasses the ORM session
  would also bypass invalidation; the TTL bounds the damage.
- A missed invalidation can show a stale tier for up to the TTL. Accepted
  as a backstop, not a design path.
- Band cut points, the floor, and the TTL are named constants, so PDD
  §18's calibration is a value change.
