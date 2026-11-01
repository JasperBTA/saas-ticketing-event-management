# ADR-002: Financial-Grade Soft Delete & Hibernate Foreign Key Handling

## Status
Accepted

## Context
In ticketing and financial platforms, physical deletion (`DELETE FROM table WHERE id = ?`) of products, events, or add-ons that have generated transactions, receipts, or access scans causes immediate regulatory, tax, and auditing violations. 

However, applying standard Hibernate soft deletion (`@SQLDelete` + `@SQLRestriction("deleted_at IS NULL")`) introduces a critical runtime failure in ORMs:
When traversing `@ManyToMany` or `@ManyToOne` relationships (such as an `EventProfile` referencing historical products, or a `Product` referencing an add-on) where the child entity has `deleted_at IS NOT NULL`, Hibernate defaults to throwing `EntityNotFoundException` because the row exists in the join table but is filtered out by `@SQLRestriction`.

## Decision
1. **Business Enforcement:**
   - Entities (`Event`, `Product`) strictly forbid hard deletion once transactions exist (`usedTickets > 0` or `salesCount > 0`), throwing a domain `ForbiddenOperationException`.
   - Modifying state delegates to soft deletion via:
     ```java
     @SQLDelete(sql = "UPDATE product SET deleted_at = NOW() WHERE id = ?")
     @SQLRestriction("deleted_at IS NULL")
     ```
2. **ORM Join Table Edge-Case Resolution:**
   - To prevent `EntityNotFoundException` when querying join tables containing soft-deleted references, we decorated relationship definitions with Hibernate's `@NotFound(action = NotFoundAction.IGNORE)`:
     ```java
     @ManyToOne(fetch = FetchType.LAZY)
     @JoinColumn(name = "profile_id")
     @NotFound(action = NotFoundAction.IGNORE)
     private ProductProfile productProfile;
     ```
   - This treats soft-deleted relations as `null` or silently omits them from collections in-memory, providing a natural "Trash Bin / Archive" state without crashing entity hydration.

## Consequences

### Positive:
- **Financial & Audit Compliance:** Transaction history remains immutable in the database catalog.
- **Resilient Entity Hydration:** Aggregates and collections can be fetched without cascading `EntityNotFoundException` when legacy items are placed in the trash bin.

### Negative / Trade-offs:
- Hibernate forces `@ManyToOne` associations annotated with `@NotFound` to `FetchType.EAGER` to verify existence, requiring conscious indexing on foreign keys (`idx_product_profile_id`).

