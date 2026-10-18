# Part 2, Q7: Order Status Understanding Question

E-commerce System MVP – Product Manager Assignment

## 1. Initial status after a customer submits an order

**PENDING_PAYMENT**

- The order record has been created, stock is locked, and the system is waiting for the customer to pay.
- The order gets a **payment deadline** (e.g., 30 minutes) at this moment.
- No money has moved yet, so the order is not PAID.

## 2. Status after payment is successful

**PAID**

- The status changes **only after the payment gateway's asynchronous callback has been verified** (signature, order number and amount). The "payment successful" page in the browser is never trusted on its own.
- Side effects: the locked stock is converted into an actual deduction, and the order appears in the operations team's "To Ship" list.

## 3. Status after the merchant ships the order

**SHIPPED**

- Operations selects the carrier and enters the tracking number, then confirms shipment.
- The tracking number is saved and shown to the customer.
- The change is conditional: it is only allowed from PAID. If the customer cancelled a moment earlier, the shipment action fails with "Order has been cancelled" and the merchant stops packing.

## 4. Status if the customer does not pay within the time limit

**CANCELLED**

- A scheduled job closes the order after the deadline, with the reason "Payment timeout".
- Locked stock is released back to sellable inventory.
- The customer is notified, and the order stays in the order history.
- **Edge case:** If the customer pays at the same moment the order closes, the payment callback arrives for a CANCELLED order. The safe rule is an **automatic full refund** to the original payment method, so the customer is never charged without an order.

## 5. In which statuses can the customer request a refund?

| Status | Can customer request a refund? | Handling |
|---|---|---|
| PENDING_PAYMENT | No | Nothing has been paid. The customer cancels the order instead, and it becomes CANCELLED. |
| **PAID** | **Yes (core MVP case)** | Goods have not shipped. The request is auto-approved, and the order moves to REFUNDING. |
| SHIPPED | Not in MVP (later iteration) | Goods are on the way. A refund here needs a return process and merchant review, so it belongs to the after-sales flow (P2). The customer should be told "Please request after-sales once you receive the item." |
| COMPLETED | Not in MVP (later iteration) | Possible within an after-sales window (e.g., 7 days) with review and return logistics. Outside the MVP scope. |
| CANCELLED | No | The order is already closed with nothing to refund (or an automatic refund was already triggered). |
| REFUNDING | No | A refund is already in progress. Duplicate requests are blocked. |
| REFUNDED | No | Already refunded. |

**Summary:** In the MVP, a customer can request a refund **only in PAID**. After shipment, refunds move to the after-sales process, which I scoped out of the first version.

## 6. Order status transition flow

### Main flow (happy path)

```
PENDING_PAYMENT --(pay success)--> PAID --(merchant ships)--> SHIPPED --(customer confirms / auto-confirm)--> COMPLETED
```

### Exception and branch flows

```
PENDING_PAYMENT --(customer cancels or payment timeout)--> CANCELLED

PAID --(customer cancels before shipment, or merchant cancels e.g. out of stock)--> REFUNDING --(gateway confirms)--> REFUNDED

REFUNDING --(refund call fails)--> stays REFUNDING, system retries, then manual handling by admin --> REFUNDED
```

### Full transition table

| From | Trigger | To | Side Effects |
|---|---|---|---|
| (none) | Customer submits order, stock lock succeeds | PENDING_PAYMENT | Stock locked, payment deadline set |
| PENDING_PAYMENT | Verified payment success callback | PAID | Stock deduction confirmed, order appears in "To Ship" list |
| PENDING_PAYMENT | Customer cancels, or payment deadline passes | CANCELLED | Locked stock released |
| PAID | Merchant confirms shipment | SHIPPED | Tracking number saved, customer notified |
| PAID | Customer cancels (auto-approved), or merchant cancels | REFUNDING | Stock restored, refund request sent to gateway, removed from "To Ship" list |
| REFUNDING | Gateway confirms refund | REFUNDED | Customer notified, order closed |
| SHIPPED | Customer confirms receipt, or auto-confirm after N days | COMPLETED | Order closed |

**Terminal statuses (no further transitions):** COMPLETED, CANCELLED, REFUNDED.

### Rules that keep the status machine safe

1. **Every transition is conditional.** A change only succeeds if the order is currently in the expected "from" status. This resolves conflicts such as "customer cancels while merchant ships": exactly one action wins.
2. **No skipping and no going backward.** For example, an order cannot jump from PENDING_PAYMENT to SHIPPED, and SHIPPED can never return to PAID.
3. **Payment callbacks and refund requests are idempotent.** Repeated callbacks or double taps do not create duplicate state changes or duplicate refunds.
4. **Every change is logged** with timestamp, operator and reason, which supports customer service and dispute handling.
5. **Why CANCELLED and REFUNDED are separate end states:** CANCELLED means the order was closed **without any money being collected**. REFUNDED means money **was collected and then returned**. Finance needs this distinction for reconciliation.

### Consistency note with my earlier process design (Part 1.3)

Part 1.3 used a separate refund status next to the order status. This question defines a single status list that includes REFUNDING and REFUNDED, so here I merged refund progress into the order status. The business behavior is the same: a paid order that is cancelled goes through a refund before it is closed.
