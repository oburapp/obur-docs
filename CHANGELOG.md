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
