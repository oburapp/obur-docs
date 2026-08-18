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
    string username
    string email
    string bio
    string avatar_url
    string city
    char country_code
    string locale
    string timezone
    string role
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
    char country_code
    string timezone
    string status
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
  USER ||--o{ CHECKIN : "creates"
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
  LIST ||--o{ LIST_ITEM : "contains"
  LIST ||--o{ LIST_LIKE : "receives"
  LIST ||--o{ LIST_BOOKMARK : "receives"
  BADGE ||--o{ USER_BADGE : "awarded as"
  BADGE ||--o{ BADGE_TRANSLATION : "translated by"
```

---

## Design Decisions

**Coordinate-based venue identity.** `lat` and `lng` are the primary identifier for a venue, not the name. Duplicate detection queries for existing venues within a 50-metre radius before allowing a new insert.

**`VENUE.location` is database-generated, not application-maintained.** It's a PostGIS `geography` computed from `lat`/`lng` via `GENERATED ALWAYS AS (...) STORED`, so it can never drift out of sync with the columns it's derived from. It backs the 50-metre duplicate check (`ST_DWithin`).

**Venue name search uses a `pg_trgm` trigram index on `name`, not full-text search.** See [ADR-0003](../adr/0003-trigram-venue-name-search.md): trigram similarity is language-agnostic and typo-tolerant, which a language-specific `tsvector` config isn't — a requirement given venue names appear in many languages and Obur's global-expansion plans.

**CHECKIN and CHECKIN_PRODUCT are separated.** A single check-in can contain multiple products (a user may eat several dishes in one visit). Each product carries its own rating. Venue-level ratings (service, ambiance, value) are stored once per check-in.

**Historical data is never deleted.** When a venue closes, `status` is set to `closed`. When a product is removed from a menu, `is_available` is set to `false`. Past check-ins remain visible as an archive. A check-in itself is never truly deleted either — `CHECKIN.deleted_at` marks it removed without dropping the row, so a badge or aggregate already computed from it can't be retroactively corrupted. Only a separate admin action can permanently purge one, for moderation/takedown cases.

**A check-in's own fields are editable; its rated products are not.** See [ADR-0005](../adr/0005-checkin-fields-editable-products-immutable.md): fixing a wrong product or rating means deleting the check-in and creating a new one, not editing it in place — verified against how comparable apps (Untappd) handle this.

**`visibility` is a shared three-tier field (`public` / `close_friends` / `private`) on `CHECKIN`, `LIST`, and `VENUE_SAVE` alike.** See [ADR-0006](../adr/0006-three-tier-visibility-and-close-friends.md), which supersedes [ADR-0004](../adr/0004-checkin-visibility-single-toggle.md)'s single boolean: a "followers-only" tier was considered and rejected, since Obur's one-directional, no-approval follow model gives it no real access-control meaning — anyone can grant themselves "follower" status just by following. `close_friends` is a manually curated allow-list instead (see `CLOSE_FRIEND` below), modeled on Letterboxd's real-world equivalent. `CHECKIN`/`LIST` default to `public`; `VENUE_SAVE` defaults to `private` — saving a venue is a personal tracking action first.

**`CLOSE_FRIEND` requires an existing `FOLLOW` row and is revoked automatically when that follow is removed.** `(friend_id, user_id)` is a composite foreign key into `FOLLOW.(follower_id, following_id)` with `ON DELETE CASCADE` — a single database constraint that both enforces "must currently be a follower to be added" and guarantees a close friend can never outlive the follow relationship it was built on. Not symmetric: `A` adding `B` as a close friend has no bearing on whether `B` has done the same for `A`.

**Likes are a visible signal; bookmarks are always private — kept as separate tables, not a shared table with a mode flag.** `CHECKIN_LIKE`/`LIST_LIKE` are public counts anyone can see (subject to the target's own visibility); `CHECKIN_BOOKMARK`/`LIST_BOOKMARK` are personal save-for-later notes nobody but the bookmarker can ever see — there is deliberately no "how many bookmarked this" count anywhere. See [ADR-0006](../adr/0006-three-tier-visibility-and-close-friends.md). Liking or bookmarking something is only possible if the actor can already see it.

**`LIST_ITEM.position` is a fractional-indexing string, not an integer `order`.** See [ADR-0007](../adr/0007-fractional-indexing-for-list-ordering.md): inserting, moving, or removing an item writes only that one row, never renumbering neighbors — and critically requires `COLLATE "C"` (byte-wise ordering), since the algorithm's correctness silently breaks under a locale-aware default collation.

**`NOTIFICATION` is created synchronously, in the same transaction as the event it describes — no queue, no worker.** See [ADR-0008](../adr/0008-synchronous-in-app-notifications.md). `read_at` lives on the backend row, not per-device, so read-state is automatically consistent across every client. `target_type`/`target_id` is deliberately not a real foreign key, unlike `CHECKIN_LIKE`/`CHECKIN_BOOKMARK` — a notification is a transient record, not data whose own correctness depends on the target still existing.

**Authorization has two layers: ownership, and an admin override.** A user may act on their own resources; `USER.role = 'admin'` may act on anyone's. The same rule — and the same `can_view` function — governs `CHECKIN`, `LIST`, and `VENUE_SAVE` visibility checks alike. `role` is never settable through any user-facing endpoint or the Clerk webhook — the first admin account is set directly in the database, by hand.

**`visited_at` and `created_at` are separate fields.** A user may log a visit days after it occurred. Badge calculations use `visited_at`. The `visited_tz` field stores the IANA timezone at the time of visit for correct local-time calculations.

**Global-ready by design.** All timestamps stored in UTC. User-facing labels (`VENUE_CATEGORY`, `GLOBAL_PRODUCT_TYPE`, `BADGE`) live in separate translation tables keyed by BCP 47 locale. Adding a new language requires only new translation rows. `slug` fields provide language-independent references in code. `country_code` follows ISO 3166-1 alpha-2, `locale` follows BCP 47, `timezone` follows IANA tz database.

**Auth identity is provider-agnostic by field name.** `USER.auth_provider` ("clerk" today) and `auth_provider_id` (that provider's user ID) are deliberately not named after Clerk specifically. Application code outside the auth module never sees provider-specific types — only this pair of columns. Switching or adding an auth provider later needs new rows, not a rename migration. `UNIQUE (auth_provider, auth_provider_id)`.

**Translation tables over embedded strings.** `VENUE_CATEGORY`, `GLOBAL_PRODUCT_TYPE`, and `BADGE` store only a `slug` and metadata. All display names live in `*_TRANSLATION` tables. This avoids duplication and allows the same category system to serve multiple languages without schema changes.