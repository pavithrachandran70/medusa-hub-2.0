# architecture-per-396.md — PER-396 HLD

**Type:** feature  
**Status:** draft  
**Updated:** 2025-01-28  
**Covers:** PER-396 — Product Comparison Feature

---

## Overview

The Product Comparison feature enables customers to select up to 4 products and view them side-by-side on a dedicated comparison page. The architecture uses React Context for state management, localStorage for persistence, and integrates with existing product card and product detail page components via a new `CompareButton`.

---

## Paradigm

**Client-Side State with Persistent Storage**: The comparison list lives entirely in the browser — React Context provides in-memory state across components, localStorage ensures the list survives page refreshes, and no server-side session or API is required. Product data is fetched on-demand when the comparison page loads. This keeps the feature lightweight and aligns with the storefront's stateless browsing model.

---

## Architectural Decisions

### AD-1: Comparison State Management
**Binds:** State must be accessible from product cards, product detail pages, and the comparison page without prop-drilling.  
**Prevents:** State drift when comparison selections are made from multiple entry points (e.g. PLP and PDP).  
**Rule:** Use React Context (`ComparisonContext`) to hold the comparison list (`productIds: string[]`) and operations (`add`, `remove`, `clear`). The context provider wraps the storefront application root. Components consume it via `useComparison()` hook.

**[ADOPTED]** — Medusa storefront uses React, and Context is the standard pattern for cross-component state.

---

### AD-2: LocalStorage Schema and Sync
**Binds:** Comparison list must persist across browser sessions and sync automatically with in-memory state.  
**Prevents:** Loss of selections on page refresh; manual sync logic scattered across components.  
**Rule:** Store the comparison list as `comparison_products: string[]` in localStorage under the key `medusa_comparison`. The `ComparisonContext` reads this key on mount (hydration) and writes to it on every state change (`add`, `remove`, `clear`). The sync is unidirectional: state → storage.

**[ADOPTED]** — localStorage is universally available and fits the stateless storefront model.

---

### AD-3: 4-Product Limit Enforcement
**Binds:** Only 4 products can be in the comparison list at a time.  
**Prevents:** UI overflow and degraded comparison UX with too many columns.  
**Rule:** The `add` operation checks `productIds.length < 4` before inserting. If the limit is reached, `add` returns a failure indicator (e.g. `{ success: false, reason: "limit" }`), and the UI displays a notification (e.g. toast). The button remains visible but disabled when the limit is hit.

**[ADOPTED]** — Per AC2.

---

### AD-4: Duplicate Prevention
**Binds:** A product can appear in the comparison list at most once.  
**Prevents:** Redundant comparison columns and confusing UX.  
**Rule:** The `add` operation checks `productIds.includes(id)` before inserting. If the product is already present, `add` returns a failure indicator (e.g. `{ success: false, reason: "duplicate" }`), and the button reflects the "already selected" state (e.g. different label/icon, possibly disabled).

**[ADOPTED]** — Per AC3.

---

### AD-5: CompareButton Integration
**Binds:** The Compare button appears in product cards (PLP) and product detail pages (PDP).  
**Prevents:** Inconsistent UI/behavior when adding products from different pages.  
**Rule:** Create a single `CompareButton` component that accepts `productId` as a prop and uses `useComparison()` to call `add` or `remove`. The button's visual state (label, icon, disabled) reflects whether the product is in the list, and whether the 4-product limit is reached. Product cards and PDP import and render `<CompareButton productId={product.id} />`.

**[ADOPTED]** — Reuse across entry points ensures consistency.

---

### AD-6: Comparison Page Data Fetching
**Binds:** The comparison page displays product details (image, title, price, variants, availability) for all selected products.  
**Prevents:** Stale data or missing products when the comparison list references IDs that no longer exist.  
**Rule:** The comparison page reads `productIds` from `ComparisonContext` and fetches full product objects via the existing product query (e.g. `useProducts({ id: productIds })`). Missing products (deleted or unavailable) are silently filtered out of the display, and the context is not automatically updated. The page renders a loading state while fetching, and an empty state if `productIds.length === 0` after filtering.

**[ADOPTED]** — Uses existing product data fetching patterns.

---

### AD-7: Comparison Page Layout
**Binds:** Each product is displayed in a column; the page must handle 1–4 columns responsively.  
**Prevents:** Horizontal scroll issues on mobile, cramped comparison on desktop.  
**Rule:** Use CSS Grid with `grid-template-columns: repeat(auto-fit, minmax(200px, 1fr))` to auto-wrap columns on smaller screens. Each column contains: product image (top), title, price, variant selector (if multiple variants), availability status. Mobile (<768px) shows 1–2 columns; desktop shows all selected products in one row.

**[ADOPTED]** — Grid layout is flexible and aligns with existing UI patterns.

---

### AD-8: Routing
**Binds:** The comparison page is accessible at `/compare` and can be linked from anywhere in the storefront.  
**Prevents:** Deep-linking and navigation issues.  
**Rule:** Add a new route `/compare` to the Next.js `pages` directory (or `app` router equivalent): `pages/compare.tsx`. The route renders `ComparisonPage`, which is a client component (`use client`) that uses `useComparison()` and product queries. The route is public (no auth required).

**[ADOPTED]** — Standard Next.js routing pattern.

---

### AD-9: Clear All Operation
**Binds:** Customers can remove all products from the comparison list in one action.  
**Prevents:** Tedious individual removals when resetting the comparison.  
**Rule:** The `clear` operation in `ComparisonContext` sets `productIds` to `[]` and updates localStorage. The comparison page renders a "Clear All" button that calls `clear`. After clearing, the page shows the empty state.

**[ADOPTED]** — Per AC6 and AC7.

---

## Component Hierarchy

```
App Root (or Storefront Layout)
  └─ ComparisonProvider (wraps entire app)
       ├─ ProductCard (PLP, displays many)
       │    └─ CompareButton (uses useComparison)
       ├─ ProductDetailPage (PDP, displays one)
       │    └─ CompareButton (uses useComparison)
       └─ ComparisonPage (at /compare)
            ├─ ComparisonList (renders columns)
            │    └─ ComparisonItem (one per product)
            │         ├─ Remove Button (calls remove(productId))
            │         └─ Product Details (image, title, price, variants, availability)
            └─ Clear All Button (calls clear())
```

---

## Data Flow

1. **Adding a product:**
   - User clicks `CompareButton` on a product card or PDP.
   - `CompareButton` calls `add(productId)` from `useComparison()`.
   - `ComparisonContext` checks duplicate and limit rules (AD-3, AD-4).
   - If valid, `productIds` is updated, and localStorage is synced (AD-2).
   - Button state updates to reflect "selected".

2. **Navigating to comparison page:**
   - User navigates to `/compare`.
   - `ComparisonPage` reads `productIds` from `useComparison()`.
   - Page fetches product details for all IDs (AD-6).
   - `ComparisonList` renders columns (AD-7).

3. **Removing a product:**
   - User clicks "Remove" on a `ComparisonItem`.
   - `remove(productId)` is called.
   - `ComparisonContext` updates `productIds` and syncs to localStorage.
   - Page re-renders without that product.

4. **Clearing all:**
   - User clicks "Clear All" on `ComparisonPage`.
   - `clear()` is called.
   - `productIds` becomes `[]`, localStorage is cleared.
   - Page shows empty state.

---

## LocalStorage Schema

```json
{
  "medusa_comparison": {
    "comparison_products": ["prod_123", "prod_456", "prod_789"]
  }
}
```

**Key:** `medusa_comparison`  
**Field:** `comparison_products` (array of product IDs)  
**Max length:** 4 items (enforced by AD-3)

---

## Integration Points

- **Product Cards (PLP):** Existing product card component (verify path in `repo-outline/medusa-appli`) adds `<CompareButton productId={product.id} />` as a sibling to "Add to Cart".
- **Product Detail Page (PDP):** Existing PDP component (verify path) adds `<CompareButton productId={product.id} />` near the main CTA section.
- **Product Data Fetching:** Uses existing product query hook (e.g. `useProducts` from `@medusajs/js-sdk` or storefront data layer) — verify exact hook name in existing code.
- **Routing:** Next.js `pages/compare.tsx` (or `app/compare/page.tsx` if using App Router) is a new file.

---

## Browser Compatibility

- **localStorage:** Supported in all modern browsers. Gracefully degrades if unavailable: comparison state works in-session but does not persist.
- **React Context:** Standard React feature, no polyfill needed.

---

## Performance Considerations

- **LocalStorage writes:** Occur on every `add`/`remove`/`clear` operation. This is acceptable because localStorage writes are synchronous and fast for small payloads (max 4 IDs).
- **Product fetching on comparison page:** Fetches up to 4 products. If using a batch query, this is a single API call. If individual queries, consider batching.

---

## Error Handling

- **Deleted/unavailable products:** Filtered out silently on the comparison page. The comparison list is not automatically cleaned up — this is deferred to a future enhancement. Users can manually remove items.
- **LocalStorage unavailable:** Comparison state works in-session but does not persist. No error shown to user.

---

## Deferred Decisions

- **Comparison list access from other pages:** Should there be a persistent "Compare" button in the header or footer that shows the count of selected products and links to `/compare`? **Deferred** — not required by AC, but a likely enhancement.
- **Variant selection on comparison page:** AC5 says "show variants", but does not specify whether users can switch which variant is compared. **Deferred** — initially display the default variant; variant switching is a future enhancement.
- **Handling unavailable products:** Should the comparison list automatically remove products that are no longer available (deleted, out of stock)? **Deferred** — initially, they remain in the list; the comparison page shows a "not available" message in that column.
- **Comparison page URL with product IDs:** Should `/compare` accept product IDs as query params (e.g. `/compare?ids=prod_123,prod_456`) to support deep-linking and sharing? **Deferred** — not required by AC; can be added later.

---

## Risks

1. **LocalStorage size limit:** Most browsers allow 5–10MB per origin. A list of 4 product IDs is negligible (~100 bytes). Risk: **low**.
2. **Concurrency across tabs:** If a user opens multiple tabs and modifies the comparison list in both, localStorage updates may clobber each other. Mitigation: Use `storage` event listener to sync state across tabs (deferred enhancement). Risk: **low** (rare use case).
3. **Product data staleness:** If a product's price or availability changes after being added to the comparison list, the comparison page may show outdated data. Mitigation: Fetches are made when the comparison page loads, so data is fresh at that moment. Risk: **low**.

---

## Open Questions

1. What is the exact path and name of the product card component in the storefront codebase?
2. What is the exact path and name of the product detail page component?
3. What is the exact name of the product query hook used in the storefront (e.g. `useProducts`, `useProductDetail`, etc.)?
4. Should the comparison page handle variant selection (allow users to switch which variant is compared), or is the default variant sufficient for now?

---

## Verification

- **AC1:** `CompareButton` is added to product cards and PDP, visible and functional.
- **AC2:** Attempting to add a 5th product is blocked; a notification is shown.
- **AC3:** Clicking "Compare" on an already-selected product does nothing (or toggles it off).
- **AC4–AC5:** Comparison page displays selected products with all required fields.
- **AC6:** "Remove" button on each product column removes that product.
- **AC7:** "Clear All" button removes all products; page shows empty state.
- **AC8:** Closing and reopening the browser retains the comparison list.

---

**End of HLD**
