# 03 — Multi-Tenancy Strategy

## Overview

Punta uses **Shared Schema + PostgreSQL Row-Level Security (RLS)** to isolate tenant data. All tenants share the same database tables, but PostgreSQL enforces that each tenant can only access their own rows.

## Tenant Resolution: Subdomain-Based

Each merchant gets a unique subdomain: `storename.punta.shop`

```
┌─────────────────────────────────────────────────────┐
│                  DNS Configuration                   │
│                                                      │
│  *.punta.shop  →  CNAME  →  app.punta.shop          │
│  app.punta.shop →  A     →  <server IP>             │
│                                                      │
│  Cloudflare handles wildcard DNS + SSL automatically │
└─────────────────────────────────────────────────────┘
```

### Resolution Flow

```
Request: GET https://acme.punta.shop/api/v1/products
                        │
                        ▼
              ┌─────────────────┐
              │ Extract Host    │
              │ header value    │
              │ "acme.punta.shop│
              └────────┬────────┘
                       │
                       ▼
              ┌─────────────────┐
              │ Parse subdomain │
              │ "acme"          │
              └────────┬────────┘
                       │
                       ▼
              ┌─────────────────┐     ┌──────────────┐
              │ Check Redis     │────▶│ Cache HIT    │──▶ TenantContext
              │ cache           │     │ tenant_id    │
              └────────┬────────┘     └──────────────┘
                       │
                       │ Cache MISS
                       ▼
              ┌─────────────────┐
              │ Query PostgreSQL│
              │ SELECT * FROM   │
              │ tenants WHERE   │
              │ slug = 'acme'   │
              └────────┬────────┘
                       │
                       ▼
              ┌─────────────────┐
              │ Cache in Redis  │
              │ TTL: 5 minutes  │
              └────────┬────────┘
                       │
                       ▼
              TenantContext {
                id: "t_abc123",
                slug: "acme",
                is_active: true,
              }
```

### Reserved Subdomains

These subdomains are reserved and cannot be used by tenants:

```
app, www, api, admin, dashboard, status, docs, help,
support, blog, mail, smtp, ftp, cdn, assets, static,
billing, auth, login, register, signup, store
```

### Custom Domains (Future)

Tenants who unlock the custom domain premium add-on can map their own domain:

```
www.acmeclothing.ng  →  CNAME  →  acme.punta.shop
```

A background job periodically verifies DNS and provisions SSL via Cloudflare.

---

## Row-Level Security (RLS)

### How RLS Works

Every table that contains tenant data includes a `tenant_id UUID NOT NULL` column. PostgreSQL RLS policies restrict all operations to the current tenant.

### Setup Pattern (applied to every tenant table)

```sql
-- 1. Add tenant_id column
ALTER TABLE products ADD COLUMN tenant_id UUID NOT NULL REFERENCES tenants(id);

-- 2. Create index for performance
CREATE INDEX idx_products_tenant_id ON products(tenant_id);

-- 3. Enable RLS
ALTER TABLE products ENABLE ROW LEVEL SECURITY;

-- 4. Force RLS even for table owner (critical for security)
ALTER TABLE products FORCE ROW LEVEL SECURITY;

-- 5. Create isolation policy
CREATE POLICY tenant_isolation_policy ON products
    FOR ALL
    USING (tenant_id = current_setting('app.current_tenant', true)::UUID)
    WITH CHECK (tenant_id = current_setting('app.current_tenant', true)::UUID);
```

### How the Rust App Sets Tenant Context

```rust
/// Acquires a connection from the pool and sets the tenant context
/// for the duration of the transaction.
pub async fn tenant_connection(
    pool: &PgPool,
    tenant_id: &Uuid,
) -> Result<PgConnection, AppError> {
    let mut conn = pool.acquire().await?;
    
    // Set the tenant context for this connection
    // `true` = LOCAL — only applies to the current transaction
    sqlx::query("SELECT set_config('app.current_tenant', $1, true)")
        .bind(tenant_id.to_string())
        .execute(&mut *conn)
        .await?;
    
    Ok(conn)
}
```

### Safety Guarantees

| Threat | Mitigation |
|:-------|:-----------|
| Developer forgets WHERE clause | RLS automatically filters — no tenant leak possible |
| SQL injection | SQLx parameterized queries prevent injection |
| Connection pool leaks tenant | `set_config(..., true)` scopes to transaction only |
| Admin bypass | `FORCE ROW LEVEL SECURITY` prevents even table owners from bypassing |
| Missing tenant_id on INSERT | `WITH CHECK` policy rejects inserts without matching tenant_id |

### Tables WITHOUT RLS (Global Tables)

These tables are platform-level and not tenant-scoped:

```
tenants              — Platform-wide tenant registry
platform_settings    — Global configuration
audit_log            — Platform-wide audit trail
```

---

## Rate Limiting (Per-Tenant)

Each tenant has a default rate limit to prevent noisy-neighbor issues, with messaging limits controlled by pay-as-you-grow wallet balances:

```
┌──────────────┬──────────────┬──────────────────────────────────┐
│ Tier         │ API Requests │ Messaging Limits                 │
├──────────────┼──────────────┼──────────────────────────────────┤
│ Standard     │ 300/min      │ Pay-As-You-Grow (Wallet Balance) │
└──────────────┴──────────────┴──────────────────────────────────┘
```

### Implementation

```rust
// Redis sliding window rate limiter
pub async fn check_rate_limit(
    redis: &RedisPool,
    tenant_id: &Uuid,
    limit: u32,
    window_secs: u64,
) -> Result<bool, AppError> {
    let key = format!("punta:ratelimit:{}:{}", tenant_id, window_secs);
    let now = SystemTime::now().duration_since(UNIX_EPOCH)?.as_secs();
    
    // MULTI: remove old entries, add current, count
    let count: u64 = redis::pipe()
        .zremrangebyscore(&key, 0, (now - window_secs) as f64)
        .zadd(&key, now as f64, uuid::Uuid::new_v4().to_string())
        .zcard(&key)
        .expire(&key, window_secs as i64)
        .query_async(redis)
        .await?;
    
    Ok(count <= limit as u64)
}
```

---

## Tenant Lifecycle

```
┌──────────┐     ┌─────────────┐     ┌────────────────────────────────┐
│  Signup  │────▶│ Active Free │────▶│ Premium Unlocks (Custom Domain)│
│          │     │             │     │ (One-time or Annual Fee)       │
└──────────┘     └──────┬──────┘     └────────────────────────────────┘
                        │
                        ▼
                 ┌─────────────┐
                 │ Deactivated │
                 │ (by admin)  │
                 └─────────────┘

States:
- ACTIVE_FREE: Core catalog, inventory, storefront and ordering services enabled.
- PREMIUM: Custom domain and multi-location features enabled via one-time/annual payment unlocks.
- DEACTIVATED: Admin-deactivated (e.g. Terms of Service violation).
```
