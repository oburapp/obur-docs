# ADR-0008: Light, Synchronous In-App Notifications (No Queue, No Push Yet)

**Date:** 2026-08-18
**Status:** Accepted

## Context

Phase 4 needed a notification system for the PDD's §11 triggers (new
follower, check-in like, list like — badge-earned and venue-verification
triggers are deferred to the phases that produce those events). The
product priority set for this system, stated explicitly ahead of the
design work, was synchronization and consistency across devices over
richness of delivery: a user reading a notification on their phone must
see it as read on web too, and the data must never be lost or duplicated
between the action that caused it and the record of it.

Obur targets web, iOS, and Android from one backend. A full push
delivery system (Firebase Cloud Messaging fan-out, delivery receipts,
per-platform payload formatting) is real, separable work that isn't
needed to satisfy the actual requirement here: a backend record of what
happened and whether it's been read, queryable by any client.

## Decision

A `NOTIFICATION` row is created synchronously, in the same database
transaction as the action that triggers it (e.g. `follow_user`,
`like_checkin`), by the service function performing that action — not by
a queue, a background worker, or an event bus. `read_at` is a backend
column, not client-local state: since there is exactly one backend
record per notification, "read" is trivially the same value on every
device that queries it — cross-device consistency falls out of the data
model rather than needing to be separately engineered.

`target_type` / `target_id` (pointing at the check-in, list, or user a
notification is about) is deliberately *not* a real foreign key, unlike
`CheckinLike`/`CheckinBookmark`/`CheckinProduct`, which are. A
notification is a transient, denormalized record of something that
happened — losing referential integrity on it (e.g. the target check-in
is later hard-deleted) is not a correctness problem the way it would be
for a rating or a like, where the row's entire meaning depends on the
thing it references still existing.

Push delivery (FCM) and any batching/digest behavior are explicitly out
of scope for this phase — this ADR covers only the backend data model and
API (list, unread count, mark-all-read), which push delivery would sit
on top of later without changing.

## Rationale

Alternatives considered:

1. **Queue/worker-based creation** (e.g. an outbox pattern, a message
   broker). Rejected for now: it solves a scaling and delivery-guarantee
   problem Obur doesn't have yet — there's no background worker
   infrastructure in the stack at all today, and introducing one for
   this alone would be new operational surface with no corresponding
   need. The synchronous approach also structurally can't drop a
   notification or create a duplicate the way an at-least-once queue
   without idempotency handling could, since it lives in the same
   transaction as the triggering write.
2. **`read_at` (or equivalent) stored client-side per device.** Rejected
   directly by the stated priority: this is the one design that
   *doesn't* give cross-device consistency for free — each device would
   need its own sync logic to reconcile local read-state with any other
   device's.
3. **Real FK on `target_type`/`target_id`.** Considered and rejected
   specifically because the target can be one of several different
   tables (check-in, list, and later others), which a single FK column
   can't point at simultaneously without a separate FK column per
   possible target type — added schema complexity for a record whose
   correctness doesn't depend on the reference staying valid.

## Consequences

**Positive:**

- Notification and the event it describes always commit together — no
  separate failure mode where the action succeeds but the notification
  is silently lost, or vice versa.
- Cross-device read-state consistency is free: one backend row, one
  truth, no client-side reconciliation logic anywhere.
- Zero new infrastructure (no queue, no worker) for this phase.

**Negative / trade-offs:**

- No push delivery yet — a user who isn't actively looking at the app
  won't be alerted in real time. Acceptable for MVP; FCM integration is
  a planned, separable addition that layers on top of this data model
  without requiring a redesign.
- Creating the notification synchronously inside the triggering
  request adds one extra write to that request's transaction. Not a
  practical concern at MVP scale; revisit if a specific action's
  notification fan-out (e.g. one event notifying many recipients) ever
  becomes a real hot path.
- A hard-deleted target leaves a notification pointing at nothing
  (no FK to enforce otherwise) — acceptable per Decision above, since the
  notification's own text/type still conveys what happened even if the
  target is gone; the client is expected to handle a missing target
  gracefully (e.g. "this check-in is no longer available") rather than
  assume it always resolves.
