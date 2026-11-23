---
key: PER-391
type: story
status: todo
blocks: []
blocked_by: []
labels: []
repos: []
summary: Fix stale inventory location levels after bulk stock quantity update
---

**Description**

When an admin updates the stocked quantity of an inventory item through the Edit stock levels drawer, the update is successfully saved in the backend. However, the Locations table continues displaying the previous quantity because the inventory item levels query cache is not invalidated after the update.

Update the batch inventory location levels mutation to invalidate the inventory item levels query cache after a successful update.

**Acceptance Criteria**

1. Update the inventory item's stocked quantity from the Admin Dashboard.
2. The updated quantity must appear immediately in the Locations table.
3. No manual browser refresh should be required.
4. The available quantity should reflect the updated inventory data.
5. Existing inventory update functionality must continue working.
6. Add regression tests for query cache invalidation.
