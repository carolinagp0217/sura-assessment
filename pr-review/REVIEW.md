# Review — PR-214

> Replace the placeholders. Order findings from most to least severe. Reference lines as `path:line` using the line numbers of the **new** file (as in the `+` side of the diff).

## Verdict

- [ ] Approve
- [ ] Approve with comments
- [x] Request changes

## Summary for the author

<!-- A few sentences addressed to @dvaldes: the overall assessment, what must change before merge, and what is good. -->
Hello @dvaldes. The PR is small, focused and aligned with the intended goal: process Austral notifications quickly, deduplicate events and keep secrets out of the code.

However, I cannot approve it in its current state. There are critical issues around security, data loss, payment consistency and Salesforce API consumption.

## Findings

| # | Severity | File:line | Finding | Why it matters in this scenario | Suggested fix |
|---|---|---|---|---|---|
| 1 | Blocker | `src/main/java/com/meridian/payments/cl/AustralWebhookController.java:44` | Signature verification is skipped when the header is missing. | Anyone with the webhook URL can submit forged payment events, potentially activating or suspending customer policies. | Reject missing, empty or invalid signatures with `401`. Add tests for all three cases. |
| 2 | Blocker | `src/main/resources/application.yml:19` | The webhook secret is committed in plain text and remains exposed in Git history. | Anyone with repository access could generate valid webhook signatures. | Treat the secret as compromised: rotate it, remove it from the repository and history, and load the new value from Key Vault using managed identity. Use a dummy value in tests. |
| 3 | Blocker | `src/main/java/com/meridian/payments/cl/AustralWebhookController.java:53`<br>`src/main/java/com/meridian/payments/cl/AustralPaymentService.java:58` | The controller returns `200` before processing is durable. The service also swallows exceptions, preventing retries. | Notifications can be permanently lost during Salesforce outages, pod restarts or deployment. Austral will not retry after receiving `200`. | Persist or enqueue verified events before returning `200`. Return `503` if persistence fails. Implement retries, backoff and a dead-letter queue. Never swallow processing exceptions. |
| 4 | Blocker | `src/main/java/com/meridian/payments/cl/AustralNotification.java:18`<br>`src/main/java/com/meridian/payments/cl/AustralWebhookController.java:50` | The model includes payer name, RUT and email, and the controller logs the entire notification. | Personal data and policy information end up in application logs, creating unnecessary data-protection risks. | Remove payer data from the model and never log payloads. Log only correlation ID, provider, event ID and status. |
| 5 | Major | `src/main/java/com/meridian/payments/cl/AustralPaymentService.java:43` | Each notification makes two Salesforce API calls: one find and one write. | At peak volume, this can exceed the shared limit of 150,000 calls per rolling 24 hours. At 120,000 notifications per day, the integration alone could generate 240,000 calls. | Replace find-then-write with composite upserts in batches of up to 200 records. Add API budget protection and a circuit breaker for `403` and `5xx` responses. |
Severity scale:

| Severity | Meaning |
|---|---|
| Blocker | Must be fixed before merge: data loss, security exposure, wrong business outcome for customers, or compliance breach |
| Major | Should be fixed before merge: reliability, correctness in edge cases, or maintainability risk with real impact |
| Minor | Can be fixed in this PR or a follow-up |
| Nit | Style or naming; optional |

## What looks good

- The PR is small, has a clear goal and is easy to review, and the description is honest about the Copilot tests and the pending Key Vault move.
