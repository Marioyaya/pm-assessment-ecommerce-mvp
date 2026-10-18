# Part 2, Q5: API Understanding Question

E-commerce System MVP – Product Manager Assignment

## 1. What information can be displayed on the product detail page?

| Field | Display |
|---|---|
| code / message | Not shown to users. The front end uses code = 0 to confirm the request succeeded and renders the page. A non-zero code triggers an error state ("Failed to load, please retry"). |
| productName | Product title: "Bluetooth Earphones" |
| images | Image gallery. Currently one main image, so it shows as a single image with no carousel dots. |
| price | Current selling price, shown prominently |
| originalPrice | Strikethrough "list price", and optionally a discount badge calculated from the two prices |
| salesCount | Social proof: "132 sold" |
| stock | Availability: low-stock hint ("Only 10 left" when below a threshold) or "In stock" |
| skuList → skuName | SKU selector options: "White" and "Black" |
| skuList → price, stock | Price and stock of the selected SKU |
| status | Not displayed directly. It controls page behavior: whether the purchase buttons are enabled, and which message appears. |
| productId, skuId | Not displayed. Used internally when adding to cart or submitting an order. |

## 2. How should price: 19900 be displayed?

**As 199.00**, not 19,900.

- Prices in APIs are normally returned as **integers in the smallest currency unit (cents/fen)**. This avoids floating-point errors (for example 0.1 + 0.2 ≠ 0.3) in calculations such as subtotals and refunds.
- The front end converts for display: **19900 ÷ 100 = 199.00**, shown with the currency symbol and two decimals (e.g., ¥199.00 or $199.00).
- The same rule applies to originalPrice (29900 → 299.00) and the SKU prices (19900 → 199.00, 20900 → 209.00).
- Conversion happens **only at the display layer**. All calculations (subtotal, total, refund) stay in integers, and the server remains the source of truth for the amount actually charged.
- The response contains no currency field. I assume a single currency for the MVP, and I flag this as a missing field in Q6.

**Example of the displayed price area:** 199.00, with 299.00 struck through and a "33% off" badge (calculated as 1 − 19900 ÷ 29900 ≈ 33%).

## 3. If status = OFF_SALE, how should the page handle the product?

The page should still load but clearly show the product can no longer be bought. It should not return a 404 or a blank page, because users may arrive from order history, shared links, favorites or search engines.

| Element | Behavior |
|---|---|
| Page access | Still accessible and still shows product information |
| Banner | Prominent notice: "This product is no longer available" |
| "Add to Cart" / "Buy Now" | Hidden or disabled (grey), with the label "Unavailable" |
| SKU selector | Disabled |
| Price | Either greyed out or hidden, since the price is no longer valid for purchase |
| Next step for the user | A "Back to shopping" button and, if available, recommendations of similar products |
| Product list, search, category pages | The product does not appear |
| Shopping cart | The item shows "No longer available", is excluded from the total, and offers a Remove button |
| Order submission | The back end re-checks status at submit time and rejects any order containing an off-sale item, even if the front end was stale |
| Order history | Existing orders for this product remain viewable and unaffected |

**Principle:** The front end disables the buttons for experience, and the server enforces the rule for safety.

## 4. If the total product stock is 0, how should the purchase button be displayed?

- **Button:** Disabled (greyed out), with the label **"Sold Out"**. It must not be clickable, and it should not silently fail on tap.
- **"Add to Cart" and "Buy Now"** are both disabled.
- **Stock hint:** Show "Out of stock". If the SKU selector is visible, mark all SKUs as unavailable.
- **The product page stays accessible**, since the product is still on sale (status = ON_SALE) and may be restocked.
- **Optional improvement (P2):** A "Notify me when back in stock" button to capture demand and recover the sale.
- **Server-side check:** The order submission API validates stock again, so a stale page can never create an order for unavailable stock.

**Data note:** In this response, product stock (25) equals the sum of the SKU stocks (10 + 15), so the two are consistent. The purchase decision should be based on the **SKU-level stock**. If product-level stock and the SKU sum ever disagree, the SKU data wins, and the discrepancy should be logged and raised with engineering.

## 5. If the user selects the Black SKU, how should the displayed price and inventory change?

| | Before selecting a SKU | White (skuId 20001) | Black (skuId 20002) |
|---|---|---|---|
| Price shown | Price range, e.g., 199.00 – 209.00, or "From 199.00" | 199.00 | **209.00** (20900 ÷ 100) |
| Stock shown | Total 25 (or no number until a SKU is chosen) | 10 | **15** |
| Maximum quantity | Not applicable | 10 | **15** (or the per-order limit if lower) |

Additional details:

- The displayed price **switches from the product-level 199.00 to the SKU price 209.00**. The SKU price overrides the product price once a SKU is selected.
- The stock indicator updates to **15**, and the quantity selector is capped at that number.
- The strikethrough price needs care. The API only provides one originalPrice (299.00) at product level. It is unclear whether it applies to Black, whose selling price is higher. To avoid showing a misleading discount, the page should **hide the strikethrough and the discount badge for Black** unless the API provides a SKU-level original price. This is one of the missing fields below.
- If a SKU has stock 0, it appears **greyed out and not selectable**, while other SKUs stay available.
- The "Add to Cart" and "Buy Now" actions send the **skuId (20002)**, not just the productId.
- The user must select a SKU before adding to cart. If none is selected, the button prompts "Please select a color" (or the system pre-selects the first available SKU, which is a UX decision to confirm).

## 6. Missing fields (5 recommended)

| # | Missing Field (suggested name) | Why It Is Needed |
|---|---|---|
| 1 | currency (e.g., "USD", "CNY") | Prices are integers with no unit. Without the currency, the front end cannot display the right symbol or format, and a multi-market expansion would break. It also prevents the 100x display error if the unit is misunderstood. |
| 2 | description / specifications (and detailImages) | A product page needs a text description, key specs, and detail images to help customers decide. The current response only has a title and one image, which is not enough to support a purchase decision and hurts conversion. |
| 3 | SKU-level originalPrice and images (inside each skuList item) | The Black SKU costs more than the product-level price, so a single originalPrice is ambiguous. Customers also expect the main image to change when they pick a color. Without SKU images, the gallery cannot match the selected color. |
| 4 | purchaseLimit (max quantity per order or per user) and minOrderQuantity | The quantity selector needs an upper bound beyond stock. This supports promotions, anti-hoarding rules and fair distribution. Without it, the front end cannot validate quantity before the user reaches checkout. |
| 5 | SKU-level status (e.g., ON_SALE / OFF_SALE) and shipping information (shippingFee, estimatedDeliveryDays, shipsFrom) | A single SKU may be unavailable while the product stays on sale. Shipping cost and delivery time strongly influence the decision to buy, and customers expect to see them before checkout rather than being surprised at order submission. |

**Other useful fields (nice to have):** rating and reviewCount for trust, categoryId and breadcrumb for navigation, updatedAt or version for cache and price-change detection, attributes (e.g., Bluetooth version, battery life) for comparison, and afterSalesPolicy (returns and warranty).
