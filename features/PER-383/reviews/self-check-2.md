## What would go wrong

### C1
- file: stories/01-medusa-appli.md
- section: ## File Plan › packages/medusa/src/services/wishlist.ts › Function: addToWishlist
- quote: const customer = await this.customerModuleService_.retrieveCustomer(customerId)
- match: stories/01-medusa-appli.md › ## File Plan › packages/medusa/src/services/wishlist.ts › Imports and Constructor
- severity: blocking
- earlier: C1
- blocks because: Dev would write `this.customerModuleService_.retrieveCustomer(customerId)` and `this.productModuleService_.retrieveProduct(productId)`, but the constructor now injects these without the underscore suffix as `this.customerModuleService` and `this.productModuleService`. The methods would fail to compile because the field names do not match.
- problem: The constructor sets `this.customerModuleService` and `this.productModuleService` (no underscore), but the function implementations in `addToWishlist` still reference `this.customerModuleService_` and `this.productModuleService_` with underscores.
- change: In the `addToWishlist` function implementation, replace `this.customerModuleService_` with `this.customerModuleService` and `this.productModuleService_` with `this.productModuleService` throughout. Apply the same change to `removeFromWishlist` and `listWishlistItems` wherever they reference these fields.

### C8
- file: stories/01-medusa-appli.md
- section: ## File Plan › packages/medusa/src/services/wishlist.ts › Function: listWishlistItems
- quote: const wishlists = await this.manager_.find(
- match: stories/01-medusa-appli.md › ## File Plan › packages/medusa/src/services/wishlist.ts › Imports and Constructor
- severity: blocking
- earlier: new
- blocks because: Dev would write `this.manager_.find(...)`, `this.manager_.create(...)`, and `this.manager_.persistAndFlush(...)` throughout the service methods, but the constructor now injects the EntityManager as `this.manager` without the underscore suffix. The database operations would fail to compile.
- problem: The constructor sets `this.manager` (no underscore), but the function implementations still reference `this.manager_` with an underscore in `addToWishlist`, `removeFromWishlist`, and `listWishlistItems`.
- change: Replace `this.manager_` with `this.manager` throughout all three service methods: in `addToWishlist` (for `create`, `persistAndFlush`, `findOne`), in `removeFromWishlist` (for `findOne`, `removeAndFlush`), and in `listWishlistItems` (for `find`).

### C9
- file: stories/01-medusa-appli.md
- section: ## File Plan › packages/medusa/src/services/wishlist.ts › Imports and Constructor
- quote: export class WishlistService {
- match: stories/02-medusa-appli.md › ## File Plan › packages/medusa/src/api/store/wishlist/route.ts › Function: POST
- severity: blocking
- earlier: new
- blocks because: Dev would use `req.scope.resolve(WISHLIST_SERVICE)` to inject the service in Story 2, but Story 1's service file does not export the `WISHLIST_SERVICE` constant. The route handlers would fail to compile because the constant is undefined.
- problem: Story 2's routes import and use `WISHLIST_SERVICE` as a constant for service resolution, and the notes in Story 2 say "Story 1 exports `export const WISHLIST_SERVICE = "wishlistService"`", but Story 1's service file does not actually export this constant.
- change: Add `export const WISHLIST_SERVICE = "wishlistService"` at the top of `packages/medusa/src/services/wishlist.ts`, after the imports and before the class definition. Also add a note showing the DI registration pattern used by other services in this codebase (verify in `packages/modules/customer/services/` for the exact pattern).

### C10
- file: stories/03-medusa-appli.md
- section: ## File Plan › packages/medusa/src/services/wishlist.ts › Function: listWishlistItems
- quote: const pricingModule = this.container_.resolve(Modules.PRICING)
- match: stories/01-medusa-appli.md › ## File Plan › packages/medusa/src/services/wishlist.ts › Imports and Constructor
- severity: blocking
- earlier: C5
- blocks because: Dev would use `this.container_.resolve(Modules.PRICING)` to inject the pricing module in the `listWishlistItems` enhancement, but Story 1's constructor does not store the container. The method would fail to compile or throw at runtime.
- problem: Story 3 calls `this.container_.resolve(Modules.PRICING)` and `this.container_.resolve(Modules.STORE)`, but Story 1's constructor does not inject or store the container.
- change: In Story 1's constructor implementation, add `this.container = container` after the existing module service injections. Also add `private container: MedusaContainer` to the class field declarations at the top of the service class. This stores the full container for later use in Story 3.

### C11
- file: stories/03-medusa-appli.md
- section: ## File Plan › packages/medusa/src/services/wishlist.ts › Function: listWishlistItems
- quote: const customerModule: ICustomerModuleService = this.customerModuleService_
- match: stories/01-medusa-appli.md › ## File Plan › packages/medusa/src/services/wishlist.ts › Imports and Constructor
- severity: blocking
- earlier: new
- blocks because: Dev would reference `this.customerModuleService_` in Story 3's enhanced `listWishlistItems`, but Story 1's constructor now injects it without the underscore suffix as `this.customerModuleService`. The reference would fail to compile.
- problem: Story 3's enhanced `listWishlistItems` references `this.customerModuleService_` with an underscore, but Story 1's constructor sets it without an underscore as `this.customerModuleService`.
- change: In Story 3's enhanced `listWishlistItems` implementation, replace `this.customerModuleService_` with `this.customerModuleService` to match Story 1's field name.

## Notes for the gate

**Code validation methods:** Items C1 and C2 from the previous review asked about `retrieveCustomer` and `retrieveProduct` methods on the injected module services. The fixer marked these as "decided" that validation is deferred and the exact methods are not shown in the available outline. The current LLD still calls these methods in the code examples. If these methods do not exist on `ICustomerModuleService` and `IProductModuleService`, the service will fail at runtime. The gate should verify whether customer and product validation is required, and if so, what the actual module service APIs are.

**EntityManager resolution key:** Story 1's constructor uses `container.resolve("manager")` to inject the EntityManager. The LLD notes to "verify exact resolution key in `packages/modules/customer/services/`" but does not show what that verification found. If the actual key is different (e.g., `"dbConnection"` or a module-specific key), the service will fail to resolve the EntityManager at runtime.

**Service registration:** Item C9 asks for the `WISHLIST_SERVICE` constant to be exported from Story 1 and for a note showing the DI registration pattern. The LLD does not show *how* the service is registered in the DI container (e.g., via a service loader, module definition, or configuration file). The gate should verify that the service will actually be registered and resolvable via `req.scope.resolve(WISHLIST_SERVICE)` in Story 2.

**Pricing workflow structure:** Items C6 from the previous review asked about the pricing module API, and the fixer decided to use `calculatePricesWorkflow` instead. Story 3's LLD still shows direct calls to `pricingModule.calculatePrices(...)` with specific input/output structures. The notes say to "verify exact workflow input/output structure in `packages/core/core-flows/src/product`" but the code example assumes a structure that may not match the actual workflow. The gate should verify the pricing integration before Story 3 is implemented.

VERDICT: FAIL
