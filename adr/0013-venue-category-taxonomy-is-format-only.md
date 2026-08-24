# ADR-0013: The Venue Category Tree Classifies Format, Not Cuisine or Diet

**Date:** 2026-08-24
**Status:** Accepted

## Context

[ADR-0011](0011-drop-product-layer-four-venue-criteria.md) removed the
product layer, which left `VENUE.category_id` as the platform's only
classification dimension. Three separate features now read it:

- Discover's filters
- the feed's Layer-2 taste-overlap signal (PDD §12)
- a user's own "best per category" history (PDD §13), whose whole appeal is
  the specificity of *"en iyi dönerci"*

Its granularity therefore sets a ceiling on all three, and the seeded
catalog was nine entries. Expanding it surfaced a design problem that the
original nine were too coarse to expose: **the tree had been mixing three
independent axes.**

- **Format** — what kind of place it is: dönerci, meyhane, pastane,
  kıraathane.
- **Cuisine** — what tradition the food comes from: Italian, sushi, Asian.
- **Diet** — vegetarian, vegan.

These are orthogonal. An Italian restaurant is a *restaurant* by format and
Italian by cuisine. A vegan dönerci is still a dönerci. Putting all three in
one tree made `italian` and `dönerci` siblings, which are not the same kind
of thing, and produced categories that read as nonsense under the features
that consume them: *"en iyi vejetaryen"* is not a superlative anyone asks
for.

`VENUE.category_id` is a single column and can carry exactly one axis.

## Decision

### The tree classifies venue format

The axis is format, and the test for whether something belongs is:

> **Does this name a distinct kind of outing?**

That test is sharper than "format versus property," which was the first
formulation and would have excluded the wrong things:

- **`fine dining` is in.** It cuts across cuisines, so a naive format-only
  rule would drop it. But it describes an entire kind of evening — booking,
  service, price, what you wear — not a menu. It is not "an expensive
  dönerci"; it is categorically another outing.
- **`vegetarian` is out.** It describes menu content. A vegan dönerci is a
  dönerci, so the label belongs to the food rather than to the place.
- **`fast food` is out**, for the same reason as diet: it is a speed and
  price attribute. The specific formats it would cover (`burger`, `pizza`,
  `büfe`) are already there as real venue types.

### Cuisine is admitted only where it has *become* a format

Some foreign cuisines have settled into Turkish city life as venue types in
their own right. "Çin lokantasına gidelim" and "suşiciye gidelim" are
sentences people say the same way they say "dönerciye gidelim." Those earn a
slot; the rest do not.

Admitted: `çin lokantası`, `suşi`, `italyan`, `uzak doğu`. Not admitted:
anything that only names a cuisine on a menu rather than a place people go.

This keeps the axis honest while acknowledging that the boundary between
"cuisine" and "format" is set by usage, not by logic, and that usage is
local. A second market will draw the line in a different place.

### Two layers: universal roots, local leaves

Roots are universal concepts and stay untouched when a new country is
added: **Restoran, Kafe, Bar, Tatlı**. Leaves are the local seed; adding a
market means adding that market's leaves under the same roots.

`parent_id` already exists on `VENUE_CATEGORY` and has since the table was
created, so this needs no schema change at all.

Depth is deliberately capped at two. A third level would multiply the
picker's complexity without a consumer that needs it: none of the three
readers above distinguishes more finely than "which kind of place is this."

### Every venue is assigned to a leaf

Roots are grouping, not answers. Where the generic form of a root is itself
a real venue type, it appears as a leaf under its own root: a plain "Kafe"
and a plain "Bar" are things people say and go to, so `cafe-general` and
`bar-general` exist as leaves. A plain "Restoran" or a plain "Tatlı" is not
— those are `lokanta` and `pastane` respectively — so those roots have no
generic leaf.

This is deliberately a convention rather than a constraint. A column
(`is_assignable`) was considered and rejected as disproportionate: with a
generic leaf wherever one is meaningful, a client has no reason to assign a
root, and a venue that somehow ends up on one degrades to a vaguer label
rather than breaking anything.

### Names are what people call the place

Display names follow usage, not a mechanical derivation:

- the `-ci`/`-cı` form where that is the word (`dönerci`, `kebapçı`,
  `mantıcı`, `baklavacı`)
- the dish alone where the suffix reads as forced (`pizza`, `burger`,
  `sandviç`, `lahmacun`)
- a full phrase where the short form is ambiguous: **`balık lokantası`**,
  not `balıkçı`, because a balıkçı is also someone who sells fish

Two entries were dropped on a related test — *what would someone check into?*
`ekmek fırını` and `simitçi` describe errands rather than outings, and a
check-in is a logged experience.

Slugs are stable ASCII identifiers and never change once seeded; where a
Turkish format has no English equivalent the slug stays Turkish
(`meyhane`, `esnaf-lokantasi`) and the translation carries the gloss.

## Rationale

Alternatives considered:

1. **Keep cuisine in the tree alongside format.** Rejected: it makes *"en
   iyi İtalyan"* and *"en iyi dönerci"* compete in one list while measuring
   different things, and it grows without bound as cuisines are added,
   diluting the format axis the three consumers actually use.
2. **Cuisine as a second dimension now** (a separate column or tag set).
   Rejected as premature: nothing in the MVP reads it, and adding a
   dimension no feature consumes is exactly the kind of speculative
   structure ADR-0011 just removed. Revisit if search demands it.
3. **Three levels** (root → group → format). Rejected: no consumer
   distinguishes that finely, and every extra level is a step in a picker
   on the platform's core action.
4. **`is_assignable` on `VENUE_CATEGORY`** to make "roots are grouping" a
   database rule. Rejected as disproportionate once generic leaves cover
   the cases where a root would otherwise be the only sensible answer.
5. **More than one category per venue.** Raised immediately after this
   decision, on two examples: a bar/pub hybrid, and a kebapçı that also
   sells lahmacun. Deferred rather than rejected — see the open decision
   in PDD §18 — because the two examples are different problems and
   neither is currently unsolved. Format ambiguity (bar/pub) is what the
   generic leaf exists for. Menu breadth (kebap plus lahmacun) is not a
   format question at all: "hadi kebapçıya gidelim" holds even when you
   order lahmacun there. The cost of admitting it now would be steep and
   quiet — a venue carrying three categories becomes the user's "best" in
   all three at once, which is the auto-generated jumble
   [ADR-0011](0011-drop-product-layer-four-venue-criteria.md) removed
   from the profile, reappearing through another door; and the axis
   drifts from *what kind of place* toward *what is on the menu*, undoing
   this decision one venue at a time. If real venues show a single format
   genuinely failing often, the answer is likelier a separate dimension
   than a second category.

## Consequences

**Positive:**

- The three consumers of `VENUE.category_id` all get an axis that means
  something to them: *"en iyi dönerci"* reads as a real claim, filtering
  matches how people actually search, and "you rate dönerci highly" is a
  genuine taste signal.
- Adding a market is additive: new leaves under the same four roots, no
  restructuring, no migration.
- The catalog stays seed data. Changing it is a seed edit plus a re-run of
  the seeder ([ADR-0012](0012-migrations-are-self-contained-reference-data-is-seeded.md)),
  never a schema change.

**Negative / trade-offs:**

- Searching by cuisine ("İtalyan") is unsupported at MVP beyond the four
  cuisines admitted as formats, and falls back to name search. Accepted:
  the alternative was polluting the one axis three features depend on.
- The cuisine-versus-format boundary is a judgement call and will need
  revisiting per market. It is recorded here so the next person sees a rule
  rather than an arbitrary list.
- `Kafe > Kafe` and `Bar > Bar` read as repetition in a picker. Accepted:
  both levels mean the same thing there, which is different from the
  genuinely ambiguous `Fırın & Pastane > Fırın` this taxonomy removed.
- The catalog is opinionated about Turkish venue culture and was written
  without usage data. It is a starting point, in the same sense as PDD §8's
  rating thresholds: the structure is the decision, the exact list is
  calibratable.
