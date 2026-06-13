# Week 2 — Foundation: Auth & Multi-Tenancy

> **Goal:** Build the complete authentication system and tenant resolution middleware with RLS enforcement.

---

## Monday — Password Hashing & JWT

- [ ] Create `punta-auth` crate with Cargo.toml and dependencies
- [ ] Implement `punta-auth/password.rs` — argon2id hash & verify functions
- [ ] Implement `punta-auth/jwt.rs` — `Claims` struct, `encode()`, `decode()` functions
- [ ] Configure JWT secret from environment, set 15-min access / 7-day refresh expiry
- [ ] Write unit tests for password hashing (hash + verify, wrong password fails)
- [ ] Write unit tests for JWT encode/decode (valid token, expired token, invalid signature)

**Deliverable:** Password hashing and JWT generation/validation working with tests.

---

## Tuesday — Registration & Login

- [ ] Implement `punta-auth/handlers.rs` — `POST /v1/auth/register`
  - Validate input (email format, password strength, slug uniqueness)
  - Create tenant record
  - Create user record (role: owner)
  - Create default store record
  - Create default location ("Main Store")
  - Create merchant wallet (balance: 0, default currency: NGN)
  - Initialize default features in tenant_features table
  - Generate JWT access + refresh tokens
  - Return user, tenant, tokens
- [ ] Implement `POST /v1/auth/login`
  - Find user by email
  - Verify password
  - Generate tokens
  - Update `last_login_at`
- [ ] Store refresh token (hashed) in `refresh_tokens` table
- [ ] Wire routes into `main.rs` router
- [ ] Test registration + login flow end-to-end with curl/httpie

**Deliverable:** Merchant can register and login, receives JWT tokens.

---

## Wednesday — Token Refresh & Password Reset

- [ ] Implement `POST /v1/auth/refresh`
  - Validate refresh token against DB
  - Rotate: generate new refresh token, invalidate old
  - Return new access token
- [ ] Implement `POST /v1/auth/forgot-password`
  - Generate reset token (random UUID), store hashed with 1-hour expiry
  - Log the reset URL for now (email integration comes in Week 8)
- [ ] Implement `POST /v1/auth/reset-password`
  - Validate reset token
  - Update password hash
  - Revoke all refresh tokens for user
- [ ] Implement `POST /v1/auth/logout`
  - Revoke refresh token
- [ ] Write integration tests for complete auth flows

**Deliverable:** Full auth lifecycle works (register → login → refresh → reset → logout).

---

## Thursday — Auth Middleware & Tenant Resolution

- [ ] Implement `punta-auth/middleware.rs` — `AuthMiddleware`
  - Extract `Authorization: Bearer <token>` header
  - Decode JWT, validate claims
  - Inject `AuthUser` into request extensions
- [ ] Implement `AuthUser` extractor for handlers
- [ ] Create `punta-tenant` crate
- [ ] Implement `punta-tenant/resolver.rs` — `TenantResolver` middleware
  - Parse subdomain from `Host` header
  - Check Redis cache for slug → tenant_id mapping
  - Fallback to PostgreSQL query
  - Cache result in Redis (TTL: 5 minutes)
  - Inject `TenantContext` into request extensions
- [ ] Implement `Tenant` extractor for handlers
- [ ] Implement reserved subdomain validation (app, www, api, admin, etc.)
- [ ] Wire both middlewares into the router stack (order: rate limit → auth → tenant)

**Deliverable:** Requests are authenticated and tenant-resolved before reaching handlers.

---

## Friday — RBAC & RLS Verification

- [ ] Implement `punta-auth/rbac.rs` — permission checks
  - `AuthUser::require_role(role)` → checks owner/admin/staff
  - `AuthUser::require_permission(perm)` → checks staff permissions array
- [ ] Implement `GET /v1/tenant` — return current tenant info
- [ ] Implement `PATCH /v1/tenant` — update tenant settings
- [ ] Implement `POST /v1/tenant/staff` — invite staff member (create user with staff role)
- [ ] Write **cross-tenant isolation tests**:
  - Create Tenant A and Tenant B
  - Insert users for both
  - Query as Tenant A → verify only A's users returned
  - Attempt to read Tenant B's data → verify empty/404
- [ ] Write test: staff user without ManageProducts → rejected from product endpoints
- [ ] Load test: verify RLS doesn't leak under concurrent requests

**Deliverable:** RBAC enforced, cross-tenant isolation verified with tests.

---

## Week 2 Checklist

```
✅ Full auth flow working (register, login, refresh, reset, logout)
✅ JWT access/refresh token lifecycle
✅ Passwords hashed with argon2id
✅ Tenant resolution via subdomain (cached in Redis)
✅ RLS enforcement verified with cross-tenant tests
✅ RBAC permissions enforced for staff users
✅ Auth + Tenant middlewares in request pipeline
✅ GET/PATCH /tenant endpoints working
✅ Staff invitation endpoint working
✅ Integration tests passing
```
