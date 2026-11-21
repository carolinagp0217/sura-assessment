# ADR-0002: Implementation stack and data store
 
- **Status:** Proposed
- **Date:** 2026-11-20
## Context
 
The integration has to verify provider signatures on the raw request bytes for three different schemes, translate every notification into one canonical event, keep accepted notifications durable until they are processed, keep a small piece of transactional state (duplicates, ordering, Salesforce sync bookkeeping, operations search, publication outbox), write to Salesforce in batches within a shared API budget, and be easy to test.
 
State to keep: about 1M notifications a month including peaks, one row per payment, a conditional "only if newer" update, a unique key to discard duplicates, lookups by several identifiers, and retention by age.
 
Decision drivers: correctness primitives (atomic conditional update and unique constraints); testing without credentials or licensed runtimes; fit with the rule that every API goes through the API Manager; low cost at a small steady volume with an 8x month-end peak; operability on Azure Monitor and managed identities; skills in the squad.
 
## Options considered
 
### Option 1 — Mule 4 on the integration platform
- Pros: reference stack; native API Manager fit; DataWeave is concise for mappings; platform operations already exist.
- Cons: MuleSoft Enterprise dependencies may need credentials that are not available here; the leader-elected batching loop and the budget guard are awkward in flows; unit tests need the MUnit toolchain.
### Option 2 — Java 21 + Spring Boot 3 on Azure Container Apps, published through the API Manager
- Pros: full control of raw-byte signature verification and response codes; library support for Service Bus, JDBC, Resilience4j (circuit breaker, retry) and Micrometer/OpenTelemetry; JUnit 5 and Testcontainers run anywhere; Container Apps give internal ingress for the API Manager, scaling on queue backlog, managed identity and modest cost; the scenario allows services on Azure behind the API Manager.
- Cons: we own what the platform would give (mapping layer, connectors, standard error handling); operations teams may expect Mule; two skill sets in the organization.
### Option 3 — Relational database: PostgreSQL (Azure Database for PostgreSQL Flexible Server)
- Pros: one atomic `INSERT … ON CONFLICT DO UPDATE … WHERE` with tuple comparison discards duplicates and applies "only if newer" in a single statement and returns the new version; unique keys; partial indexes for the pending queues; advisory locks for the single leader; Testcontainers for local tests.
- Cons: always-on cost even when idle; no built-in expiry, so retention is a nightly batched delete; scales vertically only (ample at this volume).

## Decision
 
- **Option 2** for compute and stack: three Container Apps built from one codebase, `intake-service`, `payment-processor` and `ops-api`, with a shared `canonical` module and one adapter per provider, published through the API Manager. The intake runs at least 2 replicas; the processor at least 2, scaled on queue backlog, with the Salesforce flusher and the publisher as leader loops on one replica at a time; the Ops API at least 1.
- **Azure Service Bus Standard** as the broker: a queue `payments-inbound` between the intake and the processor (one consumer) and a topic `payments-recorded` for Finance and Notifications (several subscribers). It provides duplicate detection, dead-letter queues and expiry with dead-lettering.
- **Option 3, PostgreSQL**, for the ledger, because the correctness rules are natural as a unique key plus one conditional upsert. Cosmos DB was rejected for the same reason. MySQL is workable but needs a three-statement transaction (no `WHERE` on `ON DUPLICATE KEY UPDATE`), has no partial indexes and no `RETURNING`, which leaves more room for mistakes in the code that decides duplicates and ordering.

## Consequences
 
- Positive:
  - Correctness rules expressed as a unique key and one conditional upsert, easy to test.
  - Everything runs and tests locally, with no dependency on credentials we do not have.
  - The provider-facing path depends only on Key Vault (cached) and Service Bus; the intake touches no database.
  - A queue where there is one consumer and a topic only where there are several keeps the messaging simple.
  - Cost scales with traffic only at the compute edge; the baseline is small.
- Negative:
  - We build and maintain what Mule would provide (connectors, mapping layer, standard error handling), so there is more custom code to review.
  - The platform operations team may not support Java services; ownership and on-call sit with the squad.
  - Always-on PostgreSQL has a fixed cost and needs patching, backups and retention jobs managed by us.
  - Service Bus Standard has a 99.9% SLA: a long broker outage forces `503` responses.
  - A burstable, non-high-availability database has lower availability than the rest of the design.
  - Two ways of building integrations in the same organization (Mule and Spring Boot) increase cognitive load.
- Risks and how they are mitigated:
  - **Broker outage longer than a provider's retry window** (Andina retries for 24 h). Mitigation: fail fast with `503`, alert on publish failures, and treat a prolonged outage as an incident, using the provider's dashboard to resend.
  - **Database outage.** Mitigation: the intake does not depend on it, the consumer pauses and the queue holds the messages (14-day expiry with dead-lettering); revisit high availability if the objectives start burning.
  - **The API Manager alters the raw body or restricts response codes.** Mitigation: confirm with the platform team before building, and add a contract test with the provider samples.
  - **Tier features and prices unverified** (Entra authentication, high-availability options). Mitigation: verify them in the Azure pricing calculator and revise this ADR if the estimate is far off.
  - **Concurrency bugs in the ordering upsert.** Mitigation: a concurrency test with many consumers and shuffled events of one payment.
 