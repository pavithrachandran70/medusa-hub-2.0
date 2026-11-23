# breakdown.md — Add Product Wishlist Functionality

**Ticket:** PER-383  
**Status:** draft  
**Updated:** 2025-01-13

## Stories

### Story 1: Database schema and wishlist service layer

**Repo:** `medusa-appli`

#### Context
Establish the foundational data model and business logic for wishlist persistence. This story creates the database table with foreign key constraints to customer and product, and implements a service layer for add/list/remove operations with validation and idempotency enforcement.

#### Scope
- `packages/medusa/src/models/wishlist.ts` (new) — MikroORM entity
- `packages/medusa/src/migrations/<timestamp>-CreateWishlistTable.ts` (new) — Migration
- `packages/medusa/src/services/wishlist.ts` (new) — DI-registered service with methods `addToWishlist`, `removeFromWishlist`, `listWishlistItems`

#### Implementation Pointers
- **Model (`packages/medusa/src/models/wishlist.ts`)**: Define `Wishlist` entity with `@Entity`, `@PrimaryKey` for `id` (uuid), `@ManyToOne` for `customer` (FK to `Customer` entity), `@ManyToOne` for `product` (FK to `Product` entity), `@Property` for `created_at` and `updated_at` timestamps. Add `@Unique({ properties: ['customer', 'product'] })` decorator to enforce unique constraint on the tuple.
- **Migration (`packages/medusa/src/migrations/<timestamp>-CreateWishlistTable.ts`)**: Use MikroORM's `Migration` class to create `wishlist` table with columns `id` (uuid PK), `customer_id` (uuid FK → `customer.id` ON DELETE CASCADE), `product_id` (uuid FK → `product.id` ON DELETE CASCADE), `created_at` (timestamp NOT NULL), `updated_at` (timestamp NOT NULL). Add unique constraint on `(customer_id, product_id)` and indexes on `customer_id` and `product_id`.
- **Service (`packages/medusa/src/services/wishlist.ts`)**: Inject `ICustomerModuleService` and `IProductModuleService` in constructor (follow DI patterns in `packages/modules/customer/services`). Implement:
  - `addToWishlist(customerId: string, productId: string): Promise<Wishlist>` — Validate customer and product exist by calling injected module services. Attempt to create wishlist entry; if unique constraint violation occurs (catch ORM error), query and return existing entry (idempotent behavior per AD-6).
  - `removeFromWishlist(customerId: string, productId: string): Promise<void>` — Delete the wishlist row matching both IDs. No error if row does not exist.
  - `listWishlistItems(customerId: string): Promise<Wishlist[]>` — Query all wishlist rows for the customer, eagerly loading the `product` relation (use ORM `populate` or equivalent to avoid N+1).

#### Acceptance Criteria
1. Customers can add a product to their wishlist (R1: validated via service method).
2. Adding the same product multiple times does not create duplicate entries (R1: unique constraint enforced, service returns existing entry on duplicate).
3. Wishlist data persists when the customer logs out and logs in again (R4: data stored in database table with foreign key to customer account).
4. Customers can remove products from their wishlist (R3: service method deletes row).
5. Only authenticated customers can manage their wishlist (enforced in Story 2 API layer; this story provides the validated service methods).

#### Dependencies
None — foundational data and logic layer.

#### Out of Scope
- API endpoints: Story 2 implements the HTTP routes that call this service.
- Authentication enforcement: Story 2 applies authentication middleware to API routes.
- UI components: Story 3 implements frontend integration.

#### Verification Approach
**3 tests** in `integration-tests/http/__tests__/wishlist/store/wishlist.spec.ts`:
1. **Service idempotency test**: Call `addToWishlist` twice with the same `customerId` and `productId`. Verify second call returns the same `id` without throwing an error, and database contains exactly one row for the tuple.
2. **Cascade delete test**: Create a wishlist entry, delete the product via product module service, verify wishlist row is automatically removed (foreign key `ON DELETE CASCADE`).
3. **List with eager load test**: Add two products to a customer's wishlist, call `listWishlistItems`, verify response includes both items with the `product` relation populated (product `title`, `thumbnail` present), no N+1 queries (check ORM query log or use a query counter).

---

### Story 2: Store and admin API endpoints with authentication

**Repo:** `medusa-appli`

#### Context
Expose wishlist operations via REST endpoints for storefront customers and admin users. Storefront endpoints require customer authentication and derive customer ID from the session; admin endpoints accept customer ID as a URL parameter and require admin authentication.

#### Scope
- `packages/medusa/src/api/store/wishlist/route.ts` (new) — Store POST (add), GET (list)
- `packages/medusa/src/api/store/wishlist/[product_id]/route.ts` (new) — Store DELETE (remove)
- `packages/medusa/src/api/admin/customers/[customer_id]/wishlist/route.ts` (new) — Admin GET (list), DELETE (remove)

#### Implementation Pointers
- **Store POST/GET (`packages/medusa/src/api/store/wishlist/route.ts`)**:
  - Apply `authenticate("customer", ["session", "bearer"])` middleware before handlers (see existing pattern in `packages/medusa/src/api/store/carts`).
  - **POST handler**: Extract `customer_id` from `req.auth.actor_id` (populated by auth middleware). Parse request body with Zod schema `{ product_id: z.string() }`. Inject `WishlistService` and call `addToWishlist(customer_id, product_id)`. Return `{ wishlist_item: <result> }` with 200 status.
  - **GET handler**: Extract `customer_id` from `req.auth.actor_id`. Inject `WishlistService` and call `listWishlistItems(customer_id)`. Return `{ wishlist_items: <result>, count: <result.length> }`. Empty array if no items (AD-7).
- **Store DELETE (`packages/medusa/src/api/store/wishlist/[product_id]/route.ts`)**:
  - Apply same authentication middleware.
  - **DELETE handler**: Extract `customer_id` from `req.auth.actor_id` and `product_id` from URL param. Inject `WishlistService` and call `removeFromWishlist(customer_id, product_id)`. Return `{ deleted: true, id: <product_id> }` with 200 status.
- **Admin GET/DELETE (`packages/medusa/src/api/admin/customers/[customer_id]/wishlist/route.ts`)**:
  - Apply admin authentication middleware (follow pattern in `packages/medusa/src/api/admin/customers`).
  - **GET handler**: Extract `customer_id` from URL param `req.params.customer_id`. Inject `WishlistService` and call `listWishlistItems(customer_id)`. Return `{ wishlist_items: <result>, count: <result.length> }`.
  - **DELETE handler**: Extract `customer_id` and `product_id` from URL params. Inject `WishlistService` and call `removeFromWishlist(customer_id, product_id)`. Return `{ deleted: true, id: <product_id> }`.

#### Acceptance Criteria
1. Customers can add a product to their wishlist using a wishlist icon or button (API: POST /store/wishlist accepts `product_id` in body).
2. Customers can view all their saved products on a dedicated Wishlist page (API: GET /store/wishlist returns all items).
3. Customers can remove products from their wishlist (API: DELETE /store/wishlist/:product_id removes item).
4. Adding the same product multiple times does not create duplicate entries (API: POST returns 200 with existing item on duplicate, enforced by Story 1 service).
5. Wishlist data persists when the customer logs out and logs in again (API: data tied to customer ID from authentication; re-authentication retrieves same data).
6. If the wishlist is empty, display an appropriate empty-state message (API: GET returns `{ wishlist_items: [], count: 0 }`; UI in Story 3 handles message).
7. Only authenticated customers can manage their wishlist (API: unauthenticated requests return 401 from auth middleware).

#### Dependencies
- **Story 1** — Requires `WishlistService` and `Wishlist` model to exist for injection and operation.

#### Out of Scope
- Product details in response (name, image, price): Story 3 adds product relation population and price calculation.
- UI components: Story 3 implements frontend integration.

#### Verification Approach
**6 tests** in `integration-tests/http/__tests__/wishlist/store/wishlist.spec.ts` and `integration-tests/http/__tests__/wishlist/admin/wishlist.spec.ts`:
1. **Authenticated add**: POST /store/wishlist with valid customer token and product ID. Verify 200 response with `wishlist_item.id` and `wishlist_item.product_id`.
2. **Unauthenticated add rejected**: POST /store/wishlist without token. Verify 401 response.
3. **List items**: POST two products, then GET /store/wishlist. Verify 200 response with `count: 2` and two items in `wishlist_items` array.
4. **Remove item**: POST product, DELETE /store/wishlist/:product_id, then GET. Verify item is gone from list.
5. **Admin list**: Admin GET /admin/customers/:customer_id/wishlist. Verify 200 response with correct items for that customer.
6. **Admin delete**: Admin DELETE /admin/customers/:customer_id/wishlist/:product_id. Verify item removed from customer's wishlist.

---

### Story 3: Product details enrichment and comprehensive test coverage

**Repo:** `medusa-appli`

#### Context
Enrich wishlist API responses with product details (name, image, price) by eagerly loading the product relation and calculating prices in the customer's context. Add end-to-end integration tests covering all acceptance criteria including authentication, idempotency, persistence, and empty state.

#### Scope
- `packages/medusa/src/services/wishlist.ts` — Modify `listWishlistItems` to populate product relation and calculate prices
- `packages/medusa/src/api/store/wishlist/route.ts` — Modify GET handler to return product details in response
- `integration-tests/http/__tests__/wishlist/store/wishlist.spec.ts` — Add full test suite
- `integration-tests/http/__tests__/wishlist/admin/wishlist.spec.ts` — Add admin test suite

#### Implementation Pointers
- **Service (`packages/medusa/src/services/wishlist.ts`)**:
  - In `listWishlistItems`, after querying wishlist rows with product relation populated, inject pricing module service or workflow (verify exact workflow in `packages/core/core-flows/src/product`; likely `calculatePricesWorkflow` or similar).
  - For each product, resolve pricing context from customer (default region, sales channel from publishable key if available, customer group memberships). Call pricing workflow with context and product variants.
  - Return wishlist items with `product` field containing `{ id, title, thumbnail, variants: [{ id, calculated_price: { amount, currency_code } }] }` (follow DTO structure in AD-5 and existing product response patterns in `packages/core/types`).
- **API route (`packages/medusa/src/api/store/wishlist/route.ts`)**:
  - GET handler now returns enriched `wishlist_items` from service with product details included.
- **Tests (`integration-tests/http/__tests__/wishlist/store/wishlist.spec.ts`)**:
  - **Persistence test**: Authenticate customer, add product, log out (clear session), log in again, GET /store/wishlist. Verify same product is present (AC 5).
  - **Empty state test**: Authenticate customer with no wishlist items, GET /store/wishlist. Verify `{ wishlist_items: [], count: 0 }` (AC 7).
  - **Product details test**: Add product, GET /store/wishlist. Verify response includes `product.title`, `product.thumbnail`, `product.variants[0].calculated_price.amount` and `currency_code` (AC 6).
  - **Idempotency test**: POST same product twice. Verify second response has same `wishlist_item.id` as first, and GET returns only one item (AC 4).
  - **Duplicate prevention at database level**: Attempt to create duplicate via service (bypass API). Verify constraint violation is caught and existing item returned (AC 4 enforcement).
  - **Authentication enforcement**: Test unauthenticated POST, GET, DELETE. Verify all return 401 (AC 8).

#### Acceptance Criteria
1. Customers can add a product to their wishlist using a wishlist icon or button (covered in Story 2 API tests; Story 3 verifies integration).
2. Customers can view all their saved products on a dedicated Wishlist page (covered in Story 2 API tests; Story 3 verifies product details).
3. Customers can remove products from their wishlist (covered in Story 2 API tests; Story 3 verifies integration).
4. Adding the same product multiple times does not create duplicate entries (Story 3 adds idempotency and duplicate prevention tests).
5. Wishlist data persists when the customer logs out and logs in again (Story 3 adds persistence test).
6. The wishlist page should display product name, image, and price (Story 3 adds product details enrichment and verification).
7. If the wishlist is empty, display an appropriate empty-state message (Story 3 adds empty state test).
8. Only authenticated customers can manage their wishlist (Story 3 adds comprehensive authentication tests).
9. Wishlist functionality should work on both desktop and mobile devices (backend is device-agnostic; API serves any client; no special backend handling required per AD-4 and R6).
10. Add appropriate API and integration tests for wishlist operations (Story 3 completes test coverage per AC 10).

#### Dependencies
- **Story 1** — Requires `WishlistService` and `Wishlist` model.
- **Story 2** — Requires API endpoints to exist for integration testing.

#### Out of Scope
- UI components: This ticket's scope is backend only. Frontend implementation (wishlist icon, page, state management) deferred to future work.
- Pagination: Assuming wishlist size < 100 items per customer per open question in HLD. If pagination needed, it's a future enhancement.

#### Verification Approach
**6 tests** (in addition to Story 2's 6 tests, total 12 integration tests):
1. **Persistence test**: Described in Implementation Pointers above.
2. **Empty state test**: Described above.
3. **Product details test**: Described above.
4. **Idempotency test**: Described above.
5. **Database-level duplicate prevention**: Described above.
6. **Authentication enforcement**: Described above (covers unauthenticated access to all endpoints).

---

## Cross-Story Dependencies

None beyond the linear Story 1 → Story 2 → Story 3 sequence.

---

## Open Questions

1. **Product variant handling (affects Story 1 and Story 3)**: The requirements say "add a product" but show "price" (requires a variant). Current plan: store only `product_id` in the wishlist table, and in Story 3's `listWishlistItems` resolve the first available or cheapest variant for price display. If per-variant wishlisting is desired later, it's a schema migration. **Decision needed** before Story 1 implementation to confirm this approach or switch to `product_variant_id` FK.

2. **Soft delete vs. hard delete (affects Story 1 migration)**: Currently using hard delete (`ON DELETE CASCADE`) per AD-1. If audit history is required, change Story 1's migration to add a `deleted_at` timestamp column and Story 1's service to soft-delete instead. **Clarify** before Story 1 if analytics on removed items are needed.

3. **Pagination (affects Story 3 response)**: The requirements don't specify pagination. Should `GET /store/wishlist` support `limit`/`offset` query params (standard Medusa pattern)? **Assume no** for initial implementation given expected list size < 100 items; defer to future work if needed.

---

## Risks

**Risk 1 (Story 1)**: Unique constraint enforcement relies on ORM error handling. If the ORM's constraint violation error signature changes, the idempotency logic breaks.  
*Mitigation*: Use a try-catch around the insert operation and check for the specific ORM unique constraint error code (verify in MikroORM docs). Add a unit test for this error path.

**Risk 2 (Story 3)**: Price calculation depends on existing pricing module workflow. If the workflow signature or context resolution changes, Story 3's implementation breaks.  
*Mitigation*: Reference existing product price calculation patterns in `packages/core/core-flows/src/product` and `packages/medusa/src/api/store/products`. Follow the same workflow and context structure. Add integration test that verifies price appears in response (guards against workflow API drift).

**Risk 3 (Story 2 and Story 3)**: Authentication middleware changes (e.g., new session handling) could break customer ID extraction.  
*Mitigation*: Follow existing store route patterns exactly (e.g., `packages/medusa/src/api/store/carts`). Reuse the same middleware and `req.auth.actor_id` extraction. Integration tests will catch auth flow changes.

**Risk 4 (Story 3)**: N+1 query problem when loading product relations for large wishlists.  
*Mitigation*: Story 1's service uses ORM's eager loading (`populate` or equivalent) to load products in a single query. Story 3's test suite includes a query counter check (see Story 1's Verification Approach test 3).
