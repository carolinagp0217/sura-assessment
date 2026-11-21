# ADR-0001: Integration with Salesforce
 
- **Status:** Proposed
- **Date:** 2026-11-20
## Context
 
Salesforce is the system of record for payments and hosts the automation that reinstates or suspends a policy from `Premium_Payment__c.Status__c`. The integration must record every payment there exactly once in business terms, without leaving a payment in a state that contradicts the provider's latest one, and without ever writing the policy status.
 
The forces at play:
 
- Availability ≈ **97%** per month (about 21.6 h down), in unplanned windows of 5–90 minutes.
- **150,000 API calls per rolling 24 h** for this integration. Each REST request counts as one call regardless of the records it carries. The org-wide limit is shared: exceeding it rejects **every** integration with `403 REQUEST_LIMIT_EXCEEDED`.
- Latency: single upsert p50 0.3 s, p95 1.2 s, up to 8 s under load; 200-record upsert p50 1.5 s, p95 4 s. Maximum **25 concurrent requests**.
- Upsert by external ID **overwrites**; there is no "update only if newer".
- Volume: 15,000 notifications a day, up to 120,000 at month-end with bursts of 40/s for 15 minutes; about 3% duplicates.
- Freshness: 95% within 5 minutes, 99.5% within 1 hour. No accepted notification may be lost. Provider timeouts of 5–10 s.
Decision drivers: do not spend the shared budget on avoidable calls; survive Salesforce downtime without losing or reordering data; never write an older state over a newer one; keep the provider response independent of Salesforce; keep it operable by a small squad.
 
## Options considered
 
### Option 1 — Synchronous, per record (inside the webhook request)
- Pros: simplest design; immediate visibility in Salesforce; no extra state to keep.
- Cons: a 97%-available system inside a 5–10 s provider timeout, so every Salesforce incident becomes provider errors and retries; 120,000 calls a day at peak is **80%** of the budget before retries or lookups; per-record throughput with 25 concurrent requests is about 21/s at p95 latency (about 3/s at 8 s), below the 40/s burst. Moving to asynchronous but still sending one record per call does not fix the budget or the throughput.
### Option 2 — Asynchronous, event by event, in batches of 200
- Pros: stays far below the budget (about 600–8,000 calls a day).
- Cons: one batch may carry two events of the same payment and concurrent batches can write an older state after a newer one, so ordering has to be solved separately.
### Option 3 — Asynchronous, state-based, batched
- Pros: a ledger keeps one row per payment; a single leader sends the **current state** of pending payments in batches of up to 200 records, one call each. Repeated events collapse into one write, ordering is decided in our ledger before writing, replays and duplicates are harmless, and Salesforce outages become a backlog. Calls: at most 8,640 a day (5.8% of the budget).
- Cons: more moving parts (leader election, pending-state bookkeeping, budget guard, circuit breaker); visibility lags by up to the flush wait; intermediate states may never reach Salesforce.
## Decision
 
**Option 3**: asynchronous, state-based, batched upsert, with these controls.
 
1. **Decoupling.** The webhook answers `2xx` once the canonical event is durable in the broker. Salesforce is never in the provider-facing path.
2. **Write the current state.** Each payment has one ledger row with a `version`. The flusher sends that row's current state in a composite upsert (`PATCH /composite/sobjects/Premium_Payment__c/Payment_Key__c`, `allOrNone: false`, up to 200 records, one call). A row is marked synced only if its version did not change meanwhile.
3. **Cadence.** Tick every 2 s; send when 200 rows are due or the oldest due row has waited 10 s. At most 8,640 calls a day from the loop.
4. **Single writer.** One leader (a PostgreSQL advisory lock held on a dedicated, direct connection) sends all batches, so no two batches carry the same payment. Concurrency is capped at 5 calls in flight (the limit is 25); with a 4 s p95 that gives about 250 records/s.
5. **Budget guard.** Per-minute call counts persisted in the ledger; soft cap of 100,000 per rolling 24 h; alert at 50%; the token is cached (about 2 h) and token requests are counted.
6. **Circuit breaker.** On `5xx` or timeouts: backoff from 2 s to 60 s and half-open probes of one call. On `403 REQUEST_LIMIT_EXCEEDED`: open for at least 15 minutes (doubling to 60) and page, because the limit is shared.
7. **Per-record errors** follow the Salesforce error table: lock conflicts retried with backoff, policy-not-found retried for up to 24 h then parked, other validation errors parked immediately, `401` retried once after refreshing the token.
8. **Least privilege.** The integration user can create and edit `Premium_Payment__c` and only read `Policy__c`.
9. **Salesforce is called through the API Manager** (assumption: the scenario rule is that systems are not called point to point).
10. **Downstream publication only after Salesforce confirms.** Finance and Notifications receive an event for each notification once Salesforce has recorded it. The confirming transaction marks the notifications `RECORDED` and flags them for publication (transactional outbox); a leader loop publishes them to the topic `payments-recorded` in payment and version order. The alternative, publishing the canonical event as soon as it reaches the broker, was rejected because consumers could act on payments Salesforce does not have and would receive duplicates and out-of-order events. The price is that they depend on Salesforce's availability.
Why it wins here: it is the only option that keeps the daily budget bounded independently of volume, absorbs the 40/s burst with about six times of headroom, turns the 97% availability into a delay instead of an error, and solves ordering in one place.
 
## Consequences
 
- Positive:
  - Budget use is bounded and independent of volume: at most about 6% on any day, so a month-end peak cannot cause a `REQUEST_LIMIT_EXCEEDED` that would stop other integrations.
  - The 40/s burst becomes about 180 calls over 15 minutes.
  - Salesforce outages become a backlog: after a 90-minute outage at the average peak rate (about 7,500 events) it drains in about 30 s; a full burst backlog in about 144 s.
  - Repeated events for one payment collapse into one write, which also avoids a suspend-and-reinstate flip-flop for a capture followed quickly by a refund.
  - Ordering does not depend on broker order; replays and duplicates are harmless.
- Negative:
  - **Eventual consistency**: visibility lags by up to the flush wait (10 s) plus the call latency, even when Salesforce is healthy.
  - **Finance and Notifications inherit Salesforce's downtime.** An outage of up to 90 minutes delays them too, and their latency is higher than with early publication.
  - **More moving parts** than a simple consumer: leader election, pending-state bookkeeping, a budget guard, a breaker, an outbox and a publisher.
  - **A single writer** concentrates the flushing in one process: if it stalls, flushing pauses until another replica takes the lock (seconds). Throughput of about 250 records/s is ample for the load.
  - **Intermediate states may never reach Salesforce** (coalescing), so the org's flows see only net transitions.
  - The ledger becomes a second place holding payment state.
- Risks and how they are mitigated:
  - **The one-hour freshness target cannot be guaranteed by the integration alone.** If most Salesforce downtime comes from windows near 90 minutes, about 1% of notifications exceed an hour against the 0.5% allowed. Mitigation: agree limits on outage windows with the Salesforce team, and alert on backlog age.
  - **Assumptions about Salesforce** (its flow handles a payment inserted directly as `Refunded`; a partial refund stays `CAPTURED`; the composite upsert reports failures per record). Mitigation: confirm with the Salesforce team and test in a sandbox before go-live.
  - **Shared limit exhausted by another integration.** Mitigation: the breaker opens on `403`, the budget guard keeps us at or below 67% of our allocation, and the on-call is paged.
  - **Ledger diverges from Salesforce.** Mitigation: the version handshake marks a row synced only for the version actually sent; alerts on rows pending beyond the objective; a reconciliation job comparing counts.
  - **Finance and Notifications SLAs missed during outages.** Mitigation: get their explicit agreement to this behavior.
  - **Load behavior unverified.** Mitigation: load-test the flusher against a Salesforce sandbox with realistic latencies.
 