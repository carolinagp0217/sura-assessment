# ADR-0003: Ordering and conflict resolution for out-of-order notifications
 
- **Status:** Proposed
- **Date:** 2026-11-20
## Context
 
None of the three providers guarantees ordering, and all deliver at least once. Salesforce's upsert by external ID **overwrites** unconditionally, and a wrong `Premium_Payment__c.Status__c` has a direct customer effect: the Salesforce flow reinstates a suspended policy when a payment becomes `Captured` and recalculates (possibly re-suspending) when it becomes `Refunded`. A payment must never be left in a state that contradicts the provider's latest real state, for example a refunded payment shown as captured, and it must be recorded exactly once in business terms.
 
Facts from the provider documentation:
 
- Quetzal: `created` is the event time; no ordering guarantee; refunds can be partial.
- Andina: `PENDIENTE`, then `APROBADA` or `RECHAZADA`, later possibly `REVERSADA`; `fecha_transaccion` is the time of the state change; every retry arrives with a new `notificacion_id`.
- Austral: `occurredAt` is the time of the state change; the signature carries no timestamp, so valid requests can be replayed.
- Timestamp precision, and what happens with contradictory events on the same payment (for example `RECHAZADA` then `APROBADA`), are not specified.
Decision drivers: a refund or reversal must never be undone by a late capture; the rule must be deterministic, identical for every provider and testable; it must tolerate clock skew and equal timestamps; it must not cost Salesforce API calls; it must be atomic under concurrent consumers.
 
## Options considered
 
### Option 1 — Arrival order, or provider timestamp only
- Pros: trivial to implement; the timestamp is the provider's own notion of time.
- Cons: arrival order lets any late event overwrite a newer state; a timestamp alone fails with skewed or equal timestamps, so a `PENDING` stamped slightly after a `CAPTURED`, or a refund stamped before its capture, would roll the state back.
### Option 2 — Compare with the stored state in Salesforce before writing
- Pros: compares against the real stored state.
- Cons: one extra call per record against the shared budget; races between concurrent writers; unusable while Salesforce is down; does not give lifecycle monotonicity.
### Option 3 — Lifecycle rank first, then provider time, evaluated in our ledger in one atomic statement
- Pros: protects the customer's coverage whatever the timestamps say; deterministic; no Salesforce calls; one atomic statement to test; also neutralizes replays and duplicates.
- Cons: `FAILED` and `CAPTURED` share a rank so time decides between them; a bogus `REFUNDED` cannot be undone by later normal events; the ordering knowledge lives in a second store beside Salesforce.
## Decision
 
**Option 3.** Each payment has one ledger row. An incoming event replaces it only if its tuple is **strictly greater**, compared in this order:
 
1. `status_rank`: `PENDING` 0, `FAILED` 1, `CAPTURED` 1, `REFUNDED` 2. This is a monotonic lifecycle rank: a payment's rank only goes up.
2. `occurred_at` (UTC provider time).
3. `refunded_amount`.
4. `event_id`, a deterministic final tie-break.
The comparison and the write are a single PostgreSQL `INSERT … ON CONFLICT DO UPDATE … WHERE (new tuple) > (stored tuple) RETURNING version`, so concurrent consumers cannot interleave incorrectly: the row lock taken by the upsert serializes events of the same payment. A returned row means the event was applied; no row means it is stale. Events that lose are stored as `SUPERSEDED`. Only the ledger's current state, never an individual event, is sent to Salesforce.
 
Canonical `REFUNDED` is used only for full refunds and reversals; a partial refund stays `CAPTURED` with the refunded amount (an assumption to confirm with the Salesforce team).
 
Why it wins here: rank dominates time, so a refund with a skewed timestamp still beats a late capture, which is what protects a customer's coverage; time then orders events of equal rank.
 
## Consequences
 
- Positive:
  - A late capture can never overwrite a refund, whatever the timestamps say.
  - Duplicates and replays are inherently harmless: an equal event does not beat itself and an old event cannot beat a newer one. This also covers Austral's lack of a signed timestamp.
  - No dependence on broker order and no extra Salesforce calls.
  - Partial refunds are ordered by amount, so a larger refund wins over a smaller one.
- Negative:
  - `FAILED` and `CAPTURED` share a rank, so for those two time decides; wrong or coarse provider timestamps may let the wrong one win.
  - Because rank dominates time, a genuinely newer re-capture after a refund under the **same** payment identifier would be ignored.
  - A bogus `REFUNDED` event cannot be undone by later normal events; it needs a manual correction through the audited operations API.
  - Equal timestamps fall through to amount and event id, which is deterministic but arbitrary.
  - The ordering knowledge lives in our ledger, a second source of truth beside Salesforce.
- Risks and how they are mitigated:
  - **Provider behavior not documented** (a failed charge retried under the same identifier, timestamp precision, whether a reversal is always total). Mitigation: confirm with each provider and keep the assumption written down in the design document.
  - **Re-capture under the same identifier.** Mitigation: assume providers use a new identifier for a new charge, and alert when an event is rejected for ranking lower than the stored state so it is visible.
  - **Ledger lost or inconsistent.** Mitigation: point-in-time backups; rebuild from Salesforce state and provider reports.
  - **Rule implemented wrongly.** Mitigation: property-based tests that apply every permutation of a payment's events and assert the same final state, plus a concurrency test with many consumers.
  - **Defense in depth.** Mitigation: ask the Salesforce team whether a guard that rejects an older `Status_Changed_At__c` can be added on their side.
 