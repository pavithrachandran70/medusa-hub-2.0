# requirements.md — Add Product Wishlist Functionality

## Overview

Provide wishlist functionality for authenticated customers to save products they want to purchase later, view them on a dedicated page, and manage their saved items across sessions.

## Requirements

### R1: Add product to wishlist (AC 1, 4)

**Given** an authenticated customer is viewing a product listing or product details page  
**When** they click the wishlist icon or button  
**Then** the product is added to their wishlist  
**And** duplicate attempts to add the same product are ignored (no duplicate entries created)

**Verify:**
- API endpoint accepts authenticated customer ID and product ID
- Database enforces unique constraint on (customer_id, product_id)
- Duplicate add request returns success without creating a second row

---

### R2: View saved products (AC 2, 6, 7)

**Given** an authenticated customer has products in their wishlist  
**When** they navigate to the Wishlist page  
**Then** all saved products are displayed with product name, image, and price

**Given** an authenticated customer has an empty wishlist  
**When** they navigate to the Wishlist page  
**Then** an appropriate empty-state message is shown

**Verify:**
- API endpoint returns all wishlist items for the authenticated customer
- Response includes product name, image URL, and price for each item
- UI renders empty-state message when list is empty

---

### R3: Remove product from wishlist (AC 3)

**Given** an authenticated customer is viewing their Wishlist page with saved products  
**When** they remove a product from the wishlist  
**Then** the product is deleted from their wishlist  
**And** the page updates to reflect the removal

**Verify:**
- API endpoint accepts customer ID and product ID to delete
- Database row is removed
- UI updates after successful deletion

---

### R4: Persist wishlist across sessions (AC 5)

**Given** an authenticated customer has products in their wishlist  
**When** they log out and log back in  
**Then** their wishlist contains the same saved products

**Verify:**
- Wishlist data is stored in the database, associated with customer account
- Wishlist items are retrieved on login
- No session-only storage is used

---

### R5: Authentication enforcement (AC 8)

**Given** an unauthenticated user attempts to manage a wishlist  
**When** they try to add, view, or remove products  
**Then** the request is rejected with an authentication error

**Verify:**
- API endpoints require valid authentication token
- Unauthenticated requests return 401 Unauthorized
- UI disables or hides wishlist controls when not authenticated

---

### R6: Cross-device compatibility (AC 9)

**Given** a customer accesses the application from desktop or mobile  
**When** they interact with wishlist functionality  
**Then** the experience works consistently across devices

**Verify:**
- UI is responsive on desktop and mobile viewports
- Wishlist icon and page are accessible on all device types
- Touch and click interactions both function correctly

---

### R7: Test coverage (AC 10)

**Given** wishlist operations are implemented  
**When** running the test suite  
**Then** API integration tests cover add, view, remove, duplicate prevention, authentication checks, and persistence  
**And** end-to-end tests cover UI flows on desktop and mobile

**Verify:**
- Integration tests exist for all API endpoints
- Tests verify authentication, idempotency, and data persistence
- E2E tests cover empty-state, add, view, and remove flows
