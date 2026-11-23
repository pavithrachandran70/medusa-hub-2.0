# Story — PER-391

**Type:** Bug Fix  
**Epic:** None (standalone backlog ticket)  
**Risk:** Low  
**Severity:** Medium

---

## Summary

After an admin updates the stocked quantity of an inventory item through the Edit stock levels drawer, the update saves successfully in the backend but the Locations table continues displaying the previous quantity. This is caused by the batch inventory location levels mutation not invalidating the inventory item levels query cache after a successful update.

---

## User Story

**As an** admin user  
**I want** the Locations table to immediately reflect updated stocked quantities  
**So that** I can verify my inventory changes without refreshing the browser

---

## Acceptance Criteria

### AC-1: Cache invalidation on successful mutation

**Given** an admin has updated the stocked quantity of an inventory item through the Edit stock levels drawer  
**When** the batch inventory location levels mutation completes successfully  
**Then** the inventory item levels query cache is invalidated

**Verify:**
- Inspect the batch inventory location levels mutation implementation
- Confirm a cache invalidation call targeting the inventory item levels query key is present after the mutation's success path
- Confirm the invalidation uses the correct query key pattern that matches the query the Locations table reads

---

### AC-2: Immediate UI update without manual refresh

**Given** an admin has updated the stocked quantity of an inventory item through the Edit stock levels drawer  
**When** the mutation completes successfully and invalidates the cache  
**Then** the Locations table immediately reflects the updated stocked quantity without requiring a manual browser refresh

**Verify:**
- Update an inventory item's stocked quantity through the Edit stock levels drawer
- Observe the Locations table updates the displayed stocked quantity within 2 seconds
- Confirm no manual refresh was performed

---

### AC-3: Available quantity consistency

**Given** the inventory item levels query cache has been invalidated  
**When** the Locations table re-fetches the inventory data  
**Then** the available quantity displayed reflects the updated inventory data returned from the backend

**Verify:**
- After updating stocked quantity, compare the available quantity shown in the Locations table with the value returned by the inventory item levels query
- Confirm they match

---

### AC-4: No regression in existing update flow

**Given** the cache invalidation logic is added to the batch inventory location levels mutation  
**When** an admin performs any inventory update operation (single item, bulk, locations update)  
**Then** all existing inventory update functionality continues working as before

**Verify:**
- Run existing automated tests for inventory updates (if any)
- Manually test: single item update, bulk quantity update, location changes
- Confirm all operations save successfully and UI reflects changes

---

### AC-5: Regression test coverage for cache invalidation

**Given** the cache invalidation fix is implemented  
**When** the codebase is tested  
**Then** regression tests exist that verify the inventory item levels query cache is invalidated after a successful batch inventory location levels mutation

**Verify:**
- Locate or create automated tests that:
  - Mock or spy on the cache invalidation mechanism
  - Execute the batch inventory location levels mutation
  - Assert the inventory item levels query key was invalidated after success
- Confirm the tests pass

---

## Technical Specification

### Problem Analysis

The batch inventory location levels mutation successfully updates the backend data but does not trigger a cache invalidation for the inventory item levels query that the Locations table depends on. This causes a stale data display until the user manually refreshes the browser or the query cache expires.

### Solution Approach

Add a cache invalidation call to the batch inventory location levels mutation's success handler. The invalidation must target the query key used by the inventory item levels query that the Locations table reads.

### Implementation Notes

#### Cache Patterns in This Codebase

Based on the repo structure (`packages/admin/dashboard/`, `packages/core/js-sdk/`), this codebase likely uses React Query or a similar cache management library. The standard pattern is:

1. **Query keys** are defined for each data-fetching query (e.g., `["inventory-item-levels", inventoryItemId]`)
2. **Mutations** return a promise and accept an `onSuccess` callback or similar hook
3. **Cache invalidation** is triggered via `queryClient.invalidateQueries()` or equivalent, targeting the query key

#### Files to Investigate

**Mutation implementation:**
- Search for the batch inventory location levels mutation in `packages/admin/dashboard/src/` or `packages/medusa/src/api/admin/`
- Keywords: "batch", "inventory", "location", "levels", "mutation", "useMutation"
- The mutation is likely a React Query `useMutation` hook or a similar pattern

**Query implementation:**
- Find the inventory item levels query in the Locations table component
- Search in `packages/admin/dashboard/src/` for components related to inventory locations
- Keywords: "inventory-item-levels", "useQuery", "Locations", "table"

**Query client access:**
- The query client instance is typically provided via context in React Query
- Look for `useQueryClient()` hook usage or a global query client export

#### Implementation Steps

1. **Locate the batch inventory location levels mutation**
   - Find the mutation hook definition (likely `useMutation`)
   - Identify where the mutation's success callback is or should be added

2. **Identify the inventory item levels query key**
   - Find the query in the Locations table component
   - Extract the exact query key structure (e.g., `["inventory-item-levels", id]` or similar)

3. **Add cache invalidation**
   - Import `useQueryClient` from React Query (or equivalent)
   - In the mutation's `onSuccess` callback, call:
     ```typescript
     queryClient.invalidateQueries({ queryKey: ["inventory-item-levels"] })
     ```
   - If the query key includes dynamic parameters (e.g., item ID), ensure the invalidation pattern matches all relevant queries

4. **Write regression tests**
   - Create a test that:
     - Renders the Edit stock levels drawer and Locations table
     - Mocks the mutation and query
     - Verifies `invalidateQueries` is called with the correct query key after mutation success
   - Place the test near existing mutation tests in the admin dashboard test suite

#### Example Implementation (Pseudocode)

```typescript
// In the mutation hook file (verify exact location in codebase)
import { useMutation, useQueryClient } from '@tanstack/react-query'

export const useBatchUpdateInventoryLocationLevels = () => {
  const queryClient = useQueryClient()
  
  return useMutation({
    mutationFn: (data) => batchUpdateInventoryLocationLevels(data),
    onSuccess: () => {
      // Invalidate the inventory item levels query cache
      queryClient.invalidateQueries({ 
        queryKey: ["inventory-item-levels"] 
      })
    }
  })
}
```

#### Verification Steps

1. Update an inventory item's stocked quantity through the Edit stock levels drawer
2. Without refreshing the browser, observe the Locations table
3. Confirm the stocked quantity updates within 2 seconds
4. Verify the available quantity also reflects the new data
5. Run the regression test suite to confirm the invalidation is called

### Open Questions

- **Exact query key structure:** What is the precise query key used by the inventory item levels query? (Will be determined during implementation by inspecting the Locations table component)
- **Mutation location:** Where exactly is the batch inventory location levels mutation defined? (Will be found by searching the admin dashboard and API routes)

### Assumptions

- The codebase uses React Query or a compatible cache management library
- The query client is accessible via `useQueryClient()` hook
- The mutation already has an `onSuccess` callback structure or supports adding one
- The inventory item levels query key follows a standard pattern like `["inventory-item-levels", ...]`

---

## Definition of Done

- [ ] AC-1: Cache invalidation call added to the batch inventory location levels mutation's success handler
- [ ] AC-2: Locations table updates immediately after stocked quantity change (verified manually)
- [ ] AC-3: Available quantity matches updated backend data (verified manually)
- [ ] AC-4: All existing inventory update operations continue working (verified via existing tests and manual testing)
- [ ] AC-5: Regression test for cache invalidation exists and passes
- [ ] Code reviewed and approved
- [ ] Merged to base branch `develop`
