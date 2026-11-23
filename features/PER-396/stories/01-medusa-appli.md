# Story 01-medusa-appli: Comparison State Management and Context Setup — LLD

**Repo:** `medusa-appli`  
**Type:** LLD  
**Status:** draft

---

## Acceptance Criteria

**From requirements/breakdown only:**

1. The comparison list can hold up to 4 products (AC2).
2. Duplicate products are prevented from being added (AC3).
3. Individual products can be removed from the comparison list (AC5).
4. All products can be cleared from the comparison list (AC6).

---

## File Plan

### File: `www/src/lib/context/comparison-context.types.ts`

**Purpose:** TypeScript type definitions for the comparison context.

**New file** — create as:

```typescript
export interface AddResult {
  success: boolean
  reason?: "limit" | "duplicate"
}

export interface ComparisonContextValue {
  productIds: string[]
  add: (productId: string) => AddResult
  remove: (productId: string) => void
  clear: () => void
}
```

**Rationale:** Separates types from implementation for clarity and reusability.

---

### File: `www/src/lib/context/comparison-context.tsx`

**Purpose:** React Context implementation for comparison state management.

**New file** — create as:

```typescript
"use client"

import React, { createContext, useContext, useState } from "react"
import { ComparisonContextValue, AddResult } from "./comparison-context.types"

const ComparisonContext = createContext<ComparisonContextValue | null>(null)

interface ComparisonProviderProps {
  children: React.ReactNode
}

export function ComparisonProvider({ children }: ComparisonProviderProps) {
  const [productIds, setProductIds] = useState<string[]>([])

  const add = (productId: string): AddResult => {
    if (productIds.length >= 4) {
      return { success: false, reason: "limit" }
    }
    if (productIds.includes(productId)) {
      return { success: false, reason: "duplicate" }
    }
    setProductIds([...productIds, productId])
    return { success: true }
  }

  const remove = (productId: string): void => {
    setProductIds(productIds.filter((id) => id !== productId))
  }

  const clear = (): void => {
    setProductIds([])
  }

  const value: ComparisonContextValue = {
    productIds,
    add,
    remove,
    clear,
  }

  return (
    <ComparisonContext.Provider value={value}>
      {children}
    </ComparisonContext.Provider>
  )
}

export function useComparison(): ComparisonContextValue {
  const context = useContext(ComparisonContext)
  if (context === null) {
    throw new Error("useComparison must be used within ComparisonProvider")
  }
  return context
}
```

**Key implementation details:**

- **`add` function:**
  - Check `productIds.length >= 4` first (AC2).
  - Check `productIds.includes(productId)` second (AC3).
  - Return `{ success: false, reason }` on failure; `{ success: true }` on success.
  - Update state via `setProductIds([...productIds, productId])` on success.
  
- **`remove` function:**
  - Filter out the given `productId` (AC5).
  - Use `filter` to avoid mutation.

- **`clear` function:**
  - Set `productIds` to `[]` (AC6).

- **`useComparison` hook:**
  - Throw error if context is null (enforces usage within provider).

**Client component:** Marked with `"use client"` directive at top of file (per Implementation Pointers).

---

### File: `www/src/lib/context/index.ts`

**Purpose:** Barrel export for context module.

**Action:** Modify existing file or create if not present.

**Add exports:**

```typescript
export {
  ComparisonProvider,
  useComparison,
} from "./comparison-context"
export type {
  ComparisonContextValue,
  AddResult,
} from "./comparison-context.types"
```

**If file does not exist:** Create as above.

**If file exists:** Append the above exports to the existing file. Do not remove existing exports.

---

## Interfaces

### `ComparisonContextValue`

```typescript
interface ComparisonContextValue {
  productIds: string[]
  add: (productId: string) => AddResult
  remove: (productId: string) => void
  clear: () => void
}
```

**Fields:**
- `productIds`: Array of product IDs currently in the comparison list (max 4).
- `add`: Adds a product ID to the list; returns `AddResult` indicating success or failure reason.
- `remove`: Removes a product ID from the list.
- `clear`: Clears all product IDs from the list.

---

### `AddResult`

```typescript
interface AddResult {
  success: boolean
  reason?: "limit" | "duplicate"
}
```

**Fields:**
- `success`: `true` if the product was added, `false` otherwise.
- `reason`: Present only when `success` is `false`; either `"limit"` (4-product max reached) or `"duplicate"` (product already in list).

---

## Algorithm Notes

### `add(productId: string): AddResult`

1. If `productIds.length >= 4`, return `{ success: false, reason: "limit" }`.
2. Else if `productIds.includes(productId)`, return `{ success: false, reason: "duplicate" }`.
3. Else:
   - Create new array: `[...productIds, productId]`.
   - Call `setProductIds` with new array.
   - Return `{ success: true }`.

**Order matters:** Check limit first, then duplicate. This ensures the limit check is enforced even if a duplicate is attempted when the list is full.

---

### `remove(productId: string): void`

1. Create new array: `productIds.filter((id) => id !== productId)`.
2. Call `setProductIds` with new array.

**Idempotent:** If `productId` is not in the list, the filtered array equals the original; `setProductIds` is called but state is unchanged.

---

### `clear(): void`

1. Call `setProductIds([])`.

**Direct replacement:** Sets state to an empty array.

---

## Tests

**Location:** `www/src/lib/context/__tests__/comparison-context.spec.tsx`

**4 unit tests (one per AC):**

### Test 1: AC2 — 4-product limit

```typescript
import { renderHook, act } from "@testing-library/react"
import { ComparisonProvider, useComparison } from "../comparison-context"

describe("ComparisonContext", () => {
  const wrapper = ({ children }) => (
    <ComparisonProvider>{children}</ComparisonProvider>
  )

  it("allows adding up to 4 products and blocks a 5th", () => {
    const { result } = renderHook(() => useComparison(), { wrapper })

    // Add 4 products successfully
    act(() => {
      expect(result.current.add("prod_1")).toEqual({ success: true })
      expect(result.current.add("prod_2")).toEqual({ success: true })
      expect(result.current.add("prod_3")).toEqual({ success: true })
      expect(result.current.add("prod_4")).toEqual({ success: true })
    })

    expect(result.current.productIds).toHaveLength(4)

    // Attempt to add a 5th product
    let addResult
    act(() => {
      addResult = result.current.add("prod_5")
    })

    expect(addResult).toEqual({ success: false, reason: "limit" })
    expect(result.current.productIds).toHaveLength(4)
    expect(result.current.productIds).not.toContain("prod_5")
  })
})
```

**Verifies:** AC2 (4-product limit enforced; 5th product blocked).

---

### Test 2: AC3 — Duplicate prevention

```typescript
  it("prevents adding the same product twice", () => {
    const { result } = renderHook(() => useComparison(), { wrapper })

    // Add product once
    act(() => {
      expect(result.current.add("prod_1")).toEqual({ success: true })
    })

    expect(result.current.productIds).toEqual(["prod_1"])

    // Attempt to add same product again
    let addResult
    act(() => {
      addResult = result.current.add("prod_1")
    })

    expect(addResult).toEqual({ success: false, reason: "duplicate" })
    expect(result.current.productIds).toEqual(["prod_1"])
  })
```

**Verifies:** AC3 (duplicate products blocked; same ID cannot be added twice).

---

### Test 3: AC5 — Individual product removal

```typescript
  it("removes an individual product from the list", () => {
    const { result } = renderHook(() => useComparison(), { wrapper })

    // Add 3 products
    act(() => {
      result.current.add("prod_1")
      result.current.add("prod_2")
      result.current.add("prod_3")
    })

    expect(result.current.productIds).toHaveLength(3)

    // Remove one product
    act(() => {
      result.current.remove("prod_2")
    })

    expect(result.current.productIds).toHaveLength(2)
    expect(result.current.productIds).toEqual(["prod_1", "prod_3"])
  })
```

**Verifies:** AC5 (individual product removed; array length decreases by 1; remaining products intact).

---

### Test 4: AC6 — Clear all products

```typescript
  it("clears all products from the list", () => {
    const { result } = renderHook(() => useComparison(), { wrapper })

    // Add 3 products
    act(() => {
      result.current.add("prod_1")
      result.current.add("prod_2")
      result.current.add("prod_3")
    })

    expect(result.current.productIds).toHaveLength(3)

    // Clear all
    act(() => {
      result.current.clear()
    })

    expect(result.current.productIds).toEqual([])
  })
```

**Verifies:** AC6 (clear sets productIds to empty array).

---

**End of LLD**
