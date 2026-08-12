# Obur — Product Design Document (PDD)

**Version:** 1.0  
**Date:** August 2026  
**Status:** Active  
**Owners:** Mehmet Eren Reyhanlıoğlu + [Friend]

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
13. [Venue and Product Architecture](#13-venue-and-product-architecture)
14. [Tech Stack](#14-tech-stack)
15. [Infrastructure and Cost](#15-infrastructure-and-cost)
16. [Go-to-Market Strategy](#16-go-to-market-strategy)
17. [Open Decisions](#17-open-decisions)


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
- Product-level rating: the only platform that actually answers "where's the best döner"
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

### Default Visibility: Public

To encourage content creation and keep the feed alive, check-in visibility defaults to public. Users can switch it to private if they want; private check-ins still contribute to aggregate statistics.

### Business Hours Out of Scope

Business hours are operational data with no bearing on the discovery experience Obur provides. Stale hours that must be kept up to date send users to a closed venue and cost the platform trust. Verification is left to Google Maps.

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
│   ├── Search (returns venue / product / user / list)
│   └── Map view (opened from list or discover)
├── + Modal
│   ├── Check-in creation flow (5 steps)
│   └── List creation
├── Notifications
├── Profile
│   ├── Check-ins tab
│   ├── Favorites tab
│   ├── Lists tab
│   ├── Achievements tab
│   └── Map tab (venues visited)
├── Venue page
│   ├── Products tab
│   ├── Check-ins tab
│   └── Photos tab
└── Product page
    ├── Check-ins at this venue
    └── Where else has this product been liked?
```

### Search Behavior

The same search box returns different results depending on the current page:

- **Search on Feed:** searches among check-ins from followed users
- **Search on Discover:** returns venues, products, users, and lists; can be narrowed with filter chips

---

## 6. Feature Catalog

### MVP (v1.0)

| Feature | Description | Priority |
|---------|----------|---------|
| Sign up / Sign in | Email + social login (Clerk) | P0 |
| Follow users | One-directional, no approval | P0 |
| Create check-in | 5-step flow | P0 |
| Feed | Chronological check-ins and lists from followed users | P0 |
| Discover | Venue / product / user search, city filter | P0 |
| Venue page | Aggregate rating, products, check-ins | P0 |
| Product page | Check-ins at that venue + same product ranked across other venues | P0 |
| Profile | Check-ins, favorites, lists, achievements | P0 |
| Badge system | Bronze / silver / gold tiers, rarity display | P0 |
| List creation | User curation, map display | P1 |
| Save venue | Been / want to go / favorite | P1 |
| Map view | Profile- and list-based | P1 |
| Notifications | Like, follow, badge, venue verification | P1 |
| Visibility control | Public / private toggle per check-in | P1 |
| Travel mode | Manual city selector | P1 |

### v2.0 (Next Release)

| Feature | Description |
|---------|-------------|
| "You Might Like" shelf | Venue suggestions based on taste overlap; not enabled until enough data has accumulated |
| Profile suggestions | User suggestions based on shared taste and followers |
| User-driven venue verification | Venue deactivated once multiple reports come in |
| Expansion to a second city | Same "enthusiast first, then organic growth" playbook |

---

## 7. Data Model

### Tables and Fields

```sql
USER
  id            UUID PK
  username      VARCHAR
  email         VARCHAR UNIQUE
  bio           TEXT
  avatar_url    VARCHAR
  city          VARCHAR
  country_code  CHAR(2)                  -- ISO 3166-1 alpha-2: "TR", "US", "DE"
  locale        VARCHAR DEFAULT 'tr'     -- BCP 47: "tr", "en", "de" — UI language
  timezone      VARCHAR                  -- IANA: "Europe/Istanbul", "America/New_York"
  created_at    TIMESTAMPTZ

FOLLOW
  follower_id   UUID FK → USER
  following_id  UUID FK → USER
  created_at    TIMESTAMPTZ
  PRIMARY KEY (follower_id, following_id)

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
  google_places_id VARCHAR               -- optional reference
  added_by      UUID FK → USER
  category_id   UUID FK → VENUE_CATEGORY
  city          VARCHAR                  -- "Istanbul", "New York", "Berlin"
  country_code  CHAR(2)                  -- ISO 3166-1 alpha-2: "TR", "US", "DE"
  timezone      VARCHAR                  -- IANA: "Europe/Istanbul" — for city-based filtering
  status        VARCHAR                  -- active | closed | unconfirmed
  created_at    TIMESTAMPTZ

VENUE_SAVE
  id            UUID PK
  user_id       UUID FK → USER
  venue_id      UUID FK → VENUE
  type          VARCHAR                  -- visited | wishlist | favorite
  created_at    TIMESTAMPTZ

GLOBAL_PRODUCT_TYPE
  id            UUID PK
  slug          VARCHAR UNIQUE           -- language-independent reference: "doner", "filter-coffee"
  category_id   UUID FK → VENUE_CATEGORY

GLOBAL_PRODUCT_TYPE_TRANSLATION
  product_type_id UUID FK → GLOBAL_PRODUCT_TYPE
  locale          VARCHAR                -- "tr", "en"
  name            VARCHAR                -- "Döner", "Doner", "Kebab"
  PRIMARY KEY (product_type_id, locale)

PRODUCT
  id            UUID PK
  venue_id      UUID FK → VENUE
  global_type_id UUID FK → GLOBAL_PRODUCT_TYPE
  name          VARCHAR                  -- venue-specific name, entered by the user
  is_available  BOOLEAN DEFAULT TRUE
  created_at    TIMESTAMPTZ

CHECKIN
  id            UUID PK
  user_id       UUID FK → USER
  venue_id      UUID FK → VENUE
  rating_service  SMALLINT              -- 1-4, nullable
  rating_ambiance SMALLINT              -- 1-4, nullable
  rating_value    SMALLINT              -- 1-4, nullable ("worth it?")
  note          TEXT
  photo_url     VARCHAR
  is_public     BOOLEAN DEFAULT TRUE
  visited_at    DATE                    -- visit date entered by the user
  visited_tz    VARCHAR                 -- IANA timezone at time of visit — used for badge calculation
  created_at    TIMESTAMPTZ             -- log timestamp (UTC)

CHECKIN_PRODUCT
  id            UUID PK
  checkin_id    UUID FK → CHECKIN
  product_id    UUID FK → PRODUCT
  rating        SMALLINT NOT NULL       -- 1-4

LIKE
  user_id       UUID FK → USER
  checkin_id    UUID FK → CHECKIN
  created_at    TIMESTAMPTZ
  PRIMARY KEY (user_id, checkin_id)

LIST
  id            UUID PK
  user_id       UUID FK → USER
  title         VARCHAR
  description   TEXT
  is_public     BOOLEAN DEFAULT TRUE
  created_at    TIMESTAMPTZ

LIST_ITEM
  id            UUID PK
  list_id       UUID FK → LIST
  venue_id      UUID FK → VENUE
  order         INTEGER

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
```

### Key Design Decisions

**Venue identity is built on coordinates.** The name can change, but coordinates are stable data. When a new venue is added, existing venues within a 50-meter radius are checked for duplicate detection.

**CHECKIN and CHECKIN_PRODUCT are separate.** A single check-in can contain multiple products; each product carries its own rating. Venue-level criteria (service, ambiance, worth it) are entered once per check-in.

**Historical data is never deleted.** If a venue closes, `status` becomes `closed` and its check-ins remain as an archive. If a product is removed from the menu, `is_available` becomes `false` — it's no longer suggested for new check-ins but still appears in past records.

**`visited_at` and `created_at` are kept separate.** A user may log last week's meal today. Badge calculations are based on `visited_at`.

**Global-ready design principles.** All timestamps are stored as UTC; the `visited_tz` field records the user's local timezone at the time of the visit and is used in badge calculations. User-facing labels (`VENUE_CATEGORY`, `GLOBAL_PRODUCT_TYPE`, `BADGE`) live in translation tables; adding a new language to the system only requires adding the relevant translation rows. `slug` fields provide a language-independent reference: the code uses `"food"`, and the display resolves independently of language. `country_code` follows ISO 3166-1 alpha-2, `locale` follows BCP 47, `timezone` follows the IANA standard.

---

## 8. Rating System

### User Input (CHECKIN_PRODUCT.rating)

Four options, no neutral option:

| Value | Label |
|-------|-------|
| 4 | Very good |
| 3 | Good |
| 2 | Average |
| 1 | Bad |

An even number of options with no neutral choice pushes the user to pick a real side. This structurally reduces the rating inflation seen in star systems (everyone giving 4-5).

### Venue Criteria (CHECKIN fields, optional)

| Field | Description |
|------|------|
| rating_service | Service quality |
| rating_ambiance | Ambiance (including music, lighting, decor) |
| rating_value | "Worth it?" — not price/performance, but the payoff of the overall experience |

### Aggregate Label (Shown on Venue and Product Pages)

Deliberately uses different vocabulary from user input. The label is determined by a ratio (%) and volume (check-in count) threshold:

| Label | Required Ratio | Min. Check-ins |
|--------|-------------|---------------|
| Excellent | ≥ 90% "very good" | 20+ |
| Nearly excellent | ≥ 80% "very good" | 8+ |
| Very good | ≥ 60% "very good" | 3+ |
| Good | ≥ 50% very good + good | 3+ |
| Above average | ~40-50% positive | 3+ |
| Average | Scattered / mixed | 3+ |
| Below average | ≥ 40% "bad" | 3+ |
| Bad | ≥ 60% "bad" | 3+ |
| *(New / low data)* | — | < 3 |

Thresholds are a starting point; they should be calibrated once initial user data comes in.

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
| First discoverer | Gold | Platform's first check-in at a venue or its products |
| Döner expert | Gold | Döner logged at 20+ different venues |
| Kadıköy explorer | Silver | 30+ different venues in Kadıköy |
| Loyal customer | Silver | 10+ visits to the same venue |
| First step | Bronze | First check-in |
| Social | Bronze | First 5 followers |

### Display Rules

- Users can pin the badges they want on their profile (`is_pinned = true`)
- The "All achievements" section shows every badge earned
- Each badge displays its rarity next to it: "held by 1.2% of users"
- The `rarity_pct` field is periodically recalculated on the backend

---

## 10. Check-in Flow

### Steps

```
Step 1: Choose venue
  → Search box
  → Coordinate-based nearby venue suggestions
  → Not found: add a new venue (name + map pin + address note)

Step 2: Choose product(s)
  → Products previously logged at the selected venue are shown as chips
  → Multiple selectable
  → Searchable
  → Not found: add a new product

Step 3: Rate the products
  → Selected products are shown as a card stack
  → Top card is active, others wait behind it
  → Four large buttons: Bad | Average | Good | Very Good (left to right)
  → Once rated, the card flies upward and the next one appears
  → Cannot proceed until every product is rated
  → Bottom section (optional): venue criteria — service, ambiance, worth it

Step 4: Tell the story
  → Note field: placeholder "what stood out to you the most?"
  → Photo: "add a photo — show us"
  → Visit date: defaults to today, editable

Step 5: Share
  → Summary card: venue, products, and ratings
  → Toggle 1: public (default: on)
  → Toggle 2: contribute to statistics (counts toward the aggregate even when off)
  → Save
```

### Multi-Product Decision

The same experience is packaged as a single check-in. The backend creates one `CHECKIN` record; each product is written as its own row in the `CHECKIN_PRODUCT` join table. The user selects multiple products at once and rates them in sequence — it feels like a single check-in while keeping the data model clean.

---

## 11. Social Graph

**One-directional follow.** No mutual approval required. If user A follows user B, A sees B's public check-ins and lists in their feed. B does not need to follow A back.

**Notification triggers:**

- New follower (with a follow-back button)
- Check-in like
- List like
- Badge earned
- Verification of an added venue

---

## 12. Main Feed Algorithm

### Two-Layer Structure

**Layer 1 — Followed users (primary):** Public check-ins and lists from users the account follows, shown chronologically. Requires no extra algorithm.

**Layer 2 — Algorithmic fill (secondary):** Kicks in once content from followed users falls below a defined threshold of the total feed. It's the cold-start solution for a new user, and for an existing user it keeps the feed alive while they follow few people.

Algorithmic content ranking signals (in priority order):

1. Check-ins with high ratings in categories/products the user rates highly
2. Content from the user's current city
3. Like count

Algorithmic content is visually distinguished: a thin line on the left edge of the card and a "suggested" label.

### Discover Page Ranking

When a user searches "Kadıköy döner":

1. Aggregate rating (primary)
2. Check-in count (secondary)
3. Check-in count among followed users (social signal, "3 friends have been" label)

Among venues with an equal rating, recently logged content ranks higher; venues that have closed or declined in quality naturally sink.

---

## 13. Venue and Product Architecture

### Venue Identity

Identity is built on coordinates, not on name. The `(lat, lng)` pair is the primary identifier. Duplicate detection: when a new venue is added, if another venue exists within a 50-meter radius, the user is asked "did you mean this one?"

For cases like malls where multiple venues can share the same coordinates, the free-text `address_note` field is used ("3rd floor", "Block B entrance").

### Product Hierarchy

```
VENUE_CATEGORY (hierarchical)
  └── food
        └── pide
              └── kuşbaşılı pide ← GLOBAL_PRODUCT_TYPE

PRODUCT (venue-specific)
  venue_id    → Karadeniz Pide
  global_type_id → kuşbaşılı pide
  name        → "Karadeniz Pide — kuşbaşılı pide"
```

This structure answers two questions at once: "what's available at this venue?" and "where's the best kuşbaşılı pide?" The category list is defined by the team initially; users can suggest a new type.

### Difference Between Venue and Product Pages

**Venue page:** every product at that venue with aggregate ratings, service/ambiance/value averages, all check-ins, the first discoverer.

**Product page:** check-ins for that specific product at that venue + how the product ranks across other venues with the same `GLOBAL_PRODUCT_TYPE`. The "first to try this product on the platform" badge is shown here.

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
| Framework | Next.js | SSR for SEO, venue/product pages are crawlable |
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

## 17. Open Decisions

The following decisions are not yet finalized and will be evaluated once initial user data comes in.

| Decision | Expected Data |
|-------|--------------|
| Calibration of aggregate rating thresholds | First 500 check-ins |
| Data volume at which the "You Might Like" shelf activates | 1000+ check-ins |
| Badge rarity calculation period | First active user cohort |
| TRY-based pricing (future revenue model) | After user growth |
| Timing of second-city expansion | After reaching critical mass in Istanbul |
| Meilisearch migration threshold | Once PostgreSQL FTS hits a performance wall |

---

---

*This document is a living document. Relevant sections must be updated as product decisions change.*
