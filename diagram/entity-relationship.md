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
    timestamp created_at
  }
  FOLLOW {
    uuid follower_id FK
    uuid following_id FK
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
    bool is_public
    date visited_at
    string visited_tz
    timestamp created_at
  }
  CHECKIN_PRODUCT {
    uuid id PK
    uuid checkin_id FK
    uuid product_id FK
    int rating
  }
  LIKE {
    uuid user_id FK
    uuid checkin_id FK
    timestamp created_at
  }
  LIST {
    uuid id PK
    uuid user_id FK
    string title
    string description
    bool is_public
    timestamp created_at
  }
  LIST_ITEM {
    uuid id PK
    uuid list_id FK
    uuid venue_id FK
    int order
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
  USER ||--o{ CHECKIN : "creates"
  USER ||--o{ LIKE : "likes"
  USER ||--o{ USER_BADGE : "earns"
  USER ||--o{ VENUE : "adds"
  USER ||--o{ VENUE_SAVE : "saves"
  USER ||--o{ LIST : "creates"
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
  CHECKIN ||--o{ LIKE : "receives"
  LIST ||--o{ LIST_ITEM : "contains"
  BADGE ||--o{ USER_BADGE : "awarded as"
  BADGE ||--o{ BADGE_TRANSLATION : "translated by"
```

---

## Design Decisions

**Coordinate-based venue identity.** `lat` and `lng` are the primary identifier for a venue, not the name. Duplicate detection queries for existing venues within a 50-metre radius before allowing a new insert.

**CHECKIN and CHECKIN_PRODUCT are separated.** A single check-in can contain multiple products (a user may eat several dishes in one visit). Each product carries its own rating. Venue-level ratings (service, ambiance, value) are stored once per check-in.

**Historical data is never deleted.** When a venue closes, `status` is set to `closed`. When a product is removed from a menu, `is_available` is set to `false`. Past check-ins remain visible as an archive.

**`visited_at` and `created_at` are separate fields.** A user may log a visit days after it occurred. Badge calculations use `visited_at`. The `visited_tz` field stores the IANA timezone at the time of visit for correct local-time calculations.

**Global-ready by design.** All timestamps stored in UTC. User-facing labels (`VENUE_CATEGORY`, `GLOBAL_PRODUCT_TYPE`, `BADGE`) live in separate translation tables keyed by BCP 47 locale. Adding a new language requires only new translation rows. `slug` fields provide language-independent references in code. `country_code` follows ISO 3166-1 alpha-2, `locale` follows BCP 47, `timezone` follows IANA tz database.

**Auth identity is provider-agnostic by field name.** `USER.auth_provider` ("clerk" today) and `auth_provider_id` (that provider's user ID) are deliberately not named after Clerk specifically. Application code outside the auth module never sees provider-specific types — only this pair of columns. Switching or adding an auth provider later needs new rows, not a rename migration. `UNIQUE (auth_provider, auth_provider_id)`.

**Translation tables over embedded strings.** `VENUE_CATEGORY`, `GLOBAL_PRODUCT_TYPE`, and `BADGE` store only a `slug` and metadata. All display names live in `*_TRANSLATION` tables. This avoids duplication and allows the same category system to serve multiple languages without schema changes.