---
key: PER-383
type: story
status: todo
blocks: []
blocked_by: []
labels: []
repos: []
summary: Add Product Wishlist Functionality
---

## Description

Implement a wishlist feature that allows customers to save products they are interested in and view them later.

Customers should be able to add products to their wishlist from the product listing or product details page. They should also be able to view their saved products and remove products from the wishlist.

The wishlist should be associated with the logged-in customer.

## Acceptance Criteria

 1. Customers can add a product to their wishlist using a wishlist icon or button.
 2. Customers can view all their saved products on a dedicated Wishlist page.
 3. Customers can remove products from their wishlist.
 4. Adding the same product multiple times should not create duplicate entries.
 5. Wishlist data should persist when the customer logs out and logs in again.
 6. The wishlist page should display product name, image, and price.
 7. If the wishlist is empty, display an appropriate empty-state message.
 8. Only authenticated customers can manage their wishlist.
 9. Wishlist functionality should work on both desktop and mobile devices.
10. Add appropriate API and integration tests for wishlist operations.
