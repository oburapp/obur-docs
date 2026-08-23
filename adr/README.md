# Architecture Decision Records

This directory contains Obur's Architecture Decision Records (ADRs). An ADR
captures a significant architectural decision, the context behind it, and the
alternatives that were considered.

See [CLAUDE.md](../CLAUDE.md#adr-architecture-decision-records) for
the template and process. An ADR is written at the moment the decision is
made — not drafted ahead of time.

## Naming

`NNNN-short-title.md`, zero-padded and sequential.

## Index

| ID | Title | Status |
|----|-------|--------|
| [0001](0001-test-database-strategy.md) | Test Database Strategy | Accepted |
| [0002](0002-defer-google-places-verification.md) | Defer Google Places Verification | Superseded by [0009](0009-venue-discovery-enrichment.md) |
| [0003](0003-trigram-venue-name-search.md) | Trigram-Based Venue Name Search | Accepted |
| [0004](0004-checkin-visibility-single-toggle.md) | Single Visibility Toggle for Check-ins | Superseded by [0006](0006-three-tier-visibility-and-close-friends.md) |
| [0005](0005-checkin-fields-editable-products-immutable.md) | Check-in Fields Are Editable, Rated Products Are Not | Superseded by [0011](0011-drop-product-layer-four-venue-criteria.md) |
| [0006](0006-three-tier-visibility-and-close-friends.md) | Three-Tier Visibility, Close Friends, and Bookmarks as a Private Signal | Accepted |
| [0007](0007-fractional-indexing-for-list-ordering.md) | Fractional Indexing for List Item Ordering | Accepted |
| [0008](0008-synchronous-in-app-notifications.md) | Light, Synchronous In-App Notifications | Accepted |
| [0009](0009-venue-discovery-enrichment.md) | Venue Discovery Enrichment: District, Duplicate Detection, and Verification | Accepted (not yet implemented) |
| [0010](0010-blocking-and-reporting-schema.md) | Blocking and Reporting Schema | Accepted (not yet implemented) |
| [0011](0011-drop-product-layer-four-venue-criteria.md) | Drop the Product Layer, Rate Venues on Four Criteria | Accepted (supersedes [0005](0005-checkin-fields-editable-products-immutable.md)) |
| [0012](0012-migrations-are-self-contained-reference-data-is-seeded.md) | Migrations Are Self-Contained; Reference Data Belongs to a Seeder | Accepted |
