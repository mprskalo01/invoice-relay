# Conventions

## Backend
**Layout:** package-by-module, Clean Architecture inside each module.
```
backend/src/main/java/dev/marioprskalo/invoicerelay/
├─ InvoiceRelayApplication.java
├─ documents/
│  ├─ domain/          entities, value objects, invariants · no Spring/JPA imports
│  ├─ application/     use cases, ports (interfaces), @Transactional boundaries
│  ├─ infrastructure/  persistence adapters, Kafka, HTTP clients
│  └─ web/             controllers, request/response DTOs, mappers
├─ tenants/ … delivery/ … outbox/ …   (same shape)
└─ shared/             tenant context, Problem Details, observability
```
- **Dependencies point inward:** `web` → `application` → `domain`; `infrastructure` implements ports defined in `application`.
- **Cross-module access:** only through another module's API package or through events. Spring Modulith's `ApplicationModules.verify()` test enforces it.
- **Null-safety:** add `@NullMarked` (JSpecify) in each `package-info.java`; mark exceptions with `@Nullable`.
- **Types:** records for DTOs, value objects and events; no Lombok on domain types.
- **Validation:** Bean Validation at the web edge for shape (sizes, formats); business invariants live in `domain/`. Both are tested.
- **Transactions:** only on application services. No HTTP calls or Kafka sends inside a DB transaction (the outbox exists for this).
- **Persistence:** TODO(Mario): Spring Data JPA vs Spring Data JDBC vs jOOQ. Decide in the Wk 1 · Day 6 step (approach + rejected alternative) and record it here.
- **Migrations:** Flyway, `V<n>__<verb>_<what>.sql`; never edit an applied migration.
- **Errors:** Problem Details (RFC 9457). `type` = `https://invoice-relay.marioprskalo.dev/problems/<slug>` (a stable identifier, which doesn't have to resolve), plus `title`, `status`, `detail`, `instance`, and `errors[]` for validation. Never stack traces. Another tenant's resource → 404.
- **Logging:** SLF4J, structured output; MDC carries `traceId`, `spanId`, `tenantId`, `documentId`. Never log bodies, tokens, cookies, secrets or signatures. Levels: `ERROR` = needs a human, `WARN` = degraded but handled.
- **Metrics:** Micrometer, `invoicerelay.<module>.<thing>` (e.g. `invoicerelay.delivery.attempts`, tag `outcome`). Low-cardinality tags only (never `documentId`).
- **Tracing:** OpenTelemetry via `spring-boot-starter-opentelemetry`; `@Observed` for custom spans; context propagated through Kafka headers.

## Tests
| Kind | Name | Runs in | Tools |
|---|---|---|---|
| Unit | `*Test` | `mvn test` (Surefire) | JUnit 5, AssertJ |
| Slice / integration | `*IT` | `mvn verify` (Failsafe) | Spring test slices, `RestTestClient`, Testcontainers (Postgres, Kafka, Keycloak), WireMock |
| Architecture | `ModularityTest` | `mvn test` | Spring Modulith |
| Frontend unit | `*.test.ts(x)` | `npm run check` | Vitest, Testing Library |
| End-to-end | `e2e/*.spec.ts` | `npx playwright test` | Playwright |

- Test names describe behaviour: `rejectsInvoiceWhenTotalsDoNotMatch`.
- Use real Postgres (Testcontainers), never H2. Use Awaitility for async code; no sleeps.
- Every security rule gets a 401 / 403 / 404 test.

## Frontend
```
frontend/src/
├─ app/            providers, router, layout
├─ features/
│  └─ documents/   components/ hooks/ routes/ types.ts
├─ lib/api/        the only place that calls fetch: client, CSRF header, Problem Details parsing
└─ components/ui/  shared presentational components
```
- Strict TypeScript, no `any`, no barrel (`index.ts`) files, components under 150 lines.
- Server state lives in TanStack Query; filters and paging live in the URL; no global store unless one is needed.
- Tailwind only, no CSS-in-JS. Accessible markup (labels, roles, focus).
- `npm run check` = `tsc --noEmit` + lint + Vitest.

## Git
- **Branches:** `step/wNN-<slug>` per build step; `fix/<slug>` and `docs/<slug>` otherwise. One PR per step, squash-merged.
- **Commits:** Conventional Commits, `type(scope): imperative summary`. Types: `feat`, `fix`, `test`, `refactor`, `docs`, `build`, `ci`, `chore`. Scope = module or area (`documents`, `frontend`, `deploy`).
- The PR description follows `.github/pull_request_template.md`.
- No secrets in git. Local dev values live in `deploy/` and are marked `# dev only`.
