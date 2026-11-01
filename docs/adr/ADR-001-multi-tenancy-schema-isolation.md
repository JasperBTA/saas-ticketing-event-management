# ADR-001: Schema-per-Tenant Multi-Tenancy Strategy

## Status
Accepted

## Context
The SaaS Ticketing & Event Management platform (codename: AnyPass) is a multi-tenant ticketing and event platform where multiple distinct organizations (tenants) manage events, confidential attendee records, ticket quotas, and financial transactions. 

When designing the multi-tenant architecture in PostgreSQL with Spring Boot and Hibernate, three primary patterns were evaluated:
1. **Shared Database, Shared Schema (Discriminator Column):** All tenants share tables with a `tenant_id` column.
2. **Shared Database, Separate Schemas (Schema-per-Tenant):** All tenants share a database instance, but each has a distinct PostgreSQL schema with identical table definitions.
3. **Database-per-Tenant:** Each tenant has an independent database instance.

## Decision
We chose **Option 2: Schema-per-Tenant**.

### Technical Implementation:
- A shared schema (`shared`) stores tenant registration metadata, master user accounts, and system configuration.
- Each tenant is assigned a dedicated PostgreSQL schema (e.g. `tenant_acme`, `tenant_apitest`).
- Dynamic database migrations are managed via **Flyway**: on application bootstrap, migrations run against `shared`, and dynamic discovery executes `V1__tenant_init.sql` across all active tenant schemas.
- In Spring Boot, the current tenant identifier is resolved from the verified JWT token (`tenant` claim) in `JwtAuthenticationFilter` and stored in a thread-safe `TenantContext`.
- Hibernate sets the schema search path per transaction (`SET search_path TO tenant_name`).

## Consequences

### Positive:
- **Zero Risk of Cross-Tenant Data Leaks:** A missing `WHERE tenant_id = ?` clause in a complex JPQL, native query, or join cannot leak data to another tenant; PostgreSQL enforces boundaries at the engine catalog level.
- **Cost Efficiency:** Hundreds of tenants can reside on a single managed PostgreSQL cluster without the overhead of multiple RDS/Cloud SQL instances.
- **Independent Tenant Backup & Archival:** Individual schemas can be dumped (`pg_dump -n tenant_x`) or restored without affecting other tenants.

### Negative / Trade-offs:
- Flyway migrations must be programmatically coordinated across all schemas on startup.
- Connection pooling requires careful schema-path resetting before returning connections to the HikariCP pool.

