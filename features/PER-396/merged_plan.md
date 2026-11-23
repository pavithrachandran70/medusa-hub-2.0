# Merged plan — PER-396

Execution order (1 ticket(s)):

1. `1`

Specs (1): `specs/1.md`

---

## Spec `specs/1.md`

# specs/1.md — PER-396 Tech Stub

**Type:** feature  
**Status:** draft  
**Updated:** 2025-01-28  
**Covers:** PER-396 — Product Comparison Feature  
**Source:** Ticket 1

---

## Overview

The Product Comparison feature allows customers to select up to 4 products from product cards (PLP) or product detail pages (PDP) and view them side-by-side on a dedicated comparison page. The feature uses React Context for state management, localStorage for persistence, and integrates with existing product card and detail page components via a new `CompareButton` component.

---

## Stack

**Frontend Framework:** React (TypeScript)  
**Routing:** Next.js (pages or app router — verify in repo)  
**State Management:** React Context API  
**Persistence:** Browser localStorage  
**Styling:** CSS Grid + existing UI patterns (verify design system usage in repo)  
**Data Fetching:** Existing product query hooks (verify exact hook name in `repo-outline/medusa-appli`)

**Assumptions:**
- The storefront is part of the `medusa-appli` monorepo under `www/` or a similar directory.
- Existing product card and PDP components are already in place (paths to be verified).
- Product data fetching uses existing hooks from `@medusajs/js-sdk` or similar (verify in repo).

---

## Schema & Contracts

### LocalStorage Schema

**Key:** `medusa_comparison`

**Structure:**
```json
{
  "comparison_products": ["prod_123", "prod_456", "prod_789"]
}
```

**Field:** `comparison_products` (array of product IDs, max length 4)

**Sync:** Unidirectional — in-memory state writes to localStorage on every `add`, `remove`, or `clear` operation. On mount, the context reads from localStorage to hydrate state.

### Context API

**Provider:** `ComparisonProvider` (wraps app root or layout)

**Context Shape:**
```typescript
interface ComparisonContextValue {
  productIds: string[]
  add: (productId: string) => { success: boolean; reason?: "limit" | "duplicate" }
  remove: (productId: string) => void
  clear: () => void
}
```

**Hook:** `useComparison()` returns `ComparisonContextValue`

### Component Props

**CompareButton:**
```typescript
interface CompareButtonProps {
  productId: string
}
```

**ComparisonPage:** No props (reads from context)

**ComparisonItem:**
```typescript
interface ComparisonItemProps {
  product: ProductDTO // from existing product query
  onRemove: (productId: string) => void
}
```

---

## File Touch List

**New Files:**

1. **Context/State Management:**
   - `src/lib/context/comparison-context.tsx` — ComparisonProvider, useComparison hook, localStorage sync logic
   - `src/lib/context/comparison-context.types.ts` — TypeScript types for context

2. **Components:**
   - `src/components/comparison/compare-button.tsx` — CompareButton component (integrates into product cards and PDP)
   - `src/components/comparison/comparison-page.tsx` — Comparison page container
   - `src/components/comparison/comparison-list.tsx` — Renders columns of products
   - `src/components/comparison/comparison-item.tsx` — Single product column with details
   - `src/components/comparison/index.ts` — Barrel export

3. **Routing:**
   - `pages/compare.tsx` (or `app/compare/page.tsx` if using App Router) — Route for `/compare`

4. **Utilities:**
   - `src/lib/utils/comparison-storage.ts` — localStorage read/write helpers (optional abstraction)

**Modified Files:**

1. **App Root/Layout:**
   - `src/app.tsx` (or `pages/_app.tsx` or `app/layout.tsx`) — Wrap with `ComparisonProvider`

2. **Product Card:**
   - `src/components/product-card.tsx` (or similar — **verify path in repo**) — Add `<CompareButton productId={product.id} />`

3. **Product Detail Page:**
   - `src/pages/product/[id].tsx` (or similar — **verify path in repo**) — Add `<CompareButton productId={product.id} />`

4. **Navigation (optional):**
   - Main navigation component (if adding a "Compare" link in header/footer) — **path to be verified**

**Note:** Exact file paths depend on the storefront's structure. The `repo-tree/medusa-appli` listing shows the monorepo structure but does not include storefront source files. Assume a typical Next.js structure under `www/` or a similar directory. During implementation, verify:
- Product card component path
- Product detail page path
- Existing product query hook name (e.g., `useProducts`, `useProductDetail`)
- App root or layout file path
- Routing pattern (pages vs. app router)

---

## Risks

### Technical Risks

1. **LocalStorage Unavailable:**
   - **Impact:** Comparison list does not persist across sessions.
   - **Mitigation:** Feature degrades gracefully — comparison works in-session, but selections are lost on page refresh. No error shown to user.
   - **Likelihood:** Low (localStorage is universally available in modern browsers).

2. **Stale Product Data:**
   - **Impact:** If a product's price or availability changes after being added to the comparison list, the comparison page may show outdated data.
   - **Mitigation:** Fetch product data fresh when the comparison page loads. The comparison list stores only IDs, not full product objects.
   - **Likelihood:** Low (data is fresh at page load; minor risk if user leaves comparison page open for a long time).

3. **Concurrency Across Tabs:**
   - **Impact:** If a user opens multiple tabs and modifies the comparison list in both, localStorage updates may clobber each other.
   - **Mitigation:** Use `storage` event listener to sync state across tabs (deferred enhancement). Current implementation accepts this risk.
   - **Likelihood:** Low (rare use case).

4. **Missing Product Data:**
   - **Impact:** A product in the comparison list may no longer exist (deleted, out of stock, etc.). The comparison page fetch could fail or return partial data.
   - **Mitigation:** Filter out missing products silently on the comparison page. The comparison list is not automatically cleaned up — users can manually remove items.
   - **Likelihood:** Low (edge case).

### Integration Risks

1. **Unknown Product Card/PDP Component Paths:**
   - **Impact:** Cannot integrate `CompareButton` without knowing where these components live.
   - **Mitigation:** Verify file paths in `repo-outline/medusa-appli` and existing storefront code before implementation.
   - **Likelihood:** Medium (paths are not visible in provided repo context).

2. **Unknown Product Query Hook:**
   - **Impact:** Cannot fetch product data on the comparison page without knowing the existing query hook name.
   - **Mitigation:** Search `repo-outline/medusa-appli` for `useProducts`, `useProductDetail`, or similar hooks. Check `packages/core/js-sdk` or storefront data layer.
   - **Likelihood:** Medium (hook name is not visible in provided repo context).

3. **Unknown Routing Pattern:**
   - **Impact:** Cannot create the `/compare` route without knowing if the app uses Next.js pages or app router.
   - **Mitigation:** Check `repo-tree/medusa-appli` for `pages/` or `app/` directory structure.
   - **Likelihood:** Low (Next.js convention is predictable).

### UX Risks

1. **4-Product Limit Not Clear:**
   - **Impact:** Users may not understand why they cannot add a 5th product.
   - **Mitigation:** Show a notification (e.g., toast) when the limit is reached. The `CompareButton` becomes disabled when the limit is hit.
   - **Likelihood:** Medium (user confusion is possible).

2. **No Persistent "Compare" Link:**
   - **Impact:** Users may not know how to navigate to the comparison page after adding products.
   - **Mitigation:** Deferred enhancement — add a persistent "Compare" button in header/footer with a count badge. For now, users can navigate via a direct link or by typing `/compare`.
   - **Likelihood:** High (poor discoverability in initial implementation).

---

## Open Questions

1. **What is the exact path and name of the product card component in the storefront codebase?**  
   → Required to integrate `CompareButton`.

2. **What is the exact path and name of the product detail page component?**  
   → Required to integrate `CompareButton`.

3. **What is the exact name of the product query hook used in the storefront (e.g., `useProducts`, `useProductDetail`, etc.)?**  
   → Required to fetch product data on the comparison page.

4. **Is the storefront using Next.js pages or app router?**  
   → Determines where to create the `/compare` route (`pages/compare.tsx` vs `app/compare/page.tsx`).

5. **Should the comparison page handle variant selection (allow users to switch which variant is compared), or is the default variant sufficient?**  
   → AC5 says "show variants", but does not specify if users can switch. Deferred to future enhancement unless clarified.

6. **Should there be a persistent "Compare" button in the header or footer to access `/compare`?**  
   → Not required by AC, but improves UX. Deferred to future enhancement.

7. **What happens when a product in the comparison list is no longer available (deleted, out of stock)?**  
   → Should it remain in the list with a visual indicator, or be removed automatically? Currently deferred — it remains in the list; the comparison page shows a "not available" message.

---

**End of Tech Stub**

---

# Story summaries

- `01-medusa-appli`: Story 01-medusa-appli: Comparison State Management and Context Setup — LLD
- `02-medusa-appli`: Story 02-medusa-appli: CompareButton Integration in Product Cards — LLD
- `03-medusa-appli`: Story 03-medusa-appli: CompareButton Integration in Product Detail Pages — LLD
- `04-medusa-appli`: Story 04-medusa-appli: Comparison Page UI and Product Display — LLD
- `05-medusa-appli`: Story 05-medusa-appli: LocalStorage Persistence and Sync — LLD
- `06-medusa-appli`: Story 06-medusa-appli: Comparison Page Routing — LLD
- `07-medusa-appli`: Story 07-medusa-appli: Wrap App Root with ComparisonProvider — LLD

---

_Single-ticket pack: order is trivial (no multi-split)._
