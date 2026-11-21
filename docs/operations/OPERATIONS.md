# Operations

**Correlation identifier.** `correlationId` is a UUID created at `intake-service` when the request arrives. It then travels with the notification.

Operations finds a notification by provider event id, payment id or policy through the Ops API, which returns its status and correlation id, so nobody searches raw logs.

**Log (structured JSON):** time, level, service, `correlationId`, `traceId`, country, provider, `eventId`, `paymentKey`, status transition and reason code, durations, delivery count, batch id and size, and the Salesforce HTTP status and error **code** (not the body).

**Trace (OpenTelemetry to Azure Monitor):** intake (verify, parse, publish), consumer (dedupe, ordering upsert), Salesforce flush (one span per batch, linked to the notification traces) and publisher.

**Never log, trace or use as a metric label:**
- Payer personal data: names, national ID numbers (such as the RUT) and emails. The canonical event and the DTOs do not carry them, so they cannot be logged by accident.
- Raw request bodies or provider payloads.
- Secrets and credentials: HMAC secrets, signatures, `Authorization` headers and Salesforce tokens.

## 3. Alerts

Pages go to the **Payments Integration on-call** (escalating to the team lead after 15 minutes without acknowledgement). The Salesforce on-call is also paged when the cause is on their side; Platform/SRE for Azure resources. Tickets go to the squad queue. Every alert exists for each country, defined as code.