# Entity-Relationship Diagram

Entity-relationship diagram of the Obur data model.
Global-ready design: translation tables for i18n, UTC timestamps, ISO-standard field codes.

Last updated: August 2026

```mermaid
erDiagram
  USER {
    uuid id PK
    string auth_provider
    string auth_provider_id
    string display_name
    string username
    string email
    string bio
    string avatar_url
    string city
    char country_code
    string locale
    string timezone
    string role
    string status
    timestamp created_at
  }
  FOLLOW {
    uuid follower_id FK
    uuid following_id FK
    timestamp created_at
  }
  CLOSE_FRIEND {
    uuid user_id FK
    uuid friend_id FK
    timestamp created_at
  }
  MUTE {
    uuid user_id FK
    uuid muted_id FK
    timestamp created_at
  }
  VENUE_CATEGORY {
    uuid id PK
    string slug
    uuid parent_id FK
  }
  VENUE_CATEGORY_TRANSLATION {
    uuid category_id FK
    string locale
    string name
  }
  VENUE {
    uuid id PK
    string name
    float lat
    float lng
    geography location
    string address_note
    string google_places_id
    uuid added_by FK
    uuid category_id FK
    string city
    string district
    char country_code
    string timezone
    bool is_active
    bool is_verified
    bool is_suspended
    timestamp created_at
  }
  VENUE_SAVE {
    uuid id PK
    uuid user_id FK
    uuid venue_id FK
    string type
    string visibility
    timestamp created_at
  }
  GLOBAL_PRODUCT_TYPE {
    uuid id PK
    string slug
    uuid category_id FK
  }
  GLOBAL_PRODUCT_TYPE_TRANSLATION {
    uuid product_type_id FK
    string locale
    string name
  }
  PRODUCT {
    uuid id PK
    uuid venue_id FK
    uuid global_type_id FK
    string name
    bool is_available
    timestamp created_at
  }
  CHECKIN {
    uuid id PK
    uuid user_id FK
    uuid venue_id FK
    int rating_service
    int rating_ambiance
    int rating_value
    string note
    string photo_url
    string visibility
    date visited_at
    string visited_tz
    timestamp deleted_at
    string idempotency_key
    timestamp created_at
  }
  CHECKIN_DRAFT {
    uuid id PK
    uuid user_id FK
    uuid venue_id FK
    int rating_service
    int rating_ambiance
    int rating_value
    string note
    string photo_url
    date visited_at
    timestamp updated_at
    timestamp created_at
  }
  CHECKIN_PRODUCT {
    uuid id PK
    uuid checkin_id FK
    uuid product_id FK
    int rating
  }
  CHECKIN_LIKE {
    uuid user_id FK
    uuid checkin_id FK
    timestamp created_at
  }
  CHECKIN_BOOKMARK {
    uuid user_id FK
    uuid checkin_id FK
    timestamp created_at
  }
  CHECKIN_MENTION {
    uuid id PK
    uuid checkin_id FK
    uuid mentioned_user_id FK
    timestamp created_at
  }
  HASHTAG {
    uuid id PK
    string tag
    timestamp created_at
  }
  CHECKIN_HASHTAG {
    uuid checkin_id FK
    uuid hashtag_id FK
  }
  LIST_HASHTAG {
    uuid list_id FK
    uuid hashtag_id FK
  }
  LIST {
    uuid id PK
    uuid user_id FK
    string title
    string description
    string visibility
    timestamp created_at
  }
  LIST_ITEM {
    uuid id PK
    uuid list_id FK
    uuid venue_id FK
    string position
    timestamp created_at
  }
  LIST_LIKE {
    uuid user_id FK
    uuid list_id FK
    timestamp created_at
  }
  LIST_BOOKMARK {
    uuid user_id FK
    uuid list_id FK
    timestamp created_at
  }
  NOTIFICATION {
    uuid id PK
    uuid user_id FK
    string type
    uuid actor_id FK
    string target_type
    uuid target_id
    timestamp read_at
    timestamp created_at
  }
  BADGE {
    uuid id PK
    string slug
    string tier
    float rarity_pct
  }
  BADGE_TRANSLATION {
    uuid badge_id FK
    string locale
    string name
    string description
  }
  USER_BADGE {
    uuid user_id FK
    uuid badge_id FK
    bool is_pinned
    timestamp earned_at
  }

  USER ||--o{ FOLLOW : "follows"
  USER ||--o{ CLOSE_FRIEND : "curates"
  USER ||--o{ MUTE : "mutes"
  USER ||--o{ CHECKIN : "creates"
  USER ||--o{ CHECKIN_DRAFT : "drafts"
  VENUE ||--o{ CHECKIN_DRAFT : "may reference"
  USER ||--o{ CHECKIN_LIKE : "likes"
  USER ||--o{ CHECKIN_BOOKMARK : "bookmarks"
  USER ||--o{ LIST_LIKE : "likes"
  USER ||--o{ LIST_BOOKMARK : "bookmarks"
  USER ||--o{ NOTIFICATION : "receives"
  USER ||--o{ USER_BADGE : "earns"
  USER ||--o{ VENUE : "adds"
  USER ||--o{ VENUE_SAVE : "saves"
  USER ||--o{ LIST : "creates"
  FOLLOW ||--o| CLOSE_FRIEND : "required by"
  VENUE ||--o{ CHECKIN : "contains"
  VENUE ||--o{ PRODUCT : "offers"
  VENUE ||--o{ VENUE_SAVE : "saved by"
  VENUE ||--o{ LIST_ITEM : "listed in"
  VENUE_CATEGORY ||--o{ VENUE : "classifies"
  VENUE_CATEGORY ||--o{ VENUE_CATEGORY : "parent of"
  VENUE_CATEGORY ||--o{ GLOBAL_PRODUCT_TYPE : "groups"
  VENUE_CATEGORY ||--o{ VENUE_CATEGORY_TRANSLATION : "translated by"
  GLOBAL_PRODUCT_TYPE ||--o{ PRODUCT : "defines"
  GLOBAL_PRODUCT_TYPE ||--o{ GLOBAL_PRODUCT_TYPE_TRANSLATION : "translated by"
  PRODUCT ||--o{ CHECKIN_PRODUCT : "rated in"
  CHECKIN ||--o{ CHECKIN_PRODUCT : "contains"
  CHECKIN ||--o{ CHECKIN_LIKE : "receives"
  CHECKIN ||--o{ CHECKIN_BOOKMARK : "receives"
  CHECKIN ||--o{ CHECKIN_MENTION : "mentions in"
  USER ||--o{ CHECKIN_MENTION : "is mentioned via"
  CHECKIN ||--o{ CHECKIN_HASHTAG : "tagged with"
  HASHTAG ||--o{ CHECKIN_HASHTAG : "applied to"
  HASHTAG ||--o{ LIST_HASHTAG : "applied to"
  LIST ||--o{ LIST_ITEM : "contains"
  LIST ||--o{ LIST_LIKE : "receives"
  LIST ||--o{ LIST_BOOKMARK : "receives"
  LIST ||--o{ LIST_HASHTAG : "tagged with"
  BADGE ||--o{ USER_BADGE : "awarded as"
  BADGE ||--o{ BADGE_TRANSLATION : "translated by"
```

---

## Design Decisions

**Coordinate-based venue identity, with a stronger identity signal layered on top where available.** `lat` and `lng` are the primary identifier for a venue, not the name. Duplicate detection is two layers, not one (see [ADR-0009](../adr/0009-venue-discovery-enrichment.md)): an exact `google_places_id` match (unique where not null) resolves silently to the existing venue with no prompt; everything else falls back to the original 50-metre radius check, which still prompts "did you mean this one?" `district` is required for every venue from ADR-0009 forward (nullable only for pre-existing rows), since `city` alone can't scope discovery/badges/rankings to something like "Kadıköy." `is_verified` is a separate, purely cosmetic signal ("this location definitely exists here") that never affects ranking or discoverability — set via a Google-match-plus-N-check-ins path, or a check-ins-plus-admin-review path where both conditions are required, never a review-everything model.

**No user edits a venue directly, not even whoever added it — every correction is report-driven, admin-only.** `is_active` and `is_suspended` replace what was originally a single `status` string: `is_active = false` is a closed business, still shown transparently as an archived record; `is_suspended = true` is an admin moderation action that hides the venue entirely, its own page returning a generic "not found" rather than an explanation — consistent with how a blocked profile behaves. Both flip only through an admin acting on a report, never directly by any user, on a real abuse precedent from an unrelated app's community-edit feature.

**`VENUE.location` is database-generated, not application-maintained.** It's a PostGIS `geography` computed from `lat`/`lng` via `GENERATED ALWAYS AS (...) STORED`, so it can never drift out of sync with the columns it's derived from. It backs the 50-metre duplicate check (`ST_DWithin`).

**Venue name search uses a `pg_trgm` trigram index on `name`, not full-text search.** See [ADR-0003](../adr/0003-trigram-venue-name-search.md): trigram similarity is language-agnostic and typo-tolerant, which a language-specific `tsvector` config isn't — a requirement given venue names appear in many languages and Obur's global-expansion plans.

**CHECKIN and CHECKIN_PRODUCT are separated.** A single check-in can contain multiple products (a user may eat several dishes in one visit). Each product carries its own rating. Venue-level ratings (service, ambiance, value) are stored once per check-in, required, not optional — every check-in guarantees the same three data points toward the venue's aggregate regardless of how many products were rated.

**Aggregate scores are computed at two distinct, deliberately separate levels, both using the same statistical procedure.** A product-level score (pooling only `CHECKIN_PRODUCT.rating`) stays pure food-quality-only, since it's the sort key for cross-venue product ranking (a venue's slow service shouldn't drag down its food's ranking against other venues). A venue-level headline score (pooling `CHECKIN_PRODUCT.rating` together with `rating_service`/`rating_ambiance`/`rating_value`) is the one number shown at the top of a venue's own page, where no cross-venue comparison is happening and one combined "how was visiting this place" number matches real-world precedent (Google Maps, Yelp) better than two competing scores would. Both levels compute a 95%-confidence lower bound on the mean (the same principle behind the Wilson score interval Reddit/Yelp use, chosen over a Bayesian/IMDb-style approach specifically because it stays anchored to the pool's own data instead of diluting toward an unrelated platform-wide average) rather than a raw mean, below a 10-rating floor beneath which no label is shown at all.

**Historical data is never deleted.** When a venue closes, `is_active` is set to `false` — it stays visible, shown transparently as inactive, not hidden or removed. When a product is removed from a menu, `is_available` is set to `false`. Past check-ins remain visible as an archive. A check-in itself is never truly deleted either — `CHECKIN.deleted_at` marks it removed without dropping the row, so a badge or aggregate already computed from it can't be retroactively corrupted. Only a separate admin action can permanently purge one, for moderation/takedown cases.

**A check-in's own fields are editable; its rated products are not.** See [ADR-0005](../adr/0005-checkin-fields-editable-products-immutable.md): fixing a wrong product or rating means deleting the check-in and creating a new one, not editing it in place — verified against how comparable apps (Untappd) handle this.

**`CHECKIN_DRAFT` is a separate table from `CHECKIN`, not an `is_draft` flag on it.** A flag would require every aggregate-rating, badge, and feed query to remember to filter it out, forever — this platform has already hit that exact failure mode twice (an existence-leak and a stale-visibility bug, both "a query somewhere forgot to filter"). A separate table makes a draft leaking into rating/badge/feed logic structurally impossible instead of a matter of developer discipline. It's server-synced (not device-local) so a draft started on mobile is resumable on web, the same cross-device-consistency standard `NOTIFICATION.read_at` already set. On submit, its data becomes a real `CHECKIN` row (via `idempotency_key`, guarding against duplicate creation on a retried submission) and the draft row is deleted.

**`visibility` is a shared three-tier field (`public` / `close_friends` / `private`) on `CHECKIN`, `LIST`, and `VENUE_SAVE` alike.** See [ADR-0006](../adr/0006-three-tier-visibility-and-close-friends.md), which supersedes [ADR-0004](../adr/0004-checkin-visibility-single-toggle.md)'s single boolean: a "followers-only" tier was considered and rejected, since Obur's one-directional, no-approval follow model gives it no real access-control meaning — anyone can grant themselves "follower" status just by following. `close_friends` is a manually curated allow-list instead (see `CLOSE_FRIEND` below), modeled on Letterboxd's real-world equivalent. `CHECKIN`/`LIST` default to `public`; `VENUE_SAVE` defaults to `private` — saving a venue is a personal tracking action first.

**`CLOSE_FRIEND` requires an existing `FOLLOW` row and is revoked automatically when that follow is removed.** `(friend_id, user_id)` is a composite foreign key into `FOLLOW.(follower_id, following_id)` with `ON DELETE CASCADE` — a single database constraint that both enforces "must currently be a follower to be added" and guarantees a close friend can never outlive the follow relationship it was built on. Not symmetric: `A` adding `B` as a close friend has no bearing on whether `B` has done the same for `A`.

**`MUTE` is deliberately not derived from `FOLLOW`, unlike `CLOSE_FRIEND`.** A user can mute anyone, followed or not — muting a stranger keeps their content out of Layer 2's algorithmic fill (§12) too, not just Layer 1. It's the lighter counterpart to blocking: one-directional, silent, and scoped to feed display only — the follow relationship, visibility, discoverability, likes, bookmarks, and notifications are all untouched. Where blocking is interpersonal-safety tooling, mute is a feed preference.

**Likes are a visible signal; bookmarks are always private — kept as separate tables, not a shared table with a mode flag.** `CHECKIN_LIKE`/`LIST_LIKE` are public counts anyone can see (subject to the target's own visibility); `CHECKIN_BOOKMARK`/`LIST_BOOKMARK` are personal save-for-later notes nobody but the bookmarker can ever see — there is deliberately no "how many bookmarked this" count anywhere. See [ADR-0006](../adr/0006-three-tier-visibility-and-close-friends.md). Liking or bookmarking something is only possible if the actor can already see it.

**`CHECKIN_MENTION` only exists between mutual followers, and never overrides the mentioned check-in's own visibility.** A structured table, not text parsed out of `CHECKIN.note` at render time, so mention creation can enforce the mutual-follow requirement, trigger a notification the same way `CHECKIN_LIKE` does, and get purged retroactively the moment the two people block each other — the same cleanup already applied to likes, bookmarks, and notifications. Requiring mutual following (not just "the author follows them," which the one-directional `FOLLOW` model would otherwise allow unilaterally) keeps mentions inside an already-established two-sided relationship, avoiding the unwanted-attention vector the platform's no-comment/no-DM stance already exists to prevent. Being mentioned in a `private`/`close_friends` check-in notifies, but doesn't grant access to see it — extending visibility this way would be a backdoor around the same authorization system kept airtight everywhere else in this document.

**`HASHTAG` is free text, shared across `CHECKIN` and `LIST` via join tables, capped at 5 per item at the application layer.** No relationship requirement, unlike a mention — a hashtag doesn't target a person. `HASHTAG.tag` is stored in a Turkish-aware normalized form, not a naive lowercase: Turkish's dotted/dotless İ-I distinction means a locale-naive case-fold can silently produce two different rows for what should be the same tag, the same class of subtle text bug `LIST_ITEM.position`'s `COLLATE "C"` requirement already caught (ADR-0007). The 5-item cap exists to keep hashtag-based discovery meaningful, not to limit expression — an unbounded list would let one post crowd dozens of unrelated discovery pages.

**`LIST_ITEM.position` is a fractional-indexing string, not an integer `order`.** See [ADR-0007](../adr/0007-fractional-indexing-for-list-ordering.md): inserting, moving, or removing an item writes only that one row, never renumbering neighbors — and critically requires `COLLATE "C"` (byte-wise ordering), since the algorithm's correctness silently breaks under a locale-aware default collation.

**`NOTIFICATION` is created synchronously, in the same transaction as the event it describes — no queue, no worker.** See [ADR-0008](../adr/0008-synchronous-in-app-notifications.md). `read_at` lives on the backend row, not per-device, so read-state is automatically consistent across every client. `target_type`/`target_id` is deliberately not a real foreign key, unlike `CHECKIN_LIKE`/`CHECKIN_BOOKMARK` — a notification is a transient record, not data whose own correctness depends on the target still existing.

**Authorization has two layers: ownership, and an admin override.** A user may act on their own resources; `USER.role = 'admin'` may act on anyone's. The same rule — and the same `can_view` function — governs `CHECKIN`, `LIST`, and `VENUE_SAVE` visibility checks alike. `role` is never settable through any user-facing endpoint or the Clerk webhook — the first admin account is set directly in the database, by hand.

**`display_name` and `username` are two separate fields, not one.** `display_name` is what's shown everywhere and can be changed freely, with no uniqueness constraint. `username` is the unique handle that search, mentions, and profile URLs key off of — edits to it are rate-limited, since an unrestricted handle is an impersonation vector in a way a display name isn't.

**`USER.status` (`active | frozen | suspended`) is standing, kept separate from `role`, which is permission.** `frozen` is self-service and reversible — a user freezes their own account and reactivates it just by logging back in. `suspended` is admin-only, set only by resolving a report (see the blocking/reporting decision above), and never user-reversible. Neither value is reachable by the other actor — a user can't suspend anyone, an admin doesn't need to freeze accounts.

**Account deletion is permanent, not a `status` value or a soft-delete — the one deliberate exception to "historical data is never deleted."** It purges every check-in (and its `CHECKIN_PRODUCT`/`CHECKIN_LIKE`/`CHECKIN_BOOKMARK` rows), every list and `LIST_ITEM`, and every `VENUE_SAVE` belonging to the account — reusing the same permanent-purge mechanism an admin takedown already needed for individual check-ins, just user-triggered instead. A venue the account added is not personal content, so it isn't deleted with it: `VENUE.added_by` is set to `NULL` (`ON DELETE SET NULL`) rather than left dangling or cascaded. Chosen over a softer "anonymize but keep the content" approach — the model blocking uses for display only — because once a user asks to be forgotten, the data itself should be gone, not just its attribution.

**`visited_at` and `created_at` are separate fields.** A user may log a visit days after it occurred. Badge calculations use `visited_at`. The `visited_tz` field stores the IANA timezone at the time of visit for correct local-time calculations.

**Global-ready by design.** All timestamps stored in UTC. User-facing labels (`VENUE_CATEGORY`, `GLOBAL_PRODUCT_TYPE`, `BADGE`) live in separate translation tables keyed by BCP 47 locale. Adding a new language requires only new translation rows. `slug` fields provide language-independent references in code. `country_code` follows ISO 3166-1 alpha-2, `locale` follows BCP 47, `timezone` follows IANA tz database.

**Auth identity is provider-agnostic by field name.** `USER.auth_provider` ("clerk" today) and `auth_provider_id` (that provider's user ID) are deliberately not named after Clerk specifically. Application code outside the auth module never sees provider-specific types — only this pair of columns. Switching or adding an auth provider later needs new rows, not a rename migration. `UNIQUE (auth_provider, auth_provider_id)`.

**Translation tables over embedded strings.** `VENUE_CATEGORY`, `GLOBAL_PRODUCT_TYPE`, and `BADGE` store only a `slug` and metadata. All display names live in `*_TRANSLATION` tables. This avoids duplication and allows the same category system to serve multiple languages without schema changes.