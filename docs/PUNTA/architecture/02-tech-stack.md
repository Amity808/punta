# 02 — Technology Stack

## Backend: Rust + Axum 0.8

### Why Rust?
- **Memory safety** without garbage collection — critical for a multi-tenant system where tenant isolation bugs can be catastrophic
- **Performance** — sub-millisecond response times, handles thousands of concurrent tenants on modest hardware
- **Type system** — compile-time guarantees prevent entire classes of bugs (null safety, exhaustive matching)
- **Async runtime** (Tokio) — efficient handling of I/O-bound workloads (DB queries, HTTP calls, WebSockets)

### Why Axum?
- Built on Tokio + Tower — the most mature async ecosystem in Rust
- **Extractor-based** handler design — clean, type-safe request parsing
- **Middleware composition** — Tower layers for auth, tenancy, rate limiting
- Excellent **WebSocket** support for real-time dashboard updates
- Active maintenance by the Tokio team

### Key Crates

| Crate | Purpose | Version |
|:------|:--------|:--------|
| `axum` | Web framework | 0.8.x |
| `tokio` | Async runtime | 1.x |
| `sqlx` | Async PostgreSQL driver (compile-time checked) | 0.8.x |
| `redis` | Redis client | 0.27.x |
| `serde` / `serde_json` | Serialization | 1.x |
| `jsonwebtoken` | JWT encoding/decoding | 9.x |
| `argon2` | Password hashing | 0.5.x |
| `uuid` | UUID generation | 1.x |
| `chrono` | Date/time handling | 0.4.x |
| `thiserror` | Error type definitions | 2.x |
| `tracing` / `tracing-subscriber` | Structured logging | 0.1.x |
| `tower` | Middleware framework | 0.5.x |
| `tower-http` | HTTP middleware (CORS, compression, tracing) | 0.6.x |
| `reqwest` | HTTP client (for external APIs) | 0.12.x |
| `aws-sdk-s3` | S3/R2 object storage | Latest |
| `lettre` | SMTP email (fallback) | 0.11.x |
| `validator` | Request validation | 0.19.x |

---

## Database: PostgreSQL 16

### Why PostgreSQL?
- **Row-Level Security (RLS)** — database-enforced tenant isolation
- **JSONB** — flexible metadata storage without schema rigidity
- **Full-text search** (`tsvector` + `GIN`) — product search without an external service
- **Trigram search** (`pg_trgm`) — fuzzy matching for customer/product lookup
- **ACID compliance** — critical for financial operations (payments, wallet, orders)
- **Mature ecosystem** — battle-tested at scale, excellent tooling

### Key PostgreSQL Features Used

| Feature | Use Case |
|:--------|:---------|
| RLS policies | Multi-tenant data isolation |
| JSONB columns | Store settings, theme config, addresses, metadata |
| `tsvector` + `GIN` | Product and customer search |
| `pg_trgm` | Fuzzy/typo-tolerant search |
| Generated columns | Auto-compute `search_vector` from name + description |
| Partial indexes | Index only active products, unpaid orders, etc. |
| `LISTEN/NOTIFY` | Real-time event triggers (backup to Redis pub/sub) |
| Table partitioning | Future: partition `orders` and `messages` by date range |

### Query Layer: SQLx

We use **SQLx** instead of an ORM (like Diesel or SeaORM) because:
- **Compile-time SQL verification** — queries checked against the actual DB schema at build time
- **Raw SQL** — full control over query optimization, no ORM abstraction leaks
- **Async-native** — built for Tokio
- **Migration system** — built-in, versioned SQL migrations

---

## Cache & Queue: Redis 7

### Responsibilities

| Function | Details |
|:---------|:--------|
| **Session store** | JWT refresh token storage with TTL |
| **Cache** | Tenant resolution cache, product catalog cache, dashboard metrics |
| **Rate limiting** | Per-tenant request rate limiting (sliding window) |
| **Job queue** | Background task queue (notification dispatch, analytics aggregation) |
| **Pub/Sub** | Internal event bus (order.created, payment.received, etc.) |
| **Distributed locks** | Prevent double-processing of webhooks |

### Cache Key Naming Convention

All cache keys are tenant-scoped to prevent cross-tenant contamination:

```
punta:{tenant_id}:{resource}:{identifier}

Examples:
punta:t_abc123:tenant:meta           # Tenant metadata
punta:t_abc123:products:list:page_1  # Product listing cache
punta:t_abc123:dashboard:overview    # Dashboard metrics
punta:global:tenant:slug:acme        # Slug → tenant_id lookup (global)
```

---

## Frontend: Next.js 15 (App Router)

### Why Next.js?
- **SSR for storefronts** — SEO-critical public store pages rendered server-side
- **App Router** — layouts, loading states, error boundaries built-in
- **React Server Components** — reduce client-side JavaScript for faster stores
- **API Routes** — BFF (Backend For Frontend) pattern for proxying to Rust API
- **Image optimization** — built-in `next/image` for product photos
- **Nigeria-friendly** — works well on slower connections with streaming SSR

### Key Frontend Libraries

| Library | Purpose |
|:--------|:--------|
| `react-hook-form` + `zod` | Form handling with schema validation |
| `@tanstack/react-query` | Server state management, caching, mutations |
| `recharts` or `nivo` | Analytics charts and dashboards |
| `zustand` | Lightweight client state (cart, UI state) |
| `sonner` | Toast notifications |
| `date-fns` | Date formatting (with locale support) |
| `next-themes` | Dark/light mode support |

---

## Object Storage: Cloudflare R2

### Why R2?
- **Zero egress fees** — critical for media-heavy e-commerce (product images)
- **S3-compatible API** — drop-in replacement, use `aws-sdk-s3` crate
- **Global CDN** — images served from Cloudflare's edge network
- **Cost-effective** — significantly cheaper than AWS S3 for read-heavy workloads

### Upload Strategy
1. Frontend requests a **presigned upload URL** from the Rust API
2. Frontend uploads directly to R2 (bypasses backend for large files)
3. On upload completion, frontend sends the object key to the backend
4. Backend validates and stores the URL reference in PostgreSQL

---

## Email: Resend

- Transactional emails (order confirmation, password reset, invoice)
- Marketing emails (campaigns, promotional)
- Template-based with variable substitution
- Delivery tracking and analytics

## SMS: Termii

- Nigeria-focused SMS provider
- OTP delivery for phone verification
- Order notifications for customers without WhatsApp
- Campaign messaging

## WhatsApp: Cloud API via BSP (Twilio recommended)

- Order confirmations and status updates
- Customer support conversations
- Marketing campaigns (opt-in required)
- Interactive buttons for quick actions

---

## Development Tooling

| Tool | Purpose |
|:-----|:--------|
| `cargo watch` | Auto-rebuild on file changes |
| `sqlx-cli` | Database migrations |
| `docker-compose` | Local PostgreSQL + Redis |
| `just` (justfile) | Task runner (build, test, migrate, seed) |
| `cargo clippy` | Linting |
| `cargo fmt` | Code formatting |
| `tracing` | Structured JSON logging |
| `cargo nextest` | Fast parallel test runner |
