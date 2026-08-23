# Changelog

All notable changes to this repository will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/).

## [Unreleased]

### Added

- Product Design Document (`pdd/obur-pdd.md`)
- Entity-relationship diagram (`diagram/entity-relationship.md`)
- Shared development standards (`CLAUDE.md`)
- ADR index (`adr/README.md`)
- Incident response runbook stub (`runbooks/incident-response.md`)
- ADR-0002: `VENUE.google_places_id` stays in the schema but deliberately
  unpopulated until a real verification/link-out feature needs it
- ADR-0003: venue name search moved from Turkish-only full-text search
  to `pg_trgm` word-similarity matching, so it works across any
  language, partial input, and typos
- ADR-0004: dropped the self-contradictory "contribute to statistics"
  toggle from the check-in share step — `is_public` alone now governs
  both feed visibility and aggregate rating inclusion
- ADR-0005: a check-in's own fields (note, photo, visit date,
  visibility, venue-level ratings) are editable after creation; its
  rated products and their ratings are not — verified against how
  Untappd and MyFitnessPal each handle this
- ADR-0006: check-in/list/saved-venue visibility replaced with a shared
  three-tier model (public / close friends / private); close friends is
  a manually curated subset of followers, not a "followers-only" tier —
  verified against Letterboxd's own close-friends feature as the closest
  real-world comparable. Also covers why likes stay a visible signal
  while bookmarks are always private, kept as separate table pairs.
  Supersedes ADR-0004.
- ADR-0007: `LIST_ITEM` ordering uses fractional indexing instead of an
  integer column, so inserting/moving/removing an item never renumbers
  its neighbors — includes the `COLLATE "C"` requirement found via an
  empirical test against the real database (the default locale-aware
  collation silently breaks the algorithm's byte-ordering assumption)
- ADR-0008: notifications are written synchronously, in the same
  transaction as the event that causes them — no queue, no background
  worker; `read_at` lives on the backend row so read state is
  automatically consistent across every device
- ADR-0009: `VENUE.district` (required going forward), a two-layer
  duplicate check (certain `google_places_id` match vs. the existing
  50m proximity fallback), and a cosmetic-only `is_verified` signal
  (Google match + check-ins, or check-ins + admin review — never a
  human-reviews-everything model, never affects ranking). Supersedes
  ADR-0002 — the "concrete verification feature" it said to wait for.
- ADR-0010: the schema behind blocking and reporting, neither of which
  had a table anywhere despite both being P0 and specified at length in
  §11. `BLOCK` stores direction (like `FOLLOW`) but enforces in both —
  only the blocker may unblock, and the blocked person must never learn
  a block exists, so a symmetric representation doesn't work. Reports
  split into `CONTENT_REPORT` and `VENUE_REPORT`: two tables by concern
  rather than three by target, since the two kinds differ in reason
  vocabulary, admin resolution, and urgency, and since splitting lets
  `VENUE_REPORT` keep a real foreign key (a venue is never deleted)
  while `CONTENT_REPORT.target_id` deliberately can't have one (a
  check-in can be admin-purged, a user row is destroyed by account
  deletion, and a real FK would force cascading away the audit trail or
  blocking the purge). Applies the same `CHECKIN_LIKE`-vs-`NOTIFICATION`
  integrity test this schema has answered before, per case rather than
  picking one pattern for both.
- ADR-0011: the product layer is removed from MVP — `PRODUCT`,
  `GLOBAL_PRODUCT_TYPE` (+ translation), and `CHECKIN_PRODUCT` are
  dropped, and `CHECKIN` gains `rating_taste` for four venue criteria
  in total. Four concerns were raised; the first turned out to be wrong
  and is recorded as such (free-text item names were never the
  aggregation key — `global_type_id` was, against a closed catalog).
  What actually decided it was density: §8 requires 10 ratings before
  any label shows, and at the PDD's own MVP targets a venue accumulates
  ~2–3 check-ins a month, so an individual item would hold roughly two
  ratings after six months. The feature would have cost every user two
  steps on the core action and returned "New / Low Data" either way,
  while `CHECKIN.note` already carries the same information in a form
  people read better. Supersedes ADR-0005, carrying forward its still-live
  half (a check-in's own fields stay editable after creation).
- ADR-0012: migrations never import from `app/`, and reference/catalog data
  belongs to an idempotent seeder rather than a migration. Written after
  removing a seed module under ADR-0011 broke the entire Alembic
  environment — one migration had imported five things from `app/`,
  coupling frozen history to code that changes with every refactor.
  Django documents this exact failure and its exact trigger ("will fail
  in the future when you try to rerun old migrations… when you set up a
  new installation"), which is what the integration suite does every
  session. The seeder half is separately grounded: EF Core renamed its
  migration-embedded seeding away from "data seeding" because it suits
  only static data "for example ZIP codes," and warns that seeding must
  not run from normal app execution because instances would race. Also
  records a second instance already queued — Phase 6's category-catalog
  growth had no path that wasn't a silent no-op — and closes the class
  with a test that scans migrations for `app/` imports.

### Changed

- Entity-relationship diagram: added `VENUE.location` (generated PostGIS
  geography), `USER.role`, and `CHECKIN.deleted_at`; replaced the stale
  `search_vector` reference with a note pointing to ADR-0003
- PDD: `USER` and `CHECKIN` schema blocks updated to match (`role`,
  `deleted_at`, the `CHECKIN_PRODUCT` uniqueness constraint); Check-in
  Flow §10 Step 5 rewritten to describe the single visibility toggle
  instead of the dropped second toggle
- Entity-relationship diagram and PDD §7 Data Model: `CHECKIN.is_public`
  and `LIST.is_public` replaced with the shared `visibility` field;
  added `CLOSE_FRIEND`, `CHECKIN_LIKE`, `CHECKIN_BOOKMARK`, `LIST_LIKE`,
  `LIST_BOOKMARK`, and `NOTIFICATION`; `LIST_ITEM.order` (integer)
  replaced with `position` (fractional-indexing string); `VENUE_SAVE`
  gained a `visibility` field, defaulting to private
- PDD §4, §6, §10, §11: visibility control, feature catalog, the
  check-in share step, and the Social Graph section all rewritten for
  the three-tier model, close friends, and the like/bookmark split
- ADR-0004's status changed to "Superseded by ADR-0006"; its own point
  (feed visibility and aggregate inclusion are one decision) still
  holds, only the tier count changed
- ADR-0002's status changed to "Superseded by ADR-0009"
- PDD §4, §6, §7, §9, §10, §11, §13: added venue verification (cosmetic
  only, no notification — corrected §11's own trigger list, which had
  listed one), cross-venue product ranking, personalized per-user "best
  of" history, and `VENUE.district`/`is_verified`; corrected two
  pre-existing inconsistencies found along the way — §9's badge example
  and §13 both referenced a "first to try this product" concept that
  was never actually decided, only ever "first discoverer" at the venue
  level exists
- Entity-relationship diagram and PDD §7: `VENUE` gained `district` and
  `is_verified`; `google_places_id` documented as unique where not null
- PDD §4, §6, §11, §17: added blocking (absolute, silent, bidirectional
  — closes all visibility including `public`, retroactively purges
  likes/bookmarks/notifications between the two people in both
  directions, anonymizes identity display like "first discoverer"
  rather than deleting the blocked person's own data) and reporting
  (check-ins/profiles, human-reviewed queue with no auto-hide
  threshold, `USER.status` as a new suspension primitive kept separate
  from `USER.role`) — both P0, needed to clear Apple/Google's
  user-generated-content store-review requirements, not optional
  polish. Flagged that the actual Content Policy text and Turkish
  "sosyal ağ sağlayıcı" obligations (Law 5651/7253) both need real
  legal review, not assumed here.
- ADR-0009 extended, and PDD §4, §6, §7, §11, §13 updated: `VENUE.status`
  (`active | closed | unconfirmed`) is dropped entirely in favor of two
  independent, admin-only booleans — `is_active` (closed businesses stay
  visible, shown transparently) and `is_suspended` (admin moderation
  action, hides the venue entirely; its page returns a generic "not
  found," never an explanation, matching how a blocked profile behaves).
  No venue field is ever directly user-editable, including by whoever
  added it — every correction is report-driven and admin-only, on a real
  abuse precedent from an unrelated app's direct-edit feature. Venue
  reports are folded into the existing report/admin-queue mechanism as a
  third, P0 category (alongside check-ins and profiles) rather than the
  stale v2.0 "auto-deactivate on report count" feature it replaces.
- PDD §6, §7, §11 and the entity-relationship diagram: account
  management basics. `USER.username` split into a freely-editable
  `display_name` and a unique, rate-limited `username` handle;
  `USER.status` (`active | frozen | suspended`) added — `frozen` is
  self-service and reversible, `suspended` is admin-only via a report,
  kept separate from `role`; account deletion is permanent (not a
  `status` value, not the soft-delete `CHECKIN.deleted_at` uses) and
  purges all personal content, reusing the existing admin-purge
  mechanism user-triggered instead — the one deliberate exception to
  "historical data is never deleted." `VENUE.added_by` gains
  `ON DELETE SET NULL`, since a venue isn't personal content and
  outlives the account that added it. Delete-account capability is
  required for store compliance (Apple §5.1.1(v), Google Play's
  equivalent).
- PDD §6, §7, §11, §12 and the entity-relationship diagram: added
  `MUTE`, the lighter counterpart to blocking — one-directional, silent,
  and scoped to feed display only (both feed layers, §12); the follow
  relationship, visibility, discoverability, likes, bookmarks, and
  notifications are all untouched, unlike blocking. Not derived from
  `FOLLOW` like `CLOSE_FRIEND` is — a user can mute someone they don't
  follow.
- PDD §4, §9: badges are permanent once earned, never auto-revoked, even
  if the earning condition later becomes false — admin-only manual
  revocation for moderation/fraud cases is the sole exception. Badge
  earning is evaluated synchronously and forward-only (no re-scan job
  needed), consistent with the rest of the stack. `rarity_pct` is the
  one exception needing periodic, not live, computation, since it's a
  percentage over the entire user base — real precedent (Steam's global
  achievement percentage, League of Legends' rank-distribution
  percentile) confirms this is normally a batch/periodic stat, not
  computed per-request; exact cadence is left as an implementation
  decision.
- PDD: new §17 Non-Functional Requirements (Open Decisions renumbered
  17 → 18). First topic covered — offline & draft reliability: a
  check-in must never be silently lost to a bad connection, an MVP
  requirement given check-in is the core action and venue interiors are
  exactly where mobile connections are weakest. Two mechanisms: a
  cross-device `CHECKIN_DRAFT` (§7) for saving in-progress check-ins
  before submission — a separate table, not an `is_draft` flag, so a
  draft can't leak into aggregate/badge/feed queries the way two real
  Phase-4 bugs already showed a forgotten filter can — and
  `CHECKIN.idempotency_key` for safe retries after submission. The
  automatic no-user-action retry-on-reconnect guarantee is mobile-only
  (checking in happens in the moment, at the venue — a mobile action);
  web still gets the draft and idempotency-key safety nets, just not
  silent background retry. Web being a narrower client than mobile is a
  deliberate scope choice, not permission to neglect what it does
  support.
- PDD §17, §18: rate limiting and fake-account resistance. A stricter
  rate-limit tier identified for check-in creation, report submission,
  venue creation, and follow — the actions where repeated abuse does
  real damage (aggregate-rating manipulation, coordinated false
  reporting, spam venues, follow-spam), not just annoyance. Sockpuppet
  risk ranked by actual damage, worst first: aggregate-rating
  manipulation, feed-ranking gaming via fake likes, badge gaming, venue
  verification gaming (least severe — verification was already designed
  cosmetic-only, §13). Deliberately no phone verification, custom SMS
  OTP, or IP tracking at MVP — each considered and set aside for a
  concrete reason (unverified phone fields add no real anti-fraud value;
  self-built OTP trades a flat managed cost for a variable one exposed
  to SMS-pumping fraud; IP correlation, even hashed, is a data-handling
  responsibility outside the team's current experience). Deferred until
  real abuse is observed, added as a new §18 open decision.
- PDD §7, §17: bulk-extraction resistance and signed photo URLs. Read
  endpoints (Discover, search, venue listing) get rate limits too, not
  just write ones; all list responses are paginated with a capped page
  size; no bulk-export endpoint exists, as a standing design principle.
  Separately, `CHECKIN.photo_url` is served as a short-lived signed URL
  for `close_friends`/`private` check-ins — closes a real gap where a
  private check-in's own visibility was enforced on the database row,
  but its photo (living in R2, a separate system) had no equivalent
  protection if its URL ever escaped. Public check-in photos and profile
  avatars are deliberately left unsigned: there's no access boundary
  left to enforce once something is public, and signing would cost real
  CDN-cacheability for no protection gained.
- PDD §13: a venue has no photo of its own — its representative image is
  derived (whichever of its own `public` check-ins has the most likes),
  never uploaded or set by anyone, consistent with "no say for
  businesses." Scoped to `public` check-ins only, since surfacing a
  `close_friends`-visibility check-in's photo on a fully public venue
  page would leak a deliberately restricted share. Computed live, not
  stored on `VENUE`. Also settles how list cards display venues — no
  separate list cover-image feature needed.
- PDD §17, §18: named the governing principle behind every authorization
  decision in this document (`can_view`, `ensure_visible_and_owned`, the
  existence-leak standard, signed photo URLs) — no access path to data
  should exist outside the application's own authorization logic — and
  drew its honest boundary: it only protects the application's own
  access path, not a direct database/backup compromise, which needs a
  separate layer (encryption at rest, network isolation to the backend
  only). Added as a §18 research item: verify what Railway's Postgres
  actually guarantees here before launch, not assumed.
- PDD §17: photo upload standards, applying uniformly to
  `CHECKIN.photo_url` and `USER.avatar_url` — accepted formats
  (JPEG/PNG/HEIC/WebP, HEIC converted server-side), client-side
  compression before upload, a server-side hard size cap as an abuse
  safety net, multiple generated resolutions instead of one fixed size
  (so mobile and desktop each get an appropriately sized image),
  context-specific cropping (avatar square, check-in natural aspect
  ratio), and full EXIF stripping on upload — not just GPS, since other
  embedded metadata can also be identifying, and Obur has no use for any
  of it.
- PDD §17, §18: performance and uptime targets, grounded in published
  external standards rather than picked arbitrarily — Core Web Vitals
  for `obur-web` (LCP < 2.5s, INP < 200ms, CLS < 0.1), a standard
  web-app-tier API latency target (P50/P90/P99: 200ms/500ms/1s) for
  reads, and a deliberate exemption for check-in creation given its
  heavier photo-processing path — justified by published perceived-wait
  research, and ties back to why the "sending" state was already worth
  building. No numeric uptime target at MVP, stated explicitly rather
  than left unaddressed, given Railway's hobby tier has no HA guarantee.
  Accessibility added as a new §18 item — a real long-term concern, not
  addressed or urgent at MVP.
- PDD §13: a venue's representative photo (added earlier this round) is
  selected per-viewer, not globally — corrected from an initial
  "untouched by blocking" framing that wrongly treated it like an
  aggregate rating. A photo is always sourced from one specific person's
  check-in, which puts it in the same category as "first discoverer"
  (already per-viewer under blocking, §11), not the aggregate rating
  (genuinely anonymous, no single attributable person). The selection
  query excludes anyone the viewer has blocked or is blocked by, reusing
  the same exclusion feed content already applies (§12) rather than
  inventing new logic — for a blocked pair this naturally surfaces the
  next-best-liked eligible photo. Scoped to blocking only, not muting
  (muting stays feed-only, §11). The photo is also now clickable through
  to its source check-in — not a "photo by @username" credit (would
  visually compete with the "first discoverer" badge), just the image
  itself linking to a check-in the venue page already lists.
- PDD §8: the aggregate rating label table was a placeholder from early
  in the project and wasn't actually a deterministic procedure — fixed.
  Three real problems: no stated rule for which row wins when a score
  satisfied several rows at once (a 90%+ ratio with too little volume
  for "Excellent" also technically satisfied several lower rows); the
  "positive" metric in one row was never defined as a formula; "scattered
  / mixed" was prose, not a computable condition. Also, four of the
  eight old labels were identical to the input scale's own words
  ("Very good," "Good," "Average," "Bad"), contradicting this section's
  own "deliberately different vocabulary" rule. Replaced with a 9-tier,
  symmetric Favorable/Unfavorable ladder (mirroring how Steam's review
  labels describe review-distribution sentiment rather than issuing a
  verdict, without borrowing its exact wording) computed from a single
  mean-score metric (1.0–4.0) over non-overlapping bands, with an
  explicit volume-floor-then-downgrade procedure so a high score backed
  by thin evidence can't claim more confidence than it earned. Also
  states, for the first time, which rating pool feeds the label on a
  venue page versus a product page. Turkish labels use "Olumlu/Olumsuz,"
  deliberately not a literal translation of the English wording.
- PDD §7, §8, §10, §13 and the entity-relationship diagram: replaced the
  band+volume-gate aggregate rating design (added earlier this round)
  with a statistically grounded procedure after stress-testing revealed
  a real problem — at a 3-rating floor, a venue with 2 "Very Good" and
  1 "Bad" (genuinely decent) could compute to the single worst label.
  Now computes a 95%-confidence lower bound on the mean (the principle
  behind the Wilson score interval Reddit/Yelp use, chosen over a
  Bayesian/IMDb-style approach since it stays anchored to a pool's own
  data rather than diluting toward an unrelated platform average) below
  a 10-rating floor (matching Steam's own "10 reviews minimum" standard)
  — verified this floor behaves sensibly across both a mixed-but-mostly-
  positive and a genuinely polarized sample. Introduces two distinct,
  deliberately separate aggregate levels: a pure product-level score
  (food quality only, the sort key for §13's cross-venue product
  ranking) and a venue-level headline score (blending product ratings
  with venue-criteria ratings into the one number shown at the top of a
  venue's page, matching how comparable apps show one overall score,
  not two). `CHECKIN.rating_service`/`rating_ambiance`/`rating_value`
  changed from optional to required, so every check-in contributes
  equally to the venue-level pool regardless of how many products were
  rated — weighting by check-in thoroughness (more products rated =
  proportionally more influence) is accepted as deliberate, not
  corrected for. Label wording also redesigned: a symmetric 9-tier
  Favorable/Unfavorable ladder (English) and Olumlu/Olumsuz ladder
  (Turkish, not a literal translation), softer than a blunt verdict —
  informed by researching Steam's and Reddit/Yelp's real rating-display
  conventions without copying their wording directly.
- PDD §10, §13 and ADR-0009: three connection gaps closed between
  sections that had been reviewed independently. Venue verification's
  `N`/`M` independent-check-in count is now explicitly `public`-only,
  the same restriction the aggregate rating already had — an
  unstated ambiguity before, since a private check-in is real evidence
  to whoever made it but shouldn't be able to earn a venue its public
  "verified" mark. Check-in Flow (§10) now references `CHECKIN_DRAFT`
  (autosave/resume across Steps 1–4) and the idempotency-key/"sending"
  state (Step 5's Save) — both built earlier this round under §17 but
  never actually surfaced in the flow description a reader would
  actually look at.
- PDD §17: local interaction feedback and celebration restraint. In-flow
  UI actions (rating selection, card transitions, §10 Step 3) never wait
  on the network — the check-in flow only calls the backend once, at
  Step 5 — so these should render essentially instantly, the same
  sub-100ms "feels instant" threshold already cited, applied here to
  local UI state rather than a round-trip; haptic feedback on card
  transitions is scoped mobile-only, since no reliable web equivalent
  exists. Celebration (check-in completion, badge earned, "first
  discoverer") is deliberately reserved for genuinely special moments
  rather than spent on every interaction, consistent with badges being
  real, permanent status (§9) rather than routine game-loop noise — and
  explicitly not manipulative engagement engineering (no streaks, no
  loss-aversion mechanics, no manufactured urgency), a line drawn
  consistent with decisions already made this round, not a new one.
- PDD §7: tightened the `CHECKIN_DRAFT` rating-field comments — they're
  nullable because a draft is incomplete by definition, not because
  they "mirror" `CHECKIN`, which now requires them (§8, §10).
- PDD §5, §6: a real navigation gap closed — `Profile` had no Settings
  destination anywhere in the page hierarchy despite this session
  designing several features that need one (edit profile, freeze/delete
  account, blocked/muted lists, close friends curation). Added a full
  Settings tree (Account, Privacy & Safety, Notifications, Appearance,
  Activity, About/Help) grouped against real precedent (Instagram,
  Letterboxd), plus new Feature Catalog rows for what it surfaced:
  a support/abuse contact page and Content Policy/Terms/Privacy Policy
  pages (P0 — both required by §11's already-established App Store
  baseline, but neither previously had anywhere to live), and P1 rows
  for notification preferences, dark mode, manual language switching,
  and a recent-likes activity view.
- PDD §5, §6, §7, §10, §11, §12 and the entity-relationship diagram:
  added @ mentions and # hashtags. Mentions (`CHECKIN_MENTION`, a
  structured table, not text parsed from `CHECKIN.note`) require mutual
  following — an open "tag anyone" model would reintroduce the
  unwanted-attention risk the platform's no-comment/no-DM stance (§4)
  exists specifically to avoid — and the requirement means blocking
  (which already auto-unfollows both directions) doubles as mention
  protection for free, with no separate rule needed. A mention
  notifies but never overrides the mentioned check-in's own visibility
  — verified against Instagram's actual behavior before writing this
  down (tagging on a private account notifies but doesn't grant
  access), which also turned out to be the more internally consistent
  choice, since the earlier instinct to grant access would have been a
  visibility backdoor. Existing mentions are purged retroactively on
  blocking, the same treatment already given to likes/bookmarks/
  notifications. Hashtags (`HASHTAG`, `CHECKIN_HASHTAG`, `LIST_HASHTAG`)
  are free text, capped at 5 per check-in/list to keep hashtag-based
  discovery pages meaningful, `public`-only when surfaced on a
  hashtag's own discovery page, and stored in a Turkish-aware
  normalized form — the same class of casing bug `LIST_ITEM.position`'s
  `COLLATE "C"` requirement already caught (ADR-0007) would otherwise
  silently split one tag into two. No new moderation path needed for
  either — abuse is reportable as part of the check-in/list it's
  attached to.
- PDD §17, §18: named PostgreSQL Row Level Security (RLS) as the
  concrete mechanism for the database-level layer the Governing
  Principle already called for, directly addressing the failure mode
  behind two real bugs already found this project (a query forgetting
  to call `can_view`). Framed as a development-time discipline, not a
  post-launch retrofit: evaluated per query/endpoint as the backend is
  actually built (deciding which queries need a bypass, like badge
  rarity or admin tooling, is far cheaper to get right once than to
  re-audit everything already written later).
- PDD §7, §11 and the entity-relationship diagram: added `BLOCK`,
  `CONTENT_REPORT`, and `VENUE_REPORT` per ADR-0010. Both features were
  already fully specified in prose and marked P0 — blocking and
  reporting are what Apple's App Store §1.2 and Google Play's UGC policy
  require of any app carrying user-generated content — but neither had a
  table in the data model or the diagram, so there was nothing for the
  backend to build against. Found while rewriting `obur-backend`'s
  roadmap to cover the PDD in full, when the phase meant to implement
  them turned out to have no schema to implement.
- PDD §2, §5, §6, §7, §8, §9, §10, §12, §13, §14, §17 and the
  entity-relationship diagram: the product layer removed per ADR-0011
  (31 tables → 27). §8 is the section that changed most, and it got
  *simpler*: the two-level split (a pure product-level score plus a
  blended venue-level one) collapses to a single venue-level score, and
  the paragraph rationalizing why a check-in rating more items carried
  proportionally more weight is gone — every check-in now contributes
  exactly four values and weighs the same. §10's flow drops from five
  steps to four, with the card-stack interaction kept and repointed at
  the four criteria. §13 loses the product page, the product hierarchy,
  and the cross-venue "best kuşbaşılı pide" ranking, and is retitled
  "Venue Architecture."
- PDD §8: `rating_value` redefined from "the payoff of the overall
  experience" to **value for money**, Turkish label *Değer* → *Fiyat*.
  With taste, service, and ambiance each rated separately, the old
  definition overlapped all three and measured nothing of its own; as a
  price signal it is the only criterion that can distinguish an
  excellent venue from an excellent venue that overcharges.
- PDD §13: Personalized History kept, re-keyed from `GLOBAL_PRODUCT_TYPE`
  to `VENUE.category_id` — "en iyi döner: Develi" becomes "en iyi
  dönerci: Develi". Its stated purpose never depended on the product
  layer, and unlike everything else derived from ratings it needs no
  volume floor at all, since it reports the user's own rating rather
  than a platform statistic. `VENUE_CATEGORY` now backs three readers
  instead of one (Discover filters, §12's Layer-2 signal, this), which
  is noted in the diagram as a consequence worth tracking.
- PDD §6: the product layer added to the v2.0 table rather than deleted
  outright — it is gated on density, not abandoned. At a scale where a
  venue sees hundreds of check-ins the question becomes answerable, and
  the notes accumulated in the meantime are what would show which items
  are worth cataloguing.
