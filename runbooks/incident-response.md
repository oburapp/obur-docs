# Incident Response Runbook

> **Status: stub.** This runbook has not been exercised yet. Flesh it out as
> the team defines a real on-call and incident process.

## Scope

System-wide incident response covering `obur-backend`, `obur-web`,
`obur-mobile`, and their infrastructure (Railway, Vercel, Clerk, Cloudflare
R2, Mapbox).

## Severity Levels

| Level | Definition | Example |
|-------|------------|---------|
| SEV1 | Full outage or data-loss risk | Backend down, database unreachable |
| SEV2 | Major feature broken, no data loss | Check-in creation failing |
| SEV3 | Minor or partial degradation | Slow search, non-critical endpoint errors |

## Response Steps

1. Acknowledge and assess severity
2. Notify the other founder
3. Mitigate (rollback, restart, disable the affected feature)
4. Resolve the root cause
5. Write a postmortem for SEV1/SEV2 incidents

## Postmortems

Every SEV1/SEV2 incident gets a postmortem: what happened, impact, root
cause, follow-up actions. Store postmortems in this directory as
`postmortem-YYYY-MM-DD-short-title.md`.

## Contacts / Escalation

*To be filled in once the team defines on-call ownership.*
