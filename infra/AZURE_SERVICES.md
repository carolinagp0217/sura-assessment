# Azure services

Services used by the design, why each was chosen, what was discarded, and how each supports reliability, security. It explains the decisions already made in Part A and in the ADRs.

## 1. Azure Container Apps
- **Why:** runs the applications as containers with internal ingress for the API Manager, scale on HTTP load or on the queue backlog, managed identity and no cluster to run. The 8x month-end peak is absorbed by scaling, not by sizing for the peak all month.
- **Reliability:** at least two replicas of the intake and the processor, so one restart never stops intake; health probes; one active revision with rolling replacement. Only one processor replica flushes to Salesforce at a time, enforced by a database lock.
- **Security:** managed identity per app, internal ingress only, images pulled with the identity, no secrets in environment variables.

## 3. Azure Service Bus
- **Why:** the design needs one **queue** (intake to processor, one consumer) and one **topic** with two subscriptions (Finance, Notifications), with **dead-letter queues, duplicate detection and message expiry with dead-lettering**. Those are the durability guarantees: the webhook answers `2xx` only after the broker accepts the message.
- **Tier:** Standard, queue and topic of 1 GB, 14-day TTL, duplicate detection 7 days, max delivery count 10, lock 1 minute.
- **Reliability:** SLA of 99.9% for Standard. A longer broker outage makes the intake answer `503` so providers retry; this limit is accepted and stated in ADR-0002.

## 4. Azure Database for PostgreSQL Flexible Server

- **Why:** the ledger: duplicates (unique key), ordering (one conditional upsert with `WHERE` and `RETURNING`), sync bookkeeping, operations search and the publication outbox. Reasons are in ADR-0002.
- **Security:** private access in the virtual network, public access disabled, Microsoft Entra authentication only (no passwords), no payer data stored.

## 5. Azure Key Vault

- **Why:** provider webhook secrets and the Salesforce client secret live here, read at runtime by the identities that need them; rotation and audit without redeploying.

## 6. Azure Monitor: Log Analytics, Application Insights, alerts

- **Why:** logs, traces (OpenTelemetry) and metrics in one place, with the alerts. The logs of Key Vault, Service Bus and PostgreSQL go to the same workspace.
- **Security:** logs never contain payer data (enforced in code, see operations); retention 30 days in dev and test, 90 in prod.