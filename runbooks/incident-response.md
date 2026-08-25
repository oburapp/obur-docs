# Incident Response Runbook

> **Status: partially verified.** The `obur-backend` / Railway commands
> below were run for real during Phase 8 deployment, not guessed. The
> rest of this runbook (`obur-web`, `obur-mobile`, and their vendors)
> is still a stub: those aren't deployed yet, so there's nothing real to
> verify commands against.

## Scope

System-wide incident response covering `obur-backend`, `obur-web`,
`obur-mobile`, and their infrastructure (Railway, Vercel, Clerk, Cloudflare
R2, Mapbox). Only `obur-backend` on Railway is live as of this writing,
see [obur-backend/docs/deployment.md](https://github.com/oburapp/obur-backend/blob/main/docs/deployment.md).

## Severity Levels

| Level | Definition | Example |
|-------|------------|---------|
| SEV1 | Full outage or data-loss risk | Backend down, database unreachable |
| SEV2 | Major feature broken, no data loss | Check-in creation failing |
| SEV3 | Minor or partial degradation | Slow search, non-critical endpoint errors |

## Response Steps

1. Acknowledge and assess severity
2. Notify the other founder
3. Mitigate (rollback, restart, disable the affected feature); see
   [Backend / Railway commands](#backend--railway-commands) below for
   `obur-backend`'s actual, tested mechanics
4. Resolve the root cause
5. Write a postmortem for SEV1/SEV2 incidents

## Backend / Railway Commands

Requires the Railway CLI (`npm install -g @railway/cli`), logged in
(`railway login`) and linked to the project from the `obur-backend`
directory (`railway link`). Every command below was actually run against
the real project during Phase 8, not copied from documentation.

**First, see what's actually happening:**

```
railway status                 # current deployment state, at a glance
railway logs                   # deploy logs for the current deployment
railway logs --deployment <id> # logs for a specific past deployment
```

**Restart** (fastest option; doesn't rebuild, doesn't fix bad code, only
helps for a hung or crashed process):

```
railway restart --service obur-backend
```

**Roll back to a specific previous version.** The CLI's `railway
redeploy` only ever redeploys the *current* latest deployment (it has no
option to target an older one), so this needs the dashboard, not the
CLI: Railway dashboard → `obur-backend` service → **Deployments** tab →
find the last known-good deployment in the list → its `⋮` menu →
**Redeploy**. Confirm the resulting deployment ID is the older one, not
a re-run of the broken one.

**Deploy the current `main` fresh** (e.g. after fixing forward instead of
rolling back):

```
railway up --detach
```

**Get a shell inside a running service** (e.g. `postgis`, to run `psql`
directly against the real database):

```
railway ssh --service <service-name>
```
First use needs an SSH key registered once: `railway ssh keys add`
(generate one first with `ssh-keygen -t ed25519` if none exists).

**Check environment variables actually in effect** (not what the
dashboard shows visually, which can misrepresent a value with embedded
whitespace: this exact issue caused a real Phase 8 incident, a hidden
newline inside `DATABASE_URL` broke database auth in a way that looked
identical to a wrong password in the dashboard UI):

```
railway variables
railway run bash -c 'echo "[$SOME_VAR]"'   # dumps the raw value, brackets make hidden whitespace visible
```

**Check whether a database is still private** (should always come back
empty for `postgis` and `Redis`; if either has an entry, that's a
security incident on its own, not just an availability one):

```
railway domain list --service postgis
railway tcp-proxy list --service postgis
```
Read-only. Do not run bare `railway domain --service <name>` with no
subcommand to "check" this: that command *creates* a public domain
rather than listing existing ones.

## Postmortems

Every SEV1/SEV2 incident gets a postmortem: what happened, impact, root
cause, follow-up actions. Store postmortems in this directory as
`postmortem-YYYY-MM-DD-short-title.md`.

## Contacts / Escalation

*To be filled in once the team defines on-call ownership.*
