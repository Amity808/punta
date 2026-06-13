# 06 — Module Breakdown (Rust Workspace)

## Workspace Dependency Graph

```
punta-core          ← No internal dependencies (shared types, config, errors)
    │
    ├── punta-db    ← Depends on: punta-core
    │       │
    │       ├── punta-auth       ← Depends on: punta-core, punta-db
    │       ├── punta-tenant     ← Depends on: punta-core, punta-db
    │       ├── punta-catalog    ← Depends on: punta-core, punta-db
    │       ├── punta-inventory  ← Depends on: punta-core, punta-db
    │       ├── punta-orders     ← Depends on: punta-core, punta-db, punta-catalog, punta-inventory
    │       ├── punta-payments   ← Depends on: punta-core, punta-db, punta-orders
    │       ├── punta-crm        ← Depends on: punta-core, punta-db
    │       ├── punta-messaging  ← Depends on: punta-core, punta-db, punta-crm
    │       ├── punta-analytics  ← Depends on: punta-core, punta-db
    │       ├── punta-expenses   ← Depends on: punta-core, punta-db
    │       ├── punta-delivery   ← Depends on: punta-core, punta-db, punta-orders
    │       ├── punta-store      ← Depends on: punta-core, punta-db
    │       ├── punta-wallet     ← Depends on: punta-core, punta-db, punta-payments
    │       └── punta-jobs       ← Depends on: punta-core, punta-db, + all service crates
    │
    └── punta-server (binary)    ← Depends on: ALL crates (assembles the app)
```

---

## Crate Details

### `punta-core` — Shared Foundation

**Purpose:** Types, configuration, and error handling shared across all crates.

```rust
// src/lib.rs — re-exports
pub mod config;
pub mod error;
pub mod types;
pub mod tenant;

// src/config.rs
pub struct AppConfig {
    pub database_url: String,
    pub redis_url: String,
    pub jwt_secret: String,
    pub jwt_expiry_secs: u64,
    pub s3_bucket: String,
    pub s3_region: String,
    pub s3_endpoint: String,
    pub paystack_secret_key: String,
    pub flutterwave_secret_key: String,
    pub whatsapp_token: String,
    pub termii_api_key: String,
    pub resend_api_key: String,
    pub base_domain: String,  // "punta.shop"
}

// src/error.rs
#[derive(Debug, thiserror::Error)]
pub enum AppError {
    #[error("Not found: {0}")]
    NotFound(String),
    #[error("Validation error: {0}")]
    Validation(String),
    #[error("Unauthorized")]
    Unauthorized,
    #[error("Forbidden: {0}")]
    Forbidden(String),
    #[error("Feature locked: {0}")]
    FeatureLocked(String),
    #[error("Conflict: {0}")]
    Conflict(String),
    #[error("Rate limited")]
    RateLimited,
    #[error("Database error: {0}")]
    Database(#[from] sqlx::Error),
    #[error("Internal error: {0}")]
    Internal(String),
}

// Implement IntoResponse for Axum
impl IntoResponse for AppError { ... }

// src/types.rs
pub type Money = i64;  // Always in kobo/cents
pub type Currency = String;

#[derive(Debug, Serialize, Deserialize)]
pub struct Pagination {
    pub cursor: Option<String>,
    pub limit: u32,
    pub total: Option<i64>,
    pub has_more: bool,
}

#[derive(Debug, Serialize)]
pub struct ApiResponse<T: Serialize> {
    pub data: T,
    #[serde(skip_serializing_if = "Option::is_none")]
    pub pagination: Option<Pagination>,
}

// src/tenant.rs
#[derive(Debug, Clone)]
pub struct TenantContext {
    pub id: Uuid,
    pub slug: String,
    pub is_active: bool,
}
```

---

### `punta-db` — Database Layer

**Purpose:** Connection pool management, RLS helpers, and database row types.

```
src/
├── lib.rs
├── pool.rs        — PgPool creation, health checks
├── rls.rs         — set_tenant_context(), RLS enforcement
└── models/        — FromRow structs (one per table)
    ├── mod.rs
    ├── tenant.rs
    ├── user.rs
    ├── product.rs
    ├── order.rs
    ├── customer.rs
    └── ...
```

Key function — every query goes through this:

```rust
/// Acquire a connection with tenant context set for RLS
pub async fn tenant_conn(
    pool: &PgPool,
    tenant_id: &Uuid,
) -> Result<PoolConnection<Postgres>, AppError> {
    let mut conn = pool.acquire().await?;
    sqlx::query("SELECT set_config('app.current_tenant', $1, true)")
        .bind(tenant_id.to_string())
        .execute(&mut *conn)
        .await?;
    Ok(conn)
}
```

---

### `punta-auth` — Authentication

```
src/
├── lib.rs
├── jwt.rs         — encode/decode JWT, claims struct
├── password.rs    — argon2 hash/verify
├── middleware.rs   — AuthMiddleware, AuthUser extractor
├── rbac.rs        — Permission enum, role-based checks
└── handlers.rs    — register, login, refresh, forgot/reset password
```

**JWT Claims:**
```rust
#[derive(Debug, Serialize, Deserialize)]
pub struct Claims {
    pub sub: Uuid,        // user_id
    pub tenant_id: Uuid,
    pub role: String,
    pub exp: usize,
    pub iat: usize,
}
```

**Permissions:**
```rust
pub enum Permission {
    ManageProducts,
    ManageOrders,
    ManageCustomers,
    ManageInventory,
    ManageMessaging,
    ViewAnalytics,
    ManageSettings,
    ManageStaff,
    ManageBilling,
}
```

---

### `punta-tenant` — Tenant Management

```
src/
├── lib.rs
├── resolver.rs    — SubdomainResolver middleware
├── service.rs     — create_tenant, update_tenant, onboard
├── validator.rs   — slug validation, reserved words check
└── handlers.rs    — GET/PATCH /tenant, POST /tenant/staff
```

---

### `punta-catalog` — Product Catalog

```
src/
├── lib.rs
├── product.rs     — ProductService (CRUD, search, bulk operations)
├── variant.rs     — VariantService (CRUD)
├── category.rs    — CategoryService (tree CRUD)
├── media.rs       — presigned URL generation, image management
├── search.rs      — full-text search queries
└── handlers.rs    — All /products and /categories routes
```

---

### `punta-inventory` — Stock Management

```
src/
├── lib.rs
├── stock.rs       — StockService (check, reserve, release, adjust)
├── transfer.rs    — Inter-location stock transfers
├── alerts.rs      — Low-stock detection and notification
├── location.rs    — LocationService (CRUD)
└── handlers.rs    — All /inventory and /locations routes
```

Key operation — **atomic stock reservation** during checkout:

```rust
/// Reserve stock for an order (within a transaction)
pub async fn reserve_stock(
    tx: &mut Transaction<Postgres>,
    product_id: &Uuid,
    variant_id: Option<&Uuid>,
    location_id: &Uuid,
    quantity: i32,
) -> Result<(), AppError> {
    let result = sqlx::query!(
        r#"
        UPDATE inventory
        SET reserved = reserved + $1, updated_at = now()
        WHERE product_id = $2
          AND variant_id IS NOT DISTINCT FROM $3
          AND location_id = $4
          AND (quantity - reserved) >= $1
        "#,
        quantity, product_id, variant_id, location_id
    )
    .execute(&mut **tx)
    .await?;

    if result.rows_affected() == 0 {
        return Err(AppError::Validation("Insufficient stock".into()));
    }
    Ok(())
}
```

---

### `punta-orders` — Order Engine

```
src/
├── lib.rs
├── order.rs       — OrderService (create, update status, lifecycle)
├── cart.rs        — CartService (Redis-backed session carts)
├── checkout.rs    — CheckoutService (validate → reserve stock → create order → init payment)
├── invoice.rs     — Invoice PDF generation
├── number.rs      — Order number generation
└── handlers.rs    — All /orders routes + storefront checkout
```

---

### `punta-payments` — Payment Processing

```
src/
├── lib.rs
├── gateway.rs     — PaymentGateway trait
├── paystack.rs    — PaystackGateway implementation
├── flutterwave.rs — FlutterwaveGateway implementation
├── wallet.rs      — MerchantWalletService (balance, credit, withdraw)
├── webhook.rs     — Webhook signature verification + event processing
└── handlers.rs    — Payment init, webhook endpoints, wallet routes
```

**Gateway trait** — abstraction over payment providers:

```rust
#[async_trait]
pub trait PaymentGateway: Send + Sync {
    /// Initialize a payment and return the checkout URL
    async fn initialize(
        &self,
        amount: Money,
        currency: &str,
        email: &str,
        reference: &str,
        callback_url: &str,
    ) -> Result<PaymentInitResponse, AppError>;

    /// Verify a payment by reference
    async fn verify(&self, reference: &str) -> Result<PaymentVerification, AppError>;

    /// Verify webhook signature
    fn verify_webhook_signature(&self, body: &[u8], signature: &str) -> bool;

    /// Initiate a bank transfer (for withdrawals)
    async fn transfer(
        &self,
        amount: Money,
        bank_code: &str,
        account_number: &str,
        reference: &str,
    ) -> Result<TransferResponse, AppError>;
}
```

---

### `punta-crm` — Customer Management

```
src/
├── lib.rs
├── customer.rs    — CustomerService (CRUD, search, import/export)
├── groups.rs      — CustomerGroupService (segmentation, dynamic filters)
├── history.rs     — Purchase history aggregation
└── handlers.rs
```

---

### `punta-messaging` — Multi-Channel Messaging

```
src/
├── lib.rs
├── engine.rs      — MessageDispatcher (routes to correct channel)
├── whatsapp.rs    — WhatsApp Cloud API client
├── sms.rs         — Termii SMS client
├── email.rs       — Resend email client
├── templates.rs   — Template rendering (variable substitution)
├── campaigns.rs   — CampaignService (schedule, execute, track)
└── handlers.rs
```

---

### `punta-analytics` — Business Analytics

```
src/
├── lib.rs
├── dashboard.rs   — DashboardService (overview metrics, comparison periods)
├── sales.rs       — SalesAnalytics (revenue, orders by period/channel)
├── products.rs    — ProductAnalytics (top sellers, slow movers)
├── customers.rs   — CustomerAnalytics (new vs returning, CLV)
├── aggregator.rs  — Background aggregation job (daily_sales_stats)
└── handlers.rs
```

---

### `punta-expenses` — Expense Tracking

```
src/
├── lib.rs
├── expense.rs     — ExpenseService (CRUD, recurring)
├── reports.rs     — P&L summary, expense breakdown by category
└── handlers.rs
```

---

### `punta-delivery` — Logistics Integration

```
src/
├── lib.rs
├── provider.rs    — LogisticsProvider trait
├── shipbubble.rs  — Shipbubble API client
├── tracking.rs    — TrackingService
├── rates.rs       — RateCalculator (multi-provider quotes)
└── handlers.rs
```

---

### `punta-store` — Storefront Management

```
src/
├── lib.rs
├── service.rs     — StoreService (settings, theme, domain management)
├── storefront.rs  — Public storefront API (products, cart, checkout proxy)
└── handlers.rs
```

---

### `punta-wallet` — Wallet & Billing

```
src/
├── lib.rs
├── service.rs     — WalletService (ledger bookkeeping, balance checks)
├── addons.rs      — AddonService (purchase and check premium feature flags)
├── enforcement.rs — FeatureEnforcement middleware (check unlocked add-ons / wallet balance limits)
├── payouts.rs     — Payout/withdrawal processing logic
└── handlers.rs    — API endpoints for wallet topup, withdrawals, and addon purchases
```

---

### `punta-jobs` — Background Processing

```
src/
├── lib.rs
├── worker.rs      — Redis-backed job worker (dequeue, execute, retry)
├── scheduler.rs   — Cron-like scheduler for recurring tasks
└── jobs/
    ├── mod.rs
    ├── send_notification.rs   — Dispatch WhatsApp/SMS/email
    ├── aggregate_analytics.rs — Roll up daily_sales_stats
    ├── process_webhook.rs     — Idempotent webhook processing
    ├── low_stock_check.rs     — Scan inventory, alert merchants
    ├── campaign_sender.rs     — Execute scheduled campaigns
    ├── addon_expiry_check.rs  — Check and revoke expired add-ons (e.g. yearly custom domains)
    └── cleanup.rs             — Purge expired sessions, old logs
```

---

### `punta-server` — Binary Entry Point

```rust
// src/main.rs
#[tokio::main]
async fn main() -> Result<()> {
    // 1. Load config
    let config = AppConfig::from_env()?;
    
    // 2. Initialize tracing
    tracing_subscriber::fmt().json().init();
    
    // 3. Connect to PostgreSQL
    let db_pool = PgPoolOptions::new()
        .max_connections(50)
        .connect(&config.database_url).await?;
    
    // 4. Connect to Redis
    let redis = redis::Client::open(&config.redis_url)?;
    
    // 5. Build shared application state
    let state = AppState { db_pool, redis, config };
    
    // 6. Assemble router from all modules
    let app = Router::new()
        .merge(punta_auth::routes())
        .merge(punta_tenant::routes())
        .merge(punta_catalog::routes())
        .merge(punta_inventory::routes())
        .merge(punta_orders::routes())
        .merge(punta_payments::routes())
        .merge(punta_crm::routes())
        .merge(punta_messaging::routes())
        .merge(punta_analytics::routes())
        .merge(punta_expenses::routes())
        .merge(punta_delivery::routes())
        .merge(punta_store::routes())
        .merge(punta_wallet::routes())
        .layer(tenant_middleware)
        .layer(auth_middleware)
        .layer(rate_limit_middleware)
        .layer(CorsLayer::permissive())
        .layer(CompressionLayer::new())
        .layer(TraceLayer::new_for_http())
        .with_state(state);
    
    // 7. Start background worker
    tokio::spawn(punta_jobs::start_worker(state.clone()));
    
    // 8. Start server
    let listener = TcpListener::bind("0.0.0.0:8080").await?;
    tracing::info!("🚀 Punta server running on :8080");
    axum::serve(listener, app).await?;
    
    Ok(())
}
```
