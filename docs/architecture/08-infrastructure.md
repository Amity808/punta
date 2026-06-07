# 08 — Infrastructure & Deployment

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                        Cloudflare                                │
│  • Wildcard DNS (*.punta.shop)                                   │
│  • SSL termination                                               │
│  • CDN for static assets                                         │
│  • DDoS protection                                               │
│  • R2 object storage (product images)                            │
└──────────────────────────┬──────────────────────────────────────-┘
                           │
              ┌────────────┴────────────┐
              │                         │
              ▼                         ▼
    ┌─────────────────┐     ┌─────────────────┐
    │   Vercel         │     │   Fly.io / ECS   │
    │   (Next.js)      │     │   (Rust API)     │
    │                  │     │                  │
    │   • SSR          │     │   • 2+ instances │
    │   • Edge runtime │     │   • Auto-scale   │
    │   • Dashboard    │     │   • Health check │
    │   • Storefront   │     │                  │
    └─────────────────┘     └────────┬─────────┘
                                     │
                    ┌────────────────┼───────────────┐
                    │                │               │
                    ▼                ▼               ▼
          ┌──────────────┐  ┌──────────────┐  ┌──────────────┐
          │  PostgreSQL   │  │    Redis      │  │ Cloudflare   │
          │  (Neon)       │  │  (Upstash)    │  │ R2           │
          │               │  │               │  │              │
          │  • Primary    │  │  • Serverless │  │  • Images    │
          │  • Read       │  │  • Global     │  │  • Invoices  │
          │    replica    │  │  • Durable    │  │  • Receipts  │
          └──────────────┘  └──────────────┘  └──────────────┘
```

---

## Recommended Stack

### Compute

| Service | Provider | Rationale |
|:--------|:---------|:----------|
| **Rust API** | Fly.io | Global deployment, easy Docker, auto-scaling, generous free tier |
| **Next.js** | Vercel | Native Next.js hosting, edge runtime, automatic preview deploys |
| **Background Worker** | Fly.io (same app) | Run as separate process in same container or dedicated machine |

**Alternative:** AWS ECS + Fargate (more control, more complex)

### Database

| Service | Provider | Rationale |
|:--------|:---------|:----------|
| **PostgreSQL** | Neon | Serverless Postgres, auto-scaling, branching for dev/staging, generous free tier |
| **Redis** | Upstash | Serverless Redis, pay-per-request, global replication |

**Alternative:** Supabase (PostgreSQL) + Redis Cloud

### Storage

| Service | Provider | Rationale |
|:--------|:---------|:----------|
| **Object Storage** | Cloudflare R2 | Zero egress fees, S3-compatible, built-in CDN |

---

## Docker Setup

### Rust API Dockerfile

```dockerfile
# Stage 1: Build
FROM rust:1.82-slim AS builder
WORKDIR /app

# Cache dependencies
COPY Cargo.toml Cargo.lock ./
COPY crates/ crates/
RUN mkdir src && echo "fn main() {}" > src/main.rs
RUN cargo build --release && rm -rf src target/release/deps/punta*

# Build actual binary
COPY src/ src/
RUN cargo build --release

# Stage 2: Runtime
FROM debian:bookworm-slim
RUN apt-get update && apt-get install -y ca-certificates && rm -rf /var/lib/apt/lists/*
COPY --from=builder /app/target/release/punta-server /usr/local/bin/punta
EXPOSE 8080
CMD ["punta"]
```

### docker-compose.yml (Local Development)

```yaml
version: '3.8'
services:
  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: punta_dev
      POSTGRES_USER: punta
      POSTGRES_PASSWORD: punta_dev_pass
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data

  minio:
    image: minio/minio
    environment:
      MINIO_ROOT_USER: minioadmin
      MINIO_ROOT_PASSWORD: minioadmin
    ports:
      - "9000:9000"
      - "9001:9001"
    command: server /data --console-address ":9001"
    volumes:
      - minio_data:/data

volumes:
  postgres_data:
  redis_data:
  minio_data:
```

---

## CI/CD Pipeline (GitHub Actions)

```yaml
# .github/workflows/ci.yml
name: CI

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  rust-check:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:16
        env:
          POSTGRES_DB: punta_test
          POSTGRES_USER: test
          POSTGRES_PASSWORD: test
        ports: ["5432:5432"]
      redis:
        image: redis:7
        ports: ["6379:6379"]
    steps:
      - uses: actions/checkout@v4
      - uses: dtolnay/rust-toolchain@stable
      - uses: Swatinem/rust-cache@v2
      - run: cargo fmt --check
      - run: cargo clippy -- -D warnings
      - run: cargo test --workspace
        env:
          DATABASE_URL: postgres://test:test@localhost/punta_test

  nextjs-check:
    runs-on: ubuntu-latest
    defaults:
      run:
        working-directory: web
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: npm
          cache-dependency-path: web/package-lock.json
      - run: npm ci
      - run: npm run lint
      - run: npm run build

  deploy:
    needs: [rust-check, nextjs-check]
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      # Deploy Rust to Fly.io
      - uses: superfly/flyctl-actions/setup-flyctl@master
      - run: flyctl deploy --remote-only
        env:
          FLY_API_TOKEN: ${{ secrets.FLY_API_TOKEN }}
      # Deploy Next.js to Vercel (auto via Vercel GitHub integration)
```

---

## Monitoring & Observability

### Logging

- **Rust:** `tracing` crate with JSON output → shipped to log aggregator
- **Next.js:** Vercel built-in logging
- **Format:** Structured JSON with `tenant_id`, `request_id`, `user_id`

```json
{
  "timestamp": "2025-01-15T10:30:00Z",
  "level": "INFO",
  "target": "punta_orders::service",
  "message": "Order created",
  "tenant_id": "t_abc123",
  "user_id": "u_xyz789",
  "order_id": "ord_456",
  "request_id": "req_111",
  "duration_ms": 45
}
```

### Metrics

| Metric | Tool |
|:-------|:-----|
| Request latency (p50, p95, p99) | Prometheus + Grafana |
| Error rate by endpoint | Prometheus |
| Active tenants | Custom gauge |
| Database connection pool usage | SQLx metrics |
| Redis hit rate | Redis INFO |
| Background job queue depth | Custom gauge |

### Alerting

| Alert | Condition |
|:------|:----------|
| High error rate | > 5% of requests returning 5xx |
| Slow responses | p95 latency > 500ms |
| Database pool exhaustion | Available connections < 5 |
| Payment webhook failures | > 3 consecutive failures |
| Background job backlog | Queue depth > 1000 |

---

## Environment Configuration

```bash
# .env.example

# Database
DATABASE_URL=postgres://punta:password@localhost:5432/punta_dev
DATABASE_MAX_CONNECTIONS=50

# Redis
REDIS_URL=redis://localhost:6379

# Auth
JWT_SECRET=your-256-bit-secret
JWT_ACCESS_EXPIRY_SECS=900       # 15 minutes
JWT_REFRESH_EXPIRY_SECS=604800   # 7 days

# Object Storage (S3/R2)
S3_ENDPOINT=http://localhost:9000
S3_BUCKET=punta-media
S3_ACCESS_KEY=minioadmin
S3_SECRET_KEY=minioadmin
S3_REGION=auto

# Payment Gateways
PAYSTACK_SECRET_KEY=sk_test_xxx
PAYSTACK_PUBLIC_KEY=pk_test_xxx
FLUTTERWAVE_SECRET_KEY=FLWSECK_TEST-xxx
FLUTTERWAVE_PUBLIC_KEY=FLWPUBK_TEST-xxx

# Messaging
WHATSAPP_PHONE_ID=xxx
WHATSAPP_TOKEN=xxx
TERMII_API_KEY=xxx
RESEND_API_KEY=re_xxx

# App
BASE_DOMAIN=punta.shop
APP_ENV=development
RUST_LOG=info,punta=debug
```

---

## Scaling Plan

| Stage | Tenants | Infrastructure |
|:------|:--------|:---------------|
| **Launch** | 0-100 | 1 Fly.io instance, Neon free tier, Upstash free tier |
| **Growth** | 100-1K | 2 Fly.io instances, Neon Scale, Upstash Pro |
| **Scale** | 1K-10K | 4+ instances, PG read replicas, Redis cluster, dedicated worker |
| **Enterprise** | 10K+ | Kubernetes, database sharding, cell-based architecture |
