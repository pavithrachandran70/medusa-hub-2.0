# hld.md — Add Product Wishlist Functionality

**Ticket:** PER-383  
**Status:** draft  
**Updated:** 2025-01-13

## Overview

Add wishlist functionality to the medusa-appli e-commerce platform, allowing authenticated customers to save products for later viewing across sessions. This feature extends the existing customer and product modules with a new data relationship and provides both admin and storefront API endpoints for wishlist management.

## Architecture Decisions

### AD-1: Database Schema — Wishlist Table

**Binds:** Data model for wishlist items  
**Prevents:** Duplicate entries, orphaned records, inconsistent foreign key relationships  
**Rule:** Store wishlist items in a dedicated `wishlist` table with composite unique constraint on `(customer_id, product_id)`. Use foreign key references to `customer` and `product` tables with `ON DELETE CASCADE` to maintain referential integrity. Include timestamps (`created_at`, `updated_at`) for audit and ordering purposes.

```
wishlist
--------
id: uuid (PK)
customer_id: uuid (FK → customer.id, NOT NULL)
product_id: uuid (FK → product.id, NOT NULL)
created_at: timestamp (NOT NULL)
updated_at: timestamp (NOT NULL)

UNIQUE (customer_id, product_id)
INDEX ON customer_id
INDEX ON product_id
```

**[ADOPTED]** — Medusa uses PostgreSQL with MikroORM; this schema follows existing patterns in the customer and product modules (verify exact ORM decorators and migration patterns in `packages/modules/customer` and `packages/modules/product`).

---

### AD-2: API Layer — RESTful Endpoints

**Binds:** HTTP contract for wishlist operations  
**Prevents:** Inconsistent endpoint design, unauthorized access, missing authentication checks  
**Rule:**

**Admin endpoints** (prefix `/admin/customers/:customer_id/wishlist`):
- `GET /admin/customers/:customer_id/wishlist` — List all wishlist items for a customer (admin view)
- `DELETE /admin/customers/:customer_id/wishlist/:product_id` — Remove an item (admin-initiated)

**Store endpoints** (prefix `/store/wishlist`):
- `POST /store/wishlist` — Add product to authenticated customer's wishlist (body: `{ product_id }`)
- `GET /store/wishlist` — List authenticated customer's wishlist items
- `DELETE /store/wishlist/:product_id` — Remove product from authenticated customer's wishlist

All store endpoints require authentication middleware (`authenticate("customer", ["session", "bearer"])` per existing cart/order patterns in `packages/medusa/src/api/store`). Admin endpoints require admin authentication. All endpoints return JSON with structure matching existing Medusa API conventions (e.g., `{ wishlist_items: [...] }` for lists).

**[ADOPTED]** — Medusa API routes live under `packages/medusa/src/api/{admin,store}/<resource>/route.ts` and use Zod schemas for validation (see `integration-tests/http/src/api/middlewares.ts` for middleware patterns).

---

### AD-3: Module Integration — Wishlist Service

**Binds:** Business logic layer for wishlist operations  
**Prevents:** Direct database access in routes, missing business rule enforcement, inconsistent error handling  
**Rule:** Implement wishlist operations as methods in a new `WishlistService` registered in the DI container. The service:
- Validates customer and product existence before creating entries
- Enforces the unique constraint (idempotent adds: if already exists, return success without error)
- Handles cascade deletes automatically via foreign key constraints
- Returns standardized DTOs matching Medusa's response format

Service methods:
- `addToWishlist(customerId: string, productId: string): Promise<WishlistItem>`
- `removeFromWishlist(customerId: string, productId: string): Promise<void>`
- `listWishlistItems(customerId: string): Promise<WishlistItem[]>`

Inject `ICustomerModuleService` and `IProductModuleService` to validate referenced entities exist.

**[ADOPTED]** — Medusa services follow DI patterns in `packages/core/framework` (see `packages/modules/customer/services` for reference).

---

### AD-4: Authentication Enforcement

**Binds:** Security model for wishlist access  
**Prevents:** Unauthorized wishlist access, cross-customer data leaks  
**Rule:** Store endpoints extract `customer_id` from the authenticated session/token (via `req.auth.actor_id` after middleware). Never accept `customer_id` from request body or params for store endpoints. Admin endpoints accept `customer_id` as a URL parameter and validate admin permissions via existing RBAC middleware.

**[ADOPTED]** — Follows existing authentication patterns in cart and order store routes (see `integration-tests/http/__tests__/cart/store/cart.spec.ts` for auth flow).

---

### AD-5: Response Format — Wishlist Item DTO

**Binds:** API response shape  
**Prevents:** Inconsistent client contracts, missing required fields  
**Rule:** Wishlist item responses include:
```json
{
  "id": "uuid",
  "customer_id": "uuid",
  "product_id": "uuid",
  "product": {
    "id": "uuid",
    "title": "string",
    "thumbnail": "string | null",
    "variants": [{ "id": "uuid", "calculated_price": { "amount": number, "currency_code": "string" } }]
  },
  "created_at": "ISO8601",
  "updated_at": "ISO8601"
}
```
The `product` relation is eagerly loaded (via ORM relations or service layer join) to avoid N+1 queries. Price calculation uses the existing pricing module workflow (`calculatePricesWorkflow` or equivalent) to respect region, customer group, and currency context.

**[ADOPTED]** — Matches existing Medusa DTO patterns (see product and cart response structures in `packages/core/types`).

---

### AD-6: Idempotent Add Operation

**Binds:** Behavior when adding duplicate items  
**Prevents:** Client-side error handling complexity, database constraint violations  
**Rule:** `POST /store/wishlist` with an already-wishlisted product returns `200 OK` with the existing wishlist item, not `409 Conflict`. The database unique constraint prevents the duplicate insert; the service catches the constraint violation and returns the existing record.

**[ADOPTED]** — Follows REST idempotency best practices and simplifies client logic (AC 4).

---

### AD-7: Empty State Handling

**Binds:** Response format for empty wishlists  
**Prevents:** Inconsistent empty-state contracts  
**Rule:** `GET /store/wishlist` with no items returns `{ wishlist_items: [], count: 0 }`, not a `404` or error. The UI layer handles empty-state messaging per AC 7; the API provides the data layer contract only.

**[ADOPTED]** — Standard Medusa list endpoint pattern (see existing list endpoints in `packages/medusa/src/api/store`).

---

## Seed (Structural Decisions)

### Database Migration
- Migration file: `packages/medusa/src/migrations/<timestamp>-CreateWishlistTable.ts`
- Follows MikroORM migration patterns in existing modules (see `packages/modules/customer/migrations`).

### File Tree
- **Model:** `packages/medusa/src/models/wishlist.ts` (MikroORM entity)
- **Service:** `packages/medusa/src/services/wishlist.ts` (DI-registered service)
- **Admin route:** `packages/medusa/src/api/admin/customers/[customer_id]/wishlist/route.ts`
- **Store routes:** 
  - `packages/medusa/src/api/store/wishlist/route.ts` (POST, GET)
  - `packages/medusa/src/api/store/wishlist/[product_id]/route.ts` (DELETE)
- **Validation schemas:** Inline Zod schemas in route files (per Medusa convention)
- **Tests:** 
  - `integration-tests/http/__tests__/wishlist/admin/wishlist.spec.ts`
  - `integration-tests/http/__tests__/wishlist/store/wishlist.spec.ts`

### Technology Stack
- **Framework:** Medusa v2 (Node.js/TypeScript)
- **ORM:** MikroORM (existing in codebase)
- **Database:** PostgreSQL (existing)
- **Validation:** Zod (existing)
- **Testing:** Jest + Supertest (existing integration test patterns)

### API Middleware
- **Store auth:** `authenticate("customer", ["session", "bearer"])` (existing in `packages/core/framework`)
- **Admin auth:** Standard admin authentication middleware (existing)
- **RBAC (optional):** If `Modules.RBAC` is installed, admin endpoints check `customers:write` permission; store endpoints implicitly require customer session (no additional permission checks per existing store patterns).

### Pricing Context
When returning product details in wishlist items, the API calculates prices using:
- Customer's default region (if set)
- Sales channel from the publishable key (store context)
- Customer group memberships (if applicable)

This leverages existing `calculatePricesWorkflow` or the pricing module's context resolution (verify exact workflow in `packages/core/core-flows/src/product`).

### Cross-Device / Mobile
No special backend handling required beyond responsive API design. The database and authentication layer already support multi-device access via session tokens; the frontend (not in scope) handles responsive UI per AC 9.

## Open Questions

1. **Product variant handling:** Should wishlist items reference a specific variant (`product_variant_id`) or just the product? The requirements say "add a product" and show "product name, image, and price"—the latter requires a variant for pricing. If we store only `product_id`, the API defaults to the first/cheapest variant for display; if we store `variant_id`, the schema changes to `(customer_id, variant_id)` unique constraint. **Decision needed** before implementation (affects AD-1 and AD-5).

2. **Soft delete vs. hard delete:** Should removed wishlist items be soft-deleted for analytics, or hard-deleted per `ON DELETE CASCADE`? The requirements don't specify; AD-1 currently assumes hard delete for simplicity. **Clarify** if audit history is required.

3. **Pagination:** The requirements don't specify pagination for the wishlist list endpoint. Should `GET /store/wishlist` support `limit`/`offset` query params (standard Medusa pattern)? **Assume yes** unless list is guaranteed small (< 100 items per customer).

## Deferred

- **Wishlist sharing:** Not in requirements; defer to future work.
- **Wishlist notifications:** No requirement for "back in stock" or price drop alerts; defer.
- **Bulk operations:** No requirement for batch add/remove; single-item API suffices.
- **Wishlist analytics:** No reporting or tracking beyond basic access logs; defer.
- **Product availability check:** Requirements don't specify whether out-of-stock products can be wishlisted; assume yes (business logic layer can filter if needed).
