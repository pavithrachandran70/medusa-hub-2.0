# breakdown.md — PER-396

**Type:** feature  
**Status:** draft  
**Updated:** 2025-01-28  
**Covers:** PER-396 — Product Comparison Feature

---

## Execution Order

Stories are sequenced provider-before-consumer: state management foundation, then entry point integrations (product card, PDP), then comparison page UI, then persistence, then routing. Each story is a self-contained repo change that can be built and tested independently.

---

### Story 1: Comparison State Management and Context Setup

**Repo:** `medusa-appli`

#### Context
Establish the React Context foundation that holds the comparison list (up to 4 product IDs) and operations (`add`, `remove`, `clear`). This state must be accessible across product cards, product detail pages, and the comparison page without prop-drilling. This story does not include localStorage persistence (Story 5) or UI components (Stories 2–4); it creates only the in-memory state infrastructure and the hook.

#### Scope
- **New:** `www/src/lib/context/comparison-context.tsx` — ComparisonProvider, ComparisonContext, useComparison hook
- **New:** `www/src/lib/context/comparison-context.types.ts` — TypeScript interfaces for context value
- **New:** `www/src/lib/context/index.ts` — Barrel export (if not already present)

#### Implementation Pointers
- Create `ComparisonContext` using `React.createContext<ComparisonContextValue | null>(null)`
- Implement `ComparisonProvider` as a client component (`"use client"`) that wraps children and provides state via `useState<string[]>([])` for `productIds`
- Implement `add(productId: string)` function: check `productIds.length < 4` (AC2), check `!productIds.includes(productId)` (AC3), return `{ success: boolean; reason?: "limit" | "duplicate" }`, update state if valid
- Implement `remove(productId: string)` function: filter `productId` from array (AC5)
- Implement `clear()` function: set `productIds` to `[]` (AC6)
- Export `useComparison()` hook that calls `useContext(ComparisonContext)` and throws if context is null
- Export types: `ComparisonContextValue`, `AddResult` interfaces

#### Acceptance Criteria
1. The comparison list can hold up to 4 products (AC2).
2. Duplicate products are prevented from being added (AC3).
3. Individual products can be removed from the comparison list (AC5).
4. All products can be cleared from the comparison list (AC6).

#### Dependencies
None — foundational story.

#### Out of Scope
- UI components (CompareButton, comparison page) — Stories 2–4
- LocalStorage persistence — Story 5
- Routing — Story 6
- Integration with product cards or PDP — Stories 2–3

#### Verification Approach
**4 unit tests** (one per AC):
1. Test: adding 4 products succeeds; attempting to add a 5th returns `{ success: false, reason: "limit" }`
2. Test: adding the same product twice returns `{ success: false, reason: "duplicate" }` on second attempt
3. Test: calling `remove(productId)` removes that product from the list; `productIds` array length decreases by 1
4. Test: calling `clear()` sets `productIds` to empty array

---

### Story 2: CompareButton Integration in Product Cards

**Repo:** `medusa-appli`

#### Context
Add a Compare button to each product card on the product listing page (PLP). The button uses the comparison context from Story 1 to add/remove products and reflects the current state (selected, limit reached, disabled). This story does not touch the product detail page (Story 3) or the comparison page (Story 4).

#### Scope
- **New:** `www/src/components/comparison/compare-button.tsx` — CompareButton component
- **New:** `www/src/components/comparison/index.ts` — Barrel export (if not already present)
- **Modified:** Product card component (verify path in `www/src/components/` or `www/app/` — likely `www/src/components/product-card.tsx` or similar) — add `<CompareButton productId={product.id} />`

#### Implementation Pointers
- Create `CompareButton` as a client component (`"use client"`) that accepts `productId: string` prop
- Call `useComparison()` hook to get `productIds`, `add`, `remove`
- Compute `isSelected = productIds.includes(productId)` and `isLimitReached = productIds.length >= 4`
- Render a button with label "Compare" (or "Added to Comparison" if `isSelected`)
- On click: if `isSelected`, call `remove(productId)`; else call `add(productId)` and show a toast/notification on failure (limit or duplicate)
- Disable button if `isLimitReached && !isSelected` (AC2)
- In product card component (verify path: search `www/` for existing product card rendering logic — likely under `www/src/components/` or `www/app/`), import and render `<CompareButton productId={product.id} />` as a sibling to "Add to Cart" or below product image

#### Acceptance Criteria
1. A Compare button is visible on each product card (AC1).
2. Customers can add up to 4 products for comparison; attempting to add a 5th is blocked and shows a message (AC2).
3. Duplicate products are prevented from being added (AC3).

#### Dependencies
- **Story 1** — Comparison context must exist for `useComparison()` hook

#### Out of Scope
- Product detail page integration — Story 3
- Comparison page UI — Story 4
- LocalStorage persistence — Story 5
- The comparison page does not exist yet; added products are held in memory only

#### Verification Approach
**3 integration tests** (one per AC):
1. Test: product card renders with a visible "Compare" button (check DOM)
2. Test: clicking "Compare" on 4 different products succeeds; clicking on a 5th shows a notification (mock `add` to return `{ success: false, reason: "limit" }`)
3. Test: clicking "Compare" on a product already in the list shows a notification or does nothing (mock `add` to return `{ success: false, reason: "duplicate" }`)

---

### Story 3: CompareButton Integration in Product Detail Pages

**Repo:** `medusa-appli`

#### Context
Add a Compare button to the product detail page (PDP), reusing the same `CompareButton` component from Story 2. The button behaves identically: adds/removes the current product from the comparison list, reflects state, and enforces limits. This story does not touch product cards (already done in Story 2) or the comparison page (Story 4).

#### Scope
- **Modified:** Product detail page component (verify path in `www/src/pages/` or `www/app/` — likely `www/src/pages/product/[id].tsx`, `www/app/product/[id]/page.tsx`, or similar) — add `<CompareButton productId={product.id} />`

#### Implementation Pointers
- In product detail page component (verify path: search `www/` for existing PDP rendering logic — likely under `www/src/pages/product/` or `www/app/product/`), import `CompareButton` from `www/src/components/comparison`
- Render `<CompareButton productId={product.id} />` near the main CTA section (e.g., below "Add to Cart")
- No changes to `CompareButton` itself; it uses `useComparison()` hook as in Story 2

#### Acceptance Criteria
1. A Compare button is visible on the product detail page (AC1 extended to PDP).
2. Customers can add products from the PDP; the same 4-product limit and duplicate prevention apply (AC2, AC3).

#### Dependencies
- **Story 1** — Comparison context must exist
- **Story 2** — `CompareButton` component must exist

#### Out of Scope
- Product card integration — already done in Story 2
- Comparison page UI — Story 4
- LocalStorage persistence — Story 5

#### Verification Approach
**2 integration tests** (one per AC):
1. Test: product detail page renders with a visible "Compare" button (check DOM)
2. Test: clicking "Compare" on the PDP adds the product to the list (verify via `useComparison()` state); same limit and duplicate logic as Story 2

---

### Story 4: Comparison Page UI and Product Display

**Repo:** `medusa-appli`

#### Context
Create the dedicated comparison page at `/compare` that displays all selected products side-by-side in columns. Each product shows image, title, price, variants, and availability. Users can remove individual products or clear all. This story does not include localStorage persistence (Story 5) or the route setup (Story 6); it creates the UI components only, which will be rendered by the route in Story 6.

#### Scope
- **New:** `www/src/components/comparison/comparison-page.tsx` — Comparison page container component
- **New:** `www/src/components/comparison/comparison-list.tsx` — Renders columns of products (CSS Grid layout)
- **New:** `www/src/components/comparison/comparison-item.tsx` — Single product column with details
- **Modified:** `www/src/components/comparison/index.ts` — Add exports for new components

#### Implementation Pointers
- Create `ComparisonPage` as a client component (`"use client"`) that calls `useComparison()` to get `productIds`, `remove`, `clear`
- Fetch product data for all `productIds` using existing product query hook (verify hook name in `www/` — likely `useProducts` from `@medusajs/js-sdk` or similar; check `www/src/lib/` or `www/src/hooks/`)
- Filter out missing products (deleted/unavailable) silently (AC4 extended)
- Render loading state while fetching; render empty state if `productIds.length === 0` after filtering
- Render "Clear All" button that calls `clear()` (AC6)
- Pass fetched products to `ComparisonList`
- Create `ComparisonList` component that accepts `products: ProductDTO[]` and `onRemove: (productId: string) => void` props
- Use CSS Grid with `grid-template-columns: repeat(auto-fit, minmax(200px, 1fr))` for responsive layout (1–2 columns on mobile, all selected products in one row on desktop)
- Render `ComparisonItem` for each product
- Create `ComparisonItem` component that accepts `product: ProductDTO` and `onRemove: (productId: string) => void` props
- Render product image (top), title, price, variant selector (if multiple variants exist — display only, no switching logic per deferred decision), availability status (AC4)
- Render "Remove" button that calls `onRemove(product.id)` (AC5)

#### Acceptance Criteria
1. Selected products are displayed on the comparison page in columns, showing product image, title, price, variants, and availability (AC4).
2. Customers can remove individual products from the comparison page (AC5).
3. Customers can clear all products via a "Clear All" button (AC6).

#### Dependencies
- **Story 1** — Comparison context must exist for `useComparison()` hook
- **Story 2 or 3** — At least one entry point must exist to add products (not a build dependency, but a testing dependency)

#### Out of Scope
- Routing setup — Story 6 creates the `/compare` route that renders `ComparisonPage`
- LocalStorage persistence — Story 5
- Variant switching on the comparison page (deferred decision)

#### Verification Approach
**3 integration tests** (one per AC):
1. Test: comparison page fetches and displays products from `productIds`; each product shows image, title, price, variants, availability (mock product query to return sample products)
2. Test: clicking "Remove" on a product column calls `remove(productId)` and re-renders without that product
3. Test: clicking "Clear All" calls `clear()` and shows empty state

---

### Story 5: LocalStorage Persistence and Sync

**Repo:** `medusa-appli`

#### Context
Persist the comparison list in browser localStorage so returning visitors on the same browser retain their selections. The context reads from localStorage on mount (hydration) and writes to it on every state change (`add`, `remove`, `clear`). This story modifies only the context from Story 1; no UI changes.

#### Scope
- **Modified:** `www/src/lib/context/comparison-context.tsx` — Add localStorage read on mount and write on state change

#### Implementation Pointers
- In `ComparisonProvider`, use `useEffect(() => { ... }, [])` to read `localStorage.getItem("medusa_comparison")` on mount
- Parse JSON to get `comparison_products: string[]` and set initial `productIds` state
- Handle parse errors gracefully (invalid JSON or missing key → start with empty array)
- Use `useEffect(() => { ... }, [productIds])` to write `localStorage.setItem("medusa_comparison", JSON.stringify({ comparison_products: productIds }))` whenever `productIds` changes
- Wrap localStorage calls in try-catch to handle unavailable localStorage (private browsing, etc.) — feature degrades gracefully

#### Acceptance Criteria
1. Comparison selections persist in browser storage and are restored when the customer returns to the site in the same browser (AC7 part 1).
2. Browser storage is updated immediately when a product is added or removed (AC7 part 2).

#### Dependencies
- **Story 1** — Comparison context must exist

#### Out of Scope
- Cross-tab sync using `storage` event listener (deferred enhancement)
- UI changes — this story only modifies context logic

#### Verification Approach
**2 integration tests** (one per AC):
1. Test: add products to comparison list; reload page (or remount `ComparisonProvider`); verify `productIds` state is restored from localStorage
2. Test: add or remove a product; verify `localStorage.getItem("medusa_comparison")` reflects the change immediately

---

### Story 6: Comparison Page Routing

**Repo:** `medusa-appli`

#### Context
Create the `/compare` route that renders the comparison page UI from Story 4. This story adds only the route file; the UI components already exist. The route is public (no auth required) and accessible via direct navigation or links elsewhere in the storefront.

#### Scope
- **New:** Route file for `/compare` — verify routing pattern in `www/`: if Next.js pages router, create `www/src/pages/compare.tsx`; if app router, create `www/app/compare/page.tsx` (check for `www/src/pages/_app.tsx` vs `www/app/layout.tsx` to determine)

#### Implementation Pointers
- Determine routing pattern: search `www/` for `pages/_app.tsx` or `app/layout.tsx`
- If pages router: create `www/src/pages/compare.tsx` as a default export function that returns `<ComparisonPage />`
- If app router: create `www/app/compare/page.tsx` as a default export function that returns `<ComparisonPage />`
- Import `ComparisonPage` from `www/src/components/comparison`
- No props needed; `ComparisonPage` uses `useComparison()` hook internally
- Ensure route is public (no auth check)

#### Acceptance Criteria
1. Navigating to `/compare` renders the comparison page UI (enables AC4–AC6).

#### Dependencies
- **Story 4** — `ComparisonPage` component must exist

#### Out of Scope
- UI components — already done in Story 4
- Deep-linking with product IDs in query params (deferred decision)
- Persistent "Compare" link in header/footer (deferred enhancement)

#### Verification Approach
**1 integration test:**
1. Test: navigating to `/compare` renders `ComparisonPage` component (verify DOM contains comparison page elements)

---

### Story 7: Wrap App Root with ComparisonProvider

**Repo:** `medusa-appli`

#### Context
Wrap the storefront application root (or layout) with `ComparisonProvider` so the comparison context is available to all pages and components. This story modifies only the app root or layout file; no logic changes.

#### Scope
- **Modified:** App root or layout file (verify path in `www/` — likely `www/src/pages/_app.tsx` for pages router, or `www/app/layout.tsx` for app router)

#### Implementation Pointers
- Determine routing pattern: search `www/` for `pages/_app.tsx` or `app/layout.tsx`
- If pages router: in `www/src/pages/_app.tsx`, import `ComparisonProvider` from `www/src/lib/context`, wrap `<Component {...pageProps} />` with `<ComparisonProvider>...</ComparisonProvider>`
- If app router: in `www/app/layout.tsx`, import `ComparisonProvider`, wrap `{children}` with `<ComparisonProvider>...</ComparisonProvider>`
- Ensure `ComparisonProvider` is a client component (`"use client"`) if app router RootLayout is a server component (wrap with a separate client component if needed)

#### Acceptance Criteria
1. The comparison context is accessible from all pages and components (enables AC1–AC7).

#### Dependencies
- **Story 1** — Comparison context must exist

#### Out of Scope
- Context logic changes — already done in Stories 1 and 5
- UI components — already done in Stories 2–4

#### Verification Approach
**1 integration test:**
1. Test: render the app root with `ComparisonProvider`; call `useComparison()` from a child component; verify context is not null

---

## Cross-Story Dependencies

None — all dependencies are listed in each story's Dependencies section.

---

## Risks

### Technical Risks

1. **Unknown Product Card/PDP Component Paths (Story 2–3):**
   - **Impact:** Cannot integrate `CompareButton` without knowing where these components live.
   - **Mitigation:** Verify file paths in `www/src/components/` and `www/src/pages/` before starting Stories 2 and 3. Search for existing product rendering logic.
   - **Likelihood:** Medium.

2. **Unknown Product Query Hook (Story 4):**
   - **Impact:** Cannot fetch product data on the comparison page without knowing the existing query hook name.
   - **Mitigation:** Search `www/` for `useProducts`, `useProductDetail`, or similar hooks in `www/src/hooks/` or `www/src/lib/`. Check `@medusajs/js-sdk` imports.
   - **Likelihood:** Medium.

3. **Unknown Routing Pattern (Story 6):**
   - **Impact:** Cannot create the `/compare` route without knowing if the app uses Next.js pages or app router.
   - **Mitigation:** Check `www/` for `pages/_app.tsx` vs `app/layout.tsx` structure before starting Story 6.
   - **Likelihood:** Low (Next.js convention is predictable).

4. **LocalStorage Unavailable (Story 5):**
   - **Impact:** Comparison list does not persist across sessions in private browsing or when localStorage is disabled.
   - **Mitigation:** Feature degrades gracefully — comparison works in-session, but selections are lost on page refresh. No error shown to user.
   - **Likelihood:** Low (localStorage is universally available in modern browsers).

### Integration Risks

1. **App Root Wrapping (Story 7):**
   - **Impact:** If app router RootLayout is a server component, `ComparisonProvider` (client component) cannot be directly nested.
   - **Mitigation:** Wrap `ComparisonProvider` in a separate client component if needed.
   - **Likelihood:** Medium.

### UX Risks

1. **No Persistent "Compare" Link:**
   - **Impact:** Users may not know how to navigate to the comparison page after adding products.
   - **Mitigation:** Deferred enhancement — add a persistent "Compare" button in header/footer with a count badge in a future story.
   - **Likelihood:** High (poor discoverability in initial implementation).

---

## Open Questions

1. **What is the exact path and name of the product card component in the storefront codebase?**  
   → Required for Story 2 to integrate `CompareButton`.

2. **What is the exact path and name of the product detail page component?**  
   → Required for Story 3 to integrate `CompareButton`.

3. **What is the exact name of the product query hook used in the storefront (e.g., `useProducts`, `useProductDetail`, etc.)?**  
   → Required for Story 4 to fetch product data on the comparison page.

4. **Is the storefront using Next.js pages or app router?**  
   → Determines where to create the `/compare` route in Story 6 (`pages/compare.tsx` vs `app/compare/page.tsx`) and how to wrap the app root in Story 7 (`pages/_app.tsx` vs `app/layout.tsx`).

5. **Should the comparison page handle variant selection (allow users to switch which variant is compared), or is the default variant sufficient?**  
   → AC4 says "show variants", but does not specify if users can switch. Currently deferred to future enhancement unless clarified.

6. **Should there be a persistent "Compare" button in the header or footer to access `/compare`?**  
   → Not required by AC, but improves UX. Deferred to future enhancement.

7. **What happens when a product in the comparison list is no longer available (deleted, out of stock)?**  
   → Should it remain in the list with a visual indicator, or be removed automatically? Currently deferred — it remains in the list; the comparison page shows a "not available" message.

---

**End of Breakdown**
