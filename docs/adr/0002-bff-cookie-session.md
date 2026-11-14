# ADR-0002: Cookie session via a BFF, not tokens in the SPA

- **Status:** proposed
- **Date:** 2026-10-29
- **Deciders:** Mario Prskalo

## Context
- The SPA needs authenticated access to the API.
- Identity provider: Keycloak (OIDC).
- The SPA and API can be served from the same origin.
- The app handles invoice data: commercial and possibly personal. Tokens that grant access to it must never be readable from JavaScript.
- The Spring app already exists, so acting as a BFF adds no new deployable.

## Options considered
| Option | Pros | Cons |
|---|---|---|
| A · BFF: Spring app does code + PKCE, keeps tokens server-side, browser holds an HttpOnly session cookie + CSRF token | Tokens never reach the browser, so an XSS can't exfiltrate them. Keycloak client is confidential (secret stays server-side). Refresh happens server-side. The SPA has no OIDC code. Same origin, so no CORS | Server-side session state: several `api` instances need a shared session store. CSRF protection is required on every state-changing request. Only fits same-site web clients. Logout has to end both the app session and the Keycloak session |
| B · SPA public client: code + PKCE in the browser, access token in memory | Stateless backend; easy to scale. Bearer header, so no CSRF. A common, well-supported SPA pattern | An XSS can read the in-memory token and use it elsewhere until it expires. The token is lost on reload, so the browser needs a refresh token or silent re-authentication. The IETF draft on OAuth for browser-based apps ranks the BFF pattern as the stronger option for sensitive data |
| C · Token in `localStorage` | Simplest to implement. Survives reloads | Any script on the page, including a compromised third-party one, can read the token at any time. No `HttpOnly` protection. Widely discouraged for anything sensitive |

## Decision
We will use the Spring app as a BFF. It runs the authorization code flow with PKCE against Keycloak as a confidential client and keeps tokens server-side in the session. The browser holds only an `HttpOnly`, `Secure`, `SameSite=Lax` session cookie plus a CSRF token.

The claim is deliberately narrow: an XSS can still act as the user while the page is open, but it can't steal a token to use elsewhere.

## Consequences
- **Good:** No token is reachable from JavaScript. The SPA stays simple (no OIDC library, no token refresh). One origin, so no CORS configuration.
- **Bad / accepted risks:**
  - Server-side session state. To run several `api` instances, sessions move to a shared store: Spring Session JDBC on the existing Postgres, so no new infrastructure *(planned)*.
  - Every state-changing request needs CSRF handling.
  - XSS still has to be prevented (CSP, output encoding), because it can act as the user in the open page.
- **Security:**
  - Cookie flags `HttpOnly`, `Secure`, `SameSite=Lax`. Lax rather than Strict, so the cookie survives the top-level redirect back from Keycloak.
  - Spring Security CSRF in SPA mode.
  - A new session ID on login; idle and absolute timeouts; logout invalidates the session server-side and at Keycloak.
  - See threat model T2–T4.
- **Observability:**
  - Login success and failure rate.
  - CSRF rejections (403 rate).
  - Active sessions and session-store latency once Spring Session is in place.
  - A test asserts that no token appears in any browser-visible response.
- **Revisit when:**
  - third-party API consumers appear (they need a token-based API, e.g. client credentials, alongside the BFF);
  - a mobile or native client, or another device type, is added;
  - the SPA and API have to be served from different sites.

## Interview soundbite
"The app handles financial data, so I didn't want any token readable from JavaScript. The Spring app acts as a BFF: it does code + PKCE with Keycloak as a confidential client and gives the browser only an HttpOnly session cookie plus a CSRF token. An XSS can still act as the user while the page is open, but it can't steal a token to use elsewhere. The cost is server-side session state and CSRF handling, which I accept for a same-origin web app."
