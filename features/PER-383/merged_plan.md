# Merged plan — PER-383

Execution order (1 ticket(s)):

1. `1`

Specs (1): `specs/1.md`

---

## Spec `specs/1.md`

# specs/1.md — Add Product Wishlist Functionality

**Ticket:** PER-383  
**Status:** draft  
**Updated:** 2025-01-13

## Overview

Implement a wishlist feature for authenticated customers to save products they are interested in and view them later. The wishlist persists across sessions, allows adding and removing products, prevents duplicate entries, and displays product details (name, image, price) on a dedicated page.

## Stack & Technology

**Framework:** Medusa v2 (Node.js/TypeScript)  
**ORM:** MikroORM (existing in codebase at `packages/medusa` and modules)  
**Database:** PostgreSQL (existing; connection configured via `DB_URL` in `medusa-config.js`)  
**Validation:** Zod (existing pattern in `packages/medusa/src/api`)  
**Testing:** Jest + Supertest (existing in `integration-tests/http`)  
**Authentication:** Existing customer authentication middleware (`authenticate("customer", ["session", "bearer"])`) found in store routes (e.g., `packages/medusa/src/api/store/carts`)

## Schema & API Contracts

### Database Schema

**Table:** `wishlist`

```
wishlist
--------
id: uuid (PK)
customer_id: uuid (FK → customer.id, NOT NULL, ON DELETE CASCADE)
product_id: uuid (FK → product.id, NOT NULL, ON DELETE CASCADE)
created_at: timestamp (NOT NULL)
updated_at: timestamp (NOT NULL)

UNIQUE (customer_id, product_id)
INDEX ON customer_id
INDEX ON product_id
```

**Migration location:** `packages/medusa/src/migrations/<timestamp>-CreateWishlistTable.ts`  
**Model location:** `packages/medusa/src/models/wishlist.ts`

### API Endpoints

**Admin Routes** (prefix `/admin/customers/:customer_id/wishlist`)

- `GET /admin/customers/:customer_id/wishlist`  
  **Auth:** Admin authentication (existing middleware)  
  **Returns:** `{ wishlist_items: WishlistItemDTO[], count: number }`  
  **Description:** List all wishlist items for a specific customer (admin view).

- `DELETE /admin/customers/:customer_id/wishlist/:product_id`  
  **Auth:** Admin authentication  
  **Returns:** `{ deleted: true, id: string }`  
  **Description:** Remove a wishlist item (admin-initiated).

**Store Routes** (prefix `/store/wishlist`)

- `POST /store/wishlist`  
  **Auth:** Customer authentication required (`authenticate("customer", ["session", "bearer"])`)  
  **Body:** `{ product_id: string }`  
  **Returns:** `{ wishlist_item: WishlistItemDTO }`  
  **Description:** Add a product to the authenticated customer's wishlist. Idempotent: returns existing item if already wishlisted.

- `GET /store/wishlist`  
  **Auth:** Customer authentication required  
  **Returns:** `{ wishlist_items: WishlistItemDTO[], count: number }`  
  **Description:** List all wishlist items for the authenticated customer.

- `DELETE /store/wishlist/:product_id`  
  **Auth:** Customer authentication required  
  **Returns:** `{ deleted: true, id: string }`  
  **Description:** Remove a product from the authenticated customer's wishlist.

**Route file locations:**
- Admin: `packages/medusa/src/api/admin/customers/[customer_id]/wishlist/route.ts`
- Store POST/GET: `packages/medusa/src/api/store/wishlist/route.ts`
- Store DELETE: `packages/medusa/src/api/store/wishlist/[product_id]/route.ts`

### Response DTO

**WishlistItemDTO:**

```json
{
  "id": "uuid",
  "customer_id": "uuid",
  "product_id": "uuid",
  "product": {
    "id": "uuid",
    "title": "string",
    "thumbnail": "string | null",
    "variants": [
      {
        "id": "uuid",
        "calculated_price": {
          "amount": "number",
          "currency_code": "string"
        }
      }
    ]
  },
  "created_at": "ISO8601",
  "updated_at": "ISO8601"
}
```

**Empty state response (no items):**

```json
{
  "wishlist_items": [],
  "count": 0
}
```

The `product` relation is eagerly loaded via ORM relations or service layer join. Price calculation uses existing pricing module workflow (`calculatePricesWorkflow` or equivalent from `packages/core/core-flows`) respecting region, customer group, and currency context.

## File Touch List

**New files:**

- `packages/medusa/src/models/wishlist.ts` — MikroORM entity
- `packages/medusa/src/services/wishlist.ts` — DI-registered service
- `packages/medusa/src/migrations/<timestamp>-CreateWishlistTable.ts` — Migration
- `packages/medusa/src/api/admin/customers/[customer_id]/wishlist/route.ts` — Admin endpoints
- `packages/medusa/src/api/store/wishlist/route.ts` — Store POST/GET
- `packages/medusa/src/api/store/wishlist/[product_id]/route.ts` — Store DELETE
- `integration-tests/http/__tests__/wishlist/admin/wishlist.spec.ts` — Admin integration tests
- `integration-tests/http/__tests__/wishlist/store/wishlist.spec.ts` — Store integration tests

**Existing modules/services referenced (no modifications needed unless integration requires adjustment):**

- `packages/modules/customer` — Customer module service (`ICustomerModuleService`) injected to validate customer existence
- `packages/modules/product` — Product module service (`IProductModuleService`) injected to validate product existence
- `packages/core/core-flows` — Pricing workflows for price calculation in wishlist item responses
- `packages/medusa/src/api/middlewares.ts` or inline — Authentication middleware patterns

## Risks & Mitigations

**Risk 1: Product variant handling**  
*Uncertainty:* Should wishlist items reference a specific variant (`product_variant_id`) or just the product? The requirements say "add a product" and show "product name, image, and price." The latter requires a variant for pricing. If we store only `product_id`, the API defaults to the first/cheapest variant for display; if we store `variant_id`, the schema changes to `(customer_id, variant_id)` unique constraint.

*Mitigation:* Decision needed before implementation. Recommend storing `product_id` only (simpler, matches user mental model of "wishlisting a product") and resolving the default variant (first available or cheapest) in the service layer for price display. If per-variant wishlisting is desired later, it's a schema migration.

**Risk 2: Pricing context resolution**  
*Uncertainty:* Calculating prices for wishlist items requires customer region, sales channel, and customer group context. If the customer's region or groups change, displayed prices may become stale.

*Mitigation:* Calculate prices on-the-fly at retrieval time (not stored in `wishlist` table). Use existing `calculatePricesWorkflow` or pricing module with customer context. This ensures prices always reflect current state. Performance acceptable for typical wishlist sizes (< 100 items).

**Risk 3: Pagination**  
*Uncertainty:* The requirements don't specify pagination for the wishlist list endpoint. Should `GET /store/wishlist` support `limit`/`offset` query params (standard Medusa pattern)?

*Mitigation:* Assume yes and implement standard pagination. If wishlist is guaranteed small (< 100 items per customer), pagination is optional but good practice. Follow existing Medusa list endpoint patterns in `packages/medusa/src/api/store/products`.

**Risk 4: Soft delete vs. hard delete**  
*Uncertainty:* Should removed wishlist items be soft-deleted for analytics, or hard-deleted per `ON DELETE CASCADE`?

*Mitigation:* The requirements don't specify audit history. Use hard delete for simplicity (matches `ON DELETE CASCADE`). If analytics are required later, add a soft-delete flag and adjust the delete logic.

**Risk 5: Cross-device behavior**  
*Uncertainty:* AC 9 says "work on both desktop and mobile devices." Backend implementation is device-agnostic, but the requirements don't specify a frontend layer.

*Mitigation:* The backend API (this spec's scope) handles cross-device via authenticated sessions (existing token-based auth supports multi-device). No special backend handling required beyond responsive API design. Frontend (out of scope) will handle responsive UI.

**Risk 6: Integration test fixture dependencies**  
*Uncertainty:* Wishlist tests depend on existing customer, product, and region fixtures. If those fixtures change, wishlist tests may break.

*Mitigation:* Use existing fixture patterns from `integration-tests/http/__tests__/cart` and `integration-tests/http/__tests__/customer`. Create isolated wishlist fixtures where needed. Seed the test database with minimal required data (customer, product, region) in the `beforeEach` block.

---

# Story summaries

- `01-medusa-appli`: Story 01-medusa-appli: Database schema and wishlist service layer
- `02-medusa-appli`: Story 02-medusa-appli: Store and admin API endpoints with authentication
- `03-medusa-appli`: Story 03-medusa-appli: Product details enrichment and comprehensive test coverage

---

_Single-ticket pack: order is trivial (no multi-split)._
