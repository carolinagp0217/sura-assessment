## 1. First 30 minutes

| Min | What I do | Why |
|---|---|---|
| 0–5 | Open an incident channel, name a lead and a communications owner, start a timeline. Known so far: Colombia only, Andina only, first failure Saturday 23:41. | One owner and one timeline before anything changes. |
| 5–10 | Check if it is still happening: dead-letters per minute on `payments-co` (last one 08:09) and the Salesforce API budget (66% against 38% normally). | The 43,120 `400` responses are exactly 10 × 4,312: each failing message is retried 10 times at once. The budget is shared, so other integrations risk `403 REQUEST_LIMIT_EXCEEDED`. |
| 10–15 | Apply the interim mitigation (section 3, step 1). | Protects other teams; no code change. |
| 15–22 | Size the impact from the DLQ: 4,198 `APROBADA`, 61 `REVERSADA`, 53 `RECHAZADA`; Mexico 2 and Chile 0 are healthy. | The 37 reports are only those who called: up to 4,198 paid policies may be suspended. |
| 22–28 | Classify the `referencia` of the dead-lettered messages and run the lookups in section 2. | Separates a format problem from missing policies. |
| 28–30 | First update to the contact center: payments are stored and will be applied, ETA to follow; send us the 37 policy numbers. | Customers are calling now and the data is safe in the queue. |

## 2. Most likely root cause

**The Colombian consumer sends Andina's `referencia` to Salesforce unchanged as `Policy_Number__c`, and some references do not match the stored format.** `referencia` is what the payer typed or confirmed at checkout (editable). Salesforce stores `POL-CO-NNNNNNNNN` in upper case. A mismatch gives `400 INVALID_FIELD ... not found`, a permanent error, but v1 treats it as transient: it redelivers 10 times immediately and dead-letters it

