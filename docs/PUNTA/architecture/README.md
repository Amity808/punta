# Punta — Architecture Documentation

> A Bumpa-inspired multi-tenant SaaS platform for Nigerian SMBs to manage their entire business operations.

## Quick Reference

| Decision | Choice |
|:---|:---|
| **Target Market** | Nigeria-first |
| **Backend** | Rust (Axum 0.8) — Modular Monolith |
| **Frontend** | Next.js 15 (App Router) — Web only |
| **Database** | PostgreSQL 16 + Redis 7 |
| **Multi-Tenancy** | Shared schema + Row-Level Security |
| **Tenant Routing** | Subdomain-based (`store.punta.shop`) |
| **Payments** | Paystack + Flutterwave |
| **Billing** | Custom-built (Starter / Pro / Growth tiers) |
| **POS** | Not in scope (future) |
| **Mobile App** | Not in scope (future) |

## Documentation Index

| # | Document | Description |
|:--|:---------|:------------|
| 01 | [System Overview](./01-system-overview.md) | High-level architecture, C4 diagrams, component map |
| 02 | [Tech Stack](./02-tech-stack.md) | Technology choices with rationale |
| 03 | [Multi-Tenancy](./03-multi-tenancy.md) | Tenant isolation strategy, RLS, subdomain routing |
| 04 | [Database Schema](./04-database-schema.md) | Complete PostgreSQL schema with all tables |
| 05 | [API Design](./05-api-design.md) | RESTful API endpoints, request/response formats |
| 06 | [Module Breakdown](./06-module-breakdown.md) | Rust workspace crate structure |
| 07 | [Frontend Architecture](./07-frontend-architecture.md) | Next.js app structure, components, state management |
| 08 | [Infrastructure](./08-infrastructure.md) | Deployment, CI/CD, monitoring, scaling |
| 09 | [Security](./09-security.md) | Auth, RBAC, data protection, compliance |
| 10 | [Integrations](./10-integrations.md) | External services (Paystack, WhatsApp, logistics) |
| 11 | [Load Balancing](./11-load-balancer.md) | Load balancing principles, algorithms, and application routing |

## Roadmap

See [../roadmap/README.md](../roadmap/README.md) for the 12-week delivery plan with day-by-day breakdowns.
