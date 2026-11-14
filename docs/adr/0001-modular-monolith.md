# ADR-0001: Modular monolith, not microservices

- **Status:** proposed
- **Date:** 2026-10-29
- **Deciders:** Mario Prskalo

## Context
- A single developer builds and runs the system; it is a practice project.
- Learning goal: practise enforced module boundaries inside one codebase, as a contrast to the cloud microservices I work with day to day.
- The pipeline has distinct stages (intake, validate, deliver) connected by Kafka, and the deliver stage calls slow, unreliable external endpoints.
- Every write must stay consistent with its event (transactional outbox). With one database, the state change and its outbox row commit in one local transaction.

## Options considered
| Option | Pros | Cons |
|---|---|---|
| A · Modular monolith: one image, `api` and `worker` roles by profile, module boundaries verified by Spring Modulith | Simple to build, run and track for one developer or a small team. One build, one deploy, one database. State change + outbox row in one local transaction. `api` and `worker` roles scale separately. Boundaries are checked by a test, so extracting a module later stays a refactor | Shared release: a bug in one module ships with all of them. Shared schema and runtime; a module can't choose its own stack. Boundaries hold only as long as the verify test runs and stays green |
| B · Microservices: one service per stage | Independent deployment per service. Fault isolation between stages. Scaling per service. Freedom to pick a stack per service | Not "no single point of failure": services still share Kafka and usually a database cluster. Network calls and partial failures between stages. An outbox and a schema per service; no shared transaction. N pipelines, deploys, dashboards and contract versions for one developer |
| C · Layered monolith with no module boundaries | Simplest to start. Familiar separation by technical layer (controller, service, repository) | Layers separate technical concerns, not business ones. Nothing stops `documents` reaching into `delivery` internals, so boundaries erode. Extracting a module later means untangling it first |

## Decision
We will build a modular monolith: one codebase and one image, with `api` and `worker` roles chosen by Spring profile. Modules talk only through another module's API package or through events, and Spring Modulith's `ApplicationModules.verify()` fails the build on any violation.

## Consequences
- **Good:** One build, deploy and database to operate. State change and outbox event commit atomically in one local transaction. The `worker` role scales without the `api` role. Extraction stays possible because boundaries are enforced.
- **Bad / accepted risks:** A shared release and a shared schema. Boundary discipline depends on the verify test. Within the `worker` role, slow tenant endpoints compete for the same resources as the validate stage.
- **Security:** One deployable to patch and scan. No network isolation between modules: a flaw in any module runs with the whole app's database rights. This is mitigated by a least-privilege DB role and Postgres RLS on `tenant_id`.
- **Observability:** `ApplicationModules.verify()` stays green on every build. `api` p95 latency stays flat while a delivery backlog builds in the `worker`. Each pipeline stage has its own metrics (`invoicerelay.<module>.*`) and spans, so a stage can be judged on its own.
- **Revisit when:**
  - one module needs to scale far beyond the others across large environments;
  - a module needs a different release cadence or runtime;
  - a second team owns a module;
  - the `worker`'s slow external calls hurt the `api` role even when the two run as separate deployments.

## Interview soundbite
"I chose a modular monolith because I'm one developer and the hard part of this system is correctness, not scale. One image with `api` and `worker` roles still lets the pipeline scale separately, and a state change and its outbox event commit in one local transaction. The trade-off is a shared release and database. Spring Modulith's verify test keeps the boundaries honest, so extracting a service later is a refactor, not a rewrite."
