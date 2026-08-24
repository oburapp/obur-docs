# ADR-0015: Error Contract, Request Correlation, and Structured Logging

**Date:** 2026-08-24
**Status:** Accepted; implemented in `obur-backend` Phase 7

## Context

The API has no error contract. It returns FastAPI's default `{"detail": "..."}`,
and four things have converged to make that inadequate:

**Two different 429s already collide.** `PATCH /users/me` returns
`429 Too Many Requests` when a handle was changed too recently, and
[ADR-0014](0014-rate-limiting-keys-and-ip-minimisation.md) adds rate
limiting, which returns the same status. A client cannot tell "slow down"
from "this field is locked for another three weeks" — and `Retry-After`
measured in weeks is meaningless for the second. No other status fits
either: `409` is already the taken-handle case, `422` is schema validation,
`403` is authorisation. The discriminator has to live in the body.

**Around fifteen call sites return `detail=str(e)`,** passing internal
exception text straight to the client. `NotListOwnerError` reaches a caller
as `"user 8f3a… may not delete list 2b1c…"`.

**Error messages cannot be localised.** Phase 6 built locale resolution, but
errors are English prose the client can only display verbatim. PDD §6 lists
Turkish and English as both MVP.

**There is no request correlation at all.** A rate-limit rejection
(ADR-0014) that leaves no trace is not debuggable, and neither is anything
else.

## Decision

### 1. RFC 9457 Problem Details, not a bespoke shape

Errors are served as `application/problem+json` per
[RFC 9457][rfc9457], which replaced RFC 7807 in 2023 and is implemented by
Spring Boot 6+, ASP.NET Core 7+, and referenced by OpenAPI 3.x.

```json
{
  "type": "urn:obur:problem:username-changed-too-recently",
  "title": "Username changed too recently",
  "status": 429,
  "detail": "You can change your username again on 12 September 2026.",
  "request_id": "01J9F2..."
}
```

**`type` is the machine-readable discriminator; there is no separate `code`
member.** The RFC already assigns that role to `type`, and adding a parallel
field would be two identifiers for one thing — the drift risk
[ADR-0012](0012-migrations-are-self-contained-reference-data-is-seeded.md)
exists to avoid, in miniature.

`request_id` is an extension member, which the RFC explicitly permits:
"Problem type definitions MAY extend the problem details object with
additional members." Clients are required to ignore extensions they don't
recognise, so this is forward-compatible by construction.

### 2. `type` URIs are URNs, not HTTPS URLs

`urn:obur:problem:{slug}`, not `https://obur.app/problems/{slug}`.

Two reasons, and the first is decisive: **there is no production domain yet.**
Committing to one in an identifier that clients branch on would make the
eventual domain choice a breaking API change. The RFC requires a URI
reference; it does not require one that dereferences — resolvable
documentation is "encouraged", not mandated.

The second is honesty. An HTTPS URL that returns 404 for a year is a
promise the API is not keeping. A URN promises nothing but identity, which
is exactly what this field is for. Developers are not left empty-handed:
`title` carries the human-readable summary in the same response.

**This cannot be revisited cheaply.** Changing a `type` value breaks every
client branching on it, so migrating to HTTPS URLs later would be a
versioned change, not an addition. That is why it is decided now, before any
client exists, rather than deferred as a cosmetic detail.

### 3. `detail` is written for a person, never derived from an exception

Every handler supplies a fixed, user-facing string. `str(exception)` never
reaches a response. RFC 9457's own security considerations are explicit
about why:

> "Risks include leaking information that can be exploited to compromise the
> system, access to the system, or the privacy of users of the system."

> "Generators providing links to occurrence information are encouraged to
> avoid making implementation details such as a stack dump available through
> the HTTP interface."

Exception messages remain what they are — developer-facing text for the
logs, where they are useful and where `request_id` connects them to the
response the user saw.

### 4. `title` and `detail` are English only; user-facing copy belongs to the UI

The API is a technical surface. Its strings are written in English, the same
as the code, the comments, the ADRs, and the logs — the repository already
requires this of everything else it contains, and an error string is not an
exception.

Text a user reads is the client's, produced by mapping `type` to its own
copy. The client can write better copy than the API can: it knows which
screen the person is on, which field failed, and how much room the message
has. "Username already taken" from the server becomes whatever reads best
inline under that field, in whatever language the person chose.

RFC 9457 supports this division — a client "MUST NOT" parse `detail` for
information, because `type` and the extension members are the
machine-readable part. `detail` is advisory: useful in a log, in a `curl`
response, or to a developer reading the API, all of which are English
audiences.

Adding a language therefore costs nothing here. No error string is
retranslated, and nothing lands in the translation tables that PDD §7
reserves for catalog labels.

**What would change this:** exposing the API to third-party clients we do
not control, or a surface where `detail` is displayed verbatim by design.
Either would make server-side localisation worth its cost — roughly twenty
strings per language, maintained in code, forever. Neither is true today.

The one accepted risk: a client that forgets to map a `type` shows English
to a Turkish-speaking user. That is a client bug, visible the first time
anyone exercises the path, and it is preferable to the alternative — logs
and API responses in Turkish while every other artefact in the repository is
in English, which would be a permanent inconsistency rather than a
correctable oversight.

### 5. FastAPI's own validation errors are normalised into the same shape

`RequestValidationError` produces `{"detail": [{"loc": …, "msg": …}]}`,
a different shape from everything else. An exception handler maps it to
`urn:obur:problem:validation-failed` with the field errors under an
`errors` extension member. One body shape means one client-side parser.

### 6. `X-Request-ID` now; `traceparent` when there is a second hop

Every request gets an id: generated, echoed back in the `X-Request-ID`
response header, and attached to every log line and every problem response.
An inbound `X-Request-ID` is honoured **only if it validates** — bounded
length and a restricted character set — and otherwise replaced. OWASP is
direct about the reason:

> "Perform sanitization on all event data to prevent log injection attacks
> e.g. carriage return (CR), line feed (LF) and delimiter characters."

An unvalidated client-supplied value written to logs is a log-injection
vector, and this one is written on every line.

**[W3C Trace Context][w3c] (`traceparent`) is the standard for correlation
across services, and is deliberately not adopted yet.** There is one
service; there is no second hop to correlate with, so its format
(`00-<32hex>-<16hex>-<flags>`), generation, and validation would be
structure serving nothing today — the same reasoning that removed the
product layer in
[ADR-0011](0011-drop-product-layer-four-venue-criteria.md).

Deferring is cheap because `traceparent` is additive: a new header alongside
`X-Request-ID`, breaking no existing client. That is what makes waiting the
right call rather than a gamble.

**Revisit when any of these is true:**

- `obur-web` starts calling this API from its server side. One user action
  then spans two services, and correlating them by hand across two
  independently generated ids is exactly the problem `traceparent` solves.
- A tracing or APM backend is adopted — Phase 8 will show what the platform
  offers. Every such tool speaks W3C Trace Context; feeding it a custom
  header means writing an adapter.
- A third service appears (a worker, a scheduled job — Phase 14's
  `rarity_pct` recalculation is the first candidate).

Until then `X-Request-ID` is sufficient and honest about what it is.

### 7. Structured logging, and a stricter exclusion list than the baseline

Logs are JSON, one object per line, always carrying `request_id`, method,
route, status, and duration. The duration field is where PDD §17's latency
targets are measured from; where those numbers are aggregated is a Phase 8
decision, once the platform's own facilities are known.

Never logged — [OWASP's exclusion list][owasp]: session identifiers, access
tokens, passwords, connection strings, encryption keys, payment data, and
sensitive PII.

**Two additions beyond that baseline, from ADR-0014:** the raw client
address, and the derived rate-limit key. OWASP does not list source IP as
excluded, so this is deliberately stricter than the standard requires — and
worth naming as a choice rather than leaving it to look like an oversight.
ADR-0014's position is that an address may be counted against, not
recorded, and a log line is a recording.

### 8. Middleware order

`request-id → logging/latency → GZip → rate limit → route`

A rejected request must still receive an id and be measured; a limiter that
runs before the id is assigned produces 429s with nothing to trace them by.

## Rationale

Alternatives considered:

1. **A bespoke `{"error": {"code", "message"}}` body.** This was the first
   proposal here and it was reinventing RFC 9457. Rejected once the standard
   was checked: identical expressiveness, no library support, nothing to
   point an OpenAPI spec at, and a shape every client has to be told about.
2. **Keep `{"detail": …}` and solve the 429 collision with a different
   status code.** Rejected: no unused status fits, and the collision is a
   symptom — the general problem is that a status code alone cannot carry
   which of several conditions occurred.
3. **HTTPS `type` URIs pointing at documentation.** Rejected for now in §2:
   no domain exists, and the value cannot be changed later without breaking
   clients.
4. **Server-side error localisation** via translation tables. Rejected:
   it puts every error string into the seed data and still produces worse
   copy than a client that knows its own context. `type` gives the client
   what it needs to do better.
5. **Adopting `traceparent` immediately.** Rejected in §6, with explicit
   revisit conditions rather than a vague "later".

## Consequences

**Positive:**

- One body shape across every error, including FastAPI's own validation
  failures, so a client writes one parser.
- Two conditions sharing a status code stop being ambiguous, and the same
  mechanism handles every future collision without further design.
- Internal exception text stops reaching users, and `request_id` connects
  what the user saw to what the logs recorded.
- Clients can localise errors without the server carrying translations.

**Negative / trade-offs:**

- `type` URNs do not resolve to anything. `title` mitigates it for a
  developer reading a response, but there is no documentation page behind
  the identifier until one is written.
- Roughly fifteen call sites currently passing `str(e)` need real
  user-facing strings written for them, which grows Phase 7 beyond its
  original scope. Accepted deliberately: the alternative is shipping a
  contract that half the endpoints do not follow.
- Every error path now goes through a handler rather than raising
  `HTTPException` inline. That is a uniform, mechanical change, but it is a
  change to every endpoint.
- Choosing `X-Request-ID` over `traceparent` means an adapter later if an
  APM tool is adopted. Bounded by the revisit conditions above.

## Sources

- [RFC 9457 — Problem Details for HTTP APIs][rfc9457] — the body format and
  its security considerations
- [Swagger — Problem Details (RFC 9457): Doing API Errors Well][swagger] —
  adoption across frameworks
- [W3C Trace Context — `traceparent` and `tracestate`][w3c] — the
  correlation standard deferred here
- [OWASP — Logging Cheat Sheet][owasp] — the exclusion list and log
  injection sanitisation

[rfc9457]: https://datatracker.ietf.org/doc/html/rfc9457
[swagger]: https://swagger.io/blog/problem-details-rfc9457-doing-api-errors-well/
[w3c]: https://www.dash0.com/knowledge/w3c-trace-context-traceparent-tracestate
[owasp]: https://cheatsheetseries.owasp.org/cheatsheets/Logging_Cheat_Sheet.html
