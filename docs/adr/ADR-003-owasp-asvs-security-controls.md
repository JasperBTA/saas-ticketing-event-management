# ADR-003: OWASP ASVS Compliance & Session Security Lifecycle

## Status
Accepted

## Context
Ticketing systems are frequent targets of credential stuffing, session hijacking, ticket scalping botnets, and CSRF exploits. The system required verifiable defense-in-depth adhering to the **OWASP Application Security Verification Standard (ASVS) Level 2/3**.

Key vulnerabilities addressed:
1. Token theft via Cross-Site Scripting (XSS) if tokens are stored in `localStorage`.
2. Cross-Site Request Forgery (CSRF) on state-modifying endpoints when using cookies.
3. Token leakage in reverse proxy and CDN access logs via query parameters (`?token=...`).
4. Race condition loops when multiple browser tabs simultaneously trigger refresh token rotation, causing unintended session revocation.

## Decision
We implemented a multi-layered security lifecycle across Spring Security, Nginx, and client coordination:

1. **HttpOnly Cookie Architecture:**
   - Access tokens (`jwt`) and refresh tokens (`refresh_token`) are issued exclusively with `HttpOnly`, `SameSite=Strict`, and `Secure` attributes. Client-side JavaScript cannot read raw cryptographic tokens.
2. **Defense-in-Depth CSRF Defense:**
   - In addition to `SameSite=Strict`, all non-GET requests require a custom header (`X-Requested-With: XMLHttpRequest`), which cannot be sent by cross-origin HTML `<form>` submissions without explicit CORS pre-flight consent.
3. **Sensitive Token Handling via URL Fragments:**
   - Password reset and account activation links encode cryptographic tokens in the URL fragment (`/reset-password#token=...`) rather than query parameters (`?token=...`).
   - Browsers never transmit URL fragments to servers in HTTP request lines or `Referer` headers, completely eliminating token leakage in web server access logs.
4. **Web Locks API Refresh Serialization:**
   - When access tokens expire, concurrent requests across multiple browser tabs are synchronized using `navigator.locks.request("anypass-refresh")`.
   - Exactly one tab performs the refresh mutation; subsequent tabs reuse the newly issued session cookie, eliminating refresh-token race conditions.
5. **Step-Up Authentication (Re-authentication):**
   - High-privilege actions (changing account email, remote session termination) require explicitly re-submitting the current password in a dedicated step-up challenge.

## Consequences

### Positive:
- Resilient against automated XSS token harvesting, CSRF, and concurrent tab revocation.
- High auditing standard suitable for enterprise and governmental compliance.

### Negative / Trade-offs:
- Requires hardened local and staging development configurations (HTTPS and cookie domain handling).

