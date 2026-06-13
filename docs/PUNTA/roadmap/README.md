# Punta — 12-Week Delivery Roadmap

## Overview

| Week | Phase | Focus |
|:-----|:------|:------|
| 1 | Foundation | Rust workspace setup, PostgreSQL schema, Docker environment |
| 2 | Foundation | Auth system, tenant resolution, RLS enforcement |
| 3 | Foundation | Next.js setup, design system, auth pages, dashboard layout |
| 4 | Core Commerce | Product catalog, categories, image uploads |
| 5 | Core Commerce | Orders, cart, checkout, invoice generation |
| 6 | Core Commerce | Payments (Paystack + Flutterwave), inventory management |
| 7 | CRM & Customers | Customer management, groups, segmentation |
| 8 | Messaging | WhatsApp, SMS, email integration, campaigns |
| 9 | Analytics & Reports | Dashboard metrics, sales/product/customer analytics |
| 10 | Expenses & Delivery | Expense tracking, logistics integration, tracking |
| 11 | Storefront & Store | Public storefront (SSR), theming, multi-location |
| 12 | Wallet & Launch | Wallet setup, pay-as-you-grow billing, polish, deploy |

## Daily Schedule

Each day targets **6-8 hours of focused work**.

See individual week files for day-by-day breakdowns:

| File | Week |
|:-----|:-----|
| [week-01.md](./week-01.md) | Foundation — Backend Setup |
| [week-02.md](./week-02.md) | Foundation — Auth & Tenancy |
| [week-03.md](./week-03.md) | Foundation — Frontend Setup |
| [week-04.md](./week-04.md) | Core Commerce — Catalog |
| [week-05.md](./week-05.md) | Core Commerce — Orders & Checkout |
| [week-06.md](./week-06.md) | Core Commerce — Payments & Inventory |
| [week-07.md](./week-07.md) | CRM & Customers |
| [week-08.md](./week-08.md) | Messaging Engine |
| [week-09.md](./week-09.md) | Analytics & Reports |
| [week-10.md](./week-10.md) | Expenses & Delivery |
| [week-11.md](./week-11.md) | Storefront & Store Management |
| [week-12.md](./week-12.md) | Wallet, Polish & Launch |

## Dependencies

```
Week 1 ──▶ Week 2 ──▶ Week 3  (Foundation must be sequential)
                         │
              ┌──────────┼──────────┐
              ▼          ▼          ▼
           Week 4     Week 7     Week 9   (Can partially parallelize)
              │          │          │
              ▼          │          │
           Week 5       │          │
              │          │          │
              ▼          ▼          │
           Week 6     Week 8      │
              │          │          │
              └──────────┼──────────┘
                         │
                         ▼
                      Week 10
                         │
                         ▼
                      Week 11
                         │
                         ▼
                      Week 12
```
