# Part 2, Q8: Bug / Exception Analysis Question

E-commerce System MVP – Product Manager Assignment

## 1. Problem Summary

| Item | Detail |
|---|---|
| Symptom | The product detail page shows stock = 1, but order submission returns "Insufficient inventory". |
| Core issue | Two parts of the system disagree about the same stock number. |
| Key principle | The **back end is the final source of truth**. A stock number on a page is only a snapshot taken at one moment, so the real question is *why the page's snapshot and the server's check differ*. |

**First, a classification question:** Is this a true bug, or expected behavior that is explained poorly? Some of the causes below (for example, another customer buying the last unit) are normal and correct system behavior. The product problem in that case is the **unclear message and experience**, not the stock logic.

## 2. Possible Causes

### Cause A: Another customer bought the last unit (most likely and normal)

- The page loaded when stock was 1. Before this customer tapped Submit, someone else ordered it, so stock became 0.
- The system worked correctly. The page was simply out of date by the time of submission.
- **Pattern:** Happens mostly with popular products and low stock, and the time between page view and submit is long (minutes).

### Cause B: Stale or cached data on the product detail page

- The page stock comes from a cache (CDN, front-end cache, or server cache) that was not refreshed after the last sale or stock change.
- **Pattern:** The same stale number appears for many users, and refreshing the page later shows 0.

### Cause C: Another user's stock lock is holding the unit

- Stock is locked when an order is submitted and stays locked until payment or timeout. Another customer may have submitted an order for that unit and not yet paid.
- The page may show **total stock (1)** while the order check uses **available stock = total − locked = 0**.
- **Pattern:** The unit becomes available again after the other order's payment deadline passes. This is expected behavior, but the page should show available stock, not total stock.

### Cause D: SKU stock and product stock are inconsistent

- The page shows product-level stock (sum of SKUs, or a separate field), while the order check uses SKU-level stock. If these differ, the page can show 1 while the selected SKU actually has 0.
- Example: Product stock = 1 comes from the White SKU, but the customer selected Black (stock 0).
- **Pattern:** The issue only appears for specific SKUs. It may come from a stock-sync bug or from different data sources for the two numbers.

### Cause E: Front-end SKU selection mismatch

- The page displays the stock of one SKU, but the request sends a different skuId (for example, a default SKU, or a stale selection after the user switched color).
- **Pattern:** The wrong SKU appears in the request payload or in the order confirmation page.

### Cause F: Quantity or cart problem

- The user's cart quantity is larger than 1 (for example, 2 units added earlier), while the page shows stock = 1. Then "insufficient inventory" is correct for that quantity.
- The cart may also contain an item that has become invalid (taken off sale or SKU disabled), and the error message is attached to the wrong item.
- **Pattern:** The submitted quantity differs from what the customer believes.

### Cause G: Inventory data inconsistency or system defect

- Stock-sync delay between the inventory system, the product system and the database (for example, after a manual adjustment or import).
- Stock locks that were never released (a bug in timeout or cancel handling), which leaves "ghost" locks that reduce available stock permanently.
- Double deduction (a deduction at lock time and again at payment).
- **Pattern:** Stock stays wrong over time, and the same product reproduces the issue repeatedly.

### Cause H: Order submission logic problem

- The submit API reads stock from a different source than the detail page (for example a replica database with replication lag).
- Rule differences (for example, the order check includes a safety buffer or reserved stock that the page does not subtract).
- The submit API checks stock **but the detail page API never re-checks**. This is the expected design, since the back end is the final check.

## 3. How I Would Investigate

### Step 1: Gather facts

| Question | Why it matters |
|---|---|
| Order number, user ID, product ID, SKU ID, quantity, exact time | Needed to trace logs and data |
| What did the customer see and submit (screenshots or recorded request)? | Separates a display problem from a logic problem |
| Reproducible? Always, sometimes, or once? For one product or many? | Distinguishes systemic bug vs. race condition |
| Time between viewing the page and tapping Submit | A long gap points to Cause A |

### Step 2: Check the data at the time of the incident

| Check | What to look for | Points to cause |
|---|---|---|
| Inventory table for the SKU: total, locked, available | available = 0 while total = 1 | C |
| Stock change log around the incident time | Another order, lock, manual adjustment or sync job | A, C, G |
| Orders for this SKU in Pending Payment status | An unpaid order holding the last unit | C |
| SKU-level stock vs. product-level stock | Mismatch between the two | D |
| Request payload of the submit call | Wrong skuId or quantity | E, F |

### Step 3: Check the system

| Check | What to look for | Points to cause |
|---|---|---|
| Detail page API response vs. database value | Page number differs from database | B |
| Cache settings and last refresh time of stock | Long cache duration or no invalidation after stock change | B |
| Submit API logs and the exact stock query it used | Which table, which field, replica or primary | H |
| Lock release job logs | Locks not released after timeout or cancel | G |
| Error code returned by the server | Which rule produced "Insufficient inventory" | Narrow to H or C |

### Step 4: Decide the verdict

| Finding | Verdict | Action |
|---|---|---|
| Another buyer took the unit, or an unpaid order holds it | **Expected behavior**, poor experience | Improve the messaging and stock display (see below) |
| Cache stale, or SKU/product mismatch | **Real bug (display data)** | Fix cache invalidation and data source, with an engineering ticket |
| Leaked locks, double deduction, replica lag | **Real bug (inventory logic)** | High priority, since it causes lost sales and wrong stock |

## 4. Product Fixes and Improvements

| # | Improvement | Why |
|---|---|---|
| 1 | **Show available stock** (total − locked) on the detail page, not raw total stock | Closes the gap between page and checkout (Cause C) |
| 2 | **Short cache or event-based invalidation** for stock, and re-fetch stock when the page returns to the foreground or when SKU changes | Reduces stale numbers (Cause B) |
| 3 | **Re-validate stock at checkout entry**, not only at submit | Customer learns earlier, before filling in the address |
| 4 | **Atomic stock lock at submission** (server-side, final authority) | Guarantees no overselling, whatever the page shows |
| 5 | **Clear, actionable error message:** "Sorry, this item was just taken by another customer. Only 0 left." with buttons such as "Choose another option" or "Notify me when available" | Turns a dead end into a next step. The current message does not say why or what to do. |
| 6 | **Show low-stock urgency** ("Only 1 left, selling fast") | Sets the expectation that stock can change at any moment |
| 7 | **Single source of truth for stock**, and a consistency check between SKU stock and product stock | Prevents Cause D permanently |
| 8 | **Monitoring and alerts:** stock-mismatch alerts, unreleased-lock alerts, and a metric for "insufficient inventory at submit" rate | A sudden rise in this rate signals a systemic bug early |
| 9 | **Optional:** a short reservation when the customer enters checkout (for example 5 minutes) for high-demand items | Reduces conflicts, but it needs care because it can block sales if abused |
| 10 | **Log the reason code** in the API response (e.g., OUT_OF_STOCK, LOCKED_BY_OTHERS, SKU_MISMATCH) | Makes future troubleshooting fast and helps the support team |

## 5. Summary

- The stock number on the product page is **a snapshot, not a guarantee**. Only the server-side check at submission is authoritative.
- The **most likely causes** are another customer buying the last unit, or an unpaid order holding a lock. Both are normal behavior, and the fix is a better experience.
- The **bugs worth ruling out** are stale cache, SKU vs. product stock mismatch, a wrong SKU in the request, and locks that were never released.
- My investigation order is: **collect the facts → compare page data, database and lock records at the incident time → read the submit API logs → classify the case as expected or a bug → fix the experience and the data**.
