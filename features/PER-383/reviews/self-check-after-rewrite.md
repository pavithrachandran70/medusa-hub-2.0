## What would go wrong

### C1
- file: stories/01-medusa-appli.md
- section: ## File Plan › packages/medusa/src/services/wishlist.ts › Imports and Constructor
- quote: private customerModuleService: ICustomerModuleService
- match: repo-file/medusa-appli/integration-tests/http/__tests__/cart/store/cart.spec.ts › line ~350
- severity: blocking
- earlier: C2
- blocks because: Dev would write a constructor with `this.customerModuleService = container.resolve(Modules.CUSTOMER)` and store module services as instance fields, but the existing pattern at line 350 of the cart test shows `const customerModule: ICustomerModuleService = appContainer.resolve(Modules.CUSTOMER)` — services are resolved inline when needed, not stored in the constructor. The service class would compile but not follow the codebase's DI pattern, and Story 2's routes would need a different resolution approach.
- problem: Story 1's constructor stores module services as instance fields (`this.customerModuleService`, `this.productModuleService`, `this.manager`) with a constructor that takes `container: MedusaContainer`, but the existing cart test code shows services are resolved inline when needed, not injected via constructor and stored as fields.
- change: Remove the constructor and all instance field declarations. In each method (`addToWishlist`, `removeFromWishlist`, `listWishlistItems`), resolve the services and manager inline when needed, following the pattern `const customerModule = appContainer.resolve(Modules.CUSTOMER)` shown in the cart tests. For Story 2's route handlers, use `const wishlistService = req.scope.resolve(WISHLIST_SERVICE)` where `WISHLIST_SERVICE` is the registration name exported from this file. Add a note referencing the cart test pattern for the developer to follow.

### C2
- file: stories/02-medusa-appli.md
- section: ## File Plan › packages/medusa/src/api/store/wishlist/route.ts › Function: POST
- quote: const wishlistService = req.scope.resolve(WISHLIST_SERVICE)
- match: repo-file/medusa-appli/integration-tests/http/__fixtures__/feature-flag/src/api/custom/route.ts
- severity: blocking
- earlier: C9
- blocks because: Dev would import and use `WISHLIST_SERVICE` as a constant for service resolution in the route handlers, but Story 1 does not export this constant. The route handler would fail to compile with "Cannot find name 'WISHLIST_SERVICE'".
- problem: Story 2's routes use `req.scope.resolve(WISHLIST_SERVICE)` to inject the service, and Story 2's Notes section says "Story 1 exports `export const WISHLIST_SERVICE = "wishlistService"`", but Story 1's service file does not include this export.
- change: In Story 1's `packages/medusa/src/services/wishlist.ts`, add `export const WISHLIST_SERVICE = "wishlistService"` at the top of the file after the imports, before the class definition. Also add a note showing how to register the service in the DI container (verify the exact pattern by checking an existing service registration in the modules, or note that this follows standard Medusa v2 service registration conventions).

### C3
- file: stories/01-medusa-appli.md
- section: ## Tests › integration-tests/http/__tests__/wishlist/store/wishlist.spec.ts
- quote: const originalQuery = dbConnection.query
- match: N/A
- severity: note
- earlier: new
- problem: The test uses `dbConnection.query` to intercept queries for N+1 detection, but the available repo outline does not show whether `dbConnection` has a `query` method or what the actual query logging API is in this codebase. The test may need adjustment to match the real database connection interface.
- change: (For the person at the gate) Verify the actual query logging or counting pattern used in this codebase's integration tests. If `dbConnection.query` is not the right API, adjust the N+1 test to use the correct method (e.g., MikroORM's query logger, or a different connection method). If no query counter exists, the test can be simplified to verify only that product relations are populated, not the query count.

### C4
- file: stories/01-medusa-appli.md
- section: ## File Plan › packages/medusa/src/services/wishlist.ts › Function: addToWishlist
- quote: this.generateId()
- match: N/A
- severity: blocking
- earlier: new
- blocks because: Dev would call `this.generateId()` to create a UUID for the wishlist entry, but the service class does not define this method, and the repo outline does not show a `generateId` method in existing services. The method call would fail to compile or throw at runtime.
- problem: The `addToWishlist` implementation calls `this.generateId()` with a note to "verify ID generation pattern in existing services", but no such method is shown in the available code. The developer needs a concrete way to generate the UUID.
- change: Replace `this.generateId()` with a direct UUID generation approach that matches this codebase. If existing services show a `generateId` utility, reference it; otherwise, use `import { v4 as uuidv4 } from "uuid"` and call `uuidv4()` for the `id` field. Add a note: "Verify UUID generation pattern in existing services; adjust if a utility function exists."

### C5
- file: stories/01-medusa-appli.md
- section: ## File Plan › packages/medusa/src/services/wishlist.ts › Function: addToWishlist
- quote: if (error.code === "23505") {
- match: N/A
- severity: note
- earlier: new
- problem: The idempotency logic catches a unique constraint violation by checking `error.code === "23505"` (PostgreSQL error code), but MikroORM may wrap or transform the error. The exact error structure is not shown in the available code examples.
- change: (For the person at the gate) Verify the actual error structure returned by MikroORM for unique constraint violations in this codebase. If the error code or field differs, adjust the catch block accordingly. The idempotency requirement (AC 4) must still be met, but the implementation may need to check `error.name`, `error.constraint`, or another field.

VERDICT: FAIL
