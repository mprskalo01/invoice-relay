## Step
<!-- e.g. v1 · Spring Boot skeleton + POST /documents -->

## Approach
<!-- What this does and how, in 3–5 lines. -->

## Rejected alternative
<!-- One option you didn't take, and why. -->

## How it's tested
<!-- The test(s) that prove the claim, including failure paths. -->

## Security and tenant isolation
<!-- New trust boundaries, authz checks, input limits; threat-model rows touched. Write "none" if none. -->

## Observability
<!-- Logs, metrics, spans added. Write "none" if none. -->

## Checklist
- [ ] Approach and rejected alternative written before coding
- [ ] I can explain every line: domain rules, transaction boundaries, idempotency, security checks
- [ ] `./mvnw -B verify` passes · `npm run check` passes (if the frontend was touched)
- [ ] `docs/` updated (architecture / domain / threat model) where affected
- [ ] ADR added or updated if a significant decision was made
- [ ] No secrets, tokens or personal data in the diff
