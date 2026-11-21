# Assumptions

> The scenario is intentionally incomplete. Record every assumption you needed to make, and the part it affects.

| # | Assumption | Why I needed it | Part(s) affected |
|---|---|---|---|
| 1 | Salesforce is called through the API Manager, not point to point, and the gateway passes the body and headers untouched and lets our service choose the response code (Andina accepts only `200` or `204`) | The scenario does not say how Salesforce is reached, and signatures are computed over the raw bytes; if the gateway rewrites them, valid requests would fail | A, B, C |
| 2 | Finance and Notifications want one event per recorded payment only after Salesforce confirms it, and accept that their latency follows Salesforce's availability (97%, outages of 5 to 90 minutes, freshness measured from the time we receive the notification) | The scenario does not say when they consume events; the choice also decides whether the one-hour freshness target can be met | A, D |
