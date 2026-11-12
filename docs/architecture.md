# Architecture

> Living document: it describes what is built, and anything not built yet is marked *(planned)*. Decisions and their reasons live in [ADRs](adr/README.md).

## Goal and quality attributes
The system must make one guarantee: **every accepted invoice is delivered to its tenant's endpoint at most once in effect.** Duplicates may be sent on the wire but are harmless, because the receiver de-duplicates on `Idempotency-Key`, and nothing is lost.

| Attribute | How it is met |
|---|---|
| No lost documents | Transactional outbox: the state change and its event are committed together *(planned)* |
| No duplicate effects | Idempotent consumers + `Idempotency-Key` on delivery *(planned)* |
| Tenant isolation | `tenant_id` on every row, checked in the application layer **and** enforced by Postgres RLS *(planned)* |
| Recoverability | Dead-letter topic with replay *(planned)* |
| Traceability | One trace from upload to delivery across HTTP and Kafka *(planned)* |
| Security | BFF: tokens stay server-side; HttpOnly session cookie + CSRF *(planned)* |

## Context
```mermaid
flowchart LR
  user([Tenant user]) -->|browser| sys[invoice-relay]
  sys -->|OIDC| kc[Keycloak]
  sys -->|signed webhook POST| ep([Tenant endpoint])
```

## Containers
```mermaid
flowchart LR
  UI[React SPA<br/>same origin] -- session cookie + CSRF --> API
  subgraph app[invoice-relay image]
    API[role: api<br/>BFF + REST]
    W[role: worker<br/>pipeline stages]
  end
  API -- OIDC code + PKCE --> KC[Keycloak]
  API --> PG[(PostgreSQL<br/>documents · outbox · RLS)]
  PG -- outbox relay --> K[(Kafka)]
  K --> W
  W --> PG
  W -- HMAC-signed POST --> EP([Tenant webhook])
  W -. exhausted retries .-> DLT[(DLT topic)]
  API & W -. OTLP .-> OC[OTel collector] --> GR[Grafana]
```
**One codebase, one image, two roles** chosen by Spring profile: `api` serves HTTP and `worker` runs the Kafka stages. Each role scales on its own (ECS services) without splitting into microservices; see ADR-0001.

## Modules
Base package `dev.marioprskalo.invoicerelay`. Each module has `domain/` · `application/` · `infrastructure/` · `web/` (see [conventions](conventions.md#backend)).

| Module | Owns | Talks to | Status |
|---|---|---|---|
| `documents` | Document intake, validation, lifecycle state | `outbox` (events), `tenants` (API) | *(planned, v1)* |
| `tenants` | Tenants, webhook endpoints, signing secrets | — | *(planned, v1)* |
| `delivery` | Delivery attempts, retries, HMAC signing, SSRF guard | `documents` (events) | *(planned, v2)* |
| `outbox` | Outbox table, relay to Kafka | Kafka | *(planned, v2)* |
| `shared` | Tenant context, Problem Details, observability helpers | — | *(planned, v1)* |

`ApplicationModules.verify()` runs as a test and fails the build on boundary violations.

## Key flows

### 1 · Submit a document (v1)
```mermaid
sequenceDiagram
  participant B as Browser
  participant A as api (BFF)
  participant DB as Postgres
  B->>A: POST /documents (cookie + X-XSRF-TOKEN, JSON invoice)
  A->>A: authn (session) → authz (role, tenant) → validate
  A->>DB: INSERT document (status=RECEIVED) [+ outbox row, v2] in one tx
  A-->>B: 201 Location: /documents/{id} · or 400/422 Problem Details
```

### 2 · Pipeline (v2) *(planned)*
```mermaid
sequenceDiagram
  participant DB as Postgres
  participant R as Outbox relay
  participant K as Kafka
  participant V as Validate stage
  participant D as Deliver stage
  participant E as Tenant endpoint
  R->>DB: poll unpublished outbox rows
  R->>K: publish document.received (key = documentId)
  R->>DB: mark published
  K->>V: consume (idempotent)
  V->>DB: RECEIVED→VALIDATED + outbox(document.validated) in one tx
  K->>D: consume document.validated (idempotent)
  D->>E: POST, HMAC signature, Idempotency-Key=documentId
  alt 2xx
    D->>DB: VALIDATED→DELIVERED
  else retries exhausted
    D->>K: document.failed → DLT
  end
```
Open design points (decided in the Wk 5 step): hand-written outbox relay vs Spring Modulith's event publication registry; polling vs CDC; how consumer de-duplication is keyed.

### 3 · Login (v1) *(planned)*
Authorization code + PKCE between the BFF and Keycloak. Tokens are stored server-side in the session; the browser only ever holds an HttpOnly, Secure, SameSite session cookie. See ADR-0002.

## Deployment (v2) *(planned)*
Terraform → AWS: ALB (path routing: `/api/*` and `/login/*` → api service, `/*` → static SPA), ECS Fargate services `api` and `worker`, RDS Postgres, Kafka (choice TBD in the Wk 8 step), CloudWatch / Grafana. GitHub Actions authenticates with OIDC, so there are no long-lived keys. A budget alarm is created first.

## Bottlenecks and scaling notes
- **Outbox relay:** one poller is a throughput ceiling and a single point of delay → batch with `FOR UPDATE SKIP LOCKED`; alert on outbox lag.
- **Deliver stage:** slow tenant endpoints hold up consumers → per-call timeouts, bounded concurrency, per-tenant isolation to consider.
- **Kafka partitioning:** key = `documentId` gives per-document ordering; parallelism is capped by the partition count.
- **Postgres:** keyset pagination on `(tenant_id, created_at, id)`; indexes verified with `EXPLAIN` (Wk 6).
