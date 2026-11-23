# Story 05-medusa-appli: LocalStorage Persistence and Sync — LLD

**Repo:** `medusa-appli`  
**Type:** LLD  
**Status:** draft

---

## Acceptance Criteria

**From requirements/breakdown only:**

1. Comparison selections persist in browser storage and are restored when the customer returns to the site in the same browser (AC7 part 1).
2. Browser storage is updated immediately when a product is added or removed (AC7 part 2).

---

## File Plan

### File: `src/lib/context/comparison-context.tsx`

**Purpose:** Add localStorage persistence to the comparison context from Story 1.

**Action:** Modify existing file created in Story 1.

**Current state (from Story 1):**
- File exports `ComparisonProvider`, `useComparison()` hook, and types
- `ComparisonProvider` manages `productIds` state using `useState<string[]>([])`
- Provides `add`, `remove`, and `clear` operations
- State is in-memory only (no persistence)

**Changes to make:**

#### 1. Add localStorage hydration on mount

**Find:** The `ComparisonProvider` function component, specifically the line where state is initialized:

```typescript
const [productIds, setProductIds] = useState<string[]>([])
```

**Replace with:**

```typescript
const [productIds, setProductIds] = useState<string[]>(() => {
  // Hydrate from localStorage on mount
  try {
    const stored = localStorage.getItem("medusa_comparison")
    if (stored) {
      const parsed = JSON.parse(stored)
      if (parsed && Array.isArray(parsed.comparison_products)) {
        return parsed.comparison_products
      }
    }
  } catch (error) {
    // Silently fail - localStorage unavailable or invalid JSON
    console.warn("Failed to hydrate comparison list from localStorage:", error)
  }
  return []
})
```

**Rationale:**
- Uses lazy initialization (`useState(() => ...)`) to read localStorage only once on mount, not on every render
- Validates that the parsed data has the expected shape (`comparison_products` array)
- Gracefully handles localStorage unavailable (private browsing), JSON parse errors, or missing data
- Returns empty array `[]` as fallback
- Logs warning to console for debugging, but does not throw or show UI error

---

#### 2. Add localStorage sync effect

**Find:** The end of the `ComparisonProvider` function body, before the `return` statement.

**Add the following code:**

```typescript
// Sync to localStorage whenever productIds changes
React.useEffect(() => {
  try {
    localStorage.setItem(
      "medusa_comparison",
      JSON.stringify({ comparison_products: productIds })
    )
  } catch (error) {
    // Silently fail - localStorage unavailable or quota exceeded
    console.warn("Failed to sync comparison list to localStorage:", error)
  }
}, [productIds])
```

**Placement:** Insert this `useEffect` after the `clear` function definition and before the `const value: ComparisonContextValue = { ... }` line.

**Rationale:**
- Runs whenever `productIds` changes (dependency array `[productIds]`)
- Writes the entire state to localStorage in the schema: `{ comparison_products: string[] }`
- Wraps in try-catch to handle quota exceeded errors or unavailable localStorage
- Logs warning to console but does not throw

---

#### 3. Import React (if not already imported)

**Check:** Verify that `React` is imported at the top of the file.

**If the file has:**
```typescript
import { createContext, useContext, useState } from "react"
```

**Replace with:**
```typescript
import React, { createContext, useContext, useState, useEffect } from "react"
```

**Or, if `React` is already a default import, add `useEffect` to the named imports:**
```typescript
import React, { createContext, useContext, useState, useEffect } from "react"
```

**Rationale:**
- Need `React.useEffect` or `useEffect` for the sync effect
- Need `React` namespace if using `React.useEffect` in the code

**Note:** The existing code from Story 1 may already import `React`. Verify and adjust as needed.

---

## Full Modified File Structure

**After applying changes, the file should have this structure:**

```typescript
"use client"

import React, { createContext, useContext, useState, useEffect } from "react"
import { ComparisonContextValue, AddResult } from "./comparison-context.types"

const ComparisonContext = createContext<ComparisonContextValue | null>(null)

interface ComparisonProviderProps {
  children: React.ReactNode
}

export function ComparisonProvider({ children }: ComparisonProviderProps) {
  // Hydrate from localStorage on mount
  const [productIds, setProductIds] = useState<string[]>(() => {
    try {
      const stored = localStorage.getItem("medusa_comparison")
      if (stored) {
        const parsed = JSON.parse(stored)
        if (parsed && Array.isArray(parsed.comparison_products)) {
          return parsed.comparison_products
        }
      }
    } catch (error) {
      console.warn("Failed to hydrate comparison list from localStorage:", error)
    }
    return []
  })

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

  // Sync to localStorage whenever productIds changes
  useEffect(() => {
    try {
      localStorage.setItem(
        "medusa_comparison",
        JSON.stringify({ comparison_products: productIds })
      )
    } catch (error) {
      console.warn("Failed to sync comparison list to localStorage:", error)
    }
  }, [productIds])

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

---

## Interfaces

**No new interfaces are introduced.** This story only modifies the implementation of `ComparisonProvider` from Story 1. The types remain unchanged:

- `ComparisonContextValue` (from Story 1)
- `AddResult` (from Story 1)
- `ComparisonProviderProps` (internal to `ComparisonProvider`)

---

## Algorithm Notes

### Hydration Logic (on mount)

1. **On component mount** (useState lazy initializer runs once):
   - Attempt to read `localStorage.getItem("medusa_comparison")`
   - If key exists:
     - Parse JSON
     - Validate structure: check that `parsed.comparison_products` exists and is an array
     - If valid, return `parsed.comparison_products` as initial state
   - If key does not exist, or parsing fails, or structure is invalid:
     - Return `[]` (empty array)
2. **Error handling:**
   - Catch any errors (localStorage unavailable, JSON parse error)
   - Log warning to console
   - Return `[]` as fallback

**Edge cases:**
- **Private browsing mode:** localStorage may throw when accessed. The try-catch handles this gracefully.
- **Invalid JSON:** `JSON.parse()` throws. Caught and handled.
- **Wrong data shape:** Validation checks for `Array.isArray(parsed.comparison_products)`. If false, returns `[]`.
- **Empty localStorage:** Key does not exist → `localStorage.getItem()` returns `null` → condition fails → returns `[]`.

---

### Sync Logic (on state change)

1. **On every render where `productIds` has changed** (useEffect dependency `[productIds]`):
   - Stringify current `productIds` as `{ comparison_products: productIds }`
   - Write to `localStorage.setItem("medusa_comparison", ...)`
2. **Error handling:**
   - Catch any errors (localStorage unavailable, quota exceeded)
   - Log warning to console
   - Do **not** throw or show UI error

**Edge cases:**
- **Quota exceeded:** Browser storage limit hit. The try-catch prevents crash; feature degrades (no persistence).
- **Private browsing:** localStorage may be read-only or throw on write. Caught and handled.
- **Rapid state changes:** useEffect runs after every state update. This is acceptable because localStorage writes are synchronous and fast for small payloads (max 4 product IDs).

**Performance note:** Each `add`, `remove`, or `clear` operation triggers a state update, which triggers the `useEffect`, which writes to localStorage. For 4 items, the payload is ~100 bytes, so writes are fast.

---

### Why Lazy Initialization for Hydration?

Using `useState(() => { ... })` (lazy initializer) ensures the localStorage read happens **only once on mount**, not on every render. If we used:

```typescript
const [productIds, setProductIds] = useState<string[]>(readFromLocalStorage())
```

The `readFromLocalStorage()` call would run **on every render**, which is inefficient. Lazy initialization runs the function **only once** when the state is first created.

---

## Tests

**Location:** `src/lib/context/__tests__/comparison-context-persistence.spec.tsx`

**2 integration tests (one per AC):**

---

### Test 1: AC7 part 1 — Persist and restore on reload

```typescript
import { renderHook, act } from "@testing-library/react"
import { ComparisonProvider, useComparison } from "../comparison-context"

describe("ComparisonContext - localStorage persistence", () => {
  const wrapper = ({ children }) => (
    <ComparisonProvider>{children}</ComparisonProvider>
  )

  beforeEach(() => {
    // Clear localStorage before each test
    localStorage.clear()
  })

  it("restores comparison list from localStorage on mount", () => {
    // Seed localStorage with initial data
    localStorage.setItem(
      "medusa_comparison",
      JSON.stringify({ comparison_products: ["prod_1", "prod_2", "prod_3"] })
    )

    // Mount the provider (which triggers hydration)
    const { result } = renderHook(() => useComparison(), { wrapper })

    // Verify state was hydrated from localStorage
    expect(result.current.productIds).toEqual(["prod_1", "prod_2", "prod_3"])
  })

  it("handles missing localStorage gracefully (starts with empty array)", () => {
    // localStorage is empty (no key exists)
    const { result } = renderHook(() => useComparison(), { wrapper })

    // Verify state starts empty
    expect(result.current.productIds).toEqual([])
  })

  it("handles invalid JSON in localStorage gracefully (starts with empty array)", () => {
    // Seed localStorage with invalid JSON
    localStorage.setItem("medusa_comparison", "not valid JSON")

    const { result } = renderHook(() => useComparison(), { wrapper })

    // Verify state starts empty despite invalid data
    expect(result.current.productIds).toEqual([])
  })

  it("handles wrong data shape in localStorage gracefully (starts with empty array)", () => {
    // Seed localStorage with valid JSON but wrong shape
    localStorage.setItem(
      "medusa_comparison",
      JSON.stringify({ wrong_key: ["prod_1"] })
    )

    const { result } = renderHook(() => useComparison(), { wrapper })

    // Verify state starts empty despite wrong shape
    expect(result.current.productIds).toEqual([])
  })

  it("persists state across remounts (simulating browser reload)", () => {
    // First mount: add products
    const { result, unmount } = renderHook(() => useComparison(), { wrapper })

    act(() => {
      result.current.add("prod_1")
      result.current.add("prod_2")
    })

    // Verify products were added
    expect(result.current.productIds).toEqual(["prod_1", "prod_2"])

    // Unmount (simulating page close)
    unmount()

    // Second mount: verify state is restored from localStorage
    const { result: result2 } = renderHook(() => useComparison(), { wrapper })
    expect(result2.current.productIds).toEqual(["prod_1", "prod_2"])
  })
})
```

**Verifies:** AC7 part 1 (Comparison selections persist in browser storage and are restored when the customer returns to the site).

**Edge cases covered:**
- Happy path: localStorage contains valid data
- Missing data: localStorage is empty
- Invalid JSON: localStorage contains unparseable data
- Wrong shape: localStorage contains valid JSON but wrong structure
- Remount simulation: state persists across unmount/remount (reload scenario)

---

### Test 2: AC7 part 2 — Immediate sync on add/remove

```typescript
import { renderHook, act } from "@testing-library/react"
import { ComparisonProvider, useComparison } from "../comparison-context"

describe("ComparisonContext - localStorage sync", () => {
  const wrapper = ({ children }) => (
    <ComparisonProvider>{children}</ComparisonProvider>
  )

  beforeEach(() => {
    localStorage.clear()
  })

  it("syncs to localStorage immediately when a product is added", () => {
    const { result } = renderHook(() => useComparison(), { wrapper })

    // Add a product
    act(() => {
      result.current.add("prod_1")
    })

    // Verify localStorage was updated immediately
    const stored = localStorage.getItem("medusa_comparison")
    expect(stored).not.toBeNull()

    const parsed = JSON.parse(stored!)
    expect(parsed.comparison_products).toEqual(["prod_1"])
  })

  it("syncs to localStorage immediately when a product is removed", () => {
    const { result } = renderHook(() => useComparison(), { wrapper })

    // Add two products
    act(() => {
      result.current.add("prod_1")
      result.current.add("prod_2")
    })

    // Verify both are in localStorage
    let stored = localStorage.getItem("medusa_comparison")
    let parsed = JSON.parse(stored!)
    expect(parsed.comparison_products).toEqual(["prod_1", "prod_2"])

    // Remove one product
    act(() => {
      result.current.remove("prod_1")
    })

    // Verify localStorage was updated immediately
    stored = localStorage.getItem("medusa_comparison")
    parsed = JSON.parse(stored!)
    expect(parsed.comparison_products).toEqual(["prod_2"])
  })

  it("syncs to localStorage immediately when all products are cleared", () => {
    const { result } = renderHook(() => useComparison(), { wrapper })

    // Add products
    act(() => {
      result.current.add("prod_1")
      result.current.add("prod_2")
    })

    // Verify products are in localStorage
    let stored = localStorage.getItem("medusa_comparison")
    let parsed = JSON.parse(stored!)
    expect(parsed.comparison_products).toEqual(["prod_1", "prod_2"])

    // Clear all
    act(() => {
      result.current.clear()
    })

    // Verify localStorage was updated to empty array immediately
    stored = localStorage.getItem("medusa_comparison")
    parsed = JSON.parse(stored!)
    expect(parsed.comparison_products).toEqual([])
  })

  it("handles localStorage quota exceeded gracefully (does not crash)", () => {
    const { result } = renderHook(() => useComparison(), { wrapper })

    // Mock localStorage.setItem to throw quota exceeded error
    const originalSetItem = Storage.prototype.setItem
    Storage.prototype.setItem = jest.fn(() => {
      throw new Error("QuotaExceededError")
    })

    // Attempt to add a product (should not crash)
    act(() => {
      result.current.add("prod_1")
    })

    // Verify state was updated in memory (even if localStorage failed)
    expect(result.current.productIds).toEqual(["prod_1"])

    // Restore original setItem
    Storage.prototype.setItem = originalSetItem
  })

  it("writes correct localStorage key and schema", () => {
    const { result } = renderHook(() => useComparison(), { wrapper })

    act(() => {
      result.current.add("prod_1")
      result.current.add("prod_2")
    })

    // Verify key name
    const stored = localStorage.getItem("medusa_comparison")
    expect(stored).not.toBeNull()

    // Verify schema
    const parsed = JSON.parse(stored!)
    expect(parsed).toHaveProperty("comparison_products")
    expect(Array.isArray(parsed.comparison_products)).toBe(true)
    expect(parsed.comparison_products).toEqual(["prod_1", "prod_2"])
  })
})
```

**Verifies:** AC7 part 2 (Browser storage is updated immediately when a product is added or removed).

**Edge cases covered:**
- Add operation triggers sync
- Remove operation triggers sync
- Clear operation triggers sync
- Quota exceeded error is handled gracefully (does not crash)
- Correct key name and schema are used

---

## Implementation Notes

### Critical Verification Points

1. **Verify Story 1 completion:**
   - The file `src/lib/context/comparison-context.tsx` must exist from Story 1
   - It must export `ComparisonProvider`, `useComparison`, and types
   - The existing code structure must match what Story 1 defines

2. **React import adjustment:**
   - Check if the file already imports `React` or uses named imports
   - Adjust the import statement to include `useEffect` (if not already present)
   - If using `React.useEffect`, ensure `React` is imported as default

3. **Hydration timing:**
   - The lazy initializer runs **only on mount**, not on every render
   - This is critical for performance — do not accidentally move the localStorage read into the render body

4. **Error handling strategy:**
   - All localStorage operations are wrapped in try-catch
   - Errors are logged to console (for debugging) but do not throw
   - The feature degrades gracefully: works in-session, but does not persist

5. **State updates trigger sync:**
   - Every call to `setProductIds` (via `add`, `remove`, or `clear`) triggers the `useEffect`
   - The `useEffect` dependency is `[productIds]`, so it runs whenever the state changes
   - This ensures immediate sync per AC7 part 2

6. **Testing approach:**
   - Tests verify the happy path (hydration and sync work)
   - Tests verify error handling (invalid data, quota exceeded)
   - Tests verify the localStorage schema matches the HLD specification
   - Tests do **not** rely on `act()` timing assumptions — they read localStorage directly after state updates

---

### Why Not Use `useLayoutEffect`?

`useEffect` is sufficient here because:
- Writing to localStorage does not affect the DOM (no visual flicker)
- The write happens asynchronously after render, which is fine for persistence
- Using `useLayoutEffect` would block rendering, which is unnecessary overhead

---

### Cross-Tab Sync (Deferred)

The HLD mentions that cross-tab sync using the `storage` event listener is a **deferred enhancement**. This story does **not** implement it. If needed in the future:

```typescript
useEffect(() => {
  const handleStorageChange = (event: StorageEvent) => {
    if (event.key === "medusa_comparison" && event.newValue) {
      try {
        const parsed = JSON.parse(event.newValue)
        if (parsed && Array.isArray(parsed.comparison_products)) {
          setProductIds(parsed.comparison_products)
        }
      } catch (error) {
        console.warn("Failed to sync from storage event:", error)
      }
    }
  }

  window.addEventListener("storage", handleStorageChange)
  return () => window.removeEventListener("storage", handleStorageChange)
}, [])
```

**But this is out of scope for Story 5.**

---

**End of LLD**
