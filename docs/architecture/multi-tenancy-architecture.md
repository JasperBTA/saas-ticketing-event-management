# Multi-Tenant Architecture (Schema-per-Tenant)

The **SaaS Ticketing & Event Management platform** *(codename: AnyPass)* utilizes a **Schema-per-Tenant** multi-tenancy model on top of PostgreSQL. This approach guarantees strong physical data isolation between organizers without the financial overhead of provisioning separate database instances.

## Architectural Overview

```mermaid
flowchart TD
    subgraph ClientLayer["Client & Ingestion"]
        Req["HTTP Request"] --> AuthCookie["JWT Cookie (with tenant claim)"]
    end

    subgraph SpringBoot["Spring Boot Backend"]
        AuthFilter["JwtAuthenticationFilter"]
        TenantContext["TenantContext (ThreadLocal)"]
        FlywayEngine["Flyway Dynamic Migration Engine"]
        HibernateRouting["Hibernate Multi-Tenant Connection Provider"]

        AuthCookie --> AuthFilter
        AuthFilter -->|"Extract tenant (e.g. tenant_apitest)"| TenantContext
        TenantContext --> HibernateRouting
    end

    subgraph PostgreSQL["PostgreSQL Database Engine"]
        subgraph SharedSchema["Schema: shared (Master Metadata)"]
            TenantRegistry[("tenant_registry")]
            SystemUsers[("system_users")]
            SharedFlyway[("flyway_schema_history")]
        end

        subgraph TenantA["Schema: tenant_acme (Tenant A)"]
            TablesA[("event, product, addon, skin, role")]
            FlywayA[("flyway_schema_history")]
        end

        subgraph TenantB["Schema: tenant_festival (Tenant B)"]
            TablesB[("event, product, addon, skin, role")]
            FlywayB[("flyway_schema_history")]
        end
    end

    HibernateRouting -->|"SET search_path TO tenant_acme"| TenantA
    HibernateRouting -->|"SET search_path TO tenant_festival"| TenantB
    FlywayEngine -->|"Migrate core"| SharedSchema
    FlywayEngine -->|"Migrate dynamically"| TenantA
    FlywayEngine -->|"Migrate dynamically"| TenantB

    classDef spring fill:#2e7d32,stroke:#1b5e20,color:#fff;
    classDef db fill:#1565c0,stroke:#0d47a1,color:#fff;
    classDef client fill:#37474f,stroke:#263238,color:#fff;

    class AuthFilter,TenantContext,FlywayEngine,HibernateRouting spring;
    class TenantRegistry,SystemUsers,SharedFlyway,TablesA,FlywayA,TablesB,FlywayB db;
    class Req,AuthCookie client;
```

## Key Engineering Highlights

1. **Physical Isolation vs Discriminator Column:**
   - Instead of appending `WHERE tenant_id = ?` to every SQL query (which is prone to catastrophic developer oversights and data leaks), PostgreSQL schemas isolate tables at the database catalog level.
2. **Automated Dynamic Schema Migrations:**
   - On application startup and upon provisioning a new organizer, Flyway executes tenant migrations dynamically across all registered schemas (`V1__tenant_init.sql`).
3. **No Cross-Tenant Data Bleed:**
   - Connection pools dynamically set the schema search path for the duration of the current transaction and reset it upon connection return to HikariCP pool.

