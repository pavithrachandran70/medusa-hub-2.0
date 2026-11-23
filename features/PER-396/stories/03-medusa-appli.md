# Story 03-medusa-appli: CompareButton Integration in Product Detail Pages — LLD

**Repo:** `medusa-appli`  
**Type:** LLD  
**Status:** draft

---

## Acceptance Criteria

**From requirements/breakdown only:**

1. A Compare button is visible on the product detail page (AC1 extended to PDP).
2. Customers can add products from the PDP; the same 4-product limit and duplicate prevention apply (AC2, AC3).

---

## File Plan

### Determine Product Detail Page Path

**Investigation required:** The `repo-tree/medusa-appli` listing does not show a `www/` directory structure. Based on the monorepo layout and the presence of a `packages/admin/dashboard` with a storefront-like structure, we must verify the actual location of the product detail page component.

**Search strategy:**
1. Check `repo-outline/medusa-appli` for references to product detail pages
2. Look for Next.js page routes under potential paths:
   - `www/src/pages/product/[id].tsx`
   - `www/app/product/[id]/page.tsx`
   - `www/src/app/product/[id]/page.tsx`
3. Search for components that render individual product details (likely named `ProductDetail`, `ProductPage`, or similar)

**Note:** The repo context shows this is a Medusa e-commerce monorepo. The storefront is likely a separate package or application within the monorepo structure. Without a visible `www/` directory in the tree, the product detail page may be under a different path or package name.

---

### File: Product Detail Page Component (path to be verified)

**Purpose:** Integrate the `CompareButton` into the product detail page UI.

**Action:** Modify existing file.

**Path verification steps:**

1. **Check for Next.js routing pattern:**
   - If pages router: Look for `pages/product/[id].tsx`, `pages/products/[id].tsx`, or `pages/p/[id].tsx`
   - If app router: Look for `app/product/[id]/page.tsx`, `app/products/[id]/page.tsx`, or `app/p/[id]/page.tsx`

2. **Search for product detail rendering logic:**
   - Find the component that accepts a product ID as a route parameter
   - Locate where product data is fetched (likely using a `useProduct` or similar hook)
   - Identify the section that renders product actions (Add to Cart, Wishlist, etc.)

3. **Likely file patterns to search for:**
   ```
   www/src/pages/product/[id].tsx
   www/src/pages/products/[id].tsx
   www/app/product/[id]/page.tsx
   www/src/app/product/[id]/page.tsx
   storefront/pages/product/[id].tsx
   storefront/app/product/[id]/page.tsx
   ```

**Once located, modify the file as follows:**

#### Function: Product detail page component (name varies)

**Find:** The component that renders the product detail page. This component likely:
- Accepts a product ID from the route parameters
- Fetches product data using a hook like `useProduct(id)` or similar
- Renders product information including image, title, price, description
- Displays action buttons like "Add to Cart"

**Common component names to look for:**
- `ProductDetailPage`, `ProductPage`, `ProductTemplate`, `ProductDetails`

**Modify:**

1. **Add import at top of file:**

```typescript
import { CompareButton } from "../../components/comparison" // adjust path relative to product detail page location
```

**Path adjustment:** The exact import path depends on where the product detail page file is located relative to `www/src/components/comparison/index.ts`. Adjust the number of `../` segments accordingly.

2. **Locate the product actions section:**

Look for the area where primary product actions are rendered. This is typically near:
- The "Add to Cart" button
- Product pricing display
- Product variant selectors (size, color, etc.)

**Example patterns to search for:**
```typescript
// Pattern 1: Explicit button section
<div className="product-actions">
  <button onClick={handleAddToCart}>Add to Cart</button>
</div>

// Pattern 2: Action buttons in a flex container
<div className="flex gap-2">
  <AddToCartButton />
</div>

// Pattern 3: Actions below product info
<ProductInfo />
<ProductActions>
  <AddToCartButton />
</ProductActions>
```

3. **Add `<CompareButton>` near the "Add to Cart" button:**

```typescript
<div className="product-actions"> {/* verify wrapper class name */}
  <button onClick={handleAddToCart}>Add to Cart</button>
  <CompareButton productId={product.id} />
</div>
```

**Placement guidelines:**
- Place `<CompareButton>` as a sibling to the "Add to Cart" button
- If actions are in a flex container, add it as the next flex item
- If there's a vertical stack of actions, add it below "Add to Cart"
- Maintain consistent spacing with existing action buttons

**Example integration scenarios:**

**Scenario A: Buttons in a horizontal layout**
```typescript
<div className="flex gap-4 items-center">
  <AddToCartButton product={product} />
  <CompareButton productId={product.id} />
  <WishlistButton productId={product.id} /> {/* if exists */}
</div>
```

**Scenario B: Buttons in a vertical stack**
```typescript
<div className="space-y-2">
  <AddToCartButton product={product} />
  <CompareButton productId={product.id} />
</div>
```

**Scenario C: Custom component structure**
```typescript
<ProductActions>
  <PrimaryAction>
    <AddToCartButton product={product} />
  </PrimaryAction>
  <SecondaryActions>
    <CompareButton productId={product.id} />
    <ShareButton />
  </SecondaryActions>
</ProductActions>
```

**Key considerations:**
- **Product ID:** Ensure `product.id` is available in scope where you add the button. If the product data is nested (e.g., `productData.product.id`), adjust accordingly.
- **Loading states:** If the product data is loading, the button should handle the case where `productId` is undefined. The `CompareButton` component from Story 2 accepts a string `productId` prop, so ensure the product is loaded before rendering the button, or conditionally render it:
  ```typescript
  {product && <CompareButton productId={product.id} />}
  ```
- **Styling:** Match the visual style of existing action buttons. If the product detail page uses a design system component like `<Button>` with specific variants, the `CompareButton` from Story 2 should be styled consistently (this is handled in the `CompareButton` component itself, but verify it matches the PDP's button style).

**Full example (pseudo-code, adjust to match actual component structure):**

```typescript
export const ProductDetailPage: React.FC<{ productId: string }> = ({ productId }) => {
  const { product, isLoading } = useProduct(productId) // verify hook name
  
  if (isLoading) return <LoadingSpinner />
  if (!product) return <NotFound />
  
  return (
    <div className="product-detail">
      <ProductImage src={product.image} />
      <div className="product-info">
        <h1>{product.title}</h1>
        <p>{product.price}</p>
        <p>{product.description}</p>
        
        <div className="product-actions">
          <button onClick={() => handleAddToCart(product)}>Add to Cart</button>
          <CompareButton productId={product.id} />
        </div>
      </div>
    </div>
  )
}
```

**Notes:**
- The exact structure depends on the product detail page implementation in the repo.
- If the page uses a template component (e.g., a Next.js page that renders a separate `<ProductTemplate>` component), modify the template component instead of the page file.
- If the product detail page is part of a larger layout component, ensure the `CompareButton` is added to the correct child component that handles product actions.

---

## Interfaces

### No new interfaces

This story reuses the `CompareButton` component from Story 2, which already defines its props interface:

```typescript
interface CompareButtonProps {
  productId: string
}
```

No additional interfaces are needed for this story.

---

## Algorithm Notes

### Integration logic

1. **Locate product actions section:** Find the DOM element or component that contains the "Add to Cart" button and other product actions.
2. **Import `CompareButton`:** Add the import statement at the top of the file, adjusting the relative path based on the file's location.
3. **Render `CompareButton`:** Add `<CompareButton productId={product.id} />` as a sibling to the "Add to Cart" button, inside the product actions container.
4. **Conditional rendering (if needed):** If the product data can be undefined during loading, wrap the button in a conditional:
   ```typescript
   {product && <CompareButton productId={product.id} />}
   ```

**No logic changes are required in the `CompareButton` component itself.** It uses the `useComparison()` hook from Story 1 to interact with the comparison state, and its behavior is identical whether rendered on a product card (PLP) or product detail page (PDP).

---

## Tests

**Location:** `www/src/components/comparison/__tests__/compare-button-pdp.spec.tsx`

**2 integration tests (one per AC):**

### Test 1: AC1 — Compare button visible on product detail page

```typescript
import { render, screen } from "@testing-library/react"
import { ProductDetailPage } from "../../pages/product/[id]" // adjust path
import { ComparisonProvider } from "../../../lib/context/comparison-context"

describe("CompareButton integration in ProductDetailPage", () => {
  const mockProduct = {
    id: "prod_123",
    title: "Test Product",
    price: "$10.00",
    description: "A test product",
    image: "/test-image.jpg",
  }

  const wrapper = ({ children }) => (
    <ComparisonProvider>{children}</ComparisonProvider>
  )

  it("renders a visible Compare button on the product detail page", () => {
    // Mock the product data fetching hook
    jest.mock("../../hooks/use-product", () => ({
      useProduct: jest.fn(() => ({
        product: mockProduct,
        isLoading: false,
      })),
    }))

    render(<ProductDetailPage productId="prod_123" />, { wrapper })

    const compareButton = screen.getByRole("button", { name: /add to comparison/i })
    expect(compareButton).toBeInTheDocument()
    expect(compareButton).toHaveTextContent("Compare")
  })
})
```

**Verifies:** AC1 (Compare button is visible on the product detail page).

**Note:** Adjust the import path for `ProductDetailPage` and the product data fetching hook based on the actual file structure. If the product detail page is a Next.js page component, you may need to mock `useRouter` and other Next.js-specific hooks.

---

### Test 2: AC2 — Adding products from PDP respects limit and duplicate rules

```typescript
import { render, screen, waitFor } from "@testing-library/react"
import userEvent from "@testing-library/user-event"
import { ComparisonProvider } from "../../../lib/context/comparison-context"
import { CompareButton } from "../compare-button"

describe("CompareButton on PDP", () => {
  const wrapper = ({ children }) => (
    <ComparisonProvider>{children}</ComparisonProvider>
  )

  it("allows adding a product from the PDP and respects 4-product limit", async () => {
    const { rerender } = render(
      <>
        <CompareButton productId="prod_1" />
        <CompareButton productId="prod_2" />
        <CompareButton productId="prod_3" />
        <CompareButton productId="prod_4" />
        <CompareButton productId="prod_5" />
      </>,
      { wrapper }
    )

    // Add 4 products
    const buttons = screen.getAllByRole("button", { name: /add to comparison/i })
    for (let i = 0; i < 4; i++) {
      await userEvent.click(buttons[i])
    }

    // Verify 4 products added (buttons show "Added to Comparison")
    await waitFor(() => {
      const addedButtons = screen.getAllByRole("button", { name: /remove from comparison/i })
      expect(addedButtons).toHaveLength(4)
    })

    // Verify 5th button is disabled
    rerender(
      <>
        <CompareButton productId="prod_1" />
        <CompareButton productId="prod_2" />
        <CompareButton productId="prod_3" />
        <CompareButton productId="prod_4" />
        <CompareButton productId="prod_5" />
      </>
    )

    const fifthButton = screen.getByRole("button", { name: /add to comparison/i })
    expect(fifthButton).toBeDisabled()
  })

  it("prevents adding a duplicate product from the PDP", async () => {
    render(<CompareButton productId="prod_1" />, { wrapper })

    const button = screen.getByRole("button", { name: /add to comparison/i })

    // Add product once
    await userEvent.click(button)

    // Button label changes to "Added to Comparison"
    await waitFor(() => {
      expect(screen.getByRole("button", { name: /remove from comparison/i })).toBeInTheDocument()
    })

    // Click button again (now it should remove the product)
    const removeButton = screen.getByRole("button", { name: /remove from comparison/i })
    await userEvent.click(removeButton)

    // Button label changes back to "Compare"
    await waitFor(() => {
      expect(screen.getByRole("button", { name: /add to comparison/i })).toBeInTheDocument()
    })

    // Add product again
    const addButton = screen.getByRole("button", { name: /add to comparison/i })
    await userEvent.click(addButton)

    // Verify product was added again (button shows "Added to Comparison")
    await waitFor(() => {
      expect(screen.getByRole("button", { name: /remove from comparison/i })).toBeInTheDocument()
    })
  })
})
```

**Verifies:** AC2 (Customers can add products from the PDP; the same 4-product limit and duplicate prevention apply).

**Note:** This test verifies the same behavior as Story 2 (AC2 and AC3), confirming that the `CompareButton` component works identically on the PDP. The duplicate prevention logic is enforced by the `add` function in the comparison context (Story 1), which the button calls regardless of where it's rendered.

---

**End of LLD**
