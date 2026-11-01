# SaaS Ticketing & Event Management Platform

[![Java](https://img.shields.io/badge/Java-25-orange.svg)](https://openjdk.org/)
[![Spring Boot](https://img.shields.io/badge/Spring_Boot-4.0-brightgreen.svg)](https://spring.io/projects/spring-boot)
[![PostgreSQL](https://img.shields.io/badge/PostgreSQL-16-blue.svg)](https://www.postgresql.org/)
[![Security](https://img.shields.io/badge/OWASP-ASVS_Level_2-red.svg)](https://owasp.org/www-project-application-security-verification-standard/)
[![API Docs](https://img.shields.io/badge/API_Docs-Interactive_Swagger-blue.svg)](https://jasperbta.github.io/saas-ticketing-event-management/)

Architecture and system design specification for a multi-tenant ticketing and event management backend.

This repository documents the architectural patterns, database schemas, security lifecycle, and technical decisions implemented in the platform. The application source code is maintained in a private repository; this repository serves as a technical design reference.

---

## Technical Overview

The system is a multi-tenant backend responsible for event scheduling, quota-controlled ticket issuance, pricing profiles, and access control. 

### Core Tech Stack
- **Language / Runtime:** Java 25
- **Framework:** Spring Boot 4.0
- **Database:** PostgreSQL 16
- **Persistence & Migrations:** Spring Data JPA / Hibernate, Flyway
- **Security:** Spring Security, Stateless JWT, OWASP ASVS Level 2
- **Testing:** Postman / Newman CLI, JUnit 5, Mockito

---

## Architecture

### Multi-Tenant Schema Isolation

Data isolation is implemented at the database catalog level using PostgreSQL schemas (Schema-per-Tenant), avoiding shared-table row filtering via discriminator columns.

```mermaid
flowchart TD
    subgraph Ingestion["Ingestion Layer"]
        Req["HTTP Request"] --> JWT["JWT Cookie (tenant claim)"]
    end

    subgraph Backend["Spring Boot Backend"]
        AuthFilter["JwtAuthenticationFilter"]
        TenantContext["TenantContext (ThreadLocal)"]
        RoutingProvider["MultiTenantConnectionProvider"]

        JWT --> AuthFilter
        AuthFilter -->|"Set tenant context"| TenantContext
        TenantContext --> RoutingProvider
    end

    subgraph PostgreSQL["PostgreSQL 16"]
        Shared[("Schema: shared\nMaster accounts, tenant registry")]
        TenantA[("Schema: tenant_acme\nEvents, products, addons, skins")]
        TenantB[("Schema: tenant_festival\nEvents, products, addons, skins")]
    end

    RoutingProvider -->|"SET search_path TO tenant_acme"| TenantA
    RoutingProvider -->|"SET search_path TO tenant_festival"| TenantB
```

Details: [docs/architecture/multi-tenancy-architecture.md](docs/architecture/multi-tenancy-architecture.md)

---

### Request & Authentication Pipeline

Authentication and session control are aligned with OWASP ASVS Level 2 and Level 3 verification requirements.

```mermaid
sequenceDiagram
    autonumber
    actor Browser as Browser Client
    participant Nginx as Reverse Proxy
    participant SpringSec as Spring Security (Port 8080)
    participant LockAPI as Web Locks API (Client)

    Browser->>Nginx: Request with HttpOnly cookies + X-Requested-With
    Nginx->>Nginx: Apply security headers (CSP, HSTS, X-Frame-Options)
    Nginx->>SpringSec: Proxy to non-root backend service
    SpringSec->>SpringSec: Verify JWT signature & populate SecurityContext
    SpringSec-->>Browser: 200 OK (JSON payload)

    alt Token Expired (401 Response)
        Browser->>LockAPI: navigator.locks.request("anypass-refresh")
        Note over LockAPI: Serializes refresh across tabs to prevent rotation race conditions
        LockAPI->>SpringSec: POST /api/v1/auth/refresh
        SpringSec-->>Browser: Set-Cookie: Rotated Refresh Token & new JWT
    end
```

Details: [docs/architecture/security-pipeline.md](docs/architecture/security-pipeline.md)

---

### Domain Model

Events, ticket products, and visual render templates (skins) are decoupled through profile entities to support series scheduling and reusable commercial profiles.

```mermaid
erDiagram
    EVENT ||--o| EVENT_PROFILE : "assigned to"
    EVENT ||--o{ ADDON : "hosts direct addons"
    EVENT_PROFILE ||--|{ EVENT_PROFILE_PRODUCT : "contains"
    PRODUCT ||--|{ EVENT_PROFILE_PRODUCT : "grouped into"
    PRODUCT }o--|| PRODUCT_PROFILE : "governed by profile"
    PRODUCT_PROFILE ||--o| SKIN : "renders with"
    PRODUCT ||--o{ PRODUCT_ADDON : "attaches"
    ADDON ||--o{ PRODUCT_ADDON : "configured on"
```

Details: [docs/architecture/domain-model-erd.md](docs/architecture/domain-model-erd.md)

---

## Architectural Decisions

Specific engineering trade-offs are documented as Architectural Decision Records:

- **[ADR-001: Schema-per-Tenant Multi-Tenancy Strategy](docs/adr/ADR-001-multi-tenancy-schema-isolation.md)**  
  Comparison of discriminator column vs. schema isolation vs. separate database instances in PostgreSQL.

- **[ADR-002: Financial-Grade Soft Delete & Foreign Key Handling](docs/adr/ADR-002-financial-soft-delete-hibernate.md)**  
  Implementation of non-destructive soft delete for financial audit compliance and resolving Hibernate `@ManyToMany` join table hydration issues using `@NotFound(action = IGNORE)`.

- **[ADR-003: OWASP ASVS Compliance & Session Security Lifecycle](docs/adr/ADR-003-owasp-asvs-security-controls.md)**  
  Defense-in-depth against XSS token exfiltration, CSRF mitigations, cross-tab refresh race conditions, and fragment-based token transmission.

---

## API Specification

The REST API interface is fully documented using OpenAPI 3.0:
- **Interactive Documentation (Live Swagger UI):** [https://jasperbta.github.io/saas-ticketing-event-management/](https://jasperbta.github.io/saas-ticketing-event-management/)
- Raw specification file: [`docs/openapi/openapi.yaml`](docs/openapi/openapi.yaml)

---

## Testing & Verification

Integration and regression testing are automated against a running environment via the Newman CLI test runner:
- Multi-suite coverage across authentication, authorization, event scheduling, quota management, and soft-delete restrictions.
- Test runner script: `./test-api.sh` executing collections against isolated tenant schemas.

---

## Contact

**Kamil Oleksiak**  
Java / Backend Engineer  
Project walk-through and technical demonstrations available upon request.
