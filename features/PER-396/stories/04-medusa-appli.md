# Story 04-medusa-appli: Comparison Page UI and Product Display — LLD

**Repo:** `medusa-appli`  
**Type:** LLD  
**Status:** draft

---

## Acceptance Criteria

**From requirements/breakdown only:**

1. Selected products are displayed on the comparison page in columns, showing product image, title, price, variants, and availability (AC4).
2. Customers can remove individual products from the comparison page (AC5).
3. Customers can clear all products via a "Clear All" button (AC6).

---

## File Plan

### Determine Project Structure and Product Query Hook

**Investigation required:** The `repo-tree/medusa-appli` listing does not show a `www/` directory structure. Based on the repo analysis, the Medusa application is a monorepo with multiple packages. The storefront appears to be under `packages/admin/dashboard` based on the code shown in `repo-file/medusa-appli/packages/admin/dashboard/src/app.tsx`.

However, the breakdown references a `www/` directory pattern. We need to verify:
1. The actual location of the storefront codebase
2. The product query hook name and import path

**Likely patterns to search for:**
- Product query hooks: `useProduct`, `useProducts`, `useProductsQuery` from `@medusajs/js-sdk` or Medusa client
- Component directory: `src/components/` or `app/components/`
- Context directory: `src/lib/context/` or `src/contexts/`

**Assumption:** Based on the HLD and standard Medusa patterns, we assume:
- Product data fetching uses hooks from `@medusajs/js-sdk` or similar
- Components live under a `src/components/` structure
- The comparison context from Story 1 is available

**Note:** The exact file paths and import statements must be verified against the actual repository structure during implementation.

---

### File: `src/components/comparison/comparison-page.tsx`

**Purpose:** Main comparison page container that fetches product data and orchestrates the UI.

**New file** — create as:

```typescript
"use client"

import React from "react"
import { useComparison } from "../../lib/context/comparison-context"
// TODO: Verify exact import path in repo - likely from @medusajs/js-sdk or similar
// import { useProducts } from "@medusajs/js-sdk"
import { ComparisonList } from "./comparison-list"

/**
 * Comparison page container component.
 * Fetches product data for selected product IDs and renders the comparison UI.
 */
export const ComparisonPage: React.FC = () => {
  const { productIds, remove, clear } = useComparison()

  // TODO: Verify exact hook name and import path
  // Placeholder for product query - actual implementation depends on repo structure
  // const { data: productsData, isLoading, error } = useProducts({
  //   id: productIds,
  // })

  // Temporary mock for structure - replace with actual query
  const isLoading = false
  const error = null
  const productsData = { products: [] }

  // Filter out missing products (deleted/unavailable)
  const products = React.useMemo(() => {
    if (!productsData?.products) return []
    return productsData.products.filter(Boolean)
  }, [productsData])

  // Loading state
  if (isLoading) {
    return (
      <div className="flex items-center justify-center min-h-screen">
        <div className="text-center">
          <p className="text-lg">Loading comparison...</p>
        </div>
      </div>
    )
  }

  // Error state
  if (error) {
    return (
      <div className="flex items-center justify-center min-h-screen">
        <div className="text-center">
          <p className="text-lg text-red-600">
            Failed to load products. Please try again.
          </p>
        </div>
      </div>
    )
  }

  // Empty state
  if (products.length === 0) {
    return (
      <div className="flex items-center justify-center min-h-screen">
        <div className="text-center">
          <h2 className="text-2xl font-semibold mb-4">
            No products to compare
          </h2>
          <p className="text-gray-600 mb-6">
            Add products from the product pages to start comparing.
          </p>
        </div>
      </div>
    )
  }

  return (
    <div className="container mx-auto px-4 py-8">
      <div className="mb-6 flex items-center justify-between">
        <h1 className="text-3xl font-bold">Compare Products</h1>
        <button
          onClick={clear}
          className="px-4 py-2 bg-red-600 text-white rounded hover:bg-red-700 transition-colors"
          aria-label="Clear all products from comparison"
        >
          Clear All
        </button>
      </div>

      <ComparisonList products={products} onRemove={remove} />
    </div>
  )
}
```

**Key implementation details:**

- **Client component:** Marked with `"use client"` directive at top (per Implementation Pointers).
- **State management:** Uses `useComparison()` hook from Story 1 to get `productIds`, `remove`, and `clear`.
- **Product fetching:** TODO placeholder for actual product query hook — must be verified in repo:
  - Check `packages/core/js-sdk/` for available hooks
  - Check existing product components for usage patterns
  - Common patterns: `useProducts({ id: [...] })` or `useProduct(id)` called multiple times
- **Product filtering:** `useMemo` filters out falsy values (missing products) silently (AC4 extended).
- **Loading state:** Displays centered loading message while fetching.
- **Error state:** Displays error message with retry suggestion.
- **Empty state:** Displays when no products are in the comparison list (after filtering).
- **Clear All button:** Calls `clear()` from context (AC6).
- **Accessibility:** Button has `aria-label` describing the action.

**Verification points:**
- The product query hook must match the actual API in the repo
- CSS classes assume Tailwind CSS (verify styling approach in repo)
- Product data structure must match actual `ProductDTO` type

---

### File: `src/components/comparison/comparison-list.tsx`

**Purpose:** Renders a responsive grid of product comparison columns.

**New file** — create as:

```typescript
import React from "react"
import { ComparisonItem } from "./comparison-item"

interface ComparisonListProps {
  products: ProductDTO[]
  onRemove: (productId: string) => void
}

/**
 * Renders a responsive grid of product comparison columns.
 * Uses CSS Grid to auto-wrap columns on smaller screens.
 */
export const ComparisonList: React.FC<ComparisonListProps> = ({
  products,
  onRemove,
}) => {
  return (
    <div
      className="grid gap-6"
      style={{
        gridTemplateColumns: "repeat(auto-fit, minmax(200px, 1fr))",
      }}
    >
      {products.map((product) => (
        <ComparisonItem
          key={product.id}
          product={product}
          onRemove={onRemove}
        />
      ))}
    </div>
  )
}
```

**Key implementation details:**

- **Props:** Accepts `products` array and `onRemove` callback (per Implementation Pointers).
- **Grid layout:** Uses inline `style` for CSS Grid with `repeat(auto-fit, minmax(200px, 1fr))` (per AD-7 from HLD).
  - `auto-fit`: Grid automatically wraps columns as screen size decreases.
  - `minmax(200px, 1fr)`: Each column is at least 200px wide, expands equally up to available space.
  - On mobile (<768px): Shows 1–2 columns (auto-wraps).
  - On desktop: Shows all selected products in one row (up to 4).
- **Gap:** `gap-6` (Tailwind class) provides spacing between columns.
- **Key prop:** Each `ComparisonItem` has a stable `key` (product ID) for React reconciliation.

**Note:** The `ProductDTO` type is assumed to be available from Medusa's type definitions. Verify exact import path.

---

### File: `src/components/comparison/comparison-item.tsx`

**Purpose:** Displays a single product's details in the comparison grid.

**New file** — create as:

```typescript
import React from "react"

interface ComparisonItemProps {
  product: ProductDTO
  onRemove: (productId: string) => void
}

/**
 * Single product column in the comparison view.
 * Displays product image, title, price, variants, and availability.
 */
export const ComparisonItem: React.FC<ComparisonItemProps> = ({
  product,
  onRemove,
}) => {
  // Extract product details
  const image = product.thumbnail || product.images?.[0]?.url || ""
  const title = product.title || "Untitled Product"
  const price = formatPrice(product) // TODO: Implement price formatting
  const variants = product.variants || []
  const availability = getAvailability(product) // TODO: Implement availability logic

  return (
    <div className="border border-gray-300 rounded-lg p-4 flex flex-col h-full">
      {/* Product Image */}
      <div className="mb-4">
        {image ? (
          <img
            src={image}
            alt={title}
            className="w-full h-48 object-cover rounded"
          />
        ) : (
          <div className="w-full h-48 bg-gray-200 rounded flex items-center justify-center">
            <span className="text-gray-500">No image</span>
          </div>
        )}
      </div>

      {/* Product Title */}
      <h3 className="text-lg font-semibold mb-2">{title}</h3>

      {/* Product Price */}
      <p className="text-xl font-bold text-gray-900 mb-4">{price}</p>

      {/* Variants */}
      {variants.length > 0 && (
        <div className="mb-4">
          <h4 className="text-sm font-medium text-gray-700 mb-2">Variants:</h4>
          <ul className="text-sm text-gray-600 space-y-1">
            {variants.map((variant) => (
              <li key={variant.id}>
                {variant.title || `Variant ${variant.id}`}
              </li>
            ))}
          </ul>
        </div>
      )}

      {/* Availability */}
      <div className="mb-4">
        <span
          className={`inline-block px-3 py-1 rounded text-sm font-medium ${
            availability.inStock
              ? "bg-green-100 text-green-800"
              : "bg-red-100 text-red-800"
          }`}
        >
          {availability.label}
        </span>
      </div>

      {/* Remove Button */}
      <button
        onClick={() => onRemove(product.id)}
        className="mt-auto px-4 py-2 bg-gray-200 text-gray-800 rounded hover:bg-gray-300 transition-colors"
        aria-label={`Remove ${title} from comparison`}
      >
        Remove
      </button>
    </div>
  )
}

/**
 * Format product price for display.
 * TODO: Implement actual price formatting logic based on product data structure.
 */
function formatPrice(product: ProductDTO): string {
  // Placeholder - actual implementation depends on product price structure
  // Medusa products may have multiple prices (variants, regions, etc.)
  // Need to determine which price to display (e.g., default variant, lowest price)
  return "$0.00"
}

/**
 * Determine product availability.
 * TODO: Implement actual availability logic based on product/variant inventory.
 */
function getAvailability(product: ProductDTO): {
  inStock: boolean
  label: string
} {
  // Placeholder - actual implementation depends on inventory structure
  // May need to check variant inventory levels
  return {
    inStock: true,
    label: "In Stock",
  }
}
```

**Key implementation details:**

- **Props:** Accepts `product` (ProductDTO) and `onRemove` callback (per Implementation Pointers).
- **Layout:** Flexbox column (`flex flex-col`) with `h-full` to ensure all cards have equal height.
- **Product Image:**
  - Displays `thumbnail` or first image URL.
  - Shows placeholder if no image exists.
  - Fixed height (`h-48`) with `object-cover` to prevent distortion.
- **Product Title:** `text-lg font-semibold` for prominence.
- **Product Price:** `text-xl font-bold` for emphasis (AC4).
- **Variants:**
  - Conditionally rendered if `variants.length > 0`.
  - Displays as a simple list (AC4 — "show variants").
  - **No variant switching logic** (per deferred decision in HLD).
- **Availability:**
  - Badge-style display with color coding (green for in stock, red for out of stock).
  - Actual logic TODO (depends on inventory data structure).
- **Remove Button:**
  - Positioned at bottom with `mt-auto` (pushes to bottom of flex container).
  - Calls `onRemove(product.id)` (AC5).
  - Accessible with `aria-label` including product title.

**TODO items:**
- **`formatPrice` function:** Must be implemented based on actual product price structure.
  - Medusa products have complex pricing (variants, regions, currencies).
  - Decide which price to display (e.g., default variant, lowest price, price for current region).
  - Format with currency symbol and locale.
- **`getAvailability` function:** Must be implemented based on inventory data.
  - Check product/variant inventory levels.
  - Consider inventory location (stock location, fulfillment availability).
  - Return appropriate label ("In Stock", "Out of Stock", "Limited", etc.).

**Verification points:**
- The `ProductDTO` type must include `thumbnail`, `images`, `title`, `variants`, `id` fields.
- Styling classes assume Tailwind CSS.
- Price and availability logic must match actual Medusa data structures.

---

### File: `src/components/comparison/index.ts`

**Purpose:** Barrel export for comparison components.

**Action:** Modify existing file or create if not present.

**Add exports:**

```typescript
export { ComparisonPage } from "./comparison-page"
export { ComparisonList } from "./comparison-list"
export { ComparisonItem } from "./comparison-item"
```

**If file does not exist:** Create as above (also includes `CompareButton` from Story 2 if not already present).

**If file exists:** Append the above exports to the existing file. Do not remove existing exports (e.g., `CompareButton` from Story 2).

**Full file content (if creating new):**

```typescript
export { CompareButton } from "./compare-button" // From Story 2
export { ComparisonPage } from "./comparison-page"
export { ComparisonList } from "./comparison-list"
export { ComparisonItem } from "./comparison-item"
```

---

## Interfaces

### `ComparisonListProps`

```typescript
interface ComparisonListProps {
  products: ProductDTO[]
  onRemove: (productId: string) => void
}
```

**Fields:**
- `products`: Array of product objects to display in the comparison grid.
- `onRemove`: Callback function to remove a product from the comparison list by ID.

---

### `ComparisonItemProps`

```typescript
interface ComparisonItemProps {
  product: ProductDTO
  onRemove: (productId: string) => void
}
```

**Fields:**
- `product`: The product object to display in this column.
- `onRemove`: Callback function to remove this product from the comparison list.

---

### `ProductDTO` (assumed from Medusa types)

**Note:** This type is expected to be available from Medusa's type definitions (e.g., `@medusajs/types` or `@medusajs/js-sdk`). Verify the exact import path.

**Expected structure (based on Medusa documentation):**

```typescript
interface ProductDTO {
  id: string
  title: string
  description?: string
  thumbnail?: string
  images?: Array<{ id: string; url: string }>
  variants?: Array<{
    id: string
    title: string
    prices?: Array<{ amount: number; currency_code: string }>
    inventory_quantity?: number
  }>
  // ... other fields
}
```

**Verification:** Check `packages/core/types/` or Medusa SDK documentation for the exact shape.

---

## Algorithm Notes

### Product Filtering in `ComparisonPage`

**Purpose:** Filter out missing products (deleted/unavailable) silently (AC4 extended).

**Logic:**
1. After fetching product data, check if `productsData?.products` exists.
2. Use `Array.filter(Boolean)` to remove falsy values (null, undefined).
3. This handles cases where:
   - A product ID in the comparison list no longer exists (deleted).
   - The API returns null/undefined for a missing product.

**Edge case:** If all products are filtered out, the empty state is shown.

---

### Price Formatting (`formatPrice`)

**Purpose:** Display product price in a user-friendly format.

**Current implementation:** Placeholder returning `"$0.00"`.

**Required logic (TODO):**
1. Determine which price to display:
   - **Option A:** Default variant price (if product has variants).
   - **Option B:** Lowest variant price (requires iterating variants).
   - **Option C:** Price for current region/currency (requires region context).
2. Extract price amount and currency code.
3. Format using `Intl.NumberFormat`:
   ```typescript
   const formatter = new Intl.NumberFormat("en-US", {
     style: "currency",
     currency: currencyCode,
   })
   return formatter.format(priceAmount / 100) // Medusa stores prices in cents
   ```
4. Handle missing price gracefully (return "Price not available").

**Verification:** Check Medusa product data structure for price fields.

---

### Availability Determination (`getAvailability`)

**Purpose:** Display product availability status.

**Current implementation:** Placeholder returning `{ inStock: true, label: "In Stock" }`.

**Required logic (TODO):**
1. Check product inventory:
   - **Option A:** Aggregate variant inventory quantities.
   - **Option B:** Check product-level availability flag (if exists).
2. Determine status:
   - `inStock: true` if any variant has inventory > 0.
   - `inStock: false` if all variants have inventory === 0.
3. Return appropriate label:
   - "In Stock" (inStock === true)
   - "Out of Stock" (inStock === false)
   - "Limited" (inventory < threshold, e.g., < 5)
4. Handle missing inventory data gracefully (default to "In Stock" or "Availability Unknown").

**Verification:** Check Medusa inventory data structure (may be nested under variants).

---

### Responsive Grid Layout

**CSS Grid configuration:**
- `grid-template-columns: repeat(auto-fit, minmax(200px, 1fr))`

**Behavior:**
- **Desktop (>768px):** All products (up to 4) displayed in one row.
  - Example: 4 products → 4 equal-width columns.
- **Tablet (768px–480px):** Products wrap to 2 columns.
  - Example: 4 products → 2 rows of 2 columns each.
- **Mobile (<480px):** Products wrap to 1 column (vertical stack).
  - Example: 4 products → 4 rows of 1 column each.

**Minimum column width:** 200px ensures product details remain readable on smaller screens.

---

## Tests

**Location:** `src/components/comparison/__tests__/comparison-page.spec.tsx`

**3 integration tests (one per AC):**

### Test 1: AC4 — Display products with all required fields

```typescript
import { render, screen } from "@testing-library/react"
import { ComparisonProvider } from "../../../lib/context/comparison-context"
import { ComparisonPage } from "../comparison-page"

// Mock product query hook
jest.mock("@medusajs/js-sdk", () => ({
  useProducts: jest.fn(),
}))

import { useProducts } from "@medusajs/js-sdk" // Verify import path

describe("ComparisonPage", () => {
  const mockProducts = [
    {
      id: "prod_1",
      title: "Product 1",
      thumbnail: "https://example.com/product1.jpg",
      variants: [
        { id: "var_1", title: "Small" },
        { id: "var_2", title: "Large" },
      ],
      // Mock price and inventory data
    },
    {
      id: "prod_2",
      title: "Product 2",
      thumbnail: "https://example.com/product2.jpg",
      variants: [{ id: "var_3", title: "Medium" }],
    },
  ]

  const wrapper = ({ children }) => (
    <ComparisonProvider>{children}</ComparisonProvider>
  )

  beforeEach(() => {
    jest.clearAllMocks()
  })

  it("displays selected products with image, title, price, variants, and availability", async () => {
    // Mock useComparison to return product IDs
    const mockUseComparison = {
      productIds: ["prod_1", "prod_2"],
      remove: jest.fn(),
      clear: jest.fn(),
    }

    // Mock useProducts to return product data
    ;(useProducts as jest.Mock).mockReturnValue({
      data: { products: mockProducts },
      isLoading: false,
      error: null,
    })

    // Mock useComparison hook
    jest
      .spyOn(require("../../../lib/context/comparison-context"), "useComparison")
      .mockReturnValue(mockUseComparison)

    render(<ComparisonPage />, { wrapper })

    // Verify products are displayed
    expect(screen.getByText("Product 1")).toBeInTheDocument()
    expect(screen.getByText("Product 2")).toBeInTheDocument()

    // Verify images are rendered
    const images = screen.getAllByRole("img")
    expect(images).toHaveLength(2)
    expect(images[0]).toHaveAttribute("src", "https://example.com/product1.jpg")
    expect(images[1]).toHaveAttribute("src", "https://example.com/product2.jpg")

    // Verify variants are displayed
    expect(screen.getByText("Small")).toBeInTheDocument()
    expect(screen.getByText("Large")).toBeInTheDocument()
    expect(screen.getByText("Medium")).toBeInTheDocument()

    // Verify availability badges are present
    const availabilityBadges = screen.getAllByText(/Stock/)
    expect(availabilityBadges.length).toBeGreaterThan(0)

    // Verify price is displayed (placeholder logic)
    // TODO: Update when actual price formatting is implemented
    const prices = screen.getAllByText(/\$/)
    expect(prices.length).toBeGreaterThan(0)
  })

  it("handles missing products gracefully (filters them out)", async () => {
    const mockUseComparison = {
      productIds: ["prod_1", "prod_missing"],
      remove: jest.fn(),
      clear: jest.fn(),
    }

    // Mock useProducts to return only one product (second is missing)
    ;(useProducts as jest.Mock).mockReturnValue({
      data: { products: [mockProducts[0], null] },
      isLoading: false,
      error: null,
    })

    jest
      .spyOn(require("../../../lib/context/comparison-context"), "useComparison")
      .mockReturnValue(mockUseComparison)

    render(<ComparisonPage />, { wrapper })

    // Verify only the existing product is displayed
    expect(screen.getByText("Product 1")).toBeInTheDocument()
    expect(screen.queryByText("Product 2")).not.toBeInTheDocument()

    // Verify only one product card is rendered
    const removeButtons = screen.getAllByRole("button", { name: /Remove .+ from comparison/ })
    expect(removeButtons).toHaveLength(1)
  })
})
```

**Verifies:** AC4 (Selected products are displayed with all required fields: image, title, price, variants, availability).

---

### Test 2: AC5 — Remove individual products

```typescript
import { render, screen, waitFor } from "@testing-library/react"
import userEvent from "@testing-library/user-event"
import { ComparisonProvider } from "../../../lib/context/comparison-context"
import { ComparisonPage } from "../comparison-page"

jest.mock("@medusajs/js-sdk", () => ({
  useProducts: jest.fn(),
}))

import { useProducts } from "@medusajs/js-sdk"

describe("ComparisonPage - Remove Products", () => {
  const mockProducts = [
    { id: "prod_1", title: "Product 1", thumbnail: "url1", variants: [] },
    { id: "prod_2", title: "Product 2", thumbnail: "url2", variants: [] },
  ]

  const wrapper = ({ children }) => (
    <ComparisonProvider>{children}</ComparisonProvider>
  )

  it("calls remove(productId) when clicking Remove button on a product", async () => {
    const mockRemove = jest.fn()
    const mockUseComparison = {
      productIds: ["prod_1", "prod_2"],
      remove: mockRemove,
      clear: jest.fn(),
    }

    ;(useProducts as jest.Mock).mockReturnValue({
      data: { products: mockProducts },
      isLoading: false,
      error: null,
    })

    jest
      .spyOn(require("../../../lib/context/comparison-context"), "useComparison")
      .mockReturnValue(mockUseComparison)

    render(<ComparisonPage />, { wrapper })

    // Find the Remove button for Product 1
    const removeButton = screen.getByRole("button", {
      name: /Remove Product 1 from comparison/,
    })

    // Click the Remove button
    await userEvent.click(removeButton)

    // Verify remove was called with the correct product ID
    await waitFor(() => {
      expect(mockRemove).toHaveBeenCalledWith("prod_1")
      expect(mockRemove).toHaveBeenCalledTimes(1)
    })
  })
})
```

**Verifies:** AC5 (Customers can remove individual products from the comparison page by clicking the Remove button).

---

### Test 3: AC6 — Clear all products

```typescript
import { render, screen, waitFor } from "@testing-library/react"
import userEvent from "@testing-library/user-event"
import { ComparisonProvider } from "../../../lib/context/comparison-context"
import { ComparisonPage } from "../comparison-page"

jest.mock("@medusajs/js-sdk", () => ({
  useProducts: jest.fn(),
}))

import { useProducts } from "@medusajs/js-sdk"

describe("ComparisonPage - Clear All", () => {
  const mockProducts = [
    { id: "prod_1", title: "Product 1", thumbnail: "url1", variants: [] },
    { id: "prod_2", title: "Product 2", thumbnail: "url2", variants: [] },
  ]

  const wrapper = ({ children }) => (
    <ComparisonProvider>{children}</ComparisonProvider>
  )

  it("calls clear() when clicking Clear All button", async () => {
    const mockClear = jest.fn()
    const mockUseComparison = {
      productIds: ["prod_1", "prod_2"],
      remove: jest.fn(),
      clear: mockClear,
    }

    ;(useProducts as jest.Mock).mockReturnValue({
      data: { products: mockProducts },
      isLoading: false,
      error: null,
    })

    jest
      .spyOn(require("../../../lib/context/comparison-context"), "useComparison")
      .mockReturnValue(mockUseComparison)

    render(<ComparisonPage />, { wrapper })

    // Find the Clear All button
    const clearAllButton = screen.getByRole("button", { name: /Clear all products from comparison/ })

    // Click the Clear All button
    await userEvent.click(clearAllButton)

    // Verify clear was called
    await waitFor(() => {
      expect(mockClear).toHaveBeenCalledTimes(1)
    })
  })

  it("displays empty state when no products are in the comparison list", async () => {
    const mockUseComparison = {
      productIds: [],
      remove: jest.fn(),
      clear: jest.fn(),
    }

    ;(useProducts as jest.Mock).mockReturnValue({
      data: { products: [] },
      isLoading: false,
      error: null,
    })

    jest
      .spyOn(require("../../../lib/context/comparison-context"), "useComparison")
      .mockReturnValue(mockUseComparison)

    render(<ComparisonPage />, { wrapper })

    // Verify empty state is displayed
    expect(screen.getByText("No products to compare")).toBeInTheDocument()
    expect(
      screen.getByText(/Add products from the product pages to start comparing/)
    ).toBeInTheDocument()

    // Verify Clear All button is not present in empty state
    expect(
      screen.queryByRole("button", { name: /Clear all/ })
    ).not.toBeInTheDocument()
  })
})
```

**Verifies:** AC6 (Customers can clear all products via the Clear All button; page shows empty state after clearing).

---

## Implementation Notes

### Critical TODO Items (Must Resolve Before Implementation)

1. **Product Query Hook:**
   - Verify exact import path and hook name in the repository.
   - Check `packages/core/js-sdk/` for available hooks
   - Check existing product components for usage patterns
   - Replace placeholder `useProducts` with actual implementation.

2. **Product Data Structure:**
   - Confirm `ProductDTO` type and import path.
   - Verify fields: `id`, `title`, `thumbnail`, `images`, `variants`.
   - Check variant structure and inventory fields.

3. **Price Formatting:**
   - Implement `formatPrice` function based on actual product price structure.
   - Determine which price to display (default variant, lowest, region-specific).
   - Handle currency formatting and locale.

4. **Availability Logic:**
   - Implement `getAvailability` function based on inventory data.
   - Determine how to aggregate variant inventory.
   - Handle edge cases (no inventory data, all variants out of stock).

5. **Styling Approach:**
   - Verify if Tailwind CSS is used in the project.
   - If not, adapt class names to match the project's styling approach.
   - Check for existing design system components (Button, Badge, etc.).

6. **File Structure:**
   - Confirm the actual component directory path (likely not `www/`).
   - Verify context import path from Story 1.
   - Check if barrel exports (`index.ts`) are used in the project.

---

**End of LLD**
