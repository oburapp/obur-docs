# ADR-0017: Database Backup Strategy

**Date:** 2026-08-24
**Status:** Accepted (not yet implemented), planned for `obur-backend` Phase 8

## Context

Phase 8's roadmap text does not mention backups at all, an omission surfaced during Phase 8 planning research rather than a deferred-on-purpose item like the ones the roadmap documents elsewhere. Railway's own documentation describes its Postgres offering as **unmanaged**: "you have total control over their configuration and maintenance," and backups are not automatic, the user has to set them up. Without one, a bad migration or an accidental destructive query has no recovery path; the data is gone. This holds at any scale, including pre-launch: from Phase 5 onward the team is writing directly against a real schema during development, and post-launch it's real user data.

Railway does offer a native solution, Point-in-Time Recovery, backing up via `pgBackRest` with roughly a four-week retention window. It is a Pro-plan feature, requiring the $20/month base plan plus its own storage and egress metering. The team is bootstrapped and cost-conscious at this stage, and has already chosen the smallest viable footprint for Phase 8's compute (one Railway replica, one worker, see `docs/deployment.md`) for the same reason.

## Decision

A scheduled backup, not Railway's paid PITR product:

- A **Railway Cron Job** (a service with a `Cron Schedule` set in its settings, Railway's own documented use-case list names "a daily database backup" explicitly) runs `pg_dump` against the production database on a daily schedule.
- The dump is uploaded to **Cloudflare R2**, not a new storage vendor. R2 is already a configured project dependency for Phase 15's photo storage, so this adds no new third-party account or bill line, only additional usage of one already justified.
- Old dumps are pruned on a retention window (a named constant, not a magic number, per this repo's own standard) so storage cost stays bounded instead of growing forever.
- **The restore path is written down and actually exercised at least once before this is considered done.** A backup that has never been restored is unverified, not proven to work. This is the more realistic failure mode: a corrupt or incomplete dump discovered only during an actual emergency.

## Rationale

Alternatives considered:

1. **Railway's native PITR.** Rejected for now on cost: Pro-plan-gated, and finer-grained recovery (restore to any second, not just the last daily dump) is not a requirement this team's current budget justifies. Revisit once revenue or funding changes the calculus; the gap this ADR closes doesn't disappear, it's just currently met the cheaper way.
2. **No backup, revisit later.** Rejected outright, this is the gap that prompted the ADR. Every other Phase 8 decision has a real trade-off; this one does not, the only real question was mechanism and cost, not whether to have one at all.
3. **A different storage destination** (S3, a dedicated backup service). Rejected in favor of R2 specifically, to avoid a new vendor relationship for a bootstrapped team when an equivalent, already-justified one exists.

## Consequences

**Positive:**

- Closes a real gap the Phase 8 roadmap text left open: a destructive mistake (bad migration, accidental delete) is now recoverable instead of permanent.
- No new vendor, credential, or billing relationship; reuses R2, which Phase 15 already commits to.

**Negative / trade-offs:**

- Coarser recovery granularity than PITR: restorable to the last daily dump, not to an arbitrary second. A bad write in the middle of a day can still cost up to a day's data.
- Operational ownership sits entirely with the team. Railway's PITR is a supported product with its own failure handling; a `pg_dump` cron job is code this team owns, including noticing if it silently stops running. A monitoring/alerting story for "did last night's backup actually succeed" is a real follow-up, not covered by this ADR.
- Restoring is a manual procedure until it's written down and drilled; the first real restore should not be the first time anyone has done one.

## Sources

- [Railway PostgreSQL docs](https://docs.railway.com/databases/postgresql), "unmanaged," backups not automatic
- [Railway Point-in-Time Recovery](https://docs.railway.com/volumes/point-in-time-recovery), `pgBackRest`, roughly four-week window, Pro-plan billing
- [Railway Cron Jobs](https://docs.railway.com/cron-jobs), daily database backup named as a documented use case, five-field crontab, UTC, minimum five-minute frequency
