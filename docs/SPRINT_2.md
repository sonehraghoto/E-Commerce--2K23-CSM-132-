Sprint 2 — Catalog Data Foundation
1. Sprint Goal & Boundary
Purpose
Sprint 2 is the first implementation sprint, transforming the FastAPI and PostgreSQL architecture defined in Sprint 1 into a robust, database-backed catalog foundation. The system must successfully persist categories, products, variants, and SKUs without losing identity, relationship, price, or inventory meaning. The outcome will provide the coherent data model, RESTful API administration, and automated verification necessary for future storefront consumption.
Scope Limits
 * In Scope: Category tree management with stable slugs, product creation and editing, variant and SKU records with unique codes and independent pricing, JWT-authenticated administrative routes, SQLAlchemy database constraints, Alembic migrations, seed data, and focused tests.
 * Out of Scope: Dynamic specifications, asset uploading, public catalog search, publication workflows, checkout flows, and payment integrations are deferred to Sprint 3 or later.
2. Functional Requirements
The following requirements represent the mandatory baseline for this sprint. All features must enforce required constraints before further additions.
| ID | Capability | Acceptance Requirement |
|---|---|---|
| CAT-01 | Categories | An administrator can create, update, deactivate, and list categories. A category has a unique slug, an optional parent, and cannot become its own ancestor. |
| CAT-02 | Product Identity | An administrator can create and edit a product with a name, slug, description, status, and category assignment. Slugs must be unique. |
| CAT-03 | Variants & SKUs | A product can have zero or more variants and one or more sellable SKUs. Each SKU has a unique code, price, and stock quantity. |
| CAT-04 | Variant Combinations | The system represents valid combinations only. Missing combinations are not created as fake or zero-stock SKUs. |
| CAT-05 | Data Integrity | PostgreSQL enforces required uniqueness and foreign-key constraints; Pydantic validation alone is not sufficient. |
| CAT-06 | Administrative Access | Administrative write operations strictly reject requests without a valid admin JWT. |
3. Required Data Model (ERD)
This schema extends the Sprint 1 PostgreSQL architecture. To fulfill Sprint 2 requirements, VARIANTS and SKUS are introduced. The purchasing identity in CART_ITEMS and ORDER_ITEMS is shifted from PRODUCTS to SKUS to lock in exact combinations. Field names follow the established Python snake_case conventions.
erDiagram
    CATEGORIES {
        serial category_id PK
        int parent_id FK
        varchar category_name
        varchar category_slug "UNIQUE"
        boolean is_active
        timestamp created_at
    }
    
    PRODUCTS {
        serial product_id PK
        int category_id FK
        varchar brand
        varchar model_name
        varchar product_slug "UNIQUE"
        text full_description
        varchar status
        jsonb specifications
    }
    
    VARIANTS {
        serial variant_id PK
        int product_id FK
        jsonb option_values
    }
    
    SKUS {
        serial sku_id PK
        int variant_id FK
        varchar sku_code "UNIQUE"
        int price "Integer minor units"
        int stock_quantity "CHECK >= 0"
        boolean is_active
    }
    
    ASSETS {
        serial asset_id PK
        int product_id FK
        varchar storage_key
        varchar role
        varchar alt_text
        int sort_order
    }

    CATEGORIES ||--o{ CATEGORIES : "parent of"
    CATEGORIES ||--o{ PRODUCTS: "contains"
    PRODUCTS ||--o{ VARIANTS: "has"
    VARIANTS ||--o{ SKUS: "materializes"
    PRODUCTS ||--o{ ASSETS: "displays"
    SKUS ||--o{ CART_ITEMS: "selected_as"
    SKUS ||--o{ ORDER_ITEMS: "sold_as"

Data Dictionary & Decisions
 * Price Format: Handled strictly as an integer (minor units/cents) to prevent floating-point calculation errors.
 * Stock Control: A PostgreSQL CHECK (stock_quantity >= 0) constraint ensures inventory cannot become negative during standard updates.
 * Specifications: Product-level attributes are maintained in the specifications JSONB column as defined in Sprint 1, complementing the specific variant options.
4. Minimum Administration Contract
All endpoints require a valid JWT with the admin role. The routes follow the FastAPI application structure.
| Method | Route | Purpose |
|---|---|---|
| POST | /api/v1/admin/categories | Create a category. |
| GET | /api/v1/admin/categories | Return the category tree. |
| POST | /api/v1/admin/products | Create a draft product. |
| GET | /api/v1/admin/products | Return administrative product records. |
| PATCH | /api/v1/admin/products/{id} | Update product content or status. |
| POST | /api/v1/admin/products/{id}/skus | Add a validated SKU to a product. |
| PATCH | /api/v1/admin/skus/{id} | Update price, stock, or active status. |
5. Business Rules & Edge Cases
The backend logic and database constraints enforce the following rules:
 * Draft vs. Published: A draft product is permitted to have zero SKUs. A published product must have at least one sellable SKU attached.
 * Taxonomy Assignment: A product belongs to exactly one canonical category to streamline querying for the electronics catalog.
 * Category Deactivation: If a parent category is deactivated, API reads dynamically hide its descendants and associated products.
 * Stock Handling: An out-of-stock SKU remains in the system and is returned in public responses, explicitly flagged as unavailable with a stock count of zero.
 * Independent Pricing: Every SKU maintains its own distinct price in the database. Price overrides are executed by directly updating the target SKU record.
 * Constraint Protection: Database-level uniqueness constraints block duplicate SKU codes and duplicate product slugs.
 * Order History Integrity: Products and SKUs referenced by previous orders (snapshot_price in ORDER_ITEMS) are never hard-deleted; their status is updated to inactive to preserve financial and historical data.
6. Seed Data & Testing
Reproducible Demonstration
The repository contains an Alembic revision and a Python seeding script (seed.py) that populates:
 * At least two levels in the category tree (e.g., Computers -> Laptops).
 * Three products, including one with multiple variants (e.g., RAM size configurations).
 * Four valid SKUs and one intentionally unavailable SKU combination.
Automated Testing
 * Test Command: pytest
 * Coverage: Tests validate both successful operations (via FastAPI TestClient) and required rejection paths, including duplicate slugs, duplicate SKU codes, negative stock, and category cycle prevention. Authentication middleware is verified to ensure 401/403 responses for unauthorized administrative attempts.
7. Sprint 3 Hand-off
This sprint completes the administrative catalog foundation. Sprint 3 will consume these specific SKU identities in the Next.js frontend rather than duplicating product or pricing logic. The immediate backlog for Sprint 3 begins with dynamic specifications, asset uploads, public catalog read endpoints, and catalog-to-cart API readiness.
