# ADR-0014: Rate Limiting — Keys, IP Minimisation, and Failure Modes

**Date:** 2026-08-24
**Status:** Accepted; implemented in `obur-backend` Phase 7

## Context

PDD §17 requires rate limits on read endpoints, not only writes. Most of the
read surface is reachable without authentication — deliberately, because a
client needs the category catalog before anyone signs in and because §14
chose Next.js so venue pages would be crawlable.

That creates a problem the PDD did not resolve: **an anonymous caller has no
identity to key a limit on.** The only thing distinguishing one anonymous
caller from another is the origin of the request.

And §17 appears, on a plain reading, to forbid using it:

> "IP correlation (even privacy-preserving, HMAC-hashed, never storing the
> raw address) is a real, well-understood technique, but the team made a
> deliberate bootstrap-stage call not to take on a data-handling
> responsibility in an area neither owner has direct experience with."

So the document requires a limit it also appears to forbid the means of
implementing. This ADR resolves that, and settles the mechanics — which
turned out to matter more than the policy, because the most common way to
implement this is one that does not work at all.

## Decision

### 1. Two key namespaces

| Caller | Key | Notes |
|---|---|---|
| Authenticated | `user_id` | Stable, already known, no privacy question |
| Anonymous | `HMAC-SHA256(secret, client_ip)`, truncated | Server-side secret; the raw address is never stored, logged, or returned |

The secret is a new required setting. Without it the derivation is a plain
hash of a small, enumerable space — every IPv4 address can be hashed in
seconds — which would make the stored value trivially reversible.

### 2. The narrow exception to §17's position on IP data

§17 rejected IP **correlation**: linking accounts through a shared address
to infer one person behind several. That rejection stands and this ADR does
not touch it. There is no cross-account analysis, no persistence beyond the
counting window, no geolocation, no joining a derived key to a `user_id`,
and no writing it to logs.

What is permitted is narrower: a counter that answers one question — *has
this origin made too many requests in the last minute* — and forgets.

The justification is a comparison of harms rather than a reinterpretation.
Refusing the counter means declining to protect the behavioural and location
profile that PDD §17's Bulk-Extraction Resistance now describes, in order to
avoid deriving an ephemeral value from an address. The data being protected
is materially more sensitive than the data being touched. Recording this
explicitly matters because the reverse reading — that §17 forbids all IP
processing — would leave the required protection unbuildable.

### 3. The client IP must be resolved rightmost-ish, never leftmost

Behind a proxy, `request.client.host` is the proxy. The client address has
to come from `X-Forwarded-For`, and **the obvious way to read that header is
a vulnerability, not an implementation detail.**

`X-Forwarded-For` is client-controllable: anything to the left of what our
own proxy appended was written by whoever sent the request. Taking the
leftmost entry means an attacker spoofs a different prefix per request, each
lands in its own counter, and no counter ever fills:

> "If you use the leftmost-ish IP for this, an attacker can just spoof a
> different XFF prefix value with each request and *completely avoid being
> limited*."
> — [adam-p, *The perils of the "real" client IP*][xff]

This is not hypothetical. It has been filed as a security advisory against
real frameworks, including [Litestar][litestar], where spoofing the header
bypassed rate limiting outright.

The rule, therefore:

- The trusted proxy count is **configuration**, with no default. A wrong
  value fails open silently, so it is verified against the real deployment
  topology in Phase 8 rather than guessed at in docker-compose.
- Read the address by counting in from the right by that many hops; the
  first entry we did not receive from a proxy we control is the answer.
- Validate the result is a well-formed address before it is used as part of
  a cache key — an unvalidated value is a memory-exhaustion vector, since
  the attacker chooses the string.

The cited source warns about the failure mode this creates: "network
architecture changes break this silently." A change in the number of proxy
hops disables the limiter without any error. The proxy count therefore needs
a comment where it is configured saying what it counts, and a check when the
deployment changes.

### 4. Fixed window, not a sliding window log

A sliding window log is more precise and is the usual recommendation for
accurate quota enforcement. It is the wrong choice here, for a security
reason rather than a simplicity one.

A sliding window log stores one entry per request per key — O(requests) per
key. With keys derived from addresses, an attacker rotating addresses
controls the number of keys, so a precise limiter becomes a memory
exhaustion vector against the store protecting us.

A fixed window counter is O(1) per key and self-cleaning: `INCR` plus a TTL
([Redis's own documented pattern][redis]). Its known weakness is the
boundary burst — up to twice the limit across a window edge — which is
irrelevant to this threat model. We are raising the cost of bulk harvesting,
not metering a paid quota.

**The increment and the expiry must be atomic.** Redis's guidance puts them
in one transaction, and the reason is a real failure rather than
tidiness: a counter that is incremented but never given a TTL is permanent,
and the caller it belongs to is locked out forever.

### 5. Failure mode differs by tier, because the threat does

If the counter store is unavailable, the two tiers behave differently:

| Tier | Behaviour | Why |
|---|---|---|
| **Strict** — check-in, report, and venue creation, follow | **Fail closed** | PDD §17 calls aggregate-rating manipulation a direct attack on the platform's core credibility. That is a data-integrity harm, worse than the harvesting this otherwise guards against, and it closes the "degrade the counter, then spam" path. Writes are a small share of traffic, so refusing them is "you can't post right now", not an outage |
| **Baseline** — reads | **Fail open**, logged loudly | Browsing and crawlability keep working. The exposure is bounded by how long the store is down |

A blanket fail-closed was rejected: it makes the counter store a hard
dependency of every request, so a system that is currently optional to
serving becomes one that can take the API down. On a tier with no
high-availability guarantee (§15), that trades a slow-burn privacy risk for
a self-inflicted outage.

### 6. Limits are named constants, and their honest effect is recorded

Concrete numbers are a starting point, in the same sense as §8's rating
thresholds: the mechanism is the decision, the values are calibratable and
live in one place.

Starting values, recorded here so the next person sees where the numbers
came from rather than only what they are:

| Tier | Limit | Applies to |
|---|---|---|
| Baseline | 600 / hour | Every endpoint, authenticated and anonymous alike |
| Strict | 30 / hour | Check-in creation, venue creation, follow — and report submission once it exists (Phase 10) |

Anonymous and authenticated callers share the baseline figure: both are
browsing, and there is no reason to assume a signed-in visitor reads more
pages than a signed-out one. Account deletion is deliberately *not* strict —
a rate limit does not protect an action that only needs to succeed once, and
repeated attempts against it are pointless rather than harmful.

They are chosen against two boundaries. Real anonymous browsing is a few
hundred requests an hour, because one page view is several API calls — the
figure commonly cited for anonymous API consumers, [200 requests an
hour][owasp], would break ordinary visitors here. And any limit permissive
enough for that also permits a single-address full enumeration in hours.
Hours rather than minutes is the achievable goal, and PDD §17 now says so
rather than implying prevention.

### 7. Crawlers get no special treatment yet

Verified crawler allowlisting (published address ranges, confirmed by
reverse DNS) is the correct mechanism if it is needed, and trusting the
`User-Agent` is not — it is attacker-controlled, so a "crawler" tier keyed
on it is a hole rather than an exemption.

For now crawlers are ordinary anonymous callers: a search engine paces
itself and will not approach these limits at this catalog size. The trigger
for revisiting is explicit, because the failure would otherwise be silent —
pages dropping out of an index is not something anyone notices. If verified
crawler addresses start receiving 429s, allowlisting follows. `robots.txt`
with a crawl delay is the cheaper first lever and belongs to `obur-web`.

### 8. Response contract

A refused request returns `429` with `Retry-After`, plus the widely
implemented subset of the [IETF RateLimit header fields][ietf] —
limit, remaining, and reset. The full draft is not yet an RFC, so this
follows the parts clients already understand rather than tracking a moving
specification.

## Rationale

Alternatives considered:

1. **Require authentication on the read endpoints that matter.** Removes the
   anonymous key problem entirely. Rejected: the category catalog is needed
   before sign-in, and gating venue pages ends the crawlability §14 chose
   the web stack for.
2. **A global bucket per endpoint** rather than a per-caller key. Rejected:
   one harvester exhausts everyone's quota, which converts a scraping
   problem into a denial-of-service one.
3. **Defer anonymous read limiting until the dataset is worth taking.** This
   was the first proposal here and it was wrong. It holds for catalog theft,
   which scales with the catalog, and fails for the harm that actually
   matters: a small user base is easier to deanonymise, so the profile-
   harvesting risk is worst exactly when the "too small to bother" argument
   sounds most convincing.
4. **Blanket fail-closed on store unavailability.** Rejected in §5 above.

## Consequences

**Positive:**

- The anonymous read surface gets the protection PDD §17 asked for, without
  contradicting its position on IP data — because that position has been
  read precisely rather than broadly.
- Choosing the fixed window on memory-safety grounds means the limiter
  cannot be turned into a weapon against the store it depends on.
- Splitting the failure mode by tier protects the more serious harm without
  making a non-HA dependency able to take the API down.

**Negative / trade-offs:**

- A new required secret, and a proxy-count setting whose wrong value fails
  open silently. Both are deployment configuration, which is why the
  verification lands in Phase 8 alongside the real topology.
- IP-derived keys mean addresses are processed, however briefly. This is a
  real if narrow expansion of data handling and needs to be reflected in the
  privacy policy when that text is written (§18).
- Harvesting is made expensive, not impossible. Pattern detection stays
  deferred (§17) and closing the anonymous surface stays rejected, so a
  determined actor rotating addresses still succeeds — slowly.
- Shared addresses (carrier NAT, offices, universities) share a counter, so
  a limit tuned for one visitor can be reached by several genuine ones
  behind one address. This is inherent to address-keyed limiting and is a
  reason to keep the baseline generous.

## Sources

- [The perils of the "real" client IP][xff] — the rightmost-ish algorithm
  and why leftmost defeats rate limiting
- [Litestar security advisory GHSA-hm36-ffrh-c77c][litestar] — the same
  failure in a shipped framework
- [Redis — Rate Limiting][redis] — the `INCR` + `EXPIRE` pattern and why the
  pair must be atomic
- [IETF httpapi — RateLimit header fields for HTTP][ietf] — the response
  contract this follows a subset of
- [MDN — X-Forwarded-For][mdn] — header semantics and its
  client-controllable nature
- [OWASP API4 — Lack of Resources & Rate Limiting][owasp] — the commonly
  cited anonymous/authenticated limit figures

[xff]: https://adam-p.ca/blog/2022/03/x-forwarded-for/
[litestar]: https://github.com/litestar-org/litestar/security/advisories/GHSA-hm36-ffrh-c77c
[redis]: https://redis.io/glossary/rate-limiting/
[ietf]: https://datatracker.ietf.org/doc/draft-ietf-httpapi-ratelimit-headers/
[mdn]: https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/X-Forwarded-For
[owasp]: https://www.indusface.com/blog/api42019-lack-of-resources-rate-limiting/
