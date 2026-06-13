# 09 — Security

## Authentication

### Password Security
- **Hashing:** Argon2id (memory-hard, GPU-resistant)
- **Parameters:** 19 MiB memory, 2 iterations, 1 parallelism
- **Min password length:** 8 characters
- **Breach check:** Optional — check against HaveIBeenPwned API on registration

### JWT Tokens
- **Access token:** 15-minute expiry, signed with HS256
- **Refresh token:** 7-day expiry, stored hashed in database, rotated on use
- **Claims:** `sub` (user_id), `tenant_id`, `role`, `iat`, `exp`
- **Revocation:** Refresh token revocation on password change or explicit logout

### Session Security
- Refresh tokens are stored as **bcrypt hashes** in the database (never plaintext)
- Token rotation: each refresh generates a new refresh token and invalidates the old one
- Concurrent session limit: default 5 concurrent active sessions per user (configurable for enterprise overrides)

---

## Authorization (RBAC)

### Roles

| Role | Description | Scope |
|:-----|:------------|:------|
| `owner` | Full access, wallet settings, team management | Everything |
| `admin` | Full access except wallet/billing settings | All features except wallet/billing settings |
| `staff` | Limited access based on granted permissions | Specific permissions only |

### Permissions (for `staff` role)

```rust
pub enum Permission {
    // Products
    ViewProducts,
    ManageProducts,     // create, edit, delete
    
    // Orders
    ViewOrders,
    ManageOrders,       // update status, refund
    CreateManualOrders,
    
    // Customers
    ViewCustomers,
    ManageCustomers,
    
    // Inventory
    ViewInventory,
    ManageInventory,    // adjustments, transfers
    
    // Messaging
    ViewMessages,
    SendMessages,
    ManageCampaigns,
    
    // Analytics
    ViewAnalytics,
    
    // Expenses
    ViewExpenses,
    ManageExpenses,
    
    // Delivery
    ViewDeliveries,
    ManageDeliveries,
    
    // Store
    ManageStore,        // theme, settings
}
```

### Middleware Enforcement

```rust
// Example: only owner/admin or staff with ManageProducts can create products
pub async fn create_product(
    auth: AuthUser,
    tenant: Tenant,
    Json(body): Json<CreateProductRequest>,
) -> Result<Json<Product>, AppError> {
    auth.require_permission(Permission::ManageProducts)?;
    // ... handler logic
}
```

---

## Data Protection

### Tenant Isolation
- **Database-level:** PostgreSQL RLS policies on every tenant table
- **Application-level:** Tenant context set at the start of every request
- **Cache-level:** All Redis keys prefixed with `tenant_id`
- **Storage-level:** Object storage paths include tenant_id (`tenants/{id}/...`)

### Cross-Tenant Testing
Automated integration tests that:
1. Create two tenants (A and B)
2. Insert data for both
3. Query as Tenant A → verify only A's data returned
4. Attempt to access B's data directly → verify 404 or empty result

### Sensitive Data
- **Passwords:** Argon2id hashed, never logged or returned in API
- **Payment details:** Never stored — delegated to Paystack/Flutterwave
- **Bank accounts:** Stored encrypted at rest (AES-256-GCM)
- **API keys:** Stored hashed, displayed only once on creation

---

## API Security

### Input Validation
- All request bodies validated using `validator` crate
- SQL injection prevented by SQLx parameterized queries (compile-time checked)
- XSS prevented by JSON API (no HTML rendering on backend)

### Rate Limiting
- Per-tenant sliding window (Redis)
- Per-IP rate limiting for auth endpoints (prevent brute force)
- Webhook endpoints rate-limited per source IP

### CORS
- Dashboard: `app.punta.shop`
- Storefronts: `*.punta.shop`
- API: strict origin checking

### Webhook Security
- **Paystack:** Verify `x-paystack-signature` (HMAC SHA512)
- **Flutterwave:** Verify `verifi-hash` header
- **WhatsApp:** Verify hub signature
- **Idempotency:** Store processed webhook IDs in Redis to prevent double-processing

---

## Compliance (Nigeria)

### NDPR (Nigeria Data Protection Regulation)
- **Consent:** Explicit opt-in for marketing messages
- **Data access:** Customers can request their data
- **Data deletion:** Support for customer data removal
- **Data portability:** Export customer data in standard format
- **Privacy policy:** Required for each tenant's store

### Tax
- VAT support (7.5% for Nigeria)
- Stamp duty tracking capability
- Invoice generation with proper tax breakdown

---

## Infrastructure Security

| Layer | Measure |
|:------|:--------|
| **Network** | Cloudflare WAF, DDoS protection |
| **Transport** | TLS 1.3 everywhere (Cloudflare SSL) |
| **Database** | Connection via SSL, IP allowlisting |
| **Secrets** | Environment variables, never in code or logs |
| **Logging** | Sensitive fields (password, token, card) auto-redacted |
| **Backups** | Automated daily PostgreSQL backups (Neon handles this) |
