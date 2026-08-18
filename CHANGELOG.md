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

### Changed

- Entity-relationship diagram: added `VENUE.location` (generated PostGIS
  geography), `USER.role`, and `CHECKIN.deleted_at`; replaced the stale
  `search_vector` reference with a note pointing to ADR-0003
- PDD: `USER` and `CHECKIN` schema blocks updated to match (`role`,
  `deleted_at`, the `CHECKIN_PRODUCT` uniqueness constraint); Check-in
  Flow §10 Step 5 rewritten to describe the single visibility toggle
  instead of the dropped second toggle
