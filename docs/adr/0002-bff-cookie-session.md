# ADR-0002: Cookie session via a BFF, not tokens in the SPA

- **Status:** proposed
- **Date:** TODO(Mario)
- **Deciders:** Mario Prskalo

## Context
- The SPA needs authenticated access to the API.
- Identity provider: Keycloak (OIDC).
- The SPA and API can be served from the same origin.
- TODO(Mario): anything else that forces this decision.

## Options considered
| Option | Pros | Cons |
|---|---|---|
| A · BFF: Spring app does code + PKCE, keeps tokens server-side, browser holds an HttpOnly session cookie + CSRF token | TODO(Mario) | TODO(Mario) |
| B · SPA public client: code + PKCE in the browser, access token in memory | TODO(Mario) | TODO(Mario) |
| C · Token in `localStorage` | TODO(Mario) | TODO(Mario) |

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
