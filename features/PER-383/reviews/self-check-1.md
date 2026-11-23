## What would go wrong

### C1
- file: stories/01-medusa-appli.md
- section: ## File Plan › **Function:** `addToWishlist`
- quote: const wishlist = this.manager_.create(Wishlist, {
- match: integration-tests/http/__fixtures__/feature-flag/src/modules/custom/services/module-service.ts
- severity: blocking
- blocks because: Dev would use `this.manager_.create()` and `persistAndFlush()`, but the actual MikroORM EntityManager in this codebase uses `em.create()` and `em.persistAndFlush()` as instance methods. The service would fail to compile or run.
- earlier: new
- problem: The LLD shows `this.manager_.create(Wishlist, { ... })` and `this.manager_.persistAndFlush(wishlist)`, but MikroORM's EntityManager is injected as `manager` in this codebase's service constructors and used as `this.manager`, not `this.manager_`. The underscore suffix is not shown in any existing service code.
- change: Replace `this.manager_` with `this.manager` throughout the service implementation. Update the constructor to inject as `this.manager = container.resolve("manager")`. Use `this.manager.create()`, `this.manager.persistAndFlush()`, `this.manager.findOne()`, `this.manager.find()`, and `this.manager.removeAndFlush()`.

### C2
- file: stories/01-medusa-appli.md
- section: ## File Plan › **Function:** `addToWishlist`
- quote: const customer = await this.customerModuleService_.retrieveCustomer(customerId)
- match: hld.md › AD-3
- severity: blocking
- blocks because: Dev would call `retrieveCustomer()` on the injected service, but the HLD says "Inject `ICustomerModuleService`" without showing whether that service has this method. The outline shows no `retrieveCustomer` in customer module services. The service call would fail at runtime.
- earlier: new
- problem: The LLD calls `this.customerModuleService_.retrieveCustomer(customerId)` and `this.productModuleService_.retrieveProduct(productId)`, but the specs do not show these methods exist on the injected module services. The repo outline does not show `retrieveCustomer` or `retrieveProduct` methods in the module service interfaces.
- change: Verify the exact methods on `ICustomerModuleService` and `IProductModuleService` by checking `repo-outline/medusa-appli` for the customer and product module service interfaces. Use the actual method names, or omit the validation calls if the modules do not expose retrieve methods. If validation is required, state it as "verify customer and product exist (see `packages/modules/customer` and `packages/modules/product` for exact methods)" instead of inventing method signatures.

### C3
- file: stories/01-medusa-appli.md
- section: ## File Plan › **Function:** `addToWishlist`
- quote: this.generateId()
- match: hld.md › AD-3
- severity: blocking
- blocks because: Dev would call `this.generateId()` to create the primary key, but the LLD does not show where this method comes from. The service does not define it, and the specs do not import it. The service would fail to compile.
- earlier: new
- problem: The LLD calls `this.generateId()` without showing where this method is defined or imported. The HLD and breakdown do not mention ID generation, and the service implementation does not import any ID generation utility.
- change: Replace `id: this.generateId()` with a concrete approach: either import `{ generateEntityId }` from `@medusajs/utils` and use `id: generateEntityId(null, "wishlist")`, or rely on MikroORM's auto-generation by omitting the `id` field from the `create()` call. State in the notes that the exact ID generation pattern should match other entities in this codebase.

### C4
- file: stories/02-medusa-appli.md
- section: ## File Plan › packages/medusa/src/api/store/wishlist/route.ts › **Function:** POST
- quote: const wishlistService = req.scope.resolve("wishlistService")
- match: stories/01-medusa-appli.md › packages/medusa/src/services/wishlist.ts
- severity: blocking
- blocks because: Dev would resolve the service as `"wishlistService"`, but Story 1 does not show how the service is registered in the DI container. If the registration uses a different name or is missing, the route handler would throw at runtime.
- earlier: new
- problem: The route handler resolves `req.scope.resolve("wishlistService")`, but Story 1 does not show the service registration in the DI container. The service file exports the class but does not show a `REGISTRATION_NAME` constant or container registration call.
- change: Add to Story 1's service implementation: export a registration name constant `export const REGISTRATION_NAME = "wishlistService"` and show the DI registration pattern used by other services in this codebase (verify in `packages/medusa/src/services/` for the exact pattern). In Story 2, use that registration name in `req.scope.resolve()`.

### C5
- file: stories/03-medusa-appli.md
- section: ## File Plan › packages/medusa/src/services/wishlist.ts › **Function:** `listWishlistItems`
- quote: const pricingModule = this.container_.resolve(Modules.PRICING)
- match: stories/01-medusa-appli.md › packages/medusa/src/services/wishlist.ts
- severity: blocking
- blocks because: Dev would use `this.container_` to resolve the pricing module, but Story 1's service constructor does not inject the container, only specific module services and the entity manager. The service would fail to compile or throw at runtime.
- earlier: new
- problem: Story 3 calls `this.container_.resolve(Modules.PRICING)` and `this.container_.resolve(Modules.STORE)`, but Story 1's service constructor only injects `ICustomerModuleService`, `IProductModuleService`, and `EntityManager`. The container itself is not injected, so `this.container_` is undefined.
- change: Add to Story 1's service constructor: `this.container = container` (store the full container), and use `this.container.resolve(Modules.PRICING)` in Story 3. Alternatively, inject the pricing and store modules in the constructor alongside customer and product modules, following the same pattern.

### C6
- file: stories/03-medusa-appli.md
- section: ## File Plan › packages/medusa/src/services/wishlist.ts › **Function:** `listWishlistItems`
- quote: const pricingModule = this.container_.resolve(Modules.PRICING)
- match: integration-tests/http/__tests__/cart/store/cart.spec.ts › lines showing no `calculatePrices` calls
- severity: blocking
- blocks because: Dev would call `pricingModule.calculatePrices()`, but the provided cart tests show no such method being called directly on a pricing module. The actual pricing API may differ, causing the service to fail at runtime.
- earlier: new
- problem: Story 3 calls `pricingModule.calculatePrices({ variant_id: variantIds }, pricingContext)`, but the specs do not show this exact method signature on the pricing module. The HLD mentions `calculatePricesWorkflow` as an alternative, and the cart tests show pricing happening via workflows or other mechanisms, not via a direct `calculatePrices` method on a resolved module.
- change: Replace the direct `calculatePrices` call with the workflow pattern shown in the HLD: import `{ calculatePricesWorkflow }` from `@medusajs/core-flows`, and call `const { result } = await calculatePricesWorkflow(this.container).run({ input: { variant_ids: variantIds, context: pricingContext } })`. Use the workflow's output structure to populate `calculated_price` on each variant. Add a note: "Verify exact workflow input/output structure in `packages/core/core-flows/src/product`."

### C7
- file: specs/1.md
- section: ## File Touch List
- quote: packages/medusa/src/migrations/<timestamp>-CreateWishlistTable.ts
- match: hld.md › AD-1
- severity: note
- blocks because: N/A
- earlier: new
- problem: The spec lists the migration file with `<timestamp>` as a placeholder, but does not say how to generate it or when to run it. The HLD says "Migration file: `packages/medusa/src/migrations/<timestamp>-CreateWishlistTable.ts`" but gives no guidance on the timestamp format or migration execution.
- change: This is a standard pattern. The developer will generate the timestamp when creating the migration (usually via CLI or by following existing migration file naming). No change needed to the spec; it is acceptable to use `<timestamp>` as a placeholder.

## Notes for the gate

- **N1:** The LLD in Story 3 shows a large code block for `listWishlistItems` that attempts to calculate prices by resolving the pricing module and calling `calculatePrices()`. The actual pricing workflow API may differ from what is shown. The fixer should verify the exact workflow signature in `packages/core/core-flows/src/product` or `packages/medusa/src/api/store/carts` and adjust the implementation to match. If the pricing workflow requires a different input structure or returns prices in a different format, Story 3's implementation will need rewriting.
  
- **N2:** The service in Story 1 uses `this.customerModuleService_` and `this.productModuleService_` with underscore suffixes, and Story 2 continues this pattern with `this.container_`. This is inconsistent with the style shown in the provided cart test code, which uses no underscores. The developer should follow the actual codebase convention, but the specs should be internally consistent.

- **N3:** The admin DELETE endpoint in Story 2 may need a separate route file. The spec shows `packages/medusa/src/api/admin/customers/[customer_id]/wishlist/route.ts` with both GET and DELETE handlers, but then notes "the route structure is `/admin/customers/[customer_id]/wishlist` for GET and `/admin/customers/[customer_id]/wishlist/[product_id]` for DELETE (the latter may need to be a separate file)". The fixer should clarify the routing structure before Story 2 is implemented.

- **N4:** The test setup in all three stories uses `createAuthenticatedCustomer` and `generatePublishableKey` helpers that are shown in the provided cart tests. The specs assume these helpers exist and work as shown. If they change, the tests will need adjustment, but that is outside the scope of these specs.

- **N5:** The integration test file in Story 1 wraps tests in `medusaIntegrationTestRunner({ env, testSuite: ... })`, matching the pattern in the provided cart tests. The specs assume the test runner and helpers (e.g., `api.post`, `getContainer`) work as shown in the reference code. If the test framework changes, the tests will need rewriting.

VERDICT: FAIL
