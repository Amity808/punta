# Week 1 — Foundation: Backend Setup

> **Goal:** Set up the Rust workspace, PostgreSQL database, Docker environment, and core infrastructure crates.

---

## Monday — Project Initialization

- [ ] Initialize Cargo workspace with root `Cargo.toml`
- [ ] Create `punta-core` crate with basic structure
- [ ] Create `punta-db` crate with basic structure
- [ ] Create `punta-server` binary crate with `main.rs`
- [ ] Set up `docker-compose.yml` (PostgreSQL 16, Redis 7, MinIO)
- [ ] Create `.env.example` with all configuration variables
- [ ] Create `justfile` with common tasks (run, test, migrate, fmt, clippy)
- [ ] Verify `docker-compose up` and `cargo build` work end-to-end

**Deliverable:** Empty Axum server starts and responds to health check.

---

## Tuesday — Core Types & Configuration

- [ ] Implement `punta-core/config.rs` — load config from environment variables
- [ ] Implement `punta-core/error.rs` — `AppError` enum with `thiserror`
- [ ] Implement `AppError → IntoResponse` for Axum (JSON error responses)
- [ ] Implement `punta-core/types.rs` — `Money`, `Currency`, `Pagination`, `ApiResponse<T>`
- [ ] Implement `punta-core/tenant.rs` — `TenantContext` struct
- [ ] Add `tracing` + `tracing-subscriber` setup in `main.rs` (structured JSON logging)
- [ ] Add unit tests for error formatting and config loading

**Deliverable:** Core types compile, config loads from `.env`, structured logging works.

---

## Wednesday — Database Layer

- [ ] Implement `punta-db/pool.rs` — create `PgPool` with connection options
- [ ] Implement `punta-db/rls.rs` — `tenant_conn()` function that sets RLS context
- [ ] Install `sqlx-cli` and configure for migrations
- [ ] Write migration `001_create_extensions.sql` (uuid-ossp, pg_trgm)
- [ ] Write migration `002_create_tenants.sql` (tenants table — no RLS)
- [ ] Write migration `003_create_tenant_features.sql`
- [ ] Write migration `004_create_users.sql` (with RLS policy)
- [ ] Write migration `005_create_refresh_tokens.sql`
- [ ] Run all migrations, verify tables created with correct RLS policies

**Deliverable:** PostgreSQL schema for auth/tenants is live, RLS policies active.

---

## Thursday — Database Models & Health Checks

- [ ] Implement `punta-db/models/tenant.rs` — `TenantRow` (FromRow)
- [ ] Implement `punta-db/models/user.rs` — `UserRow` (FromRow)
- [ ] Implement basic CRUD queries for tenants (create, find_by_slug, find_by_id)
- [ ] Implement basic CRUD queries for users (create, find_by_email)
- [ ] Add health check endpoint: `GET /health` → checks DB + Redis connectivity
- [ ] Set up Redis connection in `main.rs` (using `redis` crate)
- [ ] Verify Redis connection works
- [ ] Write integration tests for tenant CRUD operations

**Deliverable:** Health endpoint works, basic DB operations verified.

---

## Friday — Middleware Foundation

- [ ] Implement `tower-http` layers: CORS, compression, request tracing
- [ ] Implement request ID middleware (generate UUID per request, add to tracing span)
- [ ] Implement basic rate limiting middleware structure (Redis sliding window)
- [ ] Create `AppState` struct with `PgPool`, `Redis`, `AppConfig`
- [ ] Wire up middleware stack in `main.rs`
- [ ] Write seed data script: configure default platform parameters and wallet constants
- [ ] End-to-end test: start server, hit health check, verify logs

**Deliverable:** Server starts with full middleware stack, seed data in DB, request tracing works.

---

## Week 1 Checklist

```
✅ Cargo workspace compiles
✅ Docker Compose runs PostgreSQL, Redis, MinIO
✅ 5 migrations applied successfully
✅ RLS policies active on users table
✅ Health check endpoint responds 200
✅ Structured JSON logging works
✅ Redis connection established
✅ Wallet transactions and features tables verified
✅ Integration tests pass
```
