# 01 — System Overview

## What is Punta?

Punta is a multi-tenant SaaS platform that provides Nigerian small and medium businesses with tools to:

1. **Sell online** — Create a branded storefront at `storename.punta.shop`
2. **Manage inventory** — Track stock across multiple locations
3. **Process payments** — Accept card, bank transfer, USSD via Paystack/Flutterwave
4. **Manage customers** — CRM with segmentation, purchase history, and notes
5. **Communicate** — WhatsApp, SMS, and email campaigns
6. **Track finances** — Orders, expenses, invoices, P&L reports
7. **Fulfill orders** — Integrated logistics and delivery tracking
8. **Analyze performance** — Sales dashboards, product and customer insights

## System Architecture Diagram

```
┌──────────────────────────────────────────────────────────────────────────┐
│                           CLIENT LAYER                                   │
│                                                                          │
│  ┌─────────────────────┐     ┌──────────────────────────────────────┐   │
│  │   Merchant Dashboard │     │         Public Storefront            │   │
│  │   (Next.js SSR/SPA)  │     │   (Next.js SSR - *.punta.shop)      │   │
│  │                      │     │                                      │   │
│  │  • Products          │     │  • Product listings                  │   │
│  │  • Orders            │     │  • Shopping cart                     │   │
│  │  • Customers         │     │  • Checkout                          │   │
│  │  • Inventory         │     │  • Order tracking                    │   │
│  │  • Messaging         │     │                                      │   │
│  │  • Analytics         │     │                                      │   │
│  │  • Settings          │     │                                      │   │
│  └────────┬─────────────┘     └──────────────┬───────────────────────┘   │
│           │                                   │                          │
└───────────┼───────────────────────────────────┼──────────────────────────┘
            │             HTTPS                 │
            ▼                                   ▼
┌──────────────────────────────────────────────────────────────────────────┐
│                        EDGE / CDN LAYER                                  │
│                                                                          │
│  ┌──────────────────────────────────────────────────────────────────┐   │
│  │  Cloudflare (CDN + Wildcard DNS for *.punta.shop + SSL)         │   │
│  └──────────────────────────────────────────────────────────────────┘   │
│                                                                          │
└──────────────────────────────┬───────────────────────────────────────────┘
                               │
                               ▼
┌──────────────────────────────────────────────────────────────────────────┐
│                       APPLICATION LAYER                                  │
│                       Rust (Axum 0.8)                                    │
│                                                                          │
│  ┌────────────────────────────────────────────────────────────────────┐ │
│  │                    MIDDLEWARE PIPELINE                              │ │
│  │  Request → Rate Limiter → Auth (JWT) → Tenant Resolver → Handler  │ │
│  └────────────────────────────────────────────────────────────────────┘ │
│                                                                          │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐    │
│  │  Auth    │ │ Catalog  │ │ Orders   │ │ Payments │ │   CRM    │    │
│  │ Service  │ │ Service  │ │ Service  │ │ Service  │ │ Service  │    │
│  └──────────┘ └──────────┘ └──────────┘ └──────────┘ └──────────┘    │
│                                                                          │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐    │
│  │Inventory │ │Messaging │ │Analytics │ │ Expenses │ │ Delivery │    │
│  │ Service  │ │ Service  │ │ Service  │ │ Service  │ │ Service  │    │
│  └──────────┘ └──────────┘ └──────────┘ └──────────┘ └──────────┘    │
│                                                                          │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐                               │
│  │  Store   │ │  Tenant  │ │  Wallet  │                               │
│  │ Service  │ │ Service  │ │ Service  │                               │
│  └──────────┘ └──────────┘ └──────────┘                               │
│                                                                          │
│  ┌────────────────────────────────────────────────────────────────────┐ │
│  │              Background Worker (Tokio + Redis Queue)                │ │
│  │  • Send notifications  • Aggregate analytics  • Process webhooks   │ │
│  │  • Low stock checks    • Scheduled campaigns  • Invoice generation │ │
│  └────────────────────────────────────────────────────────────────────┘ │
│                                                                          │
└─────────┬──────────────┬──────────────┬──────────────────────────────────┘
          │              │              │
          ▼              ▼              ▼
┌──────────────────────────────────────────────────────────────────────────┐
│                         DATA LAYER                                       │
│                                                                          │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐                  │
│  │ PostgreSQL 16│  │   Redis 7    │  │ Object Store │                  │
│  │              │  │              │  │  (S3 / R2)   │                  │
│  │ • All data   │  │ • Sessions   │  │              │                  │
│  │ • RLS tenant │  │ • Cache      │  │ • Images     │                  │
│  │   isolation  │  │ • Job queue  │  │ • Invoices   │                  │
│  │ • Full-text  │  │ • Rate limits│  │ • Receipts   │                  │
│  │   search     │  │ • Pub/Sub    │  │              │                  │
│  └──────────────┘  └──────────────┘  └──────────────┘                  │
│                                                                          │
└──────────────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────────────┐
│                     EXTERNAL SERVICES                                    │
│                                                                          │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐    │
│  │ Paystack │ │Flutterwave│ │ WhatsApp │ │  Termii  │ │  Resend  │    │
│  │ (Cards,  │ │ (Cards,   │ │Cloud API │ │  (SMS)   │ │ (Email)  │    │
│  │ Transfer)│ │ Transfer) │ │          │ │          │ │          │    │
│  └──────────┘ └──────────┘ └──────────┘ └──────────┘ └──────────┘    │
│                                                                          │
│  ┌──────────────┐                                                       │
│  │ Shipbubble   │                                                       │
│  │ (Logistics)  │                                                       │
│  └──────────────┘                                                       │
│                                                                          │
└──────────────────────────────────────────────────────────────────────────┘
```

## Request Flow

```
Customer visits acme.punta.shop/products
         │
         ▼
   Cloudflare DNS
   (*.punta.shop → App Server)
         │
         ▼
   Axum receives request
   Host header: "acme.punta.shop"
         │
         ▼
   Rate Limiter Middleware
   (Redis: check tenant request count)
         │
         ▼
   Tenant Resolver Middleware
   (Redis cache → PostgreSQL fallback)
   Extracts "acme" → resolves tenant_id = "t_abc123"
         │
         ▼
   Sets PostgreSQL session variable:
   SET app.current_tenant = 't_abc123'
         │
         ▼
   Handler: GET /api/v1/storefront/products
   Executes: SELECT * FROM products WHERE is_active = true
         │
   RLS automatically appends:
   AND tenant_id = 't_abc123'
         │
         ▼
   Returns only ACME's products
```

## Component Responsibilities

| Component | Responsibility |
|:----------|:--------------|
| **Auth Service** | Registration, login, JWT tokens, password reset, RBAC |
| **Tenant Service** | Tenant CRUD, onboarding wizard, feature unlocks |
| **Store Service** | Store settings, theme, custom domain, SEO |
| **Catalog Service** | Products, variants, categories, images, search |
| **Inventory Service** | Stock levels, adjustments, transfers, low-stock alerts |
| **Order Service** | Cart, checkout, order lifecycle, invoices/receipts |
| **Payment Service** | Gateway abstraction, webhook handling, merchant wallet settlements, platform fee collection |
| **CRM Service** | Customer profiles, groups/segments, purchase history |
| **Messaging Service** | WhatsApp, SMS, email — templates, campaigns, dispatch |
| **Analytics Service** | Dashboard metrics, sales/product/customer reports |
| **Expense Service** | Expense tracking, categories, P&L reporting |
| **Delivery Service** | Logistics provider integration, rate quotes with markup, tracking |
| **Wallet Service** | Merchant wallet balance ledger, KYC verification (BVN/NIN/CAC) & limits, withdrawal processing, utility billing & add-on checks |
| **Background Worker** | Async job processing, scheduled tasks, event handlers |
