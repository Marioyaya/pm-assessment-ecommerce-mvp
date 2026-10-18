# Part 3, Q9: Core Metrics Design

E-commerce System MVP – Product Manager Assignment

**Approach:** The metrics follow the transaction funnel (Browse → Cart → Order → Pay → Ship → Complete), so each stage has at least one number telling us whether it works. I chose 8 metrics. The five in the examples are the baseline, and I added three that watch fulfillment, inventory health and customer experience, since the MVP can fail after payment as well as before it.

## Core Metrics Table

| # | Metric | Definition (Formula) | Why It Matters | What to Do With It |
|---|---|---|---|---|
| 1 | **Product Detail Page (PDP) Views / Unique Visitors** | Number of product detail page views, and the number of unique users who viewed at least one | Measures traffic and product exposure. It is the top of the funnel, and every other rate depends on it. | A drop points to acquisition, search or category problems, not to checkout problems. |
| 2 | **Add-to-Cart Rate** | Users who add to cart ÷ users who view product details | Measures how attractive the product page is (price, images, description, SKU choices) | Low rate: improve content, pricing, images and trust signals. |
| 3 | **Order Conversion Rate** | Users who submit an order ÷ users who view product details (also track cart-to-order separately) | Measures purchase intent and the effectiveness of the cart and checkout flow | Big drop between cart and order points to checkout friction (address entry, unexpected shipping fees, stock errors). |
| 4 | **Payment Success Rate** | Paid orders ÷ submitted orders (also split into "payment attempts that succeed" vs. "orders abandoned") | Measures whether the payment flow works and whether customers follow through | Low rate: check payment errors, gateway stability, timeouts, and the payment deadline length. |
| 5 | **GMV and Average Order Value (AOV)** | GMV = total value of paid orders. AOV = GMV ÷ number of paid orders | Measures sales scale and order quality. GMV alone can grow through volume while AOV falls, so I watch both. | Drives decisions on pricing, bundling and free-shipping thresholds. |
| 6 | **Order Fulfillment Rate and Shipping Time** | Shipped orders ÷ paid orders within the promised time (e.g., 48 hours), plus average hours from Paid to Shipped | Measures whether operations can deliver on what customers paid for. Slow shipping creates complaints and cancellations. | Low rate: staffing or process problem. Also a leading indicator of customer satisfaction. |
| 7 | **Cancellation and Refund Rate** (with reason breakdown) | Cancelled or refunded orders ÷ total orders, split by reason: payment timeout, out of stock, customer changed mind, other | Shows lost revenue and hidden problems. A high "out of stock" share means inventory errors. A high "payment timeout" share means checkout friction. | The reason breakdown tells us whether to fix inventory, payment flow or product accuracy. |
| 8 | **Stock-out Error Rate** (insufficient-inventory failures at submit) | Order submissions rejected for insufficient stock ÷ total submission attempts | Measures inventory accuracy and the gap between page stock and real stock. A rise signals stale cache, lock leaks or sync bugs (the issue from Q8). | Spikes trigger an engineering investigation. It also shows lost sales from popular items running out. |

## How the Metrics Connect (Funnel View)

```
PDP Views → Add-to-Cart Rate → Order Conversion Rate → Payment Success Rate → Fulfillment Rate → Cancellation/Refund Rate
   (1)             (2)                  (3)                     (4)                   (6)                    (7)

Side guards:  Stock-out Error Rate (8)  |  GMV & AOV (5) = overall outcome
```

**How to read them together:**

| Pattern | Likely Problem |
|---|---|
| High views, low add-to-cart | Product page content or pricing |
| High add-to-cart, low order conversion | Checkout friction, shipping costs, address entry, stock errors |
| Order conversion fine, low payment success | Payment channel, timeout, or payment UX |
| High payment success, low fulfillment rate | Operations capacity or shipping process |
| GMV up but refund rate also up | Quality or inventory problem hidden by growth |

## Practical Notes

- **Segment every metric** by new vs. returning users, device (mobile vs. desktop), product category and traffic source. An average can hide a failing segment.
- **Use a consistent definition and time window** for each rate (for example, same-session or same-day), so numbers can be compared week over week.
- **North Star metric for the MVP:** the number of **successfully completed orders** (paid and delivered). The MVP's goal is to prove the whole loop works, not to maximize traffic.
- **Early benchmarks should be set from actual data.** In the first weeks, the trend matters more than absolute values, since there is no historical baseline.
- **Alerts:** Set thresholds on Payment Success Rate, Stock-out Error Rate and Fulfillment Rate, because these three fail silently and cost money directly.
