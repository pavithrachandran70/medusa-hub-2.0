# requirements.md — PER-396

**Type:** feature  
**Covers:** CAP-TBD (product comparison, 4-product limit, duplicate prevention, persistence)

---

## Overview

Customers can select up to 4 products for side-by-side comparison. Products are added from product cards or product detail pages via a Compare button. The comparison page displays each selected product's image, title, price, variants, and availability. Customers can remove individual products or clear all selections. Comparison selections persist in browser storage for returning visitors on the same browser.

---

## Acceptance Criteria

### AC1: Add Compare button to product cards

**Given** a product card is displayed  
**When** the customer views the card  
**Then** a Compare button is visible

**Given** the customer clicks the Compare button on a product card  
**When** the comparison list has fewer than 4 products  
**And** the product is not already in the comparison list  
**Then** the product is added to the comparison list  
**And** the button state reflects the product is selected

---

### AC2: Limit comparison list to 4 products

**Given** the comparison list already contains 4 products  
**When** the customer attempts to add another product  
**Then** the add action is blocked  
**And** a message indicates the 4-product limit

---

### AC3: Prevent duplicate products in comparison

**Given** a product is already in the comparison list  
**When** the customer clicks the Compare button for that product  
**Then** the product is not added again  
**And** the button state reflects the product is already selected

---

### AC4: Display selected products on comparison page

**Given** the customer has added at least one product to the comparison list  
**When** the customer navigates to the comparison page  
**Then** each selected product is displayed in a column

**Given** a product is displayed on the comparison page  
**Then** the following fields are shown:  
- product image  
- title  
- price  
- variants  
- availability

---

### AC5: Remove individual products from comparison

**Given** the comparison page is displayed with selected products  
**When** the customer clicks a Remove button for a specific product  
**Then** that product is removed from the comparison list  
**And** the comparison page updates to reflect the removal

---

### AC6: Clear all products from comparison

**Given** the comparison page is displayed with selected products  
**When** the customer clicks the Clear All button  
**Then** all products are removed from the comparison list  
**And** the comparison page reflects an empty state

---

### AC7: Persist comparison selections in browser storage

**Given** the customer has added products to the comparison list  
**When** the customer closes the browser or navigates away  
**And** returns to the site in the same browser  
**Then** the comparison list still contains the previously selected products

**Given** comparison data is stored in browser storage  
**When** the customer adds or removes a product  
**Then** the browser storage is updated immediately

---

## Open Questions

1. What happens when a product in the comparison list is no longer available (deleted, out of stock)? Should it remain in the list with a visual indicator, or be removed automatically?
2. Should the comparison list be accessible from all pages (e.g., via a floating button or header link), or only from product pages?
3. What is the visual state of the Compare button when the 4-product limit is reached and the button is disabled?
4. Should variant selection on the comparison page allow changing which variant is compared, or is the initially selected variant fixed?

---

## Assumptions

1. Product data displayed on the comparison page matches the data structure used on product cards and detail pages.
2. Browser storage (localStorage or sessionStorage) is available and enabled in the customer's browser.
3. The comparison feature does not require authentication; anonymous customers can compare products.
4. The comparison page is a standalone route accessible via a link from the site navigation or a floating UI element.
