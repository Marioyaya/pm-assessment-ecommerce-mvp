# Part 2, Q6: Simple Logic Reading Question

E-commerce System MVP – Product Manager Assignment

## 1. What is this function mainly used to determine?

It is an **order eligibility check** that answers one question: *"Can this customer submit an order for this SKU in this quantity right now?"*

It takes three inputs:

| Input | Meaning |
|---|---|
| productStatus | Whether the product is currently on sale |
| skuStock | Available stock of the specific SKU the customer selected |
| buyCount | The quantity the customer wants to buy |

It returns two things:

- **canSubmit** (true or false) tells the system whether to allow the order.
- **reason** is a message explaining why the order was blocked, or empty when it is allowed. The front end can show this message directly to the user.

The function checks the three conditions **in a fixed order**, and the first failed check ends the function. This means the user only ever sees one reason at a time, even if several problems exist.

## 2. Under what conditions can the user not submit the order?

| Priority | Condition | Business Meaning | Reason Returned |
|---|---|---|---|
| 1 | productStatus is not ON_SALE | The product has been taken off sale (or is in any other non-sellable state, such as draft or deleted) | "Product is off sale" |
| 2 | skuStock <= 0 | The selected SKU is sold out | "Insufficient inventory" |
| 3 | buyCount > skuStock | The SKU has some stock, but fewer units than the customer wants | "Purchase quantity exceeds available inventory" |

The order can be submitted only when **all three checks pass**: the product is on sale, the SKU has stock, and the requested quantity does not exceed the stock.

**Two details worth noting:**

- **The status check uses "not equal to ON_SALE", not "equal to OFF_SALE".** This is the safer design. Any status other than ON_SALE (for example DRAFT, DELETED, or a status added in the future) is automatically blocked, so new statuses cannot accidentally allow purchases.
- **Checks 2 and 3 are intentionally separate.** "Sold out" and "not enough for your quantity" are different customer situations with different best responses. For sold out, the customer should choose another SKU or product. For insufficient quantity, they can simply reduce the amount. Separate messages make the experience clearer.

## 3. If productStatus = 'ON_SALE', skuStock = 3, buyCount = 5, what should the function return?

Step through the logic:

| Check | Evaluation | Result |
|---|---|---|
| productStatus !== 'ON_SALE' | 'ON_SALE' !== 'ON_SALE' is false | Pass, continue |
| skuStock <= 0 | 3 <= 0 is false | Pass, continue |
| buyCount > skuStock | 5 > 3 is true | Fail, return |

**Return value:**

```
{
  canSubmit: false,
  reason: 'Purchase quantity exceeds available inventory'
}
```

**Business meaning:** The product is available and the SKU is in stock, but the customer asked for 5 and only 3 exist. The order is blocked.

**Recommended user experience:** The message should be actionable, not just a refusal. For example: *"Only 3 available. Please reduce the quantity to 3 or less."* The page can also offer a one-tap "Change quantity to 3" button.

## 4. What additional checks should be added?

I group the additions by how much they affect the business, and I keep the same "first failure wins" structure so each failure returns one clear reason.

### A. Must add (the order would be invalid or risky without them)

| Check | Rule | Reason Message (example) | Why It Matters |
|---|---|---|---|
| Login status | isLoggedIn must be true | "Please log in to continue" | An order must belong to an account. The user should be sent to login and then returned to checkout with the cart intact. |
| Address validity | A shipping address exists and all required fields are valid | "Please add a valid shipping address" | Without a deliverable address, the order cannot be fulfilled. |
| Quantity lower bound | buyCount >= 1 and is a whole number | "Please enter a valid quantity" | The current logic accepts 0, negative or fractional values. With buyCount = 0 or -3, all three current checks pass, so the function would return canSubmit: true. This is a real logic gap. |
| SKU validity | The SKU exists, belongs to this product, and is itself active | "This option is no longer available" | A SKU can be disabled while the product is still on sale. Checking only product status misses that case. |
| Atomic stock lock at submit time | The stock check must be re-run on the server and the stock locked in one atomic step | "Sorry, this item just sold out" | This function reads stock at a single moment. Two customers can pass it simultaneously for the last unit. Only an atomic lock at submission prevents overselling. |

### B. Should add (business rules and risk control)

| Check | Rule | Reason Message (example) | Why It Matters |
|---|---|---|---|
| Purchase limit | buyCount <= perOrderLimit and the user's cumulative purchases of this item stay within perUserLimit | "Maximum {n} per order" | Prevents hoarding and resale, and supports promotions and limited releases. |
| Minimum order quantity | buyCount >= minOrderQuantity | "Minimum order quantity is {n}" | Required for wholesale or bundled products. |
| Price consistency | The price shown to the customer equals the current price | "The price has changed, please review" | Avoids charging an amount the customer did not see. |
| Sale time window | The current time is within the product's sale start and end time | "This item is not on sale yet" / "Sale has ended" | Supports scheduled launches and time-limited campaigns. |
| Shipping coverage | The product can ship to the customer's region | "This item cannot be delivered to your address" | Prevents orders that cannot be fulfilled. |
| Account status | The user account is not blocked or restricted | "Your account cannot place orders" | Basic risk control against fraud and abuse. |

### C. Nice to have (later iterations)

- **Duplicate-submission check** using an idempotency token, so double taps do not create two orders.
- **Payment method availability** for the amount and region.
- **Order total limits** (minimum order amount for free shipping, maximum amount for risk review).
- **Compliance checks** such as age verification for restricted goods.

### Improved logic order (conceptual)

1. User logged in and account valid
2. Product status is ON_SALE and within the sale window
3. SKU exists and is active
4. Quantity is a valid integer, at least the minimum, and within purchase limits
5. SKU stock is greater than 0 and at least the requested quantity
6. Valid shipping address and deliverable region
7. Price unchanged since the customer last saw it
8. Server locks stock atomically and creates the order

### Product-level recommendations from this analysis

- **Return error codes along with the message** (for example OFF_SALE, OUT_OF_STOCK, EXCEEDS_STOCK). The front end can then choose the right UI behavior and language, instead of depending on English text.
- **Return all failures when useful.** The current function stops at the first failure. For a form with several problems (missing address and invalid quantity), returning a list lets the customer fix everything at once.
- **Treat this function as a UX pre-check, not as the security boundary.** The same rules must run again on the server at submission time, because the page data can be stale or tampered with.
