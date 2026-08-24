# Obur — Product Design Document (PDD)

**Version:** 1.0  
**Date:** August 2026  
**Status:** Active  

---

## Table of Contents

1. [Vision and Problem](#1-vision-and-problem)
2. [Market Positioning](#2-market-positioning)
3. [User Classes](#3-user-classes)
4. [Product Decisions and Rationale](#4-product-decisions-and-rationale)
5. [Information Architecture and Navigation](#5-information-architecture-and-navigation)
6. [Feature Catalog](#6-feature-catalog)
7. [Data Model](#7-data-model)
8. [Rating System](#8-rating-system)
9. [Badge and Achievement System](#9-badge-and-achievement-system)
10. [Check-in Flow](#10-check-in-flow)
11. [Social Graph](#11-social-graph)
12. [Main Feed Algorithm](#12-main-feed-algorithm)
13. [Venue Architecture](#13-venue-architecture)
14. [Tech Stack](#14-tech-stack)
15. [Infrastructure and Cost](#15-infrastructure-and-cost)
16. [Go-to-Market Strategy](#16-go-to-market-strategy)
17. [Non-Functional Requirements](#17-non-functional-requirements)
18. [Open Decisions](#18-open-decisions)


---

## 1. Vision and Problem

### Problem

When searching for restaurants or venues, the anonymous mass ratings offered by Google Maps are insufficient for discovery. A star average produced by hundreds of strangers who don't know each other cannot compare to a recommendation from people whose taste a user actually trusts. The younger generation has already started meeting this need through content creators on TikTok and Instagram.

### Solution

A platform that adapts Letterboxd's taste-based social curation model to restaurants, cafes, bars, and similar food and beverage venues. Users build their own profile, log their experiences, and draw recommendations not from an anonymous crowd but from people they trust.

### Positioning

Obur does not compete directly with Google Maps. User behavior has already split into "discovery on social media, verification on Google Maps." Obur takes over the discovery layer and leaves the verification layer to Google Maps.

### Success Criteria

| Metric | Target (MVP) | Measurement Method |
|--------|-------------|---------------|
| Active users | 200 MAU | Backend log |
| Check-ins / user / month | ≥ 4 | PostgreSQL query |
| D7 retention | ≥ 30% | Cohort analysis |
| Profile completion rate | ≥ 60% | At least 1 favorite + avatar |

---

## 2. Market Positioning

### Direct Competitors

| Competitor | Strength | Weakness |
|-------|-----------|-----------|
| Beli (US) | Letterboxd/Strava hybrid model | Match score unreliable, invite system drew complaints |
| Truffle | Instagram integration | No social graph |
| Google Maps | Data breadth, trust | Anonymous crowd rating, fake review problem |
| Foursquare/Swarm | Location-based check-in culture | Active development has stalled |

### Turkey Specifics

Zomato entered Turkey twice and permanently withdrew in 2021. Local players like Gurme Haritası and Damak don't offer a social graph. Fake review complaints are widespread on Google Maps.

### Differentiating Decisions

- User curation takes precedence over anonymous mass rating
- Businesses are given no say whatsoever
- Four separate venue criteria — taste, service, ambiance, and value — instead of one undifferentiated star, so "the food is great but you're paying for the view" is something the rating can actually say
- Badge system for discovery status

---

## 3. User Classes

### Enthusiast (Core User)

People who are enthusiastic about food and care about the social credit of profile curation. They actively use every feature of the platform, check in regularly, create lists, and carry discovery badges. They are the first target audience and drive the engine of organic growth.

### Follower (Casual User)

The broad audience that joins the platform organically by following Enthusiasts. Check-in production is less intense; discovery and social feed consumption dominate.

### Visitor (New / Not-yet-following User)

May have an account but hasn't followed anyone yet. The main feed algorithm (see Section 12) does not show this user an empty screen; algorithmic content kicks in as the cold-start solution. Conversion goal: follow at least 3 users.

---

## 4. Product Decisions and Rationale

This section documents the debatable decisions made during the product design process and their rationale.

### No Say for Businesses

The root cause of the trust crisis at Google Maps and Yelp is business interference. Obur's value proposition is built on "the opinion of people I trust"; a business response right would erode that trust. Technically, a business verification process would also create disproportionate workload at the bootstrap stage.

### One-Directional Follow Model

A friendship model requiring mutual approval increases cold-start friction. The asymmetric follow model proven by Letterboxd and Untappd both eases discovery and creates an influencer dynamic.

### No Comment / DM Feature

Free-text comments carry moderation overhead and toxic-content risk. DMs, combined with the asymmetric follow model, would create an unwanted-message vector. Letterboxd has never added DMs despite years of user requests. The same decision applies to Obur.

### Visibility: Three Tiers, Not a Public/Private Toggle

Check-ins, lists, and saved venues share one three-tier visibility model: **public**, **close friends**, or **private**. Check-ins and lists default to public, to encourage content creation and keep the feed alive; saved venues default to private, since saving a venue ("been here" / "want to go" / "favorite") is a personal tracking action first, not something assumed to be shared.

A "followers-only" tier was considered and rejected: Obur's follow model is one-directional with no approval step (see §11), so "followers-only" would mean "visible to literally anyone who chooses to follow" — not a real privacy boundary. **Close friends** solves this instead: a manually curated allow-list the user builds by hand from among the people who already follow them, the same model Letterboxd uses for its own diary/review privacy. It requires deliberate action from the owner, not just a follow click from the viewer, and it's revoked automatically if the underlying follow relationship ever ends. See [ADR-0006](https://github.com/oburapp/obur-docs/blob/main/adr/0006-three-tier-visibility-and-close-friends.md).

### Business Hours Out of Scope

Business hours are operational data with no bearing on the discovery experience Obur provides. Stale hours that must be kept up to date send users to a closed venue and cost the platform trust. Verification is left to Google Maps.

### Venue Verification Is Cosmetic, Not a Quality Gate

A verified checkmark on a venue means exactly one thing — "this location definitely exists here" — never a judgment about how good it is, and it never affects search ranking or discoverability. Two independent, zero-human-labor-in-the-common-case signals confirm it: a Google Places match corroborated by real check-in activity, or, for venues Google hasn't indexed, enough independent check-ins to warrant a quick admin look. A fully manual, admin-reviews-everything model was rejected outright — it doesn't scale for a two-person team. See [ADR-0009](https://github.com/oburapp/obur-docs/blob/main/adr/0009-venue-discovery-enrichment.md).

### Venue Editing Is Report-Only, Never Direct

No user can edit a venue's fields directly, not even whoever added it. Every correction — wrong address, wrong name, permanently closed, duplicate — goes through the same reporting mechanism as check-ins and profiles (§11), reviewed by an admin before anything changes. Direct community editing was considered and rejected on a real precedent: an unrelated app that let users directly edit street-animal location pins had that feature abused to relocate or delete other people's pins maliciously. A venue that has genuinely closed isn't hidden or deleted — `is_active` is set to `false` and it stays visible, shown transparently and grayed out, the same "historical data is never deleted" principle already applied elsewhere (§7). Suspension is a separate, harsher concern: `is_suspended`, set only by an admin acting on a report, hides a venue completely — its own page returns a generic "not found" rather than an explanation, the same "hidden must be indistinguishable from nonexistent" principle already applied to blocking below. See §13.

### Badges Are Permanent, Not Live Status

A badge, once earned, is never automatically revoked — even if the condition that earned it later becomes false (deleting a check-in that had crossed a threshold, for instance). It's worded and treated as a record of something that happened, not a claim about the present, the same way a Steam achievement stays unlocked forever regardless of later account activity. This was a deliberate choice over dynamic re-evaluation, which the Philosophy's "proof of status" framing might otherwise suggest: permanence matches real user expectation (an earned trophy shouldn't quietly disappear) and removes an entire class of architecture (no re-scan job needed, only forward evaluation at the moment a new action might cross a threshold). Admin moderation is the one exception — a badge resting on fraudulent activity can be revoked by hand, the same override admins retain everywhere else in this document. See §9.

### Blocking Is Harsher Than Any Privacy Setting, On Purpose

A privacy tier (§11) is about who a user chooses to share with going forward. Blocking is about safety — removing a specific person from the relationship entirely, retroactively as well as going forward. It closes visibility completely (including `public`), purges past likes/bookmarks/notifications between the two people in both directions, and is silent — the blocked person is never told, and a blocked account behaves exactly like a nonexistent one to them. Considered against a lighter option (just hide future content, leave the historical record alone) and rejected: for interpersonal-safety tooling specifically, leaving old traces behind undermines the point of the feature.

---

## 5. Information Architecture and Navigation

### Navigation Bar (5 tabs)

```
[ Feed ] [ Discover ] [ + ] [ Notifications ] [ Profile ]
```

The middle tab (+) doesn't open a page; it opens a modal offering check-in or list creation. Per Fitts's Law, the most frequently used action sits in the most reachable position.

### Page Hierarchy

```
App
├── Feed
│   └── Search (returns check-ins)
├── Discover
│   ├── Search (returns venue / user / list / hashtag)
│   ├── Hashtag page (§12) — every public check-in/list carrying it
│   └── Map view (opened from list or discover)
├── + Modal
│   ├── Check-in creation flow (4 steps)
│   └── List creation
├── Notifications
├── Profile
│   ├── Check-ins tab
│   ├── Favorites tab
│   ├── Lists tab
│   ├── Achievements tab
│   ├── Map tab (venues visited)
│   └── Settings (gear icon)
│       ├── Account
│       │   ├── Edit profile (display_name, username, avatar, bio — §7)
│       │   └── Account (email/password via Clerk; freeze; delete — §6, §7)
│       ├── Privacy & Safety
│       │   ├── Close friends (curate list — §11)
│       │   ├── Blocked users (§11)
│       │   └── Muted users (§11)
│       ├── Notifications (per-trigger preferences — §11)
│       ├── Appearance
│       │   ├── Dark mode
│       │   └── Language (`locale` — §7)
│       ├── Activity (recent likes, across check-ins and lists)
│       └── About / Help
│           ├── Content Policy, Terms, Privacy Policy (§11, §18)
│           └── Support / abuse contact — App Store §1.2 baseline (§11)
└── Venue page
    ├── Check-ins tab
    └── Photos tab
```

### Search Behavior

The same search box returns different results depending on the current page:

- **Search on Feed:** searches among check-ins from followed users
- **Search on Discover:** returns venues, users, lists, and hashtags (typing `#` scopes directly to hashtags); can be narrowed with filter chips

Tapping an `@mention` anywhere navigates straight to that person's profile; tapping a `#hashtag` anywhere navigates to its hashtag page (§12).

---

## 6. Feature Catalog

### MVP (v1.0)

| Feature | Description | Priority |
|---------|----------|---------|
| Sign up / Sign in | Email + social login (Clerk) | P0 |
| Edit profile | `display_name` freely editable; `username` unique, rate-limited changes (§7) | P0 |
| Delete account | Permanent — purges all personal content (check-ins, lists, saves); required for store compliance (Apple §5.1.1(v), Google Play's equivalent account-deletion requirement) (§7) | P0 |
| Freeze account | Self-service, reversible — logging back in reactivates; distinct from admin suspension (§7, §11) | P1 |
| Follow users | One-directional, no approval | P0 |
| Block users | Absolute, silent, bidirectional — closes visibility entirely and purges past likes/bookmarks/notifications between the two people (§11) | P0 |
| Report content / accounts / venues | Check-in, profile, or venue — human-reviewed queue, no auto-hide threshold; a venue report is the only way its details ever change after creation, no direct user-editing (§11, §13) — App Store §1.2 baseline | P0 |
| Create check-in | 4-step flow | P0 |
| Feed | Chronological check-ins and lists from followed users | P0 |
| Discover | Venue / user / list search, city filter | P0 |
| Venue page | Aggregate rating, check-ins, photos | P0 |
| Profile | Check-ins, favorites, lists, achievements | P0 |
| Badge system | Bronze / silver / gold tiers, rarity display | P0 |
| List creation | User curation, map display, freely reorderable | P1 |
| Save venue | Been / want to go / favorite; private by default, owner may share | P1 |
| Close friends | Manually curated subset of followers, grants access to "close friends"-visibility content | P1 |
| Mute users | One-directional, silent, feed-only — the relationship, visibility, and discoverability are all untouched, unlike blocking (§11) | P1 |
| Like | Check-ins and lists; a visible social signal | P1 |
| Bookmark | Private save-for-later on a check-in or list; never a social signal, never shown to anyone else | P1 |
| Map view | Profile- and list-based | P1 |
| Notifications | Like, follow, badge — deliberately *not* venue verification, see below | P1 |
| Visibility control | Public / close friends / private, per check-in, list, or saved venue | P1 |
| Travel mode | Manual city selector | P1 |
| Venue verification | Cosmetic "this location definitely exists" signal — Google match + check-ins, or check-ins + admin review; never affects ranking, no notification (§13) | P1 |
| Support / abuse contact page | Published contact info, in-app and in the store listing — App Store §1.2 baseline, non-negotiable (§11) | P0 |
| Content Policy / Terms / Privacy Policy pages | Hosts the legal text itself, which needs review before publishing (§11, §18) — the page existing is P0 even though the final text is pending | P0 |
| Notification preferences | Per-trigger opt-out (follow, like, badge — §11) | P1 |
| Dark mode | App-wide theme toggle | P1 |
| Language switching | Manual override of `locale` (§7); TR/EN both MVP | P1 |
| Recent likes view | A user's own like history, across check-ins and lists | P1 |
| @ Mentions | Mutual-followers only, notifies but never overrides visibility (§11) | P1 |
| # Hashtags | Free text, up to 5 per check-in/list, own discovery page, public-only (§11, §12) | P1 |
| Personalized history | Profile shows the user's own highest-rated venue per venue category ("en iyi dönerci: X") — see §13 | P1 |
| Share cards | Branded, shareable image cards for a check-in/list/venue, with low-friction one-tap sharing to Instagram/WhatsApp/etc. — mostly client-side, deferred until client work starts | P1 |

### v2.0 (Next Release)

| Feature | Description |
|---------|-------------|
| "You Might Like" shelf | Venue suggestions based on taste overlap; not enabled until enough data has accumulated |
| Product layer | Per-item logging and rating within a check-in, and the cross-venue "where's the best döner" ranking built on it. Removed from MVP and gated on density, not abandoned — at MVP volume no item can reach §8's rating floor, so the feature would cost two steps on the core action and return no label. See [ADR-0011](https://github.com/oburapp/obur-docs/blob/main/adr/0011-drop-product-layer-four-venue-criteria.md) |
| Profile suggestions | User suggestions based on shared taste and followers |
| Expansion to a second city | Same "enthusiast first, then organic growth" playbook |

---

## 7. Data Model

### Tables and Fields

```sql
USER
  id                UUID PK
  auth_provider     VARCHAR                  -- "clerk" today; not hardcoded elsewhere
  auth_provider_id  VARCHAR                  -- that provider's user ID
                                              -- UNIQUE (auth_provider, auth_provider_id)
  display_name  VARCHAR                  -- shown everywhere; freely editable, no
                                          -- uniqueness constraint
  username      VARCHAR UNIQUE           -- the handle; edits rate-limited (see §6) —
                                          -- unlike display_name, this is what search,
                                          -- mentions, and profile URLs key off of
  username_changed_at TIMESTAMPTZ        -- NULL until the handle is first changed;
                                          -- what the rate limit above measures
                                          -- against. Kept on the row rather than in
                                          -- a cache, since the window spans weeks
                                          -- and a cache flush must not grant a fresh
                                          -- allowance
  email         VARCHAR UNIQUE
  bio           TEXT
  avatar_url    VARCHAR
  city          VARCHAR
  country_code  CHAR(2)                  -- ISO 3166-1 alpha-2: "TR", "US", "DE"
  locale        VARCHAR DEFAULT 'tr'     -- BCP 47: "tr", "en", "de" — UI language
  timezone      VARCHAR                  -- IANA: "Europe/Istanbul", "America/New_York"
  role          VARCHAR DEFAULT 'user'   -- user | admin — never settable via any
                                          -- user-facing endpoint or the Clerk webhook
  status        VARCHAR DEFAULT 'active' -- active | frozen | suspended — standing, not
                                          -- permission; kept separate from role.
                                          -- frozen: self-service, reversible by the
                                          -- user just logging back in. suspended:
                                          -- admin-only, via a report (§11) — never
                                          -- user-reversible.
  created_at    TIMESTAMPTZ

FOLLOW
  follower_id   UUID FK → USER
  following_id  UUID FK → USER
  created_at    TIMESTAMPTZ
  PRIMARY KEY (follower_id, following_id)
                                          -- CHECK (follower_id != following_id)

CLOSE_FRIEND
  user_id       UUID                     -- the curator
  friend_id     UUID                     -- one of user_id's followers, manually added
  created_at    TIMESTAMPTZ
  PRIMARY KEY (user_id, friend_id)
  FOREIGN KEY (friend_id, user_id) REFERENCES FOLLOW (follower_id, following_id)
    ON DELETE CASCADE                    -- friend_id must currently follow user_id;
                                          -- row is auto-removed the moment that
                                          -- follow is undone — see ADR-0006

MUTE
  user_id       UUID FK → USER          -- the muting user
  muted_id      UUID FK → USER
  created_at    TIMESTAMPTZ
  PRIMARY KEY (user_id, muted_id)
                                          -- CHECK (user_id != muted_id)
                                          -- not derived from FOLLOW — a user can be
                                          -- muted whether or not they're followed,
                                          -- see §11

BLOCK
  blocker_id    UUID FK → USER ON DELETE CASCADE
  blocked_id    UUID FK → USER ON DELETE CASCADE
  created_at    TIMESTAMPTZ
  PRIMARY KEY (blocker_id, blocked_id)
                                          -- CHECK (blocker_id != blocked_id)
                                          -- index on blocked_id, for the
                                          -- reverse-direction lookup
                                          -- Stored directionally, enforced
                                          -- bidirectionally: every visibility
                                          -- query asks whether a row exists in
                                          -- either direction, but only the
                                          -- blocker may unblock, and the blocked
                                          -- person is never told a block exists
                                          -- — both need the direction. Same shape
                                          -- as FOLLOW. See ADR-0010

VENUE_CATEGORY
  id            UUID PK
  slug          VARCHAR UNIQUE           -- language-independent reference: "food", "cafe", "bar"
  parent_id     UUID FK → VENUE_CATEGORY

VENUE_CATEGORY_TRANSLATION
  category_id   UUID FK → VENUE_CATEGORY
  locale        VARCHAR                  -- "tr", "en", "de"
  name          VARCHAR                  -- "Yiyecek", "Food", "Essen"
  PRIMARY KEY (category_id, locale)

VENUE
  id            UUID PK
  name          VARCHAR
  lat           FLOAT
  lng           FLOAT
  address_note  TEXT                     -- free text: "3rd floor of the mall"
  google_places_id VARCHAR               -- UNIQUE where not null — see ADR-0009
  added_by      UUID FK → USER ON DELETE SET NULL  -- the venue outlives its adder;
                                          -- a deleted account leaves this null, not
                                          -- a dangling reference — see §7 Key Design
                                          -- Decisions, account deletion
  category_id   UUID FK → VENUE_CATEGORY
  city          VARCHAR                  -- "Istanbul", "New York", "Berlin"
  district      VARCHAR                  -- "Kadıköy", "Beşiktaş" — sub-city scope for
                                          -- discovery/badges/rankings; required for every
                                          -- venue from ADR-0009 forward, nullable only for
                                          -- venues that predate it
  country_code  CHAR(2)                  -- ISO 3166-1 alpha-2: "TR", "US", "DE"
  timezone      VARCHAR                  -- IANA: "Europe/Istanbul" — for city-based filtering
  is_active     BOOLEAN DEFAULT TRUE     -- false = business has closed; stays visible,
                                          -- shown transparently as inactive — see §13
  is_verified   BOOLEAN DEFAULT FALSE    -- cosmetic only, never affects ranking — see ADR-0009
  is_suspended  BOOLEAN DEFAULT FALSE    -- admin moderation action; true = fully hidden,
                                          -- venue page returns "not found" — see §13
  created_at    TIMESTAMPTZ

VENUE_SAVE
  id            UUID PK
  user_id       UUID FK → USER
  venue_id      UUID FK → VENUE
  type          VARCHAR                  -- visited | wishlist | favorite
  visibility    VARCHAR DEFAULT 'private' -- public | close_friends | private
                                          -- UNIQUE (user_id, venue_id, type)
  created_at    TIMESTAMPTZ

CHECKIN
  id            UUID PK
  user_id       UUID FK → USER
  venue_id      UUID FK → VENUE
  rating_taste    SMALLINT NOT NULL     -- 1-4, required — food and drink quality.
                                          -- The only field measuring the food itself;
                                          -- added when the product layer was removed,
                                          -- see ADR-0011
  rating_service  SMALLINT NOT NULL     -- 1-4, required
  rating_ambiance SMALLINT NOT NULL     -- 1-4, required
  rating_value    SMALLINT NOT NULL     -- 1-4, required — value for money. The only
                                          -- price dimension: an excellent venue that
                                          -- overcharges scores well on the other
                                          -- three and can only be marked down here
  note          TEXT
  photo_url     VARCHAR                 -- served as a short-lived signed URL when
                                          -- visibility != public; a permanent link
                                          -- otherwise — see §17
  visibility    VARCHAR DEFAULT 'public' -- public | close_friends | private — see ADR-0006
  visited_at    DATE                    -- visit date entered by the user; may not
                                          -- be in the future, checked against the
                                          -- visitor's own visited_tz, not the server's
  visited_tz    VARCHAR                 -- IANA timezone at time of visit — used for badge calculation
  deleted_at    TIMESTAMPTZ             -- NULL unless soft-deleted; a check-in already
                                          -- factored into a badge or aggregate must
                                          -- never be truly deleted (see ADR-0011,
                                          -- which supersedes ADR-0005)
  idempotency_key VARCHAR               -- client-generated; UNIQUE (user_id,
                                          -- idempotency_key) — a retried submission
                                          -- with the same key returns this row
                                          -- instead of creating a duplicate — see §17
  created_at    TIMESTAMPTZ             -- log timestamp (UTC)

CHECKIN_DRAFT
  id            UUID PK
  user_id       UUID FK → USER
  venue_id      UUID FK → VENUE          -- nullable — venue may not be chosen yet
  rating_taste    SMALLINT               -- nullable — a draft is incomplete by
                                          -- definition, unlike CHECKIN's own
                                          -- required version of this field
  rating_service  SMALLINT               -- nullable, same reason
  rating_ambiance SMALLINT               -- nullable, same reason
  rating_value    SMALLINT               -- nullable, same reason
  note          TEXT
  photo_url     VARCHAR
  visited_at    DATE
  updated_at    TIMESTAMPTZ             -- last autosave, drives cross-device sync
  created_at    TIMESTAMPTZ
                                          -- deliberately not a CHECKIN.is_draft flag —
                                          -- a separate table makes it structurally
                                          -- impossible for a draft to leak into
                                          -- aggregate/badge/feed queries; see §17.
                                          -- Promoted to a real CHECKIN row (and
                                          -- deleted) on final submit

CHECKIN_LIKE
  user_id       UUID FK → USER
  checkin_id    UUID FK → CHECKIN
  created_at    TIMESTAMPTZ
  PRIMARY KEY (user_id, checkin_id)

CHECKIN_BOOKMARK
  user_id       UUID FK → USER
  checkin_id    UUID FK → CHECKIN
  created_at    TIMESTAMPTZ
  PRIMARY KEY (user_id, checkin_id)      -- private; no count exposed anywhere

CHECKIN_MENTION
  id            UUID PK
  checkin_id    UUID FK → CHECKIN
  mentioned_user_id UUID FK → USER       -- UNIQUE (checkin_id, mentioned_user_id)
  created_at    TIMESTAMPTZ
                                          -- only creatable between mutual followers
                                          -- (author follows mentioned_user_id AND
                                          -- vice versa) — see §11. A structured
                                          -- table, not text parsed at render time,
                                          -- so mention creation, the notification
                                          -- it triggers, and its retroactive purge
                                          -- on blocking (§11) can all be enforced
                                          -- the same way CHECKIN_LIKE's are

HASHTAG
  id            UUID PK
  tag           VARCHAR UNIQUE            -- normalized form — see §7 Key Design
                                          -- Decisions, Turkish-aware normalization
  created_at    TIMESTAMPTZ

CHECKIN_HASHTAG
  checkin_id    UUID FK → CHECKIN
  hashtag_id    UUID FK → HASHTAG
  PRIMARY KEY (checkin_id, hashtag_id)   -- max 5 per check-in, enforced at the
                                          -- application layer — see §12

LIST
  id            UUID PK
  user_id       UUID FK → USER
  title         VARCHAR
  description   TEXT
  visibility    VARCHAR DEFAULT 'public' -- public | close_friends | private
  created_at    TIMESTAMPTZ

LIST_ITEM
  id            UUID PK
  list_id       UUID FK → LIST
  venue_id      UUID FK → VENUE          -- UNIQUE (list_id, venue_id)
  position      VARCHAR COLLATE "C"      -- fractional-indexing key — see ADR-0007
  created_at    TIMESTAMPTZ

LIST_LIKE
  user_id       UUID FK → USER
  list_id       UUID FK → LIST
  created_at    TIMESTAMPTZ
  PRIMARY KEY (user_id, list_id)

LIST_BOOKMARK
  user_id       UUID FK → USER
  list_id       UUID FK → LIST
  created_at    TIMESTAMPTZ
  PRIMARY KEY (user_id, list_id)         -- private; no count exposed anywhere

LIST_HASHTAG
  list_id       UUID FK → LIST
  hashtag_id    UUID FK → HASHTAG
  PRIMARY KEY (list_id, hashtag_id)      -- max 5 per list, same rule as CHECKIN_HASHTAG

NOTIFICATION
  id            UUID PK
  user_id       UUID FK → USER           -- recipient
  type          VARCHAR                  -- new_follower | checkin_like | list_like | mention
  actor_id      UUID FK → USER, nullable -- who did it
  target_type   VARCHAR                  -- checkin | list | user
  target_id     UUID                     -- not a real FK — see ADR-0008
  read_at       TIMESTAMPTZ, nullable
  created_at    TIMESTAMPTZ

BADGE
  id            UUID PK
  slug          VARCHAR UNIQUE           -- language-independent reference: "first-discoverer"
  tier          VARCHAR                  -- bronze | silver | gold
  rarity_pct    FLOAT                   -- updated periodically, shown on profile

BADGE_TRANSLATION
  badge_id      UUID FK → BADGE
  locale        VARCHAR                  -- "tr", "en"
  name          VARCHAR                  -- "İlk Keşfeden", "First Discoverer"
  description   TEXT
  PRIMARY KEY (badge_id, locale)

USER_BADGE
  user_id       UUID FK → USER
  badge_id      UUID FK → BADGE
  is_pinned     BOOLEAN DEFAULT FALSE
  earned_at     TIMESTAMPTZ
  PRIMARY KEY (user_id, badge_id)

CONTENT_REPORT
  id            UUID PK
                                          -- an interpersonal-safety report on a
                                          -- check-in or a profile — see §11
  reporter_id   UUID FK → USER ON DELETE SET NULL
                                          -- a report is a moderation record, not
                                          -- personal content: it survives the
                                          -- reporter's account deletion with the
                                          -- identity dropped, like VENUE.added_by
  target_type   VARCHAR                  -- checkin | user
  target_id     UUID                     -- deliberately not a real FK: a CHECKIN
                                          -- can be admin-purged and a USER row is
                                          -- destroyed by account deletion, so a
                                          -- real FK would force a choice between
                                          -- cascading the audit trail away and
                                          -- blocking the purge — see ADR-0010
  reason        VARCHAR                  -- spam | harassment | hate_speech |
                                          -- sexual_content | violence |
                                          -- fake_account | other
  status        VARCHAR DEFAULT 'pending' -- pending | dismissed | actioned;
                                          -- drives the admin queue, never
                                          -- accumulates toward an automatic
                                          -- action — see §11
  resolved_by   UUID FK → USER, nullable -- the admin who acted
  resolved_at   TIMESTAMPTZ, nullable
  created_at    TIMESTAMPTZ
                                          -- UNIQUE (reporter_id, target_type,
                                          -- target_id)

VENUE_REPORT
  id            UUID PK
                                          -- a data-quality report on a venue —
                                          -- see §11 and §13
  reporter_id   UUID FK → USER ON DELETE SET NULL
  venue_id      UUID FK → VENUE          -- a real FK, unlike CONTENT_REPORT's
                                          -- target: a VENUE row is never deleted
                                          -- (is_active / is_suspended instead), so
                                          -- the target's existence is guaranteed
  reason        VARCHAR                  -- wrong_address | wrong_name |
                                          -- permanently_closed | duplicate | other
  status        VARCHAR DEFAULT 'pending' -- pending | dismissed | actioned
  resolved_by   UUID FK → USER, nullable
  resolved_at   TIMESTAMPTZ, nullable
  created_at    TIMESTAMPTZ
                                          -- UNIQUE (reporter_id, venue_id)
                                          -- The only path by which any venue field
                                          -- ever changes after creation — there is
                                          -- no direct user-editing, see §13
```

### Key Design Decisions

**Venue identity is built on coordinates.** The name can change, but coordinates are stable data. When a new venue is added, existing venues within a 50-meter radius are checked for duplicate detection.

**A check-in rates the venue on four criteria, and nothing below the venue.** Taste, service, ambiance, and value are entered once per check-in and are all required. There is deliberately no per-item layer: an earlier design logged and rated each product within a check-in, and it was removed because no item could reach §8's rating floor at this platform's scale, so it cost two steps on the core action and returned no label. What was eaten belongs in `note`, which a reader gets more from anyway. `rating_taste` is what carries food quality now, and `rating_value` is the only price dimension — the rest of the criteria say nothing about what it cost. See [ADR-0011](https://github.com/oburapp/obur-docs/blob/main/adr/0011-drop-product-layer-four-venue-criteria.md).

**Historical data is never deleted.** If a venue closes, `is_active` becomes `false` — it stays fully visible, shown transparently as inactive, and its check-ins remain as an archive. This is a different state from `is_suspended`, an admin moderation action that hides a venue entirely (see §13); neither is user-settable, both change only through a report an admin acts on.

**Account deletion is the one deliberate exception to "historical data is never deleted," and it's permanent, not soft.** Unlike a single check-in (`CHECKIN.deleted_at`, recoverable in principle, only an admin can truly purge one), deleting an account purges everything personal to it outright: every check-in and its `CHECKIN_LIKE`/`CHECKIN_BOOKMARK`/`CHECKIN_MENTION` rows, every list and `LIST_ITEM`, every `VENUE_SAVE` — the same permanent-purge mechanism an admin takedown already uses, just user-triggered instead of admin-triggered. Aggregate ratings and badges derived from that content simply recompute without it; nothing needs to leave a placeholder behind. The one exception within the exception: a `VENUE` the account added is not personal content — it's a shared resource other users rely on — so it survives with `added_by` set to `NULL` rather than being deleted or orphaned. Chosen deliberately over a softer "anonymize but keep the content" approach (the model blocking uses for display, §11): once a user asks to be forgotten, the data itself is gone, not just its attribution.

**`HASHTAG.tag` is normalized with Turkish-aware casing, not a naive lowercase.** Turkish's dotted/dotless İ-I distinction (`İstanbul`.lower() and `Istanbul`.lower() diverge under a locale-naive transform) means `#AnadoluMutfağı` and `#anadolumutfağı` could silently end up as two different rows instead of one if normalization doesn't account for it — the same class of subtle text-processing bug `LIST_ITEM.position`'s `COLLATE "C"` requirement already caught elsewhere in this schema (ADR-0007). Locale-aware casing (or a Turkish-specific case-folding table) is required at write time, not left to whatever the database's default collation happens to do.

**A mention notifies but never overrides visibility.** `CHECKIN_MENTION` can only be created between mutual followers (§11) — a real, established relationship, not an open door to tag anyone. Even so, being mentioned in a `private` or `close_friends` check-in doesn't grant the mentioned person access to see it; they get a notification, not a visibility exception. Extending access this way would be a backdoor around the same visibility system this document has otherwise kept airtight (existence-leak, signed photo URLs, §17) — a mention is a pointer, not a permission grant.

**`visited_at` and `created_at` are kept separate.** A user may log last week's meal today. Badge calculations are based on `visited_at`.

**One shared `visibility` field (public / close friends / private) governs CHECKIN, LIST, and VENUE_SAVE alike, instead of a separate privacy concept per resource.** A "followers-only" tier was rejected — Obur's follow model requires no approval, so it would grant access to literally anyone who chooses to follow. `CLOSE_FRIEND` is a manually curated allow-list instead, built from a user's existing followers, revoked automatically the moment the underlying follow ends. See ADR-0006.

**Likes are a visible signal; bookmarks are private — two separate concepts, two separate table pairs.** `CHECKIN_LIKE`/`LIST_LIKE` are public counts. `CHECKIN_BOOKMARK`/`LIST_BOOKMARK` are personal save-for-later notes nobody else can ever see — there's no bookmark count anywhere in the product. See ADR-0006.

**`BLOCK` is stored directionally but enforced bidirectionally.** The row records who blocked whom, exactly like `FOLLOW`; every visibility query then asks whether a row exists in *either* direction. Storing a symmetric pair instead would have been simpler but loses two things the feature actually needs: only the blocker may unblock (§11), and the blocked person must never learn a block exists — both require knowing which way it runs. See [ADR-0010](https://github.com/oburapp/obur-docs/blob/main/adr/0010-blocking-and-reporting-schema.md).

**Reports are split into two tables by concern, not three by target.** `CONTENT_REPORT` (check-ins and profiles) and `VENUE_REPORT` are separate because §11 already treats them as different things: different reason vocabularies, different admin resolutions, and different urgency — an unreviewed harassment report is ongoing harm, a wrong address can wait. The split also lets each keep the strongest integrity it honestly can. `VENUE_REPORT.venue_id` is a real foreign key, because a `VENUE` row is never deleted; `CONTENT_REPORT.target_id` deliberately isn't, because a check-in can be admin-purged and a user row is destroyed outright by account deletion — a real foreign key there would force a choice between cascading away the record of *why* something was removed and blocking the purge itself. This is the same `CHECKIN_LIKE`-vs-`NOTIFICATION` question the schema has answered before, applied per case rather than picking one pattern for both. See [ADR-0010](https://github.com/oburapp/obur-docs/blob/main/adr/0010-blocking-and-reporting-schema.md).

**LIST_ITEM ordering uses fractional indexing, not an integer column.** Reordering, inserting, or removing an item writes only that one row — no renumbering of neighboring items regardless of list size. See ADR-0007.

**NOTIFICATION rows are written synchronously, in the same transaction as the event that causes them.** No queue, no background worker. `read_at` lives on the backend row, so read state is automatically consistent across every device a user is signed into. See ADR-0008.

**Global-ready design principles.** All timestamps are stored as UTC; the `visited_tz` field records the user's local timezone at the time of the visit and is used in badge calculations. User-facing labels (`VENUE_CATEGORY`, `BADGE`) live in translation tables; adding a new language to the system only requires adding the relevant translation rows. `slug` fields provide a language-independent reference: the code uses `"food"`, and the display resolves independently of language. `country_code` follows ISO 3166-1 alpha-2, `locale` follows BCP 47, `timezone` follows the IANA standard.

**Auth identity is provider-agnostic by field name.** `USER.auth_provider` ("clerk" today) and `auth_provider_id` are deliberately not Clerk-specific field names. Code outside the auth module never sees a provider-specific type — only this column pair. Switching or adding an auth provider later means new rows, not a rename migration. Kept in sync via the provider's webhooks (not just at first login), since identity fields can change on the provider's side independently of Obur.

---

## 8. Rating System

### User Input (the four CHECKIN rating fields)

Four options, no neutral option:

| Value | Label |
|-------|-------|
| 4 | Very good |
| 3 | Good |
| 2 | Average |
| 1 | Bad |

An even number of options with no neutral choice pushes the user to pick a real side. This structurally reduces the rating inflation seen in star systems (everyone giving 4-5).

### Venue Criteria (CHECKIN fields, required)

| Field | Türkçe | Description |
|------|--------|------|
| rating_taste | Lezzet | Food and drink quality — the only criterion measuring what was actually consumed |
| rating_service | Servis | Service quality |
| rating_ambiance | Ambiyans | Ambiance (including music, lighting, decor) |
| rating_value | Fiyat | Value for money — is it worth what it costs |

All four are required on every check-in (see §10 Step 3), so every check-in contributes exactly four data points to the venue's aggregate below — the same four, every time, with no variation between check-ins.

`rating_taste` exists because nothing else measures the food. It was added when the per-item product layer was removed ([ADR-0011](https://github.com/oburapp/obur-docs/blob/main/adr/0011-drop-product-layer-four-venue-criteria.md)), which until then had been the only place food quality was recorded at all.

`rating_value` is deliberately about **price**, and it is the only criterion that is. An earlier version defined it as "the payoff of the overall experience," which — once taste, service, and ambiance are each rated separately — overlapped all three and measured nothing of its own. A venue can be excellent on those three and still charge three times what it's worth; this is the only field that can say so.

### Aggregate Scores

Deliberately uses different vocabulary from user input — an earlier version of this table didn't actually honor that (four of its eight labels were the exact same words as the input scale). This version is a fully deterministic statistical procedure, not a table to eyeball.

**There is one score, at one level: the venue.** Pool: every `rating_taste`, `rating_service`, `rating_ambiance`, and `rating_value` value recorded in a `public` check-in at that venue, all pooled together. It's the one big number at the top of a venue's page, and it treats the four criteria as equally-weighted, interchangeable signals of "how was going here" — a product judgment, not a statistical necessity, and one that matches how virtually every comparable app (Google Maps, Yelp) shows a single overall score rather than several competing for attention.

Because every check-in contributes exactly four values, **every check-in weighs exactly the same**. An earlier version had a second, product-level score and had to accept that a check-in rating more items carried proportionally more influence; removing the product layer ([ADR-0011](https://github.com/oburapp/obur-docs/blob/main/adr/0011-drop-product-layer-four-venue-criteria.md)) removed that asymmetry along with it.

The four criteria are also shown individually on a venue's page — a venue whose food is excellent and whose value is poor is exactly the kind of thing the headline number averages away, and the whole reason for rating four separate things is to be able to say it.

**The procedure:**

1. **Volume floor.** Fewer than 10 ratings in the pool: label is *New / Low Data* (Turkish: *Yeni / Az Veri*) — the same "don't claim a tier until there's really enough evidence" standard [Steam](https://www.steampageanalyzer.com/blog/steam-review-score-thresholds) itself uses (10 reviews minimum before any label shows). Tested empirically against a smaller floor: at 3 ratings, a venue with e.g. two "Very Good" and one "Bad" (a genuinely decent showing) could compute to the single worst possible label — the interval is just too wide that early to say anything reliable. At 10, the same kind of mixed-but-mostly-positive sample lands in a sensible middle-positive band instead.
2. **Compute a 95% confidence lower bound on the mean**, not the raw mean itself: with sample mean x̄, sample standard deviation s, and n ratings, `Lower Bound = x̄ − t(0.95, n−1) × (s / √n)`, clipped to [1.0, 4.0]. This is the same principle behind the Wilson score interval Reddit and Yelp use for exactly this "don't let a handful of votes overclaim" problem — anchored to the pool's own data (not diluted toward some unrelated global average, which is why this was chosen over a Bayesian/IMDb-style approach), and it naturally pulls the result down both when there's too little data *and* when raters genuinely disagree (a high-variance pool lands closer to *Mixed* even at healthy volume) — which is also what replaces the earlier, uncomputable "scattered / mixed" row with a real formula.
3. **Place that lower-bound number in the band below** — a single pass, no separate per-tier volume gates or downgrade steps needed, since the confidence math already accounts for sample size on its own:

| English | Türkçe | Lower-Bound Range |
|---------|--------|--------------------|
| Extremely Favorable | Çok Olumlu | (3.7 – 4.0] |
| Very Favorable | Olumlu | (3.4 – 3.7] |
| Fairly Favorable | Genelde Olumlu | (3.1 – 3.4] |
| Somewhat Favorable | Kısmen Olumlu | (2.8 – 3.1] |
| Mixed | Karışık | [2.2 – 2.8] |
| Somewhat Unfavorable | Kısmen Olumsuz | [1.9 – 2.2) |
| Fairly Unfavorable | Genelde Olumsuz | [1.6 – 1.9) |
| Very Unfavorable | Olumsuz | [1.3 – 1.6) |
| Extremely Unfavorable | Çok Olumsuz | [1.0 – 1.3) |

Only the label is ever shown to users — never the raw score or the underlying math — consistent with the "different vocabulary from input" rule above. A venue's page also shows the check-in count behind its label (e.g. "47 check-in'e dayanıyor") for transparency about how much evidence backs the claim. The full text label is reserved for the one headline number; the four individual criteria are shown as compact indicators beneath it, not as four text labels competing with the headline.

The exact cut points and the volume floor are a starting point — calibrated once real usage data comes in (§18). What's fixed now is the *procedure and its statistical grounding*, not the final numbers.

---

## 9. Badge and Achievement System

### Philosophy

Badges are not a game reward — they are proof of status. Stronger than the count "I've logged 50 different venues" is the feeling: "I'm the top discoverer of venues in Kadıköy, Istanbul." Earning a badge should require platform-specific behavior, not just time spent.

### Classification

| Tier | Description |
|-------|-------------|
| Gold | Hard to earn, rare, carries real status |
| Silver | Medium difficulty, rewards consistent use |
| Bronze | Easy to earn, hooks the new user into the platform |

### Example Badges

| Badge | Tier | Earning Condition |
|-------|-------|----------------|
| First discoverer | Gold | Platform's first check-in at a venue |
| Döner expert | Gold | Check-ins at 20+ different venues in the döner category |
| Kadıköy explorer | Silver | 30+ different venues in Kadıköy |
| Loyal customer | Silver | 10+ visits to the same venue |
| First step | Bronze | First check-in |
| Social | Bronze | First 5 followers |

### Permanence and Evaluation

**A badge is permanent once earned — it documents a historical fact, not a live status.** "Reached 30+ venues in Kadıköy" stays true forever, even if the user later deletes a check-in that would drop them back below the threshold — the same way a Steam achievement is never revoked even if its underlying condition later becomes false. This resolves what would otherwise be a real tension between the Philosophy above ("proof of status") and user expectations (nobody wants an earned trophy quietly taken away): a badge is worded and treated as something that *happened*, not something that's *currently true*. The one exception is admin moderation — if a badge turns out to rest on fraudulent activity (e.g. "first discoverer" on a venue later found to be fake and suspended, §13), an admin can revoke it manually, the same override the admin always retains elsewhere in this document. There's no automatic revocation path.

Permanence simplifies evaluation considerably: a badge only ever needs checking **forward**, at the moment a new check-in (or other qualifying action) might newly cross a threshold — the same one-extra-query pattern venue verification already uses (§13, ADR-0009). There's no need to ever re-scan or "un-award," so this stays synchronous, consistent with the rest of the stack (ADR-0008's notifications, ADR-0009's verification) and needs no new queue or background-worker infrastructure.

`rarity_pct` is the one exception that does need periodic, not live, computation: it's a percentage over the *entire* user base, and recomputing that on every profile view doesn't scale the way a per-check-in threshold check does. Real precedent for exactly this pattern: Steam's "global achievement percentage" and League of Legends' rank-distribution percentile are both known to update on a periodic/batch cycle, not in real time. This is the one place in the system that genuinely needs a scheduled job rather than synchronous evaluation — the exact cadence (daily, weekly, or usage-triggered) doesn't change this architecture and is left as an implementation decision, not fixed here.

### Display Rules

- Users can pin the badges they want on their profile (`is_pinned = true`)
- The "All achievements" section shows every badge earned
- Each badge displays its rarity next to it: "held by 1.2% of users"
- The `rarity_pct` field is recalculated periodically, not live — see Permanence and Evaluation above

---

## 10. Check-in Flow

### Steps

Throughout Steps 1–3, progress is autosaved as a `CHECKIN_DRAFT` (§17) — the flow can be closed and resumed later, including on a different device, without losing anything already entered. Nothing here is a real `CHECKIN` until Step 4's Save actually runs.

```
Step 1: Choose venue
  → Search box: searches Obur's own venues first (coordinate-based
    nearby suggestions), with Google Places Autocomplete results
    blended in for venues not yet on the platform
  → Selecting a Google result carries its place_id, address, and
    district straight through — no extra typing
  → Not found in either source: add a new venue manually (name, map
    pin, free-text address, district — district is required either
    way, not just for the Autocomplete path)
  → Either path runs through the same duplicate check before creating
    a new row: exact place_id match resolves silently to the existing
    venue; otherwise the existing 50-meter "did you mean this one?"
    prompt (§13, ADR-0009)

Step 2: Rate the venue
  → Four criteria, one card each: taste, service, ambiance, value
  → Top card is active, the rest wait behind it
  → Four large buttons: Bad | Average | Good | Very Good (left to right)
  → Once rated, the card flies upward and the next one appears
  → All four are required — the flow can't advance until each is rated
    (see §8). Every check-in therefore contributes the same four data
    points to the venue's aggregate, with no variation between check-ins

Step 3: Tell the story
  → Note field: placeholder "what stood out to you the most?"
  → Typing @ autocompletes mutual followers only (§11) — no one else is
    offered as a mention target
  → Typing # opens free-text hashtag entry, capped at 5 per check-in (§7)
  → Photo: "add a photo — show us"
  → Visit date: defaults to today, editable

Step 4: Share
  → Summary card: venue and the four ratings
  → Visibility: public (default) / close friends / private — a
    public check-in counts toward the venue aggregate; a
    close-friends or private one doesn't. There is no separate
    "contribute to statistics" toggle (see ADR-0004 in obur-docs: an
    earlier draft of this flow had one, but its own description
    contradicted itself — it claimed to count toward the aggregate
    "even when off," which made it a toggle with no actual effect).
    "Close friends" resolves against the viewer's manually curated
    close-friends list on the check-in owner's account (see §11 and
    ADR-0006) — it is not a "followers-only" option.
  → Save: submits with a client-generated idempotency key (§17) so a
    dropped connection or a double-tap can't create a duplicate
    check-in; a visible "sending" state covers the gap, and on mobile
    a failed submission retries automatically once connectivity
    returns. On success the `CHECKIN_DRAFT` this flow was autosaving to
    is deleted — the real `CHECKIN` row is what remains.
```

### Why Four Steps, Not Five

An earlier version of this flow had a fifth step between the venue and the story: choosing which items were eaten and rating each one. It was removed ([ADR-0011](https://github.com/oburapp/obur-docs/blob/main/adr/0011-drop-product-layer-four-venue-criteria.md)) — at this platform's scale no individual item could accumulate enough ratings to earn a label under §8's floor, so those two steps cost every user real effort on the core action and returned nothing visible. What was eaten now lives in the note (Step 3), which tells a reader more than a number next to a dish name anyway, and the card-stack interaction that made per-item rating feel fluid was kept and pointed at the four venue criteria instead.

---

## 11. Social Graph

**One-directional follow.** No mutual approval required. If user A follows user B, A sees B's public check-ins and lists in their feed. B does not need to follow A back. A user cannot follow themselves. Either side can end the relationship: the follower unfollowing, or the followed user removing that follower from their own followers list (Instagram-style).

**Close friends: a manually curated allow-list, not a follow-graph feature.** Because following requires no approval, "my followers" isn't a meaningful privacy boundary on its own — anyone can join it just by clicking follow. A user who wants a real private-sharing tier instead builds a **close friends** list by hand, adding people from among their existing followers. Marking a check-in, list, or saved venue as "close friends" visibility shares it with exactly that curated list, not with followers in general. If the user later stops following someone (or removes them as a follower), that person drops off the close-friends list automatically — close-friend status can never outlive the follow relationship it depended on. Modeled directly on Letterboxd's own close-friends feature, the closest real-world comparable. See ADR-0006.

**Likes are public; bookmarks are private.** A like on a check-in or list is a visible social signal — anyone who can see the content can see (and add to) its like count. A bookmark is the opposite: a personal "save this for later" note that only the person who made it can ever see. There is no bookmark count shown anywhere, on any check-in or list, to anyone. Liking or bookmarking something a user can't see (a private check-in that isn't theirs, for example) is not possible — visibility is checked first either way.

**@ Mentions require mutual following — not open to anyone, unlike a like or a bookmark.** Tagging a stranger in a check-in note would reintroduce exactly the unwanted-attention risk the platform has no comment or DM feature specifically to avoid (§4); requiring that the author follows the mentioned person *and* the mentioned person follows the author back keeps mentions inside an already-established, two-sided relationship. This restriction also makes blocking's existing behavior double as mention protection for free: blocking already auto-unfollows in both directions (above), which means a blocked pair no longer satisfies the mutual-follow requirement either — there's no separate rule needed to stop a blocked person from mentioning someone. A mention triggers a notification (below) but never overrides the mentioned check-in's own visibility (§7) — the mentioned person is told they were mentioned, not granted access to something they otherwise couldn't see. Existing mentions between two people are purged the moment they block each other, the same retroactive cleanup already applied to likes, bookmarks, and notifications.

**# Hashtags are unrestricted free text, on check-ins and lists alike — up to 5 per item.** Unlike a mention, a hashtag doesn't target a person, so there's no relationship requirement; the cap exists purely to prevent keyword-stuffing a single post to game hashtag-based discovery (§12), not to limit expression. Tapping a hashtag anywhere opens a discovery page of every `public` hashtagged item — the same public-only scoping already applied everywhere else content gets aggregated or surfaced broadly (aggregate ratings, venue verification, a venue's representative photo). An offensive or spammy hashtag doesn't need its own moderation path — it's part of the check-in or list it's attached to, already reportable as that content (§11); no new report reason required.

**Mute is the lighter counterpart to blocking — the relationship stays intact, only feed content is affected.** Unfollowing someone (a coworker, an acquaintance) to stop seeing their check-ins can feel socially awkward in a way muting doesn't; mute solves that specific case without touching anything else:

- One-directional and silent: the muted person is never notified, and nothing about the follow relationship, their visibility, or their discoverability changes for them.
- Affects the muting user's own feed only (both layers, §12) — the muted user's content simply never surfaces there. Search, Discover, and venue pages are untouched; muting isn't hiding, it's a feed preference.
- Not derived from `FOLLOW` — a user can mute someone they don't follow (e.g. to keep a stranger's content out of Layer 2's algorithmic fill), unlike `CLOSE_FRIEND`.
- No retroactive effect: existing likes, bookmarks, and notifications between the two people are untouched, and neither can be affected going forward either — muting has no bearing on any interaction, only on feed display.

**Blocking is absolute, silent, and bidirectional — deliberately harsher than a visibility tier.** Where `close_friends`/`private` control who a user chooses to share with, blocking is about removing someone entirely, in both directions at once:

- Blocking auto-unfollows in both directions (and, since `CLOSE_FRIEND` already cascades off `FOLLOW`, close-friend status is removed for free — no new logic needed there).
- Visibility closes completely between the two people, `public` included — a block is not just another privacy tier a viewer can be excluded from, it overrides all three.
- Neither person can find the other via search or Discover.
- It's silent: the blocked person is never notified, and a blocked profile behaves exactly like a nonexistent one to them — the same "hidden must be indistinguishable from nonexistent" principle already applied to private check-ins (§10) extends here.
- Existing likes, bookmarks, and notifications between the two people are purged in both directions at the moment of blocking — not just prevented going forward. This is a deliberately harsh, retroactive cleanup, decided because interpersonal-safety context (unlike an ordinary privacy setting) warrants leaving no trace rather than a lighter touch.
- Where one person's identity would otherwise surface on something the other can see (e.g. a venue's "first discoverer," §13), it's anonymized to "unknown user" for that specific viewer — this changes *display*, not data. The blocked person's own check-ins, badges, and history remain fully theirs; only how their identity renders to the person who blocked them (or whom they blocked) is masked.
- Admin moderation access is never affected by a block between two other users — reports need to stay reviewable regardless of who blocked whom.
- Aggregate ratings are untouched: they're anonymous, platform-wide numbers with no per-viewer dimension, so a block (a relationship between two specific people) has nothing to act on there.

Unblocking is a real, always-available action; nothing about the earlier relationship (follow status, close-friend status) is restored automatically — same as any other reversed social action in this system. For the `BLOCK` schema this implies — directional storage, bidirectional enforcement — see [ADR-0010](https://github.com/oburapp/obur-docs/blob/main/adr/0010-blocking-and-reporting-schema.md).

**Reporting.** Three things can be reported: a check-in (its note or photo), a user profile, or a venue — each for a different reason. Check-ins and profiles carry interpersonal-safety risk; report reasons: spam, harassment, hate speech, sexual content, violence, fake account, other — matching the categories real platforms (Instagram, Twitter/X, Reddit) converge on, not invented from scratch. A venue report is a data-quality concern instead — wrong address, wrong name, permanently closed, duplicate — and is the *only* way a venue's details ever change after creation (§13): there's no direct user-editing of any venue field, not even by whoever added it, on a real abuse precedent from another app's community-edit feature (see §4). Both kinds land in the same admin-reviewed queue.

Every report goes into a queue reviewed by a human admin — deliberately **no automatic threshold-based hiding** (e.g. "N reports auto-removes it"), unlike venue verification's check-in-count filter. The two situations aren't parallel: venue verification is zero-urgency and can sit unresolved forever with no harm, while an unreviewed harassment report is a real, time-sensitive harm — but automatic hiding at low report counts is also a known abuse vector itself (coordinated false reporting to silence someone), and at Obur's expected report volume (low, at a few-hundred-user scale) manual review doesn't have the scaling problem admin-reviews-everything had for venues. A reported user can always be blocked immediately by the person affected, independent of and faster than any admin action.

An admin handling a check-in or profile report can dismiss it, remove the offending content, or suspend the account (`USER.status = suspended`, §6) — kept separate from `USER.role`, since role is permission level and status is standing, and conflating them would be a mistake. Suspension is admin-only and, unlike a user's own self-service account freeze (§6), never user-reversible. An admin handling a venue report can dismiss it, correct the reported field directly, mark the venue closed (`is_active = false` — stays visible, shown transparently, see §13), or suspend it (`is_suspended = true` — hidden entirely, see §13). Like account suspension, both are admin-only and never reachable through any user-facing endpoint. The two report kinds are stored as two separate tables — `CONTENT_REPORT` and `VENUE_REPORT` (§7) — precisely because their reasons, resolutions, and urgency differ this much; see [ADR-0010](https://github.com/oburapp/obur-docs/blob/main/adr/0010-blocking-and-reporting-schema.md).

**Minimum standards this has to satisfy, non-negotiably:** Apple's App Store Review Guidelines §1.2 (user-generated content) requires, for any app with UGC, a mechanism to report objectionable content, a mechanism to block abusive users, published contact information a user can reach the team through, and the ability to remove content or terminate accounts in a reasonably timely way. Google Play's UGC policy asks for materially the same. These aren't aspirational — without them the app doesn't clear store review. A published support/abuse contact (even just an email address, shown in-app and in the store listing) is a small, concrete requirement that's easy to forget precisely because it's not a "feature."

**A written Content Policy is a separate document from this reporting mechanism**, and still needs to exist: this system decides *how* a report gets handled, but "what actually counts as harassment, hate speech, spam" on Obur specifically needs to be written down somewhere a user (and an admin acting consistently, not case-by-case) can point to. That text should be legally reviewed before it's published, the same way Terms of Service and a Privacy Policy need to be — not drafted here as if it were a legal document. See §18.

**Notification triggers:**

- New follower (with a follow-back button)
- Check-in like
- List like
- Badge earned
- Mention (never grants visibility into what triggered it — see above)

Notifications are written synchronously by the same action that causes them (see ADR-0008) — there's no delay or background processing step between "someone followed you" and the notification existing.

Venue verification (§13) deliberately does **not** trigger a notification — it's a cosmetic, low-stakes signal ("this location definitely exists here," not a quality judgment), and notifying the adder the moment it flips felt like friction with no real payoff. It's visible passively on the venue page whenever it happens to update.

---

## 12. Main Feed Algorithm

### Two-Layer Structure

**Layer 1 — Followed users (primary):** Public check-ins and lists from users the account follows, shown chronologically. Requires no extra algorithm.

**Layer 2 — Algorithmic fill (secondary):** Kicks in once content from followed users falls below a defined threshold of the total feed. It's the cold-start solution for a new user, and for an existing user it keeps the feed alive while they follow few people.

Both layers exclude any user the account has muted (§11) or blocked (§11) — muting and blocking are feed inputs, not something layered on top after the fact.

Algorithmic content ranking signals (in priority order):

1. Check-ins at venues whose `VENUE_CATEGORY` the user rates highly — the taste-overlap signal, keyed on venue category since there is no per-item data (§13)
2. Content from the user's current city
3. Like count

Algorithmic content is visually distinguished: a thin line on the left edge of the card and a "suggested" label.

### Discover Page Ranking

When a user searches "Kadıköy dönerci" — a category plus a district, not a dish, since ranking is venue-level throughout (§13):

1. Aggregate rating (primary)
2. Check-in count (secondary)
3. Check-in count among followed users (social signal, "3 friends have been" label)

Among venues with an equal rating, recently logged content ranks higher; venues that have closed or declined in quality naturally sink.

### Hashtags

A hashtag's own page lists every `public` check-in and list carrying it, most recent first — the same public-only scoping used everywhere else content gets surfaced beyond its original audience (§11). Reachable two ways: tapping a hashtag inline wherever it appears, or typing `#` into Discover's search box (§5). The 5-per-item cap (§7, §11) exists to keep this page's ranking meaningful — an unbounded hashtag list would let a single post claim space across dozens of unrelated discovery pages, diluting the signal for everyone else using that tag genuinely.

---

## 13. Venue Architecture

### Venue Identity

Identity is built on coordinates, not on name. The `(lat, lng)` pair is the primary identifier. Duplicate detection is two layers, not one (see ADR-0009):

1. **Exact match** — if the venue being added carries a `google_places_id` that an existing venue already has, that's a certain duplicate (Google's own identity says so). Resolves silently to the existing venue, no prompt, not overridable — there's nothing to confirm.
2. **Proximity fallback** — for everything without a `google_places_id` match, if another venue exists within a 50-meter radius, the user is asked "did you mean this one?" and can confirm it's genuinely different.

For cases like malls where multiple venues can share the same coordinates, the free-text `address_note` field is used ("3rd floor", "Block B entrance") — and the exact-match layer above is exactly what keeps that same scenario from producing false "possible duplicate" prompts once a venue actually has a `google_places_id`, since two different businesses in the same mall get two different Google identities.

Every venue also records a `district` (ilçe) alongside `city` — required going forward, since "city" alone can't express "Kadıköy" as a scope for discovery, badges (§9), or category-scoped rankings (§12).

**Verification is a cosmetic signal, not a quality one.** A venue can be marked verified once either a Google Places match plus at least N independent **public** check-ins confirm it, or — for venues Google hasn't indexed — both at least M independent **public** check-ins accumulate *and* an admin separately confirms it. Both conditions are required in the no-Google-match case, not either alone — the check-in count filters which venues are even worth an admin's time, it isn't a substitute for the admin's look. Only `public` check-ins count toward N/M, the same restriction the aggregate rating already applies (§8, §10 Step 4) — a `close_friends`/`private` check-in is real evidence to the person who made it, but counting it here would let a location get the platform's public "verified" mark off the back of a share nobody outside that circle ever saw, which undercuts the point of the visibility choice the same way it would for the aggregate label. It never affects search ranking or discoverability; it only answers "does this location definitely exist here?" See ADR-0009.

**No user can edit a venue directly, not even whoever added it.** Every correction — wrong address, wrong name, permanently closed, duplicate — goes through the same report mechanism as check-ins and profiles (§11), reviewed by an admin before anything changes. Direct community editing was considered and rejected: unmoderated edits to shared location data have a real abuse precedent (see §4), and the risk isn't worth the convenience.

**Closed and suspended are two different states, not one.** `is_active = false` means the business itself has stopped operating — set by an admin acting on a report, never automatically. The venue stays fully visible, shown transparently (grayed out, clearly marked inactive), the same "historical data is never deleted" principle already applied to venues' own check-ins (§7). `is_suspended = true` is a separate, harsher admin moderation action: the venue is hidden entirely from anyone but an admin, and its own page returns a generic "venue not found" — not an explanation that it was suspended, the same "hidden must be indistinguishable from nonexistent" principle already applied to blocked profiles (§11) and private check-ins (§10), deliberately not leaking a moderation action to whoever's affected by it. Check-ins that reference a suspended venue are unaffected anywhere else they appear (a user's own feed or profile) — only the venue's own page becomes inaccessible.

**A venue has no photo of its own — its representative image is derived, not uploaded or set by anyone.** Consistent with "no say for businesses" (§4) and no-direct-editing above, a venue's face is whichever of its own **public** check-ins currently has the most likes — nobody, including whoever added the venue, chooses or uploads it directly. Scoped to `public` check-ins only: a `close_friends`-visibility check-in can be liked plenty within that circle, but surfacing its photo on the venue's fully public page would leak a deliberately restricted share to everyone, defeating the point of the visibility choice — the same leak risk already reasoned through for signed photo URLs (§17). Computed live at read time, not stored or cached on `VENUE` — cheap enough per-venue not to need it, and it means the representative photo is automatically always current (a deleted or newly-more-liked check-in updates it for free, no invalidation logic to get wrong). A venue with no public check-ins yet simply shows no photo, the same "not enough data" treatment already used for the aggregate rating label (§8). This also settles how a list displays the venues inside it (§6): each venue card reuses this same derived photo — a list itself needs no separate cover-image feature.

**The selection is per-viewer, not a single global answer — the same treatment already given to "first discoverer."** A representative photo is always sourced from one specific person's specific check-in, which puts it in the same category as "first discoverer" (§11: anonymized per-viewer when blocking is involved), not the aggregate rating (genuinely anonymous, no single attributable person, untouched by blocking). So the query is "the most-liked public check-in, excluding anyone the current viewer has blocked or is blocked by" — the same exclusion `FOLLOW`/feed content already applies for blocking (§12), reused here rather than invented fresh. For a blocked pair, this naturally surfaces the next-best-liked eligible photo instead, with no separate "pick a runner-up" logic needed. Scoped to blocking only, not muting — muting is deliberately limited to feed display (§11) and doesn't touch venue pages. Since the photo only ever comes from a `public` check-in, and blocking is the only thing that can make a `public` check-in invisible to a specific viewer (§11), the same exclusion that picks the photo also already guarantees its click-through (below) is valid for whoever it's shown to — no separate check needed at click time.

**The photo is clickable through to its source check-in.** Not a prominent "photo by @username" credit sitting next to the image — that would visually compete with the "first discoverer" badge and risk exactly the confusion it's meant to avoid — just the image itself, tappable, taking the viewer to that check-in. The venue page already lists every check-in at that venue, so this doesn't expose anything not already reachable, just shortens the path; it's also a small, low-friction reward for contributing a photo good enough to represent the place, without inventing a new badge or mechanic for it.

### Venue Categorization

```
VENUE_CATEGORY (hierarchical)
  └── food
        ├── kebap
        ├── pide
        └── doner  ← VENUE.category_id points at the deepest matching node
```

A venue carries exactly one `category_id`, and it is the platform's only classification dimension: there is no per-item layer beneath it ([ADR-0011](https://github.com/oburapp/obur-docs/blob/main/adr/0011-drop-product-layer-four-venue-criteria.md)). Three separate things read it — Discover's filters, §12's Layer-2 ranking signal, and Personalized History below — so its granularity directly determines how specific each of those can be. "Dönerci" is a useful answer; "yiyecek yeri" is not.

The catalog is defined by the team, not by users. Two things keep that from becoming the same dead end the removed product catalog had: venue types are a genuinely bounded set with published external taxonomies to draw on, and a venue is created rarely — unlike an item, which was chosen on every single check-in. Where a venue doesn't match any leaf, the parent is always a valid answer, so the flow never blocks; a category that turns out wrong is corrected by an admin through the same report mechanism as any other venue field (§11), permanently and for everyone.

### The Venue Page

Its headline aggregate rating and the four criteria behind it (§8), every check-in at the venue, its photos, and the first discoverer — the *only* "first" badge in the system (see §9).

There is no product page and no cross-venue item ranking. Both existed in an earlier version of this document and were removed with the product layer; see [ADR-0011](https://github.com/oburapp/obur-docs/blob/main/adr/0011-drop-product-layer-four-venue-criteria.md) for why, and §6's v2.0 table for the conditions under which the question becomes answerable again.

### Personalized History

A user's own profile shows their single highest-rated venue per `VENUE_CATEGORY` they've checked into ("en iyi dönerci: Develi, en iyi kafe: Petra") — a lightweight, no-new-schema read over data that already exists, meant to reinforce a sense of discovery and progress on the user's own profile rather than to rank anything for other people.

Unlike everything else derived from ratings in this document, it needs no volume floor: it reports the user's *own* rating, not a platform statistic, so it works from their first check-in. Its specificity is entirely a function of how granular the category catalog above is.

---

## 14. Tech Stack

### Backend

| Component | Technology | Rationale |
|---------|-----------|---------|
| API Framework | FastAPI | Proven experience, async, automatic OpenAPI documentation |
| Database | PostgreSQL + PostGIS | Relational model + coordinate queries |
| Auth | Clerk | Managed, email + social login ready, integrates with JWT verification middleware |
| File storage | Cloudflare R2 | S3-compatible, free egress, easy Python integration via boto3 |
| Search | PostgreSQL FTS (initial) → Meilisearch (growth) | Turkish stemming, low maintenance overhead |
| Cache | Redis (Railway add-on) | Sessions, rate limiting |

### Web Frontend

| Component | Technology | Rationale |
|---------|-----------|---------|
| Framework | Next.js | SSR for SEO, venue pages are crawlable |
| Styling | Tailwind CSS + shadcn/ui | Fast UI development, consistent component set |
| Map | Mapbox GL JS | Cheaper than Google Maps, freedom to customize |
| Deploy | Vercel | Native for Next.js, free hobby plan |

### Mobile

| Component | Technology | Rationale |
|---------|-----------|---------|
| Framework | Flutter | Native performance, single codebase for iOS + Android, strong animation capability |
| Language | Dart | Comes bundled with Flutter, easy for LLM-assisted generation |
| Map | Mapbox Flutter SDK | Consistent with web |
| State management | Riverpod | Standard in the Flutter ecosystem |

### Shared

Both frontends consume the same FastAPI endpoints. API versioning is set up from the start (`/api/v1/`).

---

## 15. Infrastructure and Cost

### Starting Point (0-200 users)

| Service | Plan | Monthly Cost |
|--------|------|---------------|
| Railway (backend + PostgreSQL) | Hobby | ~$5 |
| Vercel (web) | Hobby | $0 |
| Clerk (auth) | Free (0-10K MAU) | $0 |
| Cloudflare R2 (storage) | Free (first 10 GB) | $0 |
| Mapbox | Free (50K loads/month) | $0 |
| **Total** | | **~$5/month** |

### Growth Thresholds

- **Clerk:** moves to a paid tier past 10,000 MAU
- **R2:** paid tier kicks in at 10 GB of storage or high request volume
- **Railway:** plan upgrade needed under high CPU/memory usage
- **Vercel:** Pro plan ($20/month) for commercial use or high bandwidth

### Architecture Notes

Backend and PostgreSQL run in the same Railway region; network hop latency is minimal. Use of a distributed set of providers has been avoided.

---

## 16. Go-to-Market Strategy

### Bootstrapped (Unfunded) Start

As with Letterboxd and Untappd, the core experience is never gated. A strong, passionate "enthusiast" core audience is courted first; the rest of the audience follows this core organically.

### First 200 Users

1. The team's social circle (first 20-30, for early testing rather than genuine feedback)
2. Micro food content creators in Istanbul (under 10K followers, genuine community owners)
3. Target audience on Reddit (`r/istanbul`, `r/yemek`) and X
4. Outreach to users actively writing reviews on Google Maps

### Geographic Focus

The first city is Istanbul. 500 users in 1 city gets far closer to critical mass than 50 users spread across 10 cities. Expansion to a second city repeats the same playbook.

---

## 17. Non-Functional Requirements

### Offline & Draft Reliability

**A check-in must never be silently lost to a bad connection — this is an MVP requirement, not a client-side nicety.** The check-in flow is the platform's core action, takes real effort to complete (venue, four ratings, photo, note across 4 steps, §10), and the venue interiors it's used in are exactly where a mobile connection is least reliable. Losing that data to a dropped connection is a core-loop failure, held to the same "not optional polish" bar as blocking or reporting (§4) — not a "nice if the client gets to it" item.

Two distinct mechanisms cover two distinct moments:

- **Before submission — an explicit, cross-device draft.** A user can save an in-progress check-in and resume it on any device, including switching from mobile to web mid-flow. This has to be server-synced, not device-local: the same "state must be consistent across every device a user is signed into" standard already applied to notification read-state (ADR-0008) applies here — a draft that only exists on the phone it was started on isn't really solving the problem for a multi-platform product. Modeled as a separate `CHECKIN_DRAFT` table (§7), not an `is_draft` flag on `CHECKIN` itself: a flag would require every aggregate-rating, badge, and feed query to remember to filter it out forever, and this platform has already hit that exact failure mode twice in Phase 4 (an existence-leak and a stale-visibility bug, both "a query somewhere forgot to filter"). A separate table makes leaking a draft into rating/badge/feed logic structurally impossible rather than a matter of developer discipline. On final submit, the draft's data becomes a real `CHECKIN` row and the draft row is deleted.
- **After submission — safe retries, not duplicates.** Once the user hits submit, a client-generated idempotency key travels with the request; retrying with the same key (because the first response never arrived, or the user tapped submit twice) returns the original check-in instead of creating a second one. This is what actually prevents a flaky connection from corrupting aggregate ratings or badge counts with duplicate check-ins.

A pending submission shows a visible "sending" state to the user who created it — never to anyone else; nothing partially-submitted is ever shown as if it were a real, published check-in.

**The automatic, no-user-action retry-on-reconnect guarantee is mobile-only.** Checking in happens in the moment, at the venue — realistically a mobile action, the same way Instagram posting essentially never happens from its web client. Building true offline-first engineering (surviving a killed app, background sync) for the web client isn't worth the cost for a use case it barely serves. Web still gets the draft mechanism above (it's just a normal API call, no special offline capability needed) and the idempotency-key safety net — a failed web submission doesn't lose data, it just isn't retried automatically in the background the way a mobile one is.

**Web being a narrower client doesn't mean it's a neglected one.** Whatever web does support must stay fast and responsive — the standard here is "PC Instagram," a client so visibly an afterthought that it changed real usage behavior; Obur's web client should do less than mobile by deliberate scope, never by neglect.

`CHECKIN_DRAFT` is deliberately not generalized to list or venue creation. Both lack the combination that actually justifies it: several fields of real effort — a venue choice, four ratings, a photo, a written note — bundled into one atomic request, at the exact moment a connection is weakest. That effort dropped when the per-item step was removed from the flow (§10, ADR-0011), but it did not drop below the bar: a photo and a written note are still the most expensive things a user produces anywhere in this product, and the weak-connection argument is untouched. Lists are built incrementally (each item add is already its own small, durable, retryable action — the existing idempotent-create pattern used elsewhere, §11, already covers it); venue creation is a handful of fields, far lower effort, and typically happens as a sub-step inside the check-in flow itself, §10, inheriting whatever protection that already has.

### Rate Limiting & Fake-Account Resistance

Every public endpoint gets a baseline rate limit (a general rule, not repeated here). A stricter tier applies to actions where repeated abuse does real damage, not just annoyance:

- **Check-in creation** — the most important one. Obur's entire value proposition rests on ratings coming from real, distinct people (§2); rapid repeated check-ins from one actor can directly inflate or deflate a venue's aggregate rating (§8), which is a direct attack on the platform's core credibility, not a minor abuse.
- **Report submission** — already identified as an abuse vector in its own right (§11): unlimited reporting enables coordinated false-reporting to harass or silence someone.
- **Venue creation** — repeated spam venues pollute Discover, and can be used to game verification's check-in-count filter (§13) with venues that only exist to be checked into.
- **Follow** — lower priority, but the well-known "mass-follow for attention" pattern is worth a limit too.

**Fake accounts (sockpuppets) matter, but not equally everywhere — ranked by actual damage:** aggregate-rating manipulation is the most severe, since it attacks the same core credibility problem Obur exists to solve for (§2); gaming the Layer 2 feed-ranking like-count signal (§12) is next; a fake-follower "Social" badge (§9) is low-severity; gaming venue verification is the least severe of all, because verification was deliberately designed to be cosmetic and never affect ranking or discoverability (§13) — there's little to gain from faking it.

**No phone verification, custom SMS OTP, or IP-based tracking at MVP — a deliberate scope decision, not an oversight.** Each was considered and set aside for a concrete reason: an unverified phone field adds signup friction and a new PII surface with no real anti-fraud value (verification's value comes specifically from the OTP step, which is what actually costs money); a self-built SMS OTP flow trades Clerk's flat, managed cost for a variable, self-secured one, including real exposure to SMS-pumping fraud if the send-OTP endpoint isn't perfectly rate-limited; IP correlation (even privacy-preserving, HMAC-hashed, never storing the raw address) is a real, well-understood technique, but the team made a deliberate bootstrap-stage call not to take on a data-handling responsibility in an area neither owner has direct experience with. **That rejection is about correlation — linking accounts to each other through a shared address in order to infer that one person is behind several.** It is not a blanket refusal to derive anything from a request's origin: an ephemeral counter that only answers "has this origin made too many requests in the last minute" builds no linkage and outlives no window. The two are separated deliberately in [ADR-0014](https://github.com/oburapp/obur-docs/blob/main/adr/0014-rate-limiting-keys-and-ip-minimisation.md), which permits the second and leaves the first rejected — because refusing the counter would mean failing to protect materially more sensitive data (see Bulk-Extraction Resistance below) in order to avoid touching less sensitive data. MVP relies entirely on Clerk's existing free-tier friction — required, verified email; a 15-character minimum password with compromised-password rejection. Real fraud-detection tooling is deferred until actual abuse is observed, not built pre-emptively against a hypothetical — the same posture already taken toward rating-threshold calibration (§18).

### Bulk-Extraction Resistance

The abuse this guards against is different from fake accounts above: not someone corrupting data by creating it, but someone systematically copying data that's already meant to be visible. There are two distinct harms here, and only the first was originally written down:

**Copying the catalog** — rebuilding Obur's curated venue database and aggregate ratings via scraping, without any of the community effort that produced them. This is an effort-theft concern, and it scales with the catalog: a small dataset isn't worth taking.

**Harvesting people** — walking every public check-in to assemble who goes where, and when. A check-in carries a user id, a free-text note, a photo, a date, and a venue, which is a location. Cross-referencing the user id across venues rebuilds a behavioural and location profile. This one does **not** scale down: a small user base is *easier* to deanonymise, not harder, so "we're too small to bother scraping" is not an argument here.

The second harm is the stronger reason this section exists, and it rests on a distinction worth stating plainly: **being public piece by piece is not the same as being public in bulk.** One person reading one `public` check-in is the product working exactly as the author intended — they chose `public`. A script reading all of them produces something no individual sharing decision ever authorised. The protection therefore has to be aimed at volume, not at access.

- **Read endpoints get rate limits too, not just write ones.** Discover, search, and venue-listing calls are unlimited today only in the sense that nobody's decided otherwise; without a cap, a script can enumerate the entire dataset in minutes. Limiting an anonymous caller requires a key derived from the request's origin — see the deliberately narrow exception to this document's own position on IP data in [ADR-0014](https://github.com/oburapp/obur-docs/blob/main/adr/0014-rate-limiting-keys-and-ip-minimisation.md).
- **Every list response is paginated, with a capped page size (tens of records, not thousands) — never an unbounded "return everything" response.** This forces a scraper into thousands of individual requests, which is exactly what the read rate limit above then catches.
- **There is no bulk-export endpoint, as a standing design principle, not just an absent feature.** No user-facing "download all venues/users as CSV/JSON" capability is ever exposed — worth stating explicitly so it isn't accidentally added later as a convenience feature without this context.

**What this actually achieves, stated honestly: rate limiting raises the cost of harvesting, it does not prevent it.** Any limit permissive enough for real anonymous browsing (an engaged visitor makes a few hundred calls an hour, since one page view is several requests) also permits a single-IP full enumeration in hours rather than minutes. That is the realistic goal, and it is the one this section's own wording implies — *minutes* is what makes bulk harvesting cheap enough to be casual. Turning minutes into hours, and forcing a determined actor into rotating addresses, is worth doing; claiming it stops them would be a false comfort.

The three measures above only work together. The page-size cap is what forces a harvester into thousands of requests, which is the surface the rate limit then bites on; a rate limit without the cap would be trivially bypassed by one large request, and a cap without a limit would only slow things down.

Detecting a scraping *pattern* specifically (one account paging through the entire dataset in an unnatural sequence, as opposed to normal browsing) is a further step beyond this — deferred to the same "once real abuse is observed" bucket as fake-account detection above, not built pre-emptively. Closing the anonymous read surface entirely was also considered and rejected: it would end the crawlability §14 chose Next.js for.

### Private Check-in Photos Need Signed URLs — Public Ones Don't

A check-in's own visibility (`public` / `close_friends` / `private`) is enforced by the backend on the `CHECKIN` row — but `photo_url` points at a file in a separate system (Cloudflare R2), reachable directly if the URL itself ever escapes (browser cache, a screenshot, a log line) without ever passing through the backend's `can_view` check at all. That's a real gap: a private check-in's photo could be reachable even though the check-in itself correctly isn't. Text fields (`note`, ratings) don't have this problem — they never leave the database, so every read of them is already gated by the same authorization the rest of the app relies on; a photo is the one field type that lives outside that boundary by construction.

The fix is scoped narrowly: for `close_friends`/`private` check-ins, `photo_url` is served as a short-lived, cryptographically signed URL (R2/S3-style pre-signed URL), generated only after the backend's own `can_view` check already passed — not a permanent link. **Public check-in photos and profile avatars are deliberately left alone.** Signing them adds no real protection, since "public" already means "no one is excluded" — there's no boundary left to enforce — while it does add a real cost: signed URLs typically defeat straightforward CDN caching, since the URL itself changes on each generation. Bulk copying of public photos specifically is a scraping concern, covered above, not an access-control one.

### The Governing Principle: No Access Path Around the Application

Everything in this document that touches authorization — `can_view`, `ensure_visible_and_owned`, the existence-leak standard applied to private check-ins and blocked profiles, signed photo URLs above — is really one principle applied repeatedly: **the only way to reach any piece of data must be through the application's own authorization logic. No side door.**

That principle has a real boundary, though, worth stating explicitly rather than assuming it's covered: everything above protects the *application's own access path* — every API request passes through `can_view` because there is no other way to ask the API for data. It does nothing for someone who bypasses the application entirely and reaches the data store directly (a compromised database credential, an exposed backup, a leaked connection string). `can_view` is application code; it isn't a property of the database itself. Closing that gap needs a second, separate layer: the database encrypted at rest (so a stolen disk or backup isn't readable), and the database reachable only from the backend service, never directly from the outside. This is infrastructure/hosting configuration, not application logic — see §18 for what still needs verifying here before launch.

**PostgreSQL Row Level Security (RLS) is the concrete mechanism for that second layer, and belongs to the backend's own development discipline, not a retrofit project.** RLS lets the database itself enforce a visibility policy on a table (e.g. a `CHECKIN` row is only readable if `visibility = 'public'`, or the requester is its owner, or a qualifying close friend) so that even a query that forgets to call `can_view` — the exact failure mode behind the two real bugs already found this project (an existence-leak and a stale-visibility bug, §17 above) — still can't return a row it shouldn't. It's evaluated as each query and endpoint is written during backend development, not bolted on after the fact: some queries (badge `rarity_pct`, cross-venue product ranking, admin moderation tooling) deliberately need to see across every user's data, and deciding which access pattern each new query falls into is far cheaper to get right once, at the moment that query is first written, than to re-audit across everything already built if RLS is introduced only after the backend is otherwise complete.

### Photo Upload Standards

Applies uniformly to both upload surfaces that actually exist — `CHECKIN.photo_url` and `USER.avatar_url`. A venue has no upload surface of its own (§13); a list has no cover image at all (§6, §13).

- **Formats accepted**: JPEG, PNG, HEIC (converted server-side to a web-standard format, since HEIC doesn't render in every browser), WebP. No animated formats — a check-in or avatar photo is a static image.
- **Client-side compression before upload**: resized to a max ~2000-2500px on the longest edge, re-encoded at roughly 80-85% JPEG quality, before the request is ever sent. A raw phone photo can be 10-20MB; nothing in this product ever displays one at that size, and compressing before upload directly helps the offline/weak-connection case above (§17) — smaller payloads fail and retry faster.
- **Server-side hard cap**: any upload over ~15MB is rejected outright at the API boundary — a safety net for a misbehaving client or an abuse attempt, not a size the normal path should ever approach.
- **Multiple resolutions generated on upload, not one fixed size.** A phone's feed card and a desktop's full-screen view have very different real display sizes; serving one size to both means either wasting bandwidth on mobile or looking soft on a larger screen. Standard practice (the same approach Instagram, Letterboxd, and comparable apps use) — a small number of generated variants (e.g. thumbnail / medium / full), with the client requesting whichever fits its viewport.
- **Cropping differs by context, everything else doesn't.** `avatar_url` is square (the universal convention for a profile picture, every comparable app does this); `photo_url` keeps its natural aspect ratio — forcing a food or venue photo into a square loses real information a user chose to include.
- **All EXIF metadata is stripped on upload, not just GPS.** A phone photo can carry exact GPS coordinates, device model, and other embedded metadata a user never consciously chose to share — stripping only location data isn't enough, since other metadata fields (a device serial fragment, timestamps precise enough for correlation) can themselves be identifying. Applies to both `photo_url` and `avatar_url`, and happens server-side before anything is stored — Obur has no use for this metadata, only the risk of holding it.

### Performance & Perceived Speed

Targets grounded in published, external standards rather than picked arbitrarily:

- **`obur-web` adopts [Core Web Vitals](https://web.dev/articles/defining-core-web-vitals-thresholds) as its baseline** — Google's real-user-measured "good" thresholds, evaluated at the 75th percentile of real users over a rolling 28-day window: LCP (loading) < 2.5s, INP (responsiveness) < 200ms, CLS (visual stability) < 0.1.
- **Backend API response time targets a standard web-app tier, not a real-time (gaming/trading) one**: [P50 < 200ms, P90 < 500ms, P99 < 1s](https://medium.com/@jfindikli/the-ultimate-guide-to-faster-api-response-times-p50-p90-p99-latencies-0fb60f0a0198) for standard read endpoints (feed, search, venue/profile pages).
- **Check-in creation is deliberately exempt from that tier.** Its heavier path — photo compression, EXIF stripping, multi-resolution generation (above), plus a synchronous verification/badge check (§9, §13) — realistically lands past 1 second, and that's acceptable as long as the UI shows explicit progress. [Published research on perceived wait time](https://uxuiprinciples.com/en/principles/response-time-limits) holds that responses under 100ms feel instant, 100ms-1s stays within a user's conscious-but-uninterrupted flow, and anything in the 1-10s range needs a visible progress indicator or users abandon the action. This is exactly why the "sending" state already decided for check-in submission (Offline & Draft Reliability, above) isn't just a nice touch — it's what keeps a multi-second submission from feeling broken.
- **No numeric uptime target at MVP — stated explicitly, not left silently unaddressed.** Railway's hobby tier has no HA/failover guarantee, and building real redundancy would be wildly disproportionate to a 200-MAU target (§15). The one concrete decision made here: a friendly "we'll be back shortly"-style client message on unreachability, instead of a raw error — the actual uptime number itself is accepted as best-effort.

**Local interaction feedback is a separate concern from network latency above, and has no excuse to ever be slow.** Selecting a rating, advancing the card stack (§10 Step 2), and similar in-flow actions never wait on the network — the check-in flow only talks to the backend once, at Step 4's Save — so these should render feedback essentially instantly (the same sub-100ms "feels instant" threshold cited above, here applied to local UI state rather than a round-trip). Concretely: the active criterion card reacts visibly the moment a rating is tapped, the transition to the next card reads as fluid and directional rather than an abrupt cut, and rating buttons are large, high-contrast targets — Fitts's Law again, the same principle already applied to the navigation bar's `+` button (§5). Haptic feedback on card transitions is a mobile-only affordance (no reliable equivalent exists in a web browser) — an enhancement where the platform supports it, not a requirement web has to fake.

**Celebration is deliberately reserved for genuinely special moments, not spent on every interaction.** Completing a check-in, earning a badge, and becoming a venue's "first discoverer" each warrant a distinct, visually celebratory UI moment — consistent with badges being real, permanent status (§9), not routine game-loop noise. Everything outside those three stays calm and fluid rather than competing for the same attention: treating every tap as an occasion cheapens the ones that actually are one. None of this is manipulative engagement engineering (no streaks, no loss-aversion mechanics, no manufactured urgency — deliberately ruled out, §9) — it's proven interaction-design and perception research (the same class of finding already cited above) applied in service of a product that feels well-crafted to use, not one engineered to be hard to put down.

## 18. Open Decisions

The following decisions are not yet finalized and will be evaluated once initial user data comes in.

| Decision | Expected Data |
|-------|--------------|
| Calibration of aggregate rating thresholds | First 500 check-ins |
| Whether a venue should carry more than one category (a bar/pub hybrid, a kebapçı that also does lahmacun) — see [ADR-0013](https://github.com/oburapp/obur-docs/blob/main/adr/0013-venue-category-taxonomy-is-format-only.md) | Real venue data. Answerable only once the catalog meets actual venues: how often a single format genuinely fails to describe a place, and whether the misses are format ambiguity or menu breadth |
| Data volume at which the "You Might Like" shelf activates | 1000+ check-ins |
| Badge rarity calculation period | First active user cohort |
| TRY-based pricing (future revenue model) | After user growth |
| Timing of second-city expansion | After reaching critical mass in Istanbul |
| Meilisearch migration threshold | Once PostgreSQL FTS hits a performance wall |
| Fake-account / fraud-detection tooling (phone verification, IP correlation, etc.) | After real abuse is actually observed — see §17 |
| Database-level isolation: does Railway's Postgres encrypt at rest by default, and is it network-reachable only from the backend or directly from the outside? | Research Railway's actual guarantees before launch — see §17 |
| Row Level Security (RLS) policies, table by table | Not a "revisit later" item like the rows above — evaluated per query/endpoint as the backend is actually built, see §17 |
| Accessibility (screen readers, color contrast, keyboard navigation) | Not addressed at MVP — real long-term concern, but not urgent enough to block launch; revisit once core product is stable |

### Needs Legal Research, Not User Data

Unlike the table above, these aren't waiting on product usage — they need an actual lawyer, not more check-ins:

- **Turkish "sosyal ağ sağlayıcı" obligations** (Law No. 5651 and the Law No. 7253 amendments — local representative appointment, content-removal response-time requirements, etc.) apply above certain platform-size thresholds. Whether/when Obur crosses into scope at bootstrap stage (~200 MAU target) is genuinely unknown here and needs real legal consultation before launch, not an assumption either way.
- **The Content Policy text itself** (§11) — what specifically counts as prohibited content on Obur — needs legal review before publishing, same as Terms of Service and a Privacy Policy do.
- Whether GDPR applies at all (depends on whether/when any EU users are served) alongside KVKK, which applies regardless given the Turkey-first user base.

---

*This document is a living document. Relevant sections must be updated as product decisions change.*
