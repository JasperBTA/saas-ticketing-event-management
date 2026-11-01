# Core Domain Model (Entity Relationship Diagram)

The **SaaS Ticketing & Event Management platform** separates the concept of a **Physical Event**, a **Ticket Product**, and a **Visual Ticket Template (Skin)** into decoupled, composable entities.

## Entity-Relationship Diagram (ERD)

```mermaid
erDiagram
    EVENT {
        UUID id PK
        VARCHAR name
        TIMESTAMP date
        VARCHAR location
        BOOLEAN is_active
        INT used_tickets
        UUID event_profile_id FK "nullable"
        BOOLEAN enforce_exclusivity
        TIMESTAMP deleted_at "Soft Delete"
    }

    EVENT_PROFILE {
        UUID id PK
        VARCHAR name
        TIMESTAMP deleted_at "Soft Delete"
    }

    PRODUCT {
        UUID id PK
        VARCHAR name
        DECIMAL base_price
        DATE valid_since
        INT validity_days
        INT max_entries
        INT entries_per_scan
        BOOLEAN is_active
        BOOLEAN is_limited
        INT stock_cap
        UUID profile_id FK
        TIMESTAMP deleted_at "Soft Delete"
    }

    PRODUCT_PROFILE {
        UUID id PK
        VARCHAR name
        UUID skin_id FK
        TIMESTAMP deleted_at "Soft Delete"
    }

    SKIN {
        UUID id PK
        VARCHAR name
        VARCHAR background_image
        JSONB layout_config
        TIMESTAMP deleted_at "Soft Delete"
    }

    ADDON {
        UUID id PK
        VARCHAR name
        DECIMAL price
        UUID event_id FK "nullable"
        TIMESTAMP deleted_at "Soft Delete"
    }

    PRODUCT_ADDON {
        UUID product_id PK, FK
        UUID addon_id PK, FK
        BOOLEAN is_mandatory
    }

    EVENT_PROFILE_PRODUCT {
        UUID event_profile_id PK, FK
        UUID product_id PK, FK
    }

    ROLE {
        UUID id PK
        VARCHAR name
        VARCHAR code
        UUID permission_id FK
    }

    PERMISSION {
        UUID id PK
        BOOLEAN admin
        BOOLEAN can_scan_tickets
        BOOLEAN can_use_cash_register
        BOOLEAN can_modify_tickets
        BOOLEAN can_scan_addons
    }

    %% Relationships
    EVENT ||--o| EVENT_PROFILE : "assigned to"
    EVENT ||--o{ ADDON : "hosts direct addons"
    EVENT_PROFILE ||--|{ EVENT_PROFILE_PRODUCT : "contains"
    PRODUCT ||--|{ EVENT_PROFILE_PRODUCT : "grouped into"
    PRODUCT }o--|| PRODUCT_PROFILE : "governed by profile"
    PRODUCT_PROFILE ||--o| SKIN : "renders with"
    PRODUCT ||--o{ PRODUCT_ADDON : "attaches"
    ADDON ||--o{ PRODUCT_ADDON : "configured on"
    ROLE ||--|| PERMISSION : "defines"
```

## Architectural Design Highlights

1. **Decoupled Ticket Profiles (`EventProfile` & `ProductProfile`):**
   - Products are not hard-wired directly to single events. Instead, an `EventProfile` groups multiple ticket products that can be assigned across recurring event series.
   - A `ProductProfile` decouples the commercial configuration of a ticket from its graphical representation (`Skin`).
2. **Strict Financial Soft Delete:**
   - Entities involved in transactions (`Event`, `Product`, `Addon`, `Skin`) use `@SQLDelete` and `@SQLRestriction("deleted_at IS NULL")`.
   - Records with `salesCount > 0` or `usedTickets > 0` are protected by a business rule forbidding permanent purge to guarantee accounting integrity.
3. **Hibernate `@NotFound(action = IGNORE)` Pattern:**
   - Avoids `EntityNotFoundException` on join tables (`EVENT_PROFILE_PRODUCT`, `PRODUCT_ADDON`) when traversing historical records linked to soft-deleted items.

