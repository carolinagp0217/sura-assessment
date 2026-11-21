# Submission checklist

Tick every item before you send your repository URL. Missing items are scored as missing.

## General
- [ ] `SUBMISSION.md` completed (including the final commit SHA you will send to your recruiter)
- [x] `AI_USAGE.md` completed
- [x] `docs/ASSUMPTIONS.md` lists every assumption you made
- [ ] No secrets, tokens or real personal data anywhere in the repository or its history
- [x] Repository is **public**

## Part A — Integration and data design
- [x] `docs/design/SOLUTION_DESIGN.md` (context diagram, sequence diagram, canonical event and mapping, idempotency and ordering, error handling, capacity, data, FR7)
- [x] `docs/adr/0001-*.md` — Integration with Salesforce
- [x] `docs/adr/0002-*.md` — Implementation stack and data store
- [x] `docs/adr/0003-*.md` — optional

## Part B — Integration component
- [ ] `docs/api/webhooks.openapi.yaml`
- [ ] `docs/api/events/premium-payment.schema.json`
- [ ] `docs/api/README.md`
- [ ] Implementation in `practical/` for Quetzal Pagos (MX) and Andina Pay (CO)
- [ ] Automated tests in `practical/`
- [ ] `practical/README.md`

## Part C — Azure IaC and CI/CD
- [x] Bicep or Terraform in `infra/`, with `infra/README.md`
- [x] `docs/infra/AZURE_SERVICES.md`
- [ ] Pipeline definition in `.github/workflows/` or `pipelines/`

## Part D — Operations
- [ ] `docs/operations/OPERATIONS.md`
- [x] `reference/incident/INC-0412-RESPONSE.md`

## Part E — Code review
- [x] `pr-review/REVIEW.md`

## Part F — Responsible AI
- [x] `AI_USAGE.md`
- [ ] `docs/ai/AI_GUIDELINES.md`
