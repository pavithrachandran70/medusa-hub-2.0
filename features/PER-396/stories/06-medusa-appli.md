# Story 06-medusa-appli: Comparison Page Routing — LLD

**Repo:** `medusa-appli`  
**Type:** LLD  
**Status:** draft

---

## Acceptance Criteria

**From requirements/breakdown only:**

1. Navigating to `/compare` renders the comparison page UI (enables AC4–AC6).

---

## File Plan

### Critical Issue: Storefront Location Unknown

**Investigation required:** Based on the repo analysis, the Medusa application is a monorepo. The admin dashboard under `packages/admin/dashboard` uses a custom `DashboardApp` routing system with virtual modules, but the breakdown references a `www/` directory pattern that **does not exist** in the provided `repo-tree/medusa-appli`.

**Analysis of existing code:**
- `packages/admin/dashboard/src/app.tsx` shows a custom routing system using virtual modules (`virtual:medusa/routes`)
- The repo tree does not show a `www/` directory at the root level
- The breakdown assumes a Next.js storefront structure under `www/`
- No storefront application is visible in the monorepo structure

**Critical verification needed:**
1. Confirm if a storefront exists in a separate repository
2. Verify if the storefront should be created in this monorepo
3. Determine the actual routing pattern used in the storefront (if it exists)

**Conclusion:** This story **cannot be implemented as written** because the target files do not exist in this repository. The following sections provide conditional implementations based on potential storefront locations, but all require clarification before proceeding.

---

### File: Determine routing pattern script

**Purpose:** Helper script to determine the routing pattern before creating the route file.

**Action:** This is a manual verification step, not an actual file. During implementation:

1. **Check for pages router:**
   ```bash
   # From project root
   find www -name "_app.tsx" -o -name "_app.jsx" -o -name "_app.js" 2>/dev/null | head -1
   ```
   - If found: Pages router is used
   - Expected path: `www/src/pages/_app.tsx` or similar

2. **Check for app router:**
   ```bash
   # From project root
   find www -name "layout.tsx" -o -name "layout.jsx" -o -name "layout.js" 2>/dev/null | grep -E "app/layout" | head -1
   ```
   - If found: App router is used
   - Expected path: `www/app/layout.tsx` or similar

3. **If neither is found:**
   - The storefront may not exist in this monorepo
   - The storefront may use a different directory name
   - The storefront may use a different framework

**Result determines which of the following sections to implement.**

---

## Scenario A: Pages Router (`www/src/pages/_app.tsx` exists)

### File: `www/src/pages/compare.tsx`

**Purpose:** Route page component for `/compare` using Next.js pages router.

**New file** — create as:

```tsx
import { ComparisonPage } from "../components/comparison"

/**
 * Product comparison route.
 * Public route - no authentication required.
 */
export default function ComparePage() {
  return <ComparisonPage />
}
```

**Key implementation details:**

- **Default export:** Pages router requires a default exported React component
- **Component location:** Imports `ComparisonPage` from `../components/comparison` (adjust relative path based on actual structure)
- **No props:** The page component receives Next.js router props implicitly, but `ComparisonPage` uses `useComparison()` hook internally and needs no props
- **Public route:** No authentication wrapper or `getServerSideProps` needed
- **Static generation:** This page can be statically generated as it has no dynamic segments

**File path verification:**
- If `www/src/pages/_app.tsx` exists, create this file at `www/src/pages/compare.tsx`
- If pages are under a different structure (e.g., `www/pages/compare.tsx`), adjust accordingly

---

## Scenario B: App Router (`www/app/layout.tsx` exists)

### File: `www/app/compare/page.tsx`

**Purpose:** Route page component for `/compare` using Next.js app router.

**New file** — create as:

```tsx
import { ComparisonPage } from "../../src/components/comparison"

/**
 * Product comparison route.
 * Public route - no authentication required.
 */
export default function ComparePage() {
  return <ComparisonPage />
}
```

**Key implementation details:**

- **Default export:** App router requires a default exported React component
- **Component location:** Imports `ComparisonPage` from `../../src/components/comparison` (adjust relative path based on actual structure)
  - Assumes components live under `www/src/components/`
  - If components are under `www/app/components/`, adjust to `../components/comparison`
- **No props:** App router pages receive no special props; `ComparisonPage` uses `useComparison()` hook internally
- **Public route:** No authentication check via middleware or layout needed
- **Client component:** If `ComparisonPage` uses client-side hooks, ensure it has `"use client"` directive (this was added in Story 4)

**File path verification:**
- If `www/app/layout.tsx` exists, create this file at `www/app/compare/page.tsx`
- The directory `www/app/compare/` will be created if it doesn't exist

---

## Scenario C: Custom Routing (neither pattern found)

**If neither Next.js routing pattern is found, the following applies:**

### Investigation steps

1. **Check for alternative storefront location:**
   ```bash
   find . -type f -name "package.json" | xargs grep -l "next\|react\|vite" | grep -v node_modules
   ```
   - Look for storefront-related packages outside `www/`

2. **Check Medusa storefront structure:**
   - Medusa may use a custom storefront starter
   - Check for `storefront/`, `store/`, `frontend/` directories

3. **Check for custom routing:**
   - Some Medusa storefronts use custom routing solutions
   - Check for `routes.ts`, `router.ts`, or similar configuration files

**If a custom routing system is found:**
- Follow the existing routing conventions in that system
- Create a route configuration that maps `/compare` to `ComparisonPage`
- Ensure the route is public (no auth guards)

**If no storefront exists in this monorepo:**
- The comparison feature may need to be implemented in a separate storefront repository
- Document the route requirements for implementation in the external storefront:
  - **Route:** `/compare`
  - **Component:** `ComparisonPage` (import from Story 4)
  - **Props:** None
  - **Auth:** Public
  - **Layout:** Standard storefront layout

---

## Interfaces

**No new interfaces are introduced.** This story only creates a route file that renders an existing component (`ComparisonPage` from Story 4).

---

## Algorithm Notes

### Route rendering logic

1. **On navigation to `/compare`:**
   - Next.js (or the routing system) matches the route
   - The route component is rendered
   - The route component returns `<ComparisonPage />`

2. **`ComparisonPage` behavior (from Story 4):**
   - Calls `useComparison()` to get `productIds`, `remove`, `clear`
   - Fetches product data for selected product IDs
   - Renders comparison UI with product columns
   - Handles empty state, loading state, and errors

**No additional logic is needed in the route file.** The route is a simple pass-through to the `ComparisonPage` component.

---

## Tests

**Location:** `www/src/components/comparison/__tests__/compare-route.spec.tsx` (or similar path based on storefront structure)

**1 integration test (as specified in Verification Approach):**

### Test 1: AC1 — Navigating to `/compare` renders ComparisonPage

**Test approach depends on routing pattern:**

#### For Pages Router:

```typescript
import { render, screen } from "@testing-library/react"
import { useRouter } from "next/router"
import ComparePage from "../../pages/compare" // Adjust path
import { ComparisonProvider } from "../../../lib/context/comparison-context"

jest.mock("next/router", () => ({
  useRouter: jest.fn(),
}))

jest.mock("@medusajs/js-sdk", () => ({
  useProducts: jest.fn(),
}))

import { useProducts } from "@medusajs/js-sdk" // Verify import path

describe("Compare route (pages router)", () => {
  const wrapper = ({ children }) => (
    <ComparisonProvider>{children}</ComparisonProvider>
  )

  beforeEach(() => {
    jest.clearAllMocks()
    ;(useRouter as jest.Mock).mockReturnValue({
      query: {},
      pathname: "/compare",
      push: jest.fn(),
    })
    ;(useProducts as jest.Mock).mockReturnValue({
      data: { products: [] },
      isLoading: false,
      error: null,
    })
  })

  it("renders ComparisonPage component", () => {
    render(<ComparePage />, { wrapper })

    // Verify ComparisonPage elements are present
    // Check for empty state message (since no products are selected)
    expect(screen.getByText("No products to compare")).toBeInTheDocument()
    expect(
      screen.getByText(/Add products from the product pages to start comparing/)
    ).toBeInTheDocument()
  })

  it("renders comparison page title", () => {
    // Mock some products in the comparison list
    const mockUseComparison = jest.fn(() => ({
      productIds: ["prod_1", "prod_2"],
      remove: jest.fn(),
      clear: jest.fn(),
    }))

    jest
      .spyOn(require("../../../lib/context/comparison-context"), "useComparison")
      .mockReturnValue(mockUseComparison())

    const mockProducts = [
      { id: "prod_1", title: "Product 1", thumbnail: "url1", variants: [] },
      { id: "prod_2", title: "Product 2", thumbnail: "url2", variants: [] },
    ]

    ;(useProducts as jest.Mock).mockReturnValue({
      data: { products: mockProducts },
      isLoading: false,
      error: null,
    })

    render(<ComparePage />, { wrapper })

    // Verify comparison page elements
    expect(screen.getByText("Compare Products")).toBeInTheDocument()
    expect(screen.getByRole("button", { name: /Clear all/i })).toBeInTheDocument()
  })
})
```

#### For App Router:

```typescript
import { render, screen } from "@testing-library/react"
import ComparePage from "../../app/compare/page" // Adjust path
import { ComparisonProvider } from "../../../lib/context/comparison-context"

jest.mock("@medusajs/js-sdk", () => ({
  useProducts: jest.fn(),
}))

import { useProducts } from "@medusajs/js-sdk" // Verify import path

describe("Compare route (app router)", () => {
  const wrapper = ({ children }) => (
    <ComparisonProvider>{children}</ComparisonProvider>
  )

  beforeEach(() => {
    jest.clearAllMocks()
    ;(useProducts as jest.Mock).mockReturnValue({
      data: { products: [] },
      isLoading: false,
      error: null,
    })
  })

  it("renders ComparisonPage component", () => {
    render(<ComparePage />, { wrapper })

    // Verify ComparisonPage elements are present
    // Check for empty state message (since no products are selected)
    expect(screen.getByText("No products to compare")).toBeInTheDocument()
    expect(
      screen.getByText(/Add products from the product pages to start comparing/)
    ).toBeInTheDocument()
  })

  it("renders comparison page title", () => {
    // Mock some products in the comparison list
    const mockUseComparison = jest.fn(() => ({
      productIds: ["prod_1", "prod_2"],
      remove: jest.fn(),
      clear: jest.fn(),
    }))

    jest
      .spyOn(require("../../../lib/context/comparison-context"), "useComparison")
      .mockReturnValue(mockUseComparison())

    const mockProducts = [
      { id: "prod_1", title: "Product 1", thumbnail: "url1", variants: [] },
      { id: "prod_2", title: "Product 2", thumbnail: "url2", variants: [] },
    ]

    ;(useProducts as jest.Mock).mockReturnValue({
      data: { products: mockProducts },
      isLoading: false,
      error: null,
    })

    render(<ComparePage />, { wrapper })

    // Verify comparison page elements
    expect(screen.getByText("Compare Products")).toBeInTheDocument()
    expect(screen.getByRole("button", { name: /Clear all/i })).toBeInTheDocument()
  })
})
```

**Verifies:** AC1 (Navigating to `/compare` renders the comparison page UI).

**Test coverage:**
- Route renders without crashing
- `ComparisonPage` component is rendered (verified by checking for its unique elements)
- Both empty state and populated state are tested

---

## Implementation Notes

### Critical Blockers (Must Resolve Before Implementation)

1. **Identify the target repository:**
   - This monorepo (`medusa-appli`) does **not** contain a storefront application
   - The `www/` directory referenced in the breakdown does not exist
   - Confirm whether:
     - The storefront is in a separate repository
     - The storefront should be created in this monorepo
     - The breakdown has an error and the target is different

2. **Verify the routing pattern:**
   - Once the storefront repository is identified, check for Next.js routing pattern
   - Pages router: `pages/_app.tsx`
   - App router: `app/layout.tsx`
   - Custom routing: Different approach needed

3. **Verify Story 4 completion:**
   - The `ComparisonPage` component must exist from Story 4
   - The file `[storefront-root]/src/components/comparison/comparison-page.tsx` (or equivalent) must be present
   - The component must export `ComparisonPage`

4. **Check for existing routes:**
   - If the storefront uses a custom routing solution, determine how routes are registered
   - Follow the existing pattern for creating new routes

### Post-Implementation Verification

After creating the route file, manually verify:

1. **Development server:**
   ```bash
   # Navigate to storefront directory
   cd [storefront-root]
   
   # Start development server
   npm run dev # or yarn dev
   ```

2. **Test navigation:**
   - Open browser to storefront URL (e.g., `http://localhost:3000`)
   - Navigate to `/compare`
   - Verify the comparison page renders without errors

3. **Empty state:**
   - With no products selected, the page should show "No products to compare"
   - No JavaScript errors in console

4. **With products:**
   - Add products to comparison (via Story 2 or Story 3 implementation)
   - Navigate to `/compare`
   - Verify products are displayed in columns

### Integration with Other Stories

**Depends on:**
- **Story 4:** `ComparisonPage` component must exist and be importable

**Enables:**
- **Story 2 & 3:** `CompareButton` can link to `/compare` (via `<Link>` or `router.push`)
- **Future enhancement:** Deep-linking with product IDs in query params (e.g., `/compare?ids=prod_1,prod_2`)

**Note:** This story does not modify any existing files. It only creates a new route file that renders an existing component. The route is immediately functional once the file is created, assuming Story 4 is complete.

---

## Alternative Approaches (Not Recommended)

### Alternative 1: Implement in Admin Dashboard

Instead of creating a storefront route, implement the comparison feature in the admin dashboard under `packages/admin/dashboard`.

**Problems:**
- The admin dashboard is for store administration, not customer-facing features
- The user stories describe customer behavior ("customers can compare products"), not admin functionality
- This would be a fundamental scope change

**Verdict:** Not viable for this feature.

### Alternative 2: Create New Storefront Package

Create a new storefront package under `packages/storefront/` or `www/`.

**Problems:**
- This would be far beyond the scope of "creating a route"
- Would require creating an entire Next.js application structure
- Not described in any of the story requirements

**Verdict:** Not viable without explicit product sign-off.

---

## Troubleshooting Guide

### Error: Module not found: Can't resolve 'ComparisonPage'

**Cause:** The import path to `ComparisonPage` is incorrect.

**Fix:**
1. Verify the actual location of `ComparisonPage` from Story 4
2. Adjust the import statement to use the correct relative path
3. If `ComparisonPage` is not at `src/components/comparison/comparison-page.tsx`, update the import

### Error: Next.js 404 on `/compare`

**Cause:** The route file is not in the correct location for the routing pattern used.

**Fix:**
1. Verify the routing pattern (pages vs app router)
2. Ensure the file is in the correct location:
   - Pages router: `pages/compare.tsx`
   - App router: `app/compare/page.tsx`
3. Restart the development server

### Error: ComparisonProvider not found

**Cause:** The comparison context from Story 1 is not set up correctly, or Story 7 (wrapping the app root) has not been completed.

**Fix:**
1. Verify Story 1 is complete (context exists)
2. Verify Story 7 is complete (app root is wrapped with provider)
3. Check that the provider import path is correct

---

**End of LLD**

**Status:** This story is **blocked** pending clarification of the target repository structure. The implementation cannot proceed until the actual storefront application location is confirmed.
