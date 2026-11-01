# Security & Request Pipeline (OWASP ASVS Compliance)

The **SaaS Ticketing & Event Management platform** *(codename: AnyPass)* implements security controls aligned with **OWASP ASVS (Application Security Verification Standard) Level 2 & 3**.

## Security Flow Architecture

```mermaid
sequenceDiagram
    autonumber
    actor User as User / Browser
    participant Nginx as Hardened Nginx Proxy
    participant FilterChain as Spring Security Filter Chain
    participant JwtFilter as JwtAuthenticationFilter
    participant Controller as REST Controller (Editor / Auth)
    participant LockAPI as Web Locks API (Browser)

    Note over User, Nginx: 1. Request Ingestion & Network Hardening
    User->>Nginx: HTTPS Request (HttpOnly Cookie, X-Requested-With)
    Nginx->>Nginx: Validate Security Headers (HSTS, CSP, X-Frame-Options)
    Nginx->>Nginx: Strip internal operator headers & Rate Limit Auth

    Note over Nginx, FilterChain: 2. Filter Chain Processing
    Nginx->>FilterChain: Forward to port 8080 (Non-root daemon)
    FilterChain->>JwtFilter: Intercept request

    alt Ops API Machine Request
        JwtFilter->>FilterChain: Match X-Ops-Api-Key
    else Authenticated User Request
        JwtFilter->>JwtFilter: Extract HttpOnly JWT & verify signature
        JwtFilter->>JwtFilter: Populate SecurityContext (Roles & Granular Permissions)
    end

    FilterChain->>Controller: Route to Controller (@PreAuthorize RBAC)
    Controller-->>User: 200 OK / 201 Created (JSON Response)

    Note over User, LockAPI: 3. Session & Token Refresh Lifecycle
    alt Access Token Expired (401 Response)
        User->>LockAPI: navigator.locks.request("anypass-refresh")
        Note over LockAPI: Serializes refresh across multiple tabs (prevents race conditions)
        LockAPI->>Nginx: POST /api/v1/auth/refresh (with refresh_token cookie)
        Nginx->>Controller: Rotate Refresh Token & Issue new JWT
        Controller-->>User: Set-Cookie: new JWT (HttpOnly, SameSite=Strict)
    end

    Note over User, Controller: 4. Step-up Authentication (Re-auth)
    alt Sensitive Action (Change Email / Terminate All Sessions)
        User->>Controller: POST with Current Password
        Controller->>Controller: Verify credentials before executing state modification
    end
```

## Security Controls Summary

| Security Control | Implementation Detail | ASVS Objective |
|---|---|---|
| **Token Storage** | `HttpOnly`, `SameSite=Strict`, `Secure` cookies | Complete mitigation of XSS token exfiltration |
| **CSRF Defense** | Custom header check (`X-Requested-With`) + Strict SameSite cookies | Defense-in-depth against cross-site forged requests |
| **MFA Verification** | 6-digit cryptographic OTP via email with strict time expiry | Credential compromise mitigation |
| **Token Leakage Defense** | Activation & reset tokens delivered via URL fragment (`#token=...`) | Eliminates leakage in Server Access Logs & `Referer` headers |
| **Race-Condition Defense**| `navigator.locks` Web Locks API for token rotation | Eliminates simultaneous refresh collisions across browser tabs |
| **Input Validation** | Strict `maxLength` bounds on all endpoints and DTOs | Buffer overrun & DoS mitigation |

