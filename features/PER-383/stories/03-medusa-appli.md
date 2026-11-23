# Story 03-medusa-appli: Product details enrichment and comprehensive test coverage

**Repo:** `medusa-appli`

## Context

Enrich wishlist API responses with product details (name, image, price) by eagerly loading the product relation and calculating prices in the customer's context. Add end-to-end integration tests covering all acceptance criteria including authentication, idempotency, persistence, and empty state.

## Acceptance Criteria

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

## File Plan

### packages/medusa/src/services/wishlist.ts

Modify the existing service to enrich wishlist items with product details and calculated prices.

#### Function: listWishlistItems

Enhance the existing method to populate product relation and calculate prices in customer context.

**Current implementation** (from Story 1) queries wishlist rows with basic product relation populated:

```typescript
async listWishlistItems(customerId: string): Promise<Wishlist[]> {
  const wishlists = await this.manager_.find(
    Wishlist,
    { customer: customerId },
    { populate: ["product"] }
  )

  return wishlists
}
```

**Enhanced implementation:**

Replace the entire `listWishlistItems` method body with the following code that adds price calculation logic:

```typescript
async listWishlistItems(customerId: string): Promise<Wishlist[]> {
  const wishlists = await this.manager_.find(
    Wishlist,
    { customer: customerId },
    { populate: ["product", "product.variants"] }
  )

  // Inject pricing module to calculate prices for each product
  const pricingModule = this.container_.resolve(Modules.PRICING)
  const customerModule: ICustomerModuleService = this.customerModuleService_
  
  // Resolve customer context for pricing
  const customer = await customerModule.retrieveCustomer(customerId, {
    relations: ["groups"]
  })
  
  // Get customer's default region and sales channel context
  // Note: In a real implementation, region would come from cart/session context
  // For wishlist, we use the store's default context
  const storeModule = this.container_.resolve(Modules.STORE)
  const [defaultStore] = await storeModule.listStores({}, { take: 1 })
  
  // Calculate prices for each product's variants
  for (const wishlist of wishlists) {
    if (wishlist.product && wishlist.product.variants) {
      const variantIds = wishlist.product.variants.map((v) => v.id)
      
      // Build pricing context
      const pricingContext = {
        currency_code: defaultStore?.default_currency_code || "usd",
        customer_group_id: customer.groups?.[0]?.id,
        // Additional context from sales channel if available
      }
      
      // Calculate prices using pricing module
      const calculatedPrices = await pricingModule.calculatePrices(
        { variant_id: variantIds },
        pricingContext
      )
      
      // Attach calculated prices to variants
      wishlist.product.variants = wishlist.product.variants.map((variant) => {
        const priceData = calculatedPrices.find((p) => p.variant_id === variant.id)
        return {
          ...variant,
          calculated_price: priceData
            ? {
                amount: priceData.calculated_amount,
                currency_code: priceData.currency_code,
              }
            : null,
        }
      })
    }
  }

  return wishlists
}
```

**Key changes:**
- Change `{ populate: ["product"] }` to `{ populate: ["product", "product.variants"] }` to eagerly load variants
- Inject `Modules.PRICING` to access pricing calculations
- Inject `Modules.STORE` to get default currency context
- Retrieve customer with groups for customer group pricing
- Calculate prices for all variants of each wishlisted product
- Attach `calculated_price` to each variant with `amount` and `currency_code`

**Note:** Verify the exact pricing workflow API by checking `packages/medusa/src/api/store/carts/route.ts` for reference implementations. If a workflow like `calculatePricesWorkflow` exists instead of `pricingModule.calculatePrices`, use that pattern instead.

---

### packages/medusa/src/api/store/wishlist/route.ts

No changes needed to the GET handler. The enhanced service method from above already returns enriched data, and the GET handler already returns the service result directly (from Story 2 implementation).

---

## Tests

### integration-tests/http/__tests__/wishlist/store/wishlist.spec.ts

**Complete test file** — Extend the existing file from Story 2 with the following complete implementation including both Story 2 and Story 3 tests:

```typescript
import { medusaIntegrationTestRunner } from "@medusajs/test-utils"
import { Modules, ProductStatus } from "@medusajs/utils"
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
    let product
    let shippingProfile
    let region

    beforeAll(async () => {
      appContainer = getContainer()
      await createAdminUser(dbConnection, adminHeaders, appContainer)
      const publishableKey = await generatePublishableKey(appContainer)
      storeHeaders = generateStoreHeaders({ publishableKey })

      // Create shared test product
      shippingProfile = (
        await api.post(
          `/admin/shipping-profiles`,
          { name: "default", type: "default" },
          adminHeaders
        )
      ).data.shipping_profile

      region = (
        await api.post(
          "/admin/regions",
          {
            payment_providers: ["pp_system_default"],
            name: "US",
            currency_code: "usd",
            countries: ["us"],
          },
          adminHeaders
        )
      ).data.region

      product = (
        await api.post(
          "/admin/products",
          {
            title: "Test Wishlist Product",
            status: ProductStatus.PUBLISHED,
            shipping_profile_id: shippingProfile.id,
            options: [{ title: "Size", values: ["S", "M"] }],
            variants: [
              {
                title: "Small",
                sku: "TEST-WISHLIST-S",
                options: { Size: "S" },
                manage_inventory: false,
                prices: [{ amount: 1000, currency_code: "usd" }],
              },
              {
                title: "Medium",
                sku: "TEST-WISHLIST-M",
                options: { Size: "M" },
                manage_inventory: false,
                prices: [{ amount: 1200, currency_code: "usd" }],
              },
            ],
          },
          adminHeaders
        )
      ).data.product
    })

    describe("POST /store/wishlist (Story 2)", () => {
      it("should add a product to authenticated customer's wishlist (AC 1)", async () => {
        const result = await createAuthenticatedCustomer(api, storeHeaders, {
          first_name: "Test",
          last_name: "User",
          email: "test-add@wishlist.com",
        })

        const storeHeadersWithCustomer = {
          headers: {
            ...storeHeaders.headers,
            authorization: `Bearer ${result.jwt}`,
          },
        }

        const response = await api.post(
          `/store/wishlist`,
          { product_id: product.id },
          storeHeadersWithCustomer
        )

        expect(response.status).toEqual(200)
        expect(response.data.wishlist_item).toEqual(
          expect.objectContaining({
            id: expect.any(String),
            customer_id: result.customer.id,
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
      let customer
      let storeHeadersWithCustomer

      beforeEach(async () => {
        const result = await createAuthenticatedCustomer(api, storeHeaders, {
          first_name: "List",
          last_name: "Test",
          email: `list-test-${Date.now()}@wishlist.com`,
        })

        customer = result.customer
        storeHeadersWithCustomer = {
          headers: {
            ...storeHeaders.headers,
            authorization: `Bearer ${result.jwt}`,
          },
        }

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
              status: ProductStatus.PUBLISHED,
              shipping_profile_id: shippingProfile.id,
              options: [{ title: "Size", values: ["L"] }],
              variants: [
                {
                  title: "Large",
                  sku: "TEST-WISHLIST-L",
                  options: { Size: "L" },
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
      let customer
      let storeHeadersWithCustomer
      let addedProduct

      beforeEach(async () => {
        const result = await createAuthenticatedCustomer(api, storeHeaders, {
          first_name: "Delete",
          last_name: "Test",
          email: `delete-test-${Date.now()}@wishlist.com`,
        })

        customer = result.customer
        storeHeadersWithCustomer = {
          headers: {
            ...storeHeaders.headers,
            authorization: `Bearer ${result.jwt}`,
          },
        }

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

    describe("Wishlist persistence and product details (Story 3)", () => {
      it("should persist wishlist items when customer logs out and logs in again (AC 5)", async () => {
        const customerModule = appContainer.resolve(Modules.CUSTOMER)
        const authModule = appContainer.resolve(Modules.AUTH)

        // Create test customer with password
        const customer = await customerModule.createCustomers({
          email: `persistence-${Date.now()}@example.com`,
          first_name: "Persistence",
          last_name: "Test",
        })

        // Create auth identity for email/password login
        const authIdentity = await authModule.createAuthIdentities({
          provider_identities: [
            {
              provider: "emailpass",
              entity_id: customer.id,
              provider_metadata: {
                email: customer.email,
                password: "testpassword",
              },
            },
          ],
        })

        // First session: authenticate and add product
        const loginResponse1 = await api.post("/auth/customer/emailpass", {
          email: customer.email,
          password: "testpassword",
        })
        const token1 = loginResponse1.data.token
        const headers1 = {
          headers: {
            ...storeHeaders.headers,
            authorization: `Bearer ${token1}`,
          },
        }

        const addResponse = await api.post(
          `/store/wishlist`,
          { product_id: product.id },
          headers1
        )
        const wishlistItemId = addResponse.data.wishlist_item.id

        // Log out (clear session)
        await api.delete("/auth/session", headers1)

        // Second session: log in again
        const loginResponse2 = await api.post("/auth/customer/emailpass", {
          email: customer.email,
          password: "testpassword",
        })
        const token2 = loginResponse2.data.token
        const headers2 = {
          headers: {
            ...storeHeaders.headers,
            authorization: `Bearer ${token2}`,
          },
        }

        // Verify wishlist persisted
        const listResponse = await api.get(`/store/wishlist`, headers2)

        expect(listResponse.status).toEqual(200)
        expect(listResponse.data.count).toEqual(1)
        expect(listResponse.data.wishlist_items).toHaveLength(1)
        expect(listResponse.data.wishlist_items[0]).toEqual(
          expect.objectContaining({
            id: wishlistItemId,
            customer_id: customer.id,
            product_id: product.id,
          })
        )
      })

      it("should return empty array and count zero for customer with no wishlist items (AC 7)", async () => {
        const result = await createAuthenticatedCustomer(api, storeHeaders, {
          first_name: "Empty",
          last_name: "Wishlist",
          email: `empty-${Date.now()}@wishlist.com`,
        })

        const authHeaders = {
          headers: {
            ...storeHeaders.headers,
            authorization: `Bearer ${result.jwt}`,
          },
        }

        const response = await api.get(`/store/wishlist`, authHeaders)

        expect(response.status).toEqual(200)
        expect(response.data).toEqual({
          wishlist_items: [],
          count: 0,
        })
      })

      it("should return product details including name, image, and calculated price (AC 6)", async () => {
        // Create product with explicit thumbnail
        const detailProduct = (
          await api.post(
            "/admin/products",
            {
              title: "Product Details Test Item",
              thumbnail: "https://example.com/thumbnail.jpg",
              status: ProductStatus.PUBLISHED,
              shipping_profile_id: shippingProfile.id,
              options: [{ title: "Size", values: ["S", "M"] }],
              variants: [
                {
                  title: "Small",
                  sku: "DETAIL-TEST-S",
                  options: { Size: "S" },
                  manage_inventory: false,
                  prices: [{ amount: 1000, currency_code: "usd" }],
                },
                {
                  title: "Medium",
                  sku: "DETAIL-TEST-M",
                  options: { Size: "M" },
                  manage_inventory: false,
                  prices: [{ amount: 1200, currency_code: "usd" }],
                },
              ],
            },
            adminHeaders
          )
        ).data.product

        const result = await createAuthenticatedCustomer(api, storeHeaders, {
          first_name: "Details",
          last_name: "Test",
          email: `details-${Date.now()}@example.com`,
        })

        const authHeaders = {
          headers: {
            ...storeHeaders.headers,
            authorization: `Bearer ${result.jwt}`,
          },
        }

        // Add product to wishlist
        await api.post(
          `/store/wishlist`,
          { product_id: detailProduct.id },
          authHeaders
        )

        // Retrieve wishlist
        const response = await api.get(`/store/wishlist`, authHeaders)

        expect(response.status).toEqual(200)
        expect(response.data.count).toEqual(1)
        expect(response.data.wishlist_items[0]).toEqual(
          expect.objectContaining({
            product_id: detailProduct.id,
            product: expect.objectContaining({
              id: detailProduct.id,
              title: "Product Details Test Item",
              thumbnail: "https://example.com/thumbnail.jpg",
              variants: expect.arrayContaining([
                expect.objectContaining({
                  id: expect.any(String),
                  title: expect.any(String),
                  calculated_price: expect.objectContaining({
                    amount: expect.any(Number),
                    currency_code: "usd",
                  }),
                }),
              ]),
            }),
          })
        )

        // Verify at least one variant has calculated price
        const variants = response.data.wishlist_items[0].product.variants
        expect(variants.length).toBeGreaterThan(0)
        expect(variants[0].calculated_price).toBeDefined()
        expect(variants[0].calculated_price.amount).toBeGreaterThan(0)
      })

      it("should return same wishlist item ID when adding duplicate product (AC 4)", async () => {
        const result = await createAuthenticatedCustomer(api, storeHeaders, {
          first_name: "Idempotency",
          last_name: "Test",
          email: `idempotency-${Date.now()}@example.com`,
        })

        const authHeaders = {
          headers: {
            ...storeHeaders.headers,
            authorization: `Bearer ${result.jwt}`,
          },
        }

        // First add
        const firstAdd = await api.post(
          `/store/wishlist`,
          { product_id: product.id },
          authHeaders
        )

        expect(firstAdd.status).toEqual(200)
        const firstItemId = firstAdd.data.wishlist_item.id

        // Second add (duplicate)
        const secondAdd = await api.post(
          `/store/wishlist`,
          { product_id: product.id },
          authHeaders
        )

        expect(secondAdd.status).toEqual(200)
        expect(secondAdd.data.wishlist_item.id).toEqual(firstItemId)

        // Verify only one item in wishlist
        const listResponse = await api.get(`/store/wishlist`, authHeaders)

        expect(listResponse.status).toEqual(200)
        expect(listResponse.data.count).toEqual(1)
        expect(listResponse.data.wishlist_items).toHaveLength(1)
      })

      it("should catch unique constraint violation and return existing item (AC 4 enforcement)", async () => {
        const wishlistService = appContainer.resolve("wishlistService")
        const customerModule = appContainer.resolve(Modules.CUSTOMER)

        // Create test customer and product
        const customer = await customerModule.createCustomers({
          email: `constraint-${Date.now()}@example.com`,
          first_name: "Constraint",
          last_name: "Test",
        })

        // Add first time
        const firstAdd = await wishlistService.addToWishlist(
          customer.id,
          product.id
        )

        // Attempt to add again via service (bypassing API idempotency)
        const secondAdd = await wishlistService.addToWishlist(
          customer.id,
          product.id
        )

        // Should return same item ID without throwing error
        expect(secondAdd.id).toEqual(firstAdd.id)

        // Verify database contains only one row
        const items = await wishlistService.listWishlistItems(customer.id)
        const matchingItems = items.filter(
          (item) => item.product_id === product.id
        )
        expect(matchingItems).toHaveLength(1)
      })

      it("should reject unauthenticated requests to all wishlist endpoints (AC 8)", async () => {
        // Test POST without auth
        const postResponse = await api
          .post(
            `/store/wishlist`,
            { product_id: product.id },
            storeHeaders // No auth token
          )
          .catch((e) => e)

        expect(postResponse.response.status).toEqual(401)

        // Test GET without auth
        const getResponse = await api
          .get(`/store/wishlist`, storeHeaders)
          .catch((e) => e)

        expect(getResponse.response.status).toEqual(401)

        // Test DELETE without auth
        const deleteResponse = await api
          .delete(`/store/wishlist/${product.id}`, storeHeaders)
          .catch((e) => e)

        expect(deleteResponse.response.status).toEqual(401)
      })
    })
  },
})
```

**Test coverage summary:**
- **6 tests from Story 2** (authenticated add, unauthenticated rejection, list items, remove item from earlier describe blocks)
- **6 tests from Story 3** (persistence, empty state, product details, idempotency via API, idempotency via service, authentication enforcement)
- **Total: 12 integration tests** covering all 10 acceptance criteria

---

## Dependencies

- **Story 1** — Requires `WishlistService` with methods `addToWishlist`, `removeFromWishlist`, `listWishlistItems`, and `Wishlist` model to exist for injection and operation.
- **Story 2** — Requires API endpoints to exist for integration testing.

---

## Out of Scope

- **UI components:** This ticket's scope is backend only. Frontend implementation (wishlist icon, page, state management) deferred to future work.
- **Pagination:** Assuming wishlist size < 100 items per customer per open question in HLD. If pagination needed, it's a future enhancement.
- **Admin test suite:** The file `integration-tests/http/__tests__/wishlist/admin/wishlist.spec.ts` was created in Story 2 with basic admin CRUD tests. No additional tests are required for Story 3, as admin endpoints do not add product details enrichment (they already return the data the service provides).

---

## Notes

### Pricing Module Integration

The pricing calculation in `listWishlistItems` follows Medusa's existing patterns for product price resolution. The exact implementation depends on the pricing module version:

- **If using pricing workflows:** Call `calculatePricesWorkflow(container).run({ input: { ... } })` as shown in cart and checkout flows.
- **If using pricing module service directly:** Call `pricingModule.calculatePrices(...)` with variant IDs and context.

**Verification steps:**
1. Check `packages/medusa/src/api/store/carts/route.ts` for reference implementations (see provided `repo-file` showing cart price calculation patterns).
2. Use the same pricing context structure (currency, customer groups, region) as those endpoints.
3. Verify the actual `pricingModule` API matches the implementation by checking existing service usage in cart or product modules.

If the pricing module API differs from the implementation above, adjust to match the actual interface. The key requirements are:
- Calculate prices for all variants of each wishlisted product
- Use customer context (groups, region) for accurate pricing
- Return `calculated_price` with `amount` and `currency_code` for each variant

### Test Database Queries

The database-level duplicate prevention test verifies row uniqueness by calling `listWishlistItems` and filtering results rather than using raw SQL, which is more portable and follows the existing test patterns in this codebase (see the provided cart test examples).

### Authentication Patterns

The authentication enforcement test verifies 401 responses for unauthenticated requests, following the pattern seen in existing Medusa tests (see provided cart test file showing `.catch((e) => e)` and `expect(response.response.status).toEqual(401)` patterns).
