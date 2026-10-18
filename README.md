# IT Product Manager Assessment: E-commerce System MVP

**Candidate:** HUWEI  
**Position:** IT Product Manager  
**Contact:** 2295048910@qq.com

This repository contains my answers to the Product Manager Assignment *"E-commerce System MVP Design & Basic Technical Understanding"*.

## Overview

The assignment asks for an MVP design of a basic e-commerce system that supports the core transaction flow:

**Browse Products → Add to Cart / Buy Now → Submit Order → Pay → Ship → Complete Order**

My answers follow one principle: **the MVP must prove the core transaction loop works end to end** (a customer can discover, buy and receive a product, and the team can fulfill it) before investing in growth features. Anything outside that loop is deliberately deferred.

## Contents

### Part 1: Product Design

| # | Topic | File |
|---|---|---|
| 1.1 | User Roles and Core Scenarios | [1.1-user-roles-and-core-scenarios.md](1.1-user-roles-and-core-scenarios.md) |
| 1.2 | MVP Feature List and Priority (P0 / P1 / P2) | [1.2-mvp-feature-list-and-priority.md](1.2-mvp-feature-list-and-priority.md) (Excel version: [.xlsx](1.2-mvp-feature-list-and-priority.xlsx)) |
| 1.3 | Core Process Design: Order and Payment, Order Cancellation | [1.3-core-process-design.md](1.3-core-process-design.md) |
| 1.4 | Simplified PRD: Submit Order Page | [1.4-simplified-prd-submit-order-page.md](1.4-simplified-prd-submit-order-page.md) |

> Q1.2 is provided as a readable Markdown page, plus an optional Excel workbook with the same 34 features and an auto-calculated priority summary. GitHub does not preview `.xlsx` files, so please download the workbook to view it.

### Part 2: Code / Technical Understanding

| # | Topic | File |
|---|---|---|
| 5 | API Understanding Question | [q5-api-understanding.md](q5-api-understanding.md) |
| 6 | Simple Logic Reading Question | [q6-simple-logic-reading.md](q6-simple-logic-reading.md) |
| 7 | Order Status Understanding Question | [q7-order-status.md](q7-order-status.md) |
| 8 | Bug / Exception Analysis Question | [q8-bug-exception-analysis.md](q8-bug-exception-analysis.md) |

### Part 3: Data and Metrics

| # | Topic | File |
|---|---|---|
| 9 | Core Metrics Design | [q9-core-metrics-design.md](q9-core-metrics-design.md) |

## Key Design Decisions at a Glance

1. **Roles by flow:** Customer is the priority, Operations second, and Platform Admin kept minimal (permissions and exception handling only).
2. **Priority by dependency:** P0 means the basic transaction flow cannot work without it. Growth features such as coupons, reviews and recommendations are P2.
3. **Stock is locked at order submission, not at Add to Cart.** This prevents overselling without letting abandoned carts block real sales.
4. **Every order status change is conditional.** A transition only succeeds from the expected "from" status, so conflicts (for example, customer cancels while operations ships) have exactly one winner.
5. **Back end is the source of truth.** Page data (stock, price, status) is a snapshot, and the server re-validates everything at submission.
6. **Prices are integers in the smallest currency unit**, converted only for display.
7. **Metrics follow the funnel:** eight metrics cover traffic, conversion, payment, sales, fulfillment, cancellations and inventory accuracy.

## How to Read This Repository

- Start with Part 1.1 for the roles and scope, then 1.2 for what is in and out of the MVP.
- Part 1.3 and Part 2, Q7 describe the same order lifecycle from two angles (process design and status model).
- Part 2, Q8 connects back to the stock-lock design in Parts 1.3 and 1.4.
- All answers are written in English, as required by the assessment.

---

*Prepared for the Product Manager assessment. No confidential material from the assessment has been reproduced here, only my own answers.*
