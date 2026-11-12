# ADR-0001: Modular monolith, not microservices

- **Status:** proposed
- **Date:** TODO(Mario)
- **Deciders:** Mario Prskalo

## Context
- A single developer builds and runs the system.
- The pipeline has distinct stages (intake, validate, deliver) connected by Kafka, and the deliver stage calls slow, unreliable external endpoints.
- Every write must stay consistent with its event (transactional outbox).
- TODO(Mario): anything else that forces this decision.

## Options considered
| Option | Pros | Cons |
|---|---|---|
| A · Modular monolith: one image, `api` and `worker` roles by profile, module boundaries verified by Spring Modulith | TODO(Mario) | TODO(Mario) |
| B · Microservices: one service per stage | TODO(Mario) | TODO(Mario) |
| C · Layered monolith with no module boundaries | TODO(Mario) | TODO(Mario) |

## Decision
TODO(Mario)

## Consequences
- **Good:** TODO(Mario)
- **Bad / accepted risks:** TODO(Mario)
- **Security:** TODO(Mario)
- **Observability:** TODO(Mario)
- **Revisit when:** TODO(Mario)

## Interview soundbite
TODO(Mario)
