# Story 02-medusa-appli: Store and admin API endpoints with authentication

**Repo:** `medusa-appli`

## Context

Expose wishlist operations via REST endpoints for storefront customers and admin users. Storefront endpoints require customer authentication and derive customer ID from the session; admin endpoints accept customer ID as a URL parameter and require admin authentication.

## Acceptance Criteria

1. Customers can add a product to their wishlist using a wishlist icon or button (API: POST /store/wishlist accepts `product_id` in body).
2. Customers can view all their saved products on a dedicated Wishlist page (API: GET /store/wishlist returns all items).
3. Customers can remove products from their wishlist (API: DELETE /store/wishlist/:product_id removes item).
4. Adding the same product multiple times does not create duplicate entries (API: POST returns 200 with existing item on duplicate, enforced by Story 1 service).
5. Wishlist data persists when the customer logs out and logs in again (API: data tied to customer ID from authentication; re-authentication retrieves same data).
6. If the wishlist is empty, display an appropriate empty-state message (API: GET returns `{ wishlist_items: [], count: 0 }`; UI in Story 3 handles message).
7. Only authenticated customers can manage their wishlist (API: unauthenticated requests return 401 from auth middleware).

## File Plan

### packages/medusa/src/api/store/wishlist/route.ts

**New file** — Store POST (add to wishlist) and GET (list wishlist items) endpoints.

Implements the storefront wishlist operations per AD-2 (API Layer — RESTful Endpoints) from the HLD. Both handlers require customer authentication and extract the customer ID from the authenticated session.

#### Imports

```typescript
import { MedusaRequest, MedusaResponse } from "@medusajs/framework/http"
import { authenticate } from "@medusajs/framework/http"
import { z } from "zod"
```

Verify exact import paths in existing store routes (e.g., `packages/medusa/src/api/store/carts/route.ts`).

#### Request Schemas

```typescript
export const StoreAddWishlistItemSchema = z.object({
  product_id: z.string(),
})

export const StoreGetWishlistSchema = z.object({})
```

Zod schemas follow existing validation patterns in Medusa v2 (see cart routes for reference).

#### Function: POST

```typescript
export const POST = [
  authenticate("customer", ["session", "bearer"]),
  async (req: MedusaRequest, res: MedusaResponse) => {
    const customerId = req.auth.actor_id
    const { product_id } = req.validatedBody as z.infer<typeof StoreAddWishlistItemSchema>

    const wishlistService = req.scope.resolve(WISHLIST_SERVICE)
    
    const wishlistItem = await wishlistService.addToWishlist(
      customerId,
      product_id
    )

    res.status(200).json({ wishlist_item: wishlistItem })
  },
]
```

**Key details:**
- Authentication middleware `authenticate("customer", ["session", "bearer"])` applied per AD-4 (Authentication Enforcement). This middleware populates `req.auth.actor_id` with the authenticated customer's ID.
- Extract `customer_id` from `req.auth.actor_id` (never from request body).
- Parse request body with Zod schema via `req.validatedBody` (body parsing middleware applied by framework).
- Inject `WishlistService` using `req.scope.resolve(WISHLIST_SERVICE)` where `WISHLIST_SERVICE` is the registration name exported from Story 1's service.
- Call service method `addToWishlist(customerId, product_id)` (implemented in Story 1).
- Return `{ wishlist_item: <result> }` with 200 status per AD-2.
- Idempotency per AD-6: if product already in wishlist, service returns existing entry without error.

**Verify:** Authentication middleware pattern in `packages/medusa/src/api/store/carts/route.ts`. Service resolution pattern in existing store routes.

#### Function: GET

```typescript
export const GET = [
  authenticate("customer", ["session", "bearer"]),
  async (req: MedusaRequest, res: MedusaResponse) => {
    const customerId = req.auth.actor_id

    const wishlistService = req.scope.resolve(WISHLIST_SERVICE)
    
    const wishlistItems = await wishlistService.listWishlistItems(customerId)

    res.status(200).json({
      wishlist_items: wishlistItems,
      count: wishlistItems.length,
    })
  },
]
```

**Key details:**
- Same authentication middleware as POST.
- Extract `customerId` from `req.auth.actor_id`.
- Call service method `listWishlistItems(customerId)` (implemented in Story 1).
- Return `{ wishlist_items: <result>, count: <result.length> }` per AD-2.
- Empty state per AD-7: if no items, service returns empty array `[]`, response becomes `{ wishlist_items: [], count: 0 }`.

---

### packages/medusa/src/api/store/wishlist/[product_id]/route.ts

**New file** — Store DELETE (remove from wishlist) endpoint.

Implements the storefront wishlist removal operation per AD-2 (API Layer — RESTful Endpoints) from the HLD. Requires customer authentication and extracts both customer ID from session and product ID from URL parameter.

#### Imports

```typescript
import { MedusaRequest, MedusaResponse } from "@medusajs/framework/http"
import { authenticate } from "@medusajs/framework/http"
```

#### Function: DELETE

```typescript
export const DELETE = [
  authenticate("customer", ["session", "bearer"]),
  async (req: MedusaRequest, res: MedusaResponse) => {
    const customerId = req.auth.actor_id
    const { product_id } = req.params

    const wishlistService = req.scope.resolve(WISHLIST_SERVICE)
    
    await wishlistService.removeFromWishlist(customerId, product_id)

    res.status(200).json({ deleted: true, id: product_id })
  },
]
```

**Key details:**
- Same authentication middleware pattern as POST/GET in `route.ts`.
- Extract `customerId` from `req.auth.actor_id`.
- Extract `product_id` from URL param `req.params.product_id` (Medusa framework populates `req.params` from dynamic route segments `[product_id]`).
- Call service method `removeFromWishlist(customerId, product_id)` (implemented in Story 1).
- Return `{ deleted: true, id: <product_id> }` with 200 status per AD-2.
- Idempotent: service does not throw error if wishlist item does not exist (per Story 1 implementation).

**Verify:** Dynamic route parameter pattern in existing Medusa routes (e.g., `packages/medusa/src/api/store/carts/[id]/route.ts`).

---

### packages/medusa/src/api/admin/customers/[customer_id]/wishlist/route.ts

**New file** — Admin GET (list) and DELETE (remove) endpoints for customer wishlists.

Implements the admin wishlist operations per AD-2 (API Layer — RESTful Endpoints) from the HLD. Admin endpoints require admin authentication and accept customer ID as a URL parameter.

#### Imports

```typescript
import { MedusaRequest, MedusaResponse } from "@medusajs/framework/http"
import { authenticate } from "@medusajs/framework/http"
```

#### Function: GET

```typescript
export const GET = [
  authenticate("admin", ["session", "bearer"]),
  async (req: MedusaRequest, res: MedusaResponse) => {
    const { customer_id } = req.params

    const wishlistService = req.scope.resolve(WISHLIST_SERVICE)
    
    const wishlistItems = await wishlistService.listWishlistItems(customer_id)

    res.status(200).json({
      wishlist_items: wishlistItems,
      count: wishlistItems.length,
    })
  },
]
```

**Key details:**
- Apply admin authentication middleware `authenticate("admin", ["session", "bearer"])` per AD-4. This follows existing admin route patterns (verify in `packages/medusa/src/api/admin/customers/route.ts`).
- Extract `customer_id` from URL param `req.params.customer_id` (not from authenticated session).
- Call same service method as store GET: `listWishlistItems(customer_id)`.
- Return same response format as store GET: `{ wishlist_items: <result>, count: <result.length> }`.

#### Function: DELETE

```typescript
export const DELETE = [
  authenticate("admin", ["session", "bearer"]),
  async (req: MedusaRequest, res: MedusaResponse) => {
    const { customer_id, product_id } = req.params

    const wishlistService = req.scope.resolve(WISHLIST_SERVICE)
    
    await wishlistService.removeFromWishlist(customer_id, product_id)

    res.status(200).json({ deleted: true, id: product_id })
  },
]
```

**Key details:**
- Same admin authentication middleware as GET.
- Extract both `customer_id` and `product_id` from URL params `req.params` (Medusa framework populates both from nested route structure).
- Call same service method as store DELETE: `removeFromWishlist(customer_id, product_id)`.
- Return `{ deleted: true, id: <product_id> }` with 200 status.

**Verify:** Nested admin route parameter pattern in existing admin routes (e.g., admin customer address routes if they exist). The route structure is `/admin/customers/[customer_id]/wishlist` for GET and `/admin/customers/[customer_id]/wishlist/[product_id]` for DELETE (the latter may need to be a separate file `[product_id]/route.ts` under the `wishlist` directory—check Medusa routing conventions).

**Open question:** Medusa v2 routing conventions for nested dynamic segments. If `/admin/customers/[customer_id]/wishlist/[product_id]` requires a separate file, adjust to:
- GET at `packages/medusa/src/api/admin/customers/[customer_id]/wishlist/route.ts`
- DELETE at `packages/medusa/src/api/admin/customers/[customer_id]/wishlist/[product_id]/route.ts`

Verify by inspecting existing nested admin routes or Medusa routing documentation.

---

## Tests

### integration-tests/http/__tests__/wishlist/store/wishlist.spec.ts

**Complete test file** — Integration tests for Story 2 store API endpoints.

```typescript
import { medusaIntegrationTestRunner } from "@medusajs/test-utils"
import {
  createAdminUser,
  generatePublishableKey,
  generateStoreHeaders,
} from "../../../../helpers/create-admin-user"
import { createAuthenticatedCustomer } from "../../../../modules/helpers/create-authenticated-customer"

const env = {}
const adminHeaders = { headers: { "x-medusa-access-token": "test_token" } }

medusaIntegrationTestRunner({
  env,
  testSuite: ({ dbConnection, getContainer, api }) => {
    let appContainer
    let storeHeaders
    let storeHeadersWithCustomer
    let customer
    let product

    beforeAll(async () => {
      appContainer = getContainer()
      await createAdminUser(dbConnection, adminHeaders, appContainer)
      const publishableKey = await generatePublishableKey(appContainer)
      storeHeaders = generateStoreHeaders({ publishableKey })

      const result = await createAuthenticatedCustomer(api, storeHeaders, {
        first_name: "Test",
        last_name: "User",
        email: "test@wishlist.com",
      })

      customer = result.customer
      storeHeadersWithCustomer = {
        headers: {
          ...storeHeaders.headers,
          authorization: `Bearer ${result.jwt}`,
        },
      }

      // Create a test product
      const shippingProfile = (
        await api.post(
          `/admin/shipping-profiles`,
          { name: "default", type: "default" },
          adminHeaders
        )
      ).data.shipping_profile

      product = (
        await api.post(
          "/admin/products",
          {
            title: "Test Wishlist Product",
            status: "published",
            shipping_profile_id: shippingProfile.id,
            options: [{ title: "Size", values: ["S"] }],
            variants: [
              {
                title: "S",
                sku: "TEST-WISHLIST-S",
                options: { Size: "S" },
                manage_inventory: false,
                prices: [{ amount: 1000, currency_code: "usd" }],
              },
            ],
          },
          adminHeaders
        )
      ).data.product
    })

    describe("POST /store/wishlist (Story 2)", () => {
      it("should add a product to authenticated customer's wishlist (AC 1)", async () => {
        const response = await api.post(
          `/store/wishlist`,
          { product_id: product.id },
          storeHeadersWithCustomer
        )

        expect(response.status).toEqual(200)
        expect(response.data.wishlist_item).toEqual(
          expect.objectContaining({
            id: expect.any(String),
            customer_id: customer.id,
            product_id: product.id,
            created_at: expect.any(String),
            updated_at: expect.any(String),
          })
        )
      })

      it("should reject unauthenticated add request (AC 7)", async () => {
        const response = await api
          .post(
            `/store/wishlist`,
            { product_id: product.id },
            storeHeaders // No customer token
          )
          .catch((e) => e)

        expect(response.response.status).toEqual(401)
      })
    })

    describe("GET /store/wishlist (Story 2)", () => {
      beforeEach(async () => {
        // Add two products to wishlist for list tests
        await api.post(
          `/store/wishlist`,
          { product_id: product.id },
          storeHeadersWithCustomer
        )

        const secondProduct = (
          await api.post(
            "/admin/products",
            {
              title: "Second Test Product",
              status: "published",
              shipping_profile_id: product.variants[0].shipping_profile_id,
              options: [{ title: "Size", values: ["M"] }],
              variants: [
                {
                  title: "M",
                  sku: "TEST-WISHLIST-M",
                  options: { Size: "M" },
                  manage_inventory: false,
                  prices: [{ amount: 1500, currency_code: "usd" }],
                },
              ],
            },
            adminHeaders
          )
        ).data.product

        await api.post(
          `/store/wishlist`,
          { product_id: secondProduct.id },
          storeHeadersWithCustomer
        )
      })

      it("should list all wishlist items for authenticated customer (AC 2)", async () => {
        const response = await api.get(
          `/store/wishlist`,
          storeHeadersWithCustomer
        )

        expect(response.status).toEqual(200)
        expect(response.data.count).toEqual(2)
        expect(response.data.wishlist_items).toHaveLength(2)
        expect(response.data.wishlist_items[0]).toEqual(
          expect.objectContaining({
            id: expect.any(String),
            customer_id: customer.id,
            product_id: expect.any(String),
          })
        )
      })
    })

    describe("DELETE /store/wishlist/:product_id (Story 2)", () => {
      let addedProduct

      beforeEach(async () => {
        const response = await api.post(
          `/store/wishlist`,
          { product_id: product.id },
          storeHeadersWithCustomer
        )
        addedProduct = response.data.wishlist_item
      })

      it("should remove product from wishlist (AC 3)", async () => {
        const deleteResponse = await api.delete(
          `/store/wishlist/${product.id}`,
          storeHeadersWithCustomer
        )

        expect(deleteResponse.status).toEqual(200)
        expect(deleteResponse.data).toEqual({
          deleted: true,
          id: product.id,
        })

        // Verify item is gone from list
        const listResponse = await api.get(
          `/store/wishlist`,
          storeHeadersWithCustomer
        )

        expect(listResponse.data.count).toEqual(0)
        expect(listResponse.data.wishlist_items).toHaveLength(0)
      })
    })
  },
})
```

**Test coverage:**
1. **Authenticated add** (AC 1) — Verify POST /store/wishlist accepts product_id and returns wishlist item with correct customer_id.
2. **Unauthenticated request rejection** (AC 7) — Verify POST without authentication token returns 401.
3. **List items** (AC 2) — Verify GET /store/wishlist returns all items with correct count.
4. **Remove item** (AC 3) — Verify DELETE /store/wishlist/:product_id removes item and updates list.

---

### integration-tests/http/__tests__/wishlist/admin/wishlist.spec.ts

**Complete test file** — Integration tests for Story 2 admin API endpoints.

```typescript
import { medusaIntegrationTestRunner } from "@medusajs/test-utils"
import {
  createAdminUser,
  generatePublishableKey,
  generateStoreHeaders,
} from "../../../../helpers/create-admin-user"
import { createAuthenticatedCustomer } from "../../../../modules/helpers/create-authenticated-customer"

const env = {}
const adminHeaders = { headers: { "x-medusa-access-token": "test_token" } }

medusaIntegrationTestRunner({
  env,
  testSuite: ({ dbConnection, getContainer, api }) => {
    let appContainer
    let storeHeaders
    let storeHeadersWithCustomer
    let customer
    let product

    beforeAll(async () => {
      appContainer = getContainer()
      await createAdminUser(dbConnection, adminHeaders, appContainer)
      const publishableKey = await generatePublishableKey(appContainer)
      storeHeaders = generateStoreHeaders({ publishableKey })

      const result = await createAuthenticatedCustomer(api, storeHeaders, {
        first_name: "Test",
        last_name: "Customer",
        email: "admin-test@wishlist.com",
      })

      customer = result.customer
      storeHeadersWithCustomer = {
        headers: {
          ...storeHeaders.headers,
          authorization: `Bearer ${result.jwt}`,
        },
      }

      // Create a test product
      const shippingProfile = (
        await api.post(
          `/admin/shipping-profiles`,
          { name: "default", type: "default" },
          adminHeaders
        )
      ).data.shipping_profile

      product = (
        await api.post(
          "/admin/products",
          {
            title: "Admin Test Product",
            status: "published",
            shipping_profile_id: shippingProfile.id,
            options: [{ title: "Size", values: ["L"] }],
            variants: [
              {
                title: "L",
                sku: "ADMIN-TEST-L",
                options: { Size: "L" },
                manage_inventory: false,
                prices: [{ amount: 2000, currency_code: "usd" }],
              },
            ],
          },
          adminHeaders
        )
      ).data.product

      // Add product to customer's wishlist via store endpoint
      await api.post(
        `/store/wishlist`,
        { product_id: product.id },
        storeHeadersWithCustomer
      )
    })

    describe("GET /admin/customers/:customer_id/wishlist (Story 2)", () => {
      it("should list wishlist items for a specific customer (AC 2 admin view)", async () => {
        const response = await api.get(
          `/admin/customers/${customer.id}/wishlist`,
          adminHeaders
        )

        expect(response.status).toEqual(200)
        expect(response.data.count).toEqual(1)
        expect(response.data.wishlist_items).toHaveLength(1)
        expect(response.data.wishlist_items[0]).toEqual(
          expect.objectContaining({
            id: expect.any(String),
            customer_id: customer.id,
            product_id: product.id,
          })
        )
      })
    })

    describe("DELETE /admin/customers/:customer_id/wishlist/:product_id (Story 2)", () => {
      it("should remove product from customer's wishlist (AC 3 admin action)", async () => {
        const deleteResponse = await api.delete(
          `/admin/customers/${customer.id}/wishlist/${product.id}`,
          adminHeaders
        )

        expect(deleteResponse.status).toEqual(200)
        expect(deleteResponse.data).toEqual({
          deleted: true,
          id: product.id,
        })

        // Verify item is gone from admin list view
        const listResponse = await api.get(
          `/admin/customers/${customer.id}/wishlist`,
          adminHeaders
        )

        expect(listResponse.data.count).toEqual(0)
        expect(listResponse.data.wishlist_items).toHaveLength(0)
      })
    })
  },
})
```

**Test coverage:**
1. **Admin list** (AC 2 admin view) — Verify GET /admin/customers/:customer_id/wishlist returns items for specified customer.
2. **Admin delete** (AC 3 admin action) — Verify DELETE /admin/customers/:customer_id/wishlist/:product_id removes item.

---

## Dependencies

- **Story 1** — Requires `WishlistService` with methods `addToWishlist`, `removeFromWishlist`, `listWishlistItems`, exported `WISHLIST_SERVICE` registration name constant, and `Wishlist` model to exist for injection and operation.

---

## Out of Scope

- **Product details in response (name, image, price):** Story 3 adds product relation population and price calculation.
- **UI components:** Story 3 implements frontend integration.
- **Pagination:** Deferred per HLD open question unless needed.

---

## Notes

**Authentication patterns:** Verify exact authentication middleware patterns by inspecting:
- Store customer auth: `packages/medusa/src/api/store/carts/route.ts`
- Admin auth: `packages/medusa/src/api/admin/customers/route.ts`

The middleware pattern `authenticate("customer", ["session", "bearer"])` and `req.auth.actor_id` extraction follows existing Medusa v2 conventions. If the actual pattern differs (e.g., different field name or middleware signature), adjust the implementation accordingly.

**Nested admin route structure:** The admin DELETE endpoint requires both `customer_id` and `product_id` from the URL. If Medusa routing requires a separate file for the nested dynamic segment, structure as:
- `packages/medusa/src/api/admin/customers/[customer_id]/wishlist/route.ts` (GET)
- `packages/medusa/src/api/admin/customers/[customer_id]/wishlist/[product_id]/route.ts` (DELETE)

Verify by checking existing nested admin routes or Medusa routing documentation.

**Service resolution:** The pattern `req.scope.resolve(WISHLIST_SERVICE)` uses the registration name constant exported from Story 1's service implementation. This follows the fix for C4: Story 1 exports `export const WISHLIST_SERVICE = "wishlistService"` and shows the DI registration pattern. All route handlers import and use this constant for service resolution.
