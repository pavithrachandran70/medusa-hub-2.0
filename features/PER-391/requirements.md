# requirements.md — PER-391

**Type:** Bug Fix  
**Epic:** None (standalone backlog ticket)  
**Risk:** Low  
**Severity:** Medium

---

## Summary

After an admin updates the stocked quantity of an inventory item through the Edit stock levels drawer, the update saves successfully in the backend but the Locations table continues displaying the previous quantity. This is caused by the batch inventory location levels mutation not invalidating the inventory item levels query cache after a successful update.

---

## Requirements

### REQ-1: Cache invalidation on successful mutation

**Given** an admin has updated the stocked quantity of an inventory item through the Edit stock levels drawer  
**When** the batch inventory location levels mutation completes successfully  
**Then** the inventory item levels query cache is invalidated

**Verify:**  
- Inspect the batch inventory location levels mutation implementation
- Confirm a cache invalidation call targeting the inventory item levels query key is present after the mutation's success path
- Confirm the invalidation uses the correct query key pattern that matches the query the Locations table reads

---

### REQ-2: Immediate UI update without manual refresh

**Given** an admin has updated the stocked quantity of an inventory item through the Edit stock levels drawer  
**When** the mutation completes successfully and invalidates the cache  
**Then** the Locations table immediately reflects the updated stocked quantity without requiring a manual browser refresh

**Verify:**  
- Update an inventory item's stocked quantity through the Edit stock levels drawer
- Observe the Locations table updates the displayed stocked quantity within 2 seconds
- Confirm no manual refresh was performed

---

### REQ-3: Available quantity consistency

**Given** the inventory item levels query cache has been invalidated  
**When** the Locations table re-fetches the inventory data  
**Then** the available quantity displayed reflects the updated inventory data returned from the backend

**Verify:**  
- After updating stocked quantity, compare the available quantity shown in the Locations table with the value returned by the inventory item levels query
- Confirm they match

---

### REQ-4: No regression in existing update flow

**Given** the cache invalidation logic is added to the batch inventory location levels mutation  
**When** an admin performs any inventory update operation (single item, bulk, locations update)  
**Then** all existing inventory update functionality continues working as before

**Verify:**  
- Run existing automated tests for inventory updates (if any)
- Manually test: single item update, bulk quantity update, location changes
- Confirm all operations save successfully and UI reflects changes

---

### REQ-5: Regression test coverage for cache invalidation

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

## Technical Notes

### Cache Patterns

- Identify the query key used by the inventory item levels query that the Locations table depends on
- Identify the cache invalidation API (e.g., `queryClient.invalidateQueries()` in React Query, or equivalent in the caching layer used by this codebase)
- Determine where the batch inventory location levels mutation is implemented
- Add the invalidation call in the mutation's success handler, targeting the identified query key

### Existing Code References

- The batch inventory location levels mutation implementation can be found by searching the codebase for mutations related to inventory location updates or bulk stock updates
- The inventory item levels query can be found by inspecting the Locations table component and tracing its data fetching logic
- Cache patterns and query key conventions should be inferred from similar mutations and queries already in the codebase

### Open Questions

- None at this time; planner focus is sufficient to guide implementation

---

## Acceptance Criteria

1. Update the inventory item's stocked quantity from the Admin Dashboard.
2. The updated quantity must appear immediately in the Locations table.
3. No manual browser refresh should be required.
4. The available quantity should reflect the updated inventory data.
5. Existing inventory update functionality must continue working.
6. Add regression tests for query cache invalidation.

---

## Definition of Done

- [ ] Requirements REQ-1 through REQ-5 are verified
- [ ] All acceptance criteria are met
- [ ] Regression tests pass
- [ ] Code reviewed and approved
- [ ] Merged to base branch
