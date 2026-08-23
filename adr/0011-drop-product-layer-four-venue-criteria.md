# ADR-0011: Drop the Product Layer, Rate Venues on Four Criteria

**Date:** 2026-08-23
**Status:** Accepted (supersedes [ADR-0005](0005-checkin-fields-editable-products-immutable.md))

## Context

The PDD listed product-level rating as one of three differentiators (§2):
*"the only platform that actually answers 'where's the best döner'."* It was
implemented across three tables, shipped in Phases 2 and 3:

- `GLOBAL_PRODUCT_TYPE` — a closed, team-curated catalog (`slug` +
  translations) that is the aggregation key for cross-venue ranking.
- `PRODUCT` — venue-specific, with a mandatory `global_type_id` and a
  free-text `name`.
- `CHECKIN_PRODUCT` — the per-item rating, `NOT NULL`, at least one required
  per check-in.

Four concerns surfaced in review, in escalating order of severity. The first
turned out to be wrong, and recording why matters as much as the rest:

**1. "Free-text product names will fragment aggregation."** They wouldn't.
The free text is `PRODUCT.name`, which is never an aggregation key;
`global_type_id` is, and it is `NOT NULL` against a closed catalog. "Adana
Kebap" / "adana kebabı" / "Adana kebap (acılı)" all carry
`global_type = adana-kebap` and rank together. The normalization layer this
concern asked for was already the design. Dismissed.

**2. The closed catalog can't cover the real world, and a gap is a wall.**
This holds. The seed catalog has 12 types. The set of things a person can
order is effectively unbounded — cuisine × dish × each venue's own
invention. A user hits "it isn't in the list" *during check-in*, at the
product step, on the platform's core action, and hits it often. Unlike venue
categories (a bounded set, created rarely, with a published external
taxonomy to copy and an admin correction path), there is no version of this
catalog that is both maintainable by a small team and complete enough to
avoid the wall.

**3. Two of the three features built on it don't survive scrutiny.**

- *Cross-venue ranking* is sorted by an anonymous, platform-wide aggregate
  with no social dimension at all. That is structurally the same thing §1
  criticizes Google Maps for — *"a star average produced by hundreds of
  strangers"* — just scoped to a dish instead of a venue. The improvement is
  granularity, not the thesis. A genuinely on-thesis version ("the best
  döner among people you follow") needs follow-graph density that does not
  exist at this scale.
- *Personalized history* showed the user's top item per product type — *en
  iyi döner, en iyi latte, en iyi baklava* — an auto-generated index of
  whatever they happened to log, with no narrative and no curation. The
  PDD's whole positioning is taste-based **curation** (§1); an
  auto-generated list is its opposite.

**4. The density math, which is decisive and applies to the one feature that
did survive scrutiny.** §8 sets a 10-rating floor before any label is shown.
Against the PDD's own MVP targets (§1: 200 MAU, ≥4 check-ins per user per
month → ~800 check-ins/month), spread across a few hundred Istanbul venues,
a venue accumulates roughly 2–3 check-ins per month. After six months that
is ~16 check-ins per venue; if each rates two items across ~15 distinct
items, each item holds about **two** ratings against a floor of ten.

So even the most defensible use — *"what's good at this place"* on the venue
page, which needs no global type and no cross-venue density — would show
"New / Low Data" next to every row, for years. The product step would cost
every user two extra steps on the core action and return an empty label.

Meanwhile `CHECKIN.note` already carries the same information in a form
humans read better: *"İskender harikaydı ama ayran ekşiydi"* tells a reader
more than `İskender: 4/4`. The venue page already lists check-ins (§13), and
its representative photo is already a food photo from the most-liked one.

## Decision

**Remove the product layer entirely.** `PRODUCT`,
`GLOBAL_PRODUCT_TYPE` (and its translation table), and `CHECKIN_PRODUCT` are
dropped. A check-in is a venue, four ratings, an optional note and photo.

**`CHECKIN` gains `rating_taste`, making four venue-level criteria.** This is
not optional polish: food quality currently lives *entirely* in
`CHECKIN_PRODUCT.rating`, and §8 defines the existing three criteria as
explicitly non-food. Removing products without adding this would leave a
food-and-drink platform that rates ambiance and service but never the food.

| Field | Türkçe | English | Measures |
|---|---|---|---|
| `rating_taste` | Lezzet | Taste | food and drink quality |
| `rating_service` | Servis | Service | service quality |
| `rating_ambiance` | Ambiyans | Ambiance | atmosphere, music, lighting, decor |
| `rating_value` | Fiyat | Value | is it worth what it costs |

All four are required, all four use the existing four-point scale
(`app/core/ratings.py`), unchanged.

**`rating_value` is redefined.** §8 previously described it as *"not
price/performance, but the payoff of the overall experience."* With taste,
service, and ambiance all now rated separately, that definition overlaps all
three and measures nothing of its own. It becomes **value for money** — the
single price dimension, and the only criterion that can distinguish an
excellent venue from an excellent venue that overcharges. Its Turkish label
changes from *Değer* to **Fiyat**.

**§8 collapses to a single level.** The product-level score disappears along
with its pool. The venue-level headline score now pools exactly four values
per check-in. The confidence-bound procedure itself — the 10-rating floor,
the 95% lower bound, the nine-tier Favorable/Unfavorable ladder — is
unchanged; only the number of pools changes, from two to one.

**Personalized history survives, re-keyed.** Its stated purpose (§13) is to
reinforce a sense of discovery on a user's own profile, and that never
required products. It becomes the user's own highest-rated venue per
`VENUE.category_id`: *"en iyi dönerci: Develi"* rather than *"en iyi döner:
Develi."* Still a read over existing data with no new schema — and, unlike
everything else here, it needs no density at all, since it reports the
user's own rating rather than a platform statistic.

**The product layer moves to v2.0, gated on density, not abandoned.** §6's
v2.0 table already carries features held back until enough data accumulates
("You Might Like"). This belongs there for the same reason. At a scale where
a venue sees hundreds of check-ins, the question becomes answerable — and by
then the accumulated notes will show which items are worth curating, which
is information not available today.

## Rationale

Alternatives considered, in the order they were raised:

1. **Keep the layer as designed.** Rejected on the density math above: the
   floor cannot be cleared at MVP scale, so the feature costs two steps on
   the core action and returns nothing.
2. **Make `global_type_id` nullable** — items in the catalog rank across
   venues, items outside it are recorded but unranked. Rejected: it fixes
   the wall (concern 2) but keeps both features that fail concern 3, keeps a
   catalog to maintain, and still cannot clear the floor.
3. **Keep `PRODUCT` but drop `GLOBAL_PRODUCT_TYPE`** — venue-scoped items
   only, no cross-venue anything. Rejected: this was the strongest
   intermediate option and it fails on concern 4 alone. The venue-scoped
   "what's good here" list is exactly what cannot reach ten ratings per
   item.
4. **Remove products without adding a taste rating**, leaving the existing
   three criteria. Rejected outright: no field would measure food quality.
5. **Replace `rating_value` with taste**, keeping three criteria. Rejected:
   taste, service, and ambiance say nothing about price. A venue that is
   excellent on all three and charges three times the fair rate scores 4/4/4
   with no way to say otherwise, and price is something users actively want
   to know.

## Consequences

**Positive:**

- The check-in flow drops from five steps to four — Step 2 (choose products)
  disappears and Step 3 becomes the four criteria alone. The platform's core
  action gets materially shorter.
- **§8 gets simpler, not more complex.** It currently has to rationalize
  that a check-in rating more products carries proportionally more weight
  ("accepted deliberately"). With four values per check-in, every check-in
  weighs exactly the same and that paragraph disappears along with the
  two-level split.
- Zero taxonomy curation burden. There is no catalog to seed, grow, or
  admin-review, and no "suggest a new type" queue.
- No dead end anywhere in the flow.
- The four criteria map to the four things people actually judge a venue on,
  and `rating_value` finally measures something distinct.

**Negative / trade-offs:**

- §2 loses a differentiator, leaving two. Judged acceptable because the
  remaining two — no say for businesses, and curation by people you trust —
  reinforce each other and the whole product rests on them, while the
  product layer served a measurably different value proposition (concern 3).
- Obur moves structurally closer to a check-in log. The distance that
  remains is real, though: Swarm has no taste-based curation, no venue
  aggregate rating, and no lists.
- **Historical product data cannot be collected retroactively.** If the
  layer returns in v2.0, MVP-era check-ins will have none. Accepted: at a
  scale where the feature works at all, new data accumulates quickly, and
  the notes from the MVP period are what tell you which items to curate.
- **Load shifts onto `VENUE_CATEGORY`.** It now backs three consumers
  instead of one — Discover filtering, §12's Layer-2 ranking signal (which
  moves off products onto venue category), and personalized history. The
  seeded catalog is 9 entries and is too coarse for the specificity
  personalized history depends on; expanding it is a direct consequence of
  this decision and is tracked in `obur-backend`'s roadmap rather than here,
  since it is seed data rather than an architectural decision.
- `CHECKIN_DRAFT`'s justification in §17 rests partly on the check-in
  flow's "high per-submission effort." That effort is now lower. The draft
  is still warranted — venue selection, photo, and note remain real work,
  and the weak-connection argument is untouched — but §17's wording is
  updated so it doesn't rest on a reason that no longer holds.

## Superseded ADR

This supersedes [ADR-0005](0005-checkin-fields-editable-products-immutable.md),
whose subject was which parts of a check-in may be edited after creation:
its own fields yes, its rated products and their ratings no. The second half
has no subject left once `CHECKIN_PRODUCT` is gone.

**The first half still holds and is carried forward here:** a check-in's own
fields — `note`, `photo_url`, `visited_at`, `visibility`, and now all four
venue-level ratings — remain editable after creation, and this is what
`PATCH /api/v1/checkins/{id}` enforces. Correcting anything else still means
deleting the check-in (a soft delete, so no badge or aggregate already
computed from it is corrupted) and creating a new one.
