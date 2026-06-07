# 04 — Database Schema

## Schema Conventions

- **Primary keys**: `UUID` (generated via `gen_random_uuid()`)
- **Money**: `BIGINT` stored in **kobo** (₦1 = 100 kobo). Never use FLOAT/DECIMAL for money.
- **Timestamps**: `TIMESTAMPTZ` (timezone-aware), defaults to `now()`
- **Soft deletes**: `deleted_at TIMESTAMPTZ` (NULL = active)
- **Tenant isolation**: Every tenant-scoped table has `tenant_id UUID NOT NULL REFERENCES tenants(id)`
- **Naming**: `snake_case` for tables and columns, plural table names

---

## Global Tables (No RLS)

### tenants

```sql
CREATE TABLE tenants (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    slug            TEXT NOT NULL UNIQUE,          -- subdomain: "acme"
    business_name   TEXT NOT NULL,
    email           TEXT NOT NULL,
    phone           TEXT,
    logo_url        TEXT,
    country         TEXT NOT NULL DEFAULT 'NG',
    currency        TEXT NOT NULL DEFAULT 'NGN',
    timezone        TEXT NOT NULL DEFAULT 'Africa/Lagos',
    settings        JSONB NOT NULL DEFAULT '{}',   -- general settings
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE UNIQUE INDEX idx_tenants_slug ON tenants(slug);
CREATE INDEX idx_tenants_email ON tenants(email);
```

-- Global platform tables will reside here. Default configurations and system parameters.

---

## Auth & Users (RLS-Enabled)

### users

```sql
CREATE TABLE users (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    email           TEXT NOT NULL,
    password_hash   TEXT NOT NULL,
    full_name       TEXT NOT NULL,
    phone           TEXT,
    avatar_url      TEXT,
    role            TEXT NOT NULL DEFAULT 'owner'
                    CHECK (role IN ('owner', 'admin', 'staff')),
    permissions     JSONB NOT NULL DEFAULT '[]',     -- granular permissions for staff
    is_active       BOOLEAN NOT NULL DEFAULT true,
    last_login_at   TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    
    UNIQUE(tenant_id, email)
);

CREATE INDEX idx_users_tenant ON users(tenant_id);
CREATE INDEX idx_users_email ON users(email);

-- RLS
ALTER TABLE users ENABLE ROW LEVEL SECURITY;
ALTER TABLE users FORCE ROW LEVEL SECURITY;
CREATE POLICY tenant_isolation ON users
    FOR ALL
    USING (tenant_id = current_setting('app.current_tenant', true)::UUID)
    WITH CHECK (tenant_id = current_setting('app.current_tenant', true)::UUID);
```

### refresh_tokens

```sql
CREATE TABLE refresh_tokens (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    token_hash      TEXT NOT NULL UNIQUE,
    expires_at      TIMESTAMPTZ NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    revoked_at      TIMESTAMPTZ
);

CREATE INDEX idx_refresh_tokens_user ON refresh_tokens(user_id);
CREATE INDEX idx_refresh_tokens_tenant ON refresh_tokens(tenant_id);
```

---

## Store & Storefront

### stores

```sql
CREATE TABLE stores (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    name            TEXT NOT NULL,
    subdomain       TEXT NOT NULL UNIQUE,            -- same as tenant slug
    custom_domain   TEXT UNIQUE,                     -- e.g. "www.acmeclothing.ng"
    description     TEXT,
    logo_url        TEXT,
    favicon_url     TEXT,
    theme           JSONB NOT NULL DEFAULT '{
        "primary_color": "#6366f1",
        "secondary_color": "#ec4899",
        "font": "Inter",
        "layout": "grid",
        "hero_image": null
    }',
    seo             JSONB NOT NULL DEFAULT '{
        "title": null,
        "description": null,
        "og_image": null
    }',
    social_links    JSONB NOT NULL DEFAULT '{
        "instagram": null,
        "twitter": null,
        "facebook": null,
        "whatsapp": null
    }',
    is_published    BOOLEAN NOT NULL DEFAULT false,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- RLS
ALTER TABLE stores ENABLE ROW LEVEL SECURITY;
ALTER TABLE stores FORCE ROW LEVEL SECURITY;
CREATE POLICY tenant_isolation ON stores FOR ALL
    USING (tenant_id = current_setting('app.current_tenant', true)::UUID)
    WITH CHECK (tenant_id = current_setting('app.current_tenant', true)::UUID);
```

---

## Product Catalog

### categories

```sql
CREATE TABLE categories (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    parent_id       UUID REFERENCES categories(id) ON DELETE SET NULL,
    name            TEXT NOT NULL,
    slug            TEXT NOT NULL,
    description     TEXT,
    image_url       TEXT,
    sort_order      INTEGER NOT NULL DEFAULT 0,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    
    UNIQUE(tenant_id, slug)
);

-- RLS
ALTER TABLE categories ENABLE ROW LEVEL SECURITY;
ALTER TABLE categories FORCE ROW LEVEL SECURITY;
CREATE POLICY tenant_isolation ON categories FOR ALL
    USING (tenant_id = current_setting('app.current_tenant', true)::UUID)
    WITH CHECK (tenant_id = current_setting('app.current_tenant', true)::UUID);
```

### products

```sql
CREATE TABLE products (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    category_id     UUID REFERENCES categories(id) ON DELETE SET NULL,
    name            TEXT NOT NULL,
    slug            TEXT NOT NULL,
    description     TEXT,
    price           BIGINT NOT NULL,                  -- in kobo
    compare_at_price BIGINT,                          -- original price (for showing discounts)
    cost_price      BIGINT,                           -- merchant's cost (for profit calc)
    currency        TEXT NOT NULL DEFAULT 'NGN',
    sku             TEXT,
    barcode         TEXT,
    weight_grams    INTEGER,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    is_featured     BOOLEAN NOT NULL DEFAULT false,
    tags            TEXT[] DEFAULT '{}',
    metadata        JSONB NOT NULL DEFAULT '{}',       -- custom fields
    search_vector   TSVECTOR,                          -- full-text search
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    deleted_at      TIMESTAMPTZ,                       -- soft delete
    
    UNIQUE(tenant_id, slug)
);

-- Indexes
CREATE INDEX idx_products_tenant ON products(tenant_id);
CREATE INDEX idx_products_category ON products(category_id);
CREATE INDEX idx_products_search ON products USING GIN(search_vector);
CREATE INDEX idx_products_tags ON products USING GIN(tags);
CREATE INDEX idx_products_active ON products(tenant_id) WHERE is_active = true AND deleted_at IS NULL;

-- Auto-update search vector
CREATE OR REPLACE FUNCTION update_product_search_vector()
RETURNS TRIGGER AS $$
BEGIN
    NEW.search_vector := to_tsvector('english', 
        coalesce(NEW.name, '') || ' ' || 
        coalesce(NEW.description, '') || ' ' ||
        coalesce(NEW.sku, '') || ' ' ||
        coalesce(array_to_string(NEW.tags, ' '), '')
    );
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trigger_update_product_search
    BEFORE INSERT OR UPDATE ON products
    FOR EACH ROW EXECUTE FUNCTION update_product_search_vector();

-- RLS
ALTER TABLE products ENABLE ROW LEVEL SECURITY;
ALTER TABLE products FORCE ROW LEVEL SECURITY;
CREATE POLICY tenant_isolation ON products FOR ALL
    USING (tenant_id = current_setting('app.current_tenant', true)::UUID)
    WITH CHECK (tenant_id = current_setting('app.current_tenant', true)::UUID);
```

### product_variants

```sql
CREATE TABLE product_variants (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    product_id      UUID NOT NULL REFERENCES products(id) ON DELETE CASCADE,
    name            TEXT NOT NULL,                     -- "Large / Red"
    sku             TEXT,
    barcode         TEXT,
    price           BIGINT,                            -- NULL = use product price
    cost_price      BIGINT,
    weight_grams    INTEGER,
    attributes      JSONB NOT NULL DEFAULT '{}',       -- {"size": "L", "color": "Red"}
    sort_order      INTEGER NOT NULL DEFAULT 0,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_variants_product ON product_variants(product_id);
CREATE INDEX idx_variants_tenant ON product_variants(tenant_id);

-- RLS (same pattern for all tables below — omitting for brevity)
ALTER TABLE product_variants ENABLE ROW LEVEL SECURITY;
ALTER TABLE product_variants FORCE ROW LEVEL SECURITY;
CREATE POLICY tenant_isolation ON product_variants FOR ALL
    USING (tenant_id = current_setting('app.current_tenant', true)::UUID)
    WITH CHECK (tenant_id = current_setting('app.current_tenant', true)::UUID);
```

### product_images

```sql
CREATE TABLE product_images (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    product_id      UUID NOT NULL REFERENCES products(id) ON DELETE CASCADE,
    url             TEXT NOT NULL,
    alt_text        TEXT,
    sort_order      INTEGER NOT NULL DEFAULT 0,
    is_primary      BOOLEAN NOT NULL DEFAULT false,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_product_images_product ON product_images(product_id);
```

---

## Inventory

### locations

```sql
CREATE TABLE locations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    name            TEXT NOT NULL,                     -- "Main Store", "Lekki Warehouse"
    type            TEXT NOT NULL DEFAULT 'store'
                    CHECK (type IN ('store', 'warehouse', 'popup')),
    address_line1   TEXT,
    address_line2   TEXT,
    city            TEXT,
    state           TEXT,
    country         TEXT NOT NULL DEFAULT 'NG',
    phone           TEXT,
    is_default      BOOLEAN NOT NULL DEFAULT false,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

### inventory

```sql
CREATE TABLE inventory (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    product_id      UUID NOT NULL REFERENCES products(id) ON DELETE CASCADE,
    variant_id      UUID REFERENCES product_variants(id) ON DELETE CASCADE,
    location_id     UUID NOT NULL REFERENCES locations(id) ON DELETE CASCADE,
    quantity         INTEGER NOT NULL DEFAULT 0,
    reserved        INTEGER NOT NULL DEFAULT 0,        -- held for pending orders
    reorder_level   INTEGER NOT NULL DEFAULT 5,        -- low stock threshold
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    
    UNIQUE(tenant_id, product_id, variant_id, location_id)
);

-- Computed available stock
-- Available = quantity - reserved
CREATE INDEX idx_inventory_low_stock ON inventory(tenant_id)
    WHERE quantity - reserved <= reorder_level;
```

### inventory_movements

```sql
CREATE TABLE inventory_movements (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    inventory_id    UUID NOT NULL REFERENCES inventory(id) ON DELETE CASCADE,
    movement_type   TEXT NOT NULL
                    CHECK (movement_type IN (
                        'sale', 'restock', 'adjustment', 
                        'transfer_in', 'transfer_out', 'return', 'damage'
                    )),
    quantity_change  INTEGER NOT NULL,                  -- positive or negative
    quantity_before  INTEGER NOT NULL,
    quantity_after   INTEGER NOT NULL,
    reference_type   TEXT,                              -- 'order', 'transfer', 'manual'
    reference_id     UUID,                              -- order_id or transfer_id
    notes           TEXT,
    performed_by    UUID REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_movements_inventory ON inventory_movements(inventory_id);
CREATE INDEX idx_movements_tenant_date ON inventory_movements(tenant_id, created_at DESC);
```

---

## Customers & CRM

### customers

```sql
CREATE TABLE customers (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    first_name      TEXT,
    last_name       TEXT,
    email           TEXT,
    phone           TEXT,
    whatsapp        TEXT,
    addresses       JSONB NOT NULL DEFAULT '[]',
    -- [{ "label": "Home", "line1": "...", "city": "Lagos", "state": "Lagos", "is_default": true }]
    tags            TEXT[] DEFAULT '{}',
    notes           TEXT,
    source          TEXT DEFAULT 'manual'
                    CHECK (source IN ('manual', 'storefront', 'whatsapp', 'import')),
    
    -- Denormalized aggregates (updated by triggers/jobs)
    total_spent     BIGINT NOT NULL DEFAULT 0,         -- in kobo
    order_count     INTEGER NOT NULL DEFAULT 0,
    last_order_at   TIMESTAMPTZ,
    
    accepts_marketing BOOLEAN NOT NULL DEFAULT false,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    
    UNIQUE(tenant_id, email),
    UNIQUE(tenant_id, phone)
);

CREATE INDEX idx_customers_tenant ON customers(tenant_id);
CREATE INDEX idx_customers_search ON customers USING GIN(
    to_tsvector('simple', coalesce(first_name, '') || ' ' || coalesce(last_name, '') || ' ' || coalesce(email, '') || ' ' || coalesce(phone, ''))
);
```

### customer_groups

```sql
CREATE TABLE customer_groups (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    name            TEXT NOT NULL,
    description     TEXT,
    filter_criteria JSONB NOT NULL DEFAULT '{}',
    -- Dynamic segmentation rules:
    -- {
    --   "min_total_spent": 50000,    -- ₦500+
    --   "min_order_count": 3,
    --   "tags_include": ["vip"],
    --   "last_order_within_days": 30
    -- }
    is_default      BOOLEAN NOT NULL DEFAULT false,    -- "All Customers" group
    member_count    INTEGER NOT NULL DEFAULT 0,        -- cached count
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

### customer_group_members

```sql
CREATE TABLE customer_group_members (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    group_id        UUID NOT NULL REFERENCES customer_groups(id) ON DELETE CASCADE,
    customer_id     UUID NOT NULL REFERENCES customers(id) ON DELETE CASCADE,
    added_at        TIMESTAMPTZ NOT NULL DEFAULT now(),
    
    UNIQUE(group_id, customer_id)
);
```

---

## Orders & Checkout

### orders

```sql
CREATE TABLE orders (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    customer_id     UUID REFERENCES customers(id) ON DELETE SET NULL,
    location_id     UUID REFERENCES locations(id),
    
    order_number    TEXT NOT NULL,                      -- "PNT-0001" (auto-generated per tenant)
    status          TEXT NOT NULL DEFAULT 'pending'
                    CHECK (status IN (
                        'pending', 'confirmed', 'processing', 
                        'shipped', 'delivered', 'cancelled', 'refunded'
                    )),
    payment_status  TEXT NOT NULL DEFAULT 'unpaid'
                    CHECK (payment_status IN ('unpaid', 'paid', 'partial', 'refunded')),
    
    channel         TEXT NOT NULL DEFAULT 'web'
                    CHECK (channel IN ('web', 'whatsapp', 'instagram', 'manual', 'pos')),
    
    -- Money (all in kobo)
    subtotal        BIGINT NOT NULL DEFAULT 0,
    discount_amount BIGINT NOT NULL DEFAULT 0,
    discount_code   TEXT,
    shipping_cost   BIGINT NOT NULL DEFAULT 0,
    tax_amount      BIGINT NOT NULL DEFAULT 0,
    total           BIGINT NOT NULL DEFAULT 0,
    currency        TEXT NOT NULL DEFAULT 'NGN',
    
    -- Shipping
    shipping_address JSONB,
    billing_address  JSONB,
    
    -- Metadata
    notes           TEXT,                              -- merchant notes
    customer_notes  TEXT,                              -- customer's order notes
    metadata        JSONB NOT NULL DEFAULT '{}',
    
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    completed_at    TIMESTAMPTZ,
    cancelled_at    TIMESTAMPTZ,
    
    UNIQUE(tenant_id, order_number)
);

CREATE INDEX idx_orders_tenant_status ON orders(tenant_id, status);
CREATE INDEX idx_orders_tenant_date ON orders(tenant_id, created_at DESC);
CREATE INDEX idx_orders_customer ON orders(customer_id);

-- Auto-generate order number per tenant
CREATE OR REPLACE FUNCTION generate_order_number()
RETURNS TRIGGER AS $$
DECLARE
    next_num INTEGER;
BEGIN
    SELECT COALESCE(MAX(
        CAST(SUBSTRING(order_number FROM 'PNT-(\d+)') AS INTEGER)
    ), 0) + 1
    INTO next_num
    FROM orders
    WHERE tenant_id = NEW.tenant_id;
    
    NEW.order_number := 'PNT-' || LPAD(next_num::TEXT, 5, '0');
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trigger_order_number
    BEFORE INSERT ON orders
    FOR EACH ROW
    WHEN (NEW.order_number IS NULL)
    EXECUTE FUNCTION generate_order_number();
```

### order_items

```sql
CREATE TABLE order_items (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    order_id        UUID NOT NULL REFERENCES orders(id) ON DELETE CASCADE,
    product_id      UUID REFERENCES products(id) ON DELETE SET NULL,
    variant_id      UUID REFERENCES product_variants(id) ON DELETE SET NULL,
    
    -- Snapshot at time of order (prices may change later)
    product_name    TEXT NOT NULL,
    variant_name    TEXT,
    sku             TEXT,
    unit_price      BIGINT NOT NULL,                   -- in kobo
    quantity        INTEGER NOT NULL DEFAULT 1,
    total           BIGINT NOT NULL,                   -- unit_price * quantity
    
    metadata        JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_order_items_order ON order_items(order_id);
```

---

## Payments

### payments

```sql
CREATE TABLE payments (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    order_id        UUID REFERENCES orders(id) ON DELETE SET NULL,
    
    provider        TEXT NOT NULL
                    CHECK (provider IN ('paystack', 'flutterwave', 'bank_transfer', 'cash')),
    provider_ref    TEXT,                              -- Paystack/Flutterwave transaction reference
    
    amount          BIGINT NOT NULL,                   -- total amount paid in kobo
    platform_fee    BIGINT NOT NULL DEFAULT 0,         -- Punta commission fee (in kobo)
    gateway_fee     BIGINT NOT NULL DEFAULT 0,         -- Paystack/Flutterwave fee (in kobo)
    currency        TEXT NOT NULL DEFAULT 'NGN',
    status          TEXT NOT NULL DEFAULT 'pending'
                    CHECK (status IN ('pending', 'processing', 'success', 'failed', 'refunded')),
    
    payment_method  TEXT,                              -- 'card', 'bank_transfer', 'ussd', 'qr'
    metadata        JSONB NOT NULL DEFAULT '{}',       -- gateway response data
    
    paid_at         TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_payments_order ON payments(order_id);
CREATE INDEX idx_payments_tenant_status ON payments(tenant_id, status);
CREATE INDEX idx_payments_provider_ref ON payments(provider_ref);
```

### merchant_wallets

```sql
CREATE TABLE merchant_wallets (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL UNIQUE REFERENCES tenants(id) ON DELETE CASCADE,
    balance         BIGINT NOT NULL DEFAULT 0,         -- in kobo
    currency        TEXT NOT NULL DEFAULT 'NGN',
    bank_name       TEXT,
    account_number  TEXT,
    account_name    TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

### wallet_transactions

```sql
CREATE TABLE wallet_transactions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    wallet_id       UUID NOT NULL REFERENCES merchant_wallets(id),
    type            TEXT NOT NULL CHECK (type IN ('credit', 'debit')),
    amount          BIGINT NOT NULL,
    balance_before  BIGINT NOT NULL,
    balance_after   BIGINT NOT NULL,
    reference_type  TEXT,                              -- 'payment', 'withdrawal', 'message_charge', 'addon_purchase', 'logistics_charge'
    reference_id    UUID,
    description     TEXT,
    status          TEXT NOT NULL DEFAULT 'completed'
                    CHECK (status IN ('pending', 'completed', 'failed')),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_wallet_tx_tenant ON wallet_transactions(tenant_id, created_at DESC);
```

---

## Messaging

### message_templates

```sql
CREATE TABLE message_templates (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    name            TEXT NOT NULL,
    channel         TEXT NOT NULL CHECK (channel IN ('whatsapp', 'sms', 'email')),
    subject         TEXT,                              -- for email
    body            TEXT NOT NULL,                     -- supports {{first_name}}, {{order_number}}, etc.
    variables       TEXT[] DEFAULT '{}',               -- list of supported variables
    status          TEXT NOT NULL DEFAULT 'draft'
                    CHECK (status IN ('draft', 'pending_approval', 'approved', 'rejected')),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

### messages

```sql
CREATE TABLE messages (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    customer_id     UUID REFERENCES customers(id) ON DELETE SET NULL,
    template_id     UUID REFERENCES message_templates(id),
    campaign_id     UUID,                              -- FK added after campaigns table
    
    channel         TEXT NOT NULL CHECK (channel IN ('whatsapp', 'sms', 'email')),
    direction       TEXT NOT NULL DEFAULT 'outbound'
                    CHECK (direction IN ('inbound', 'outbound')),
    
    recipient       TEXT NOT NULL,                     -- phone or email
    subject         TEXT,
    body            TEXT NOT NULL,
    
    status          TEXT NOT NULL DEFAULT 'queued'
                    CHECK (status IN ('queued', 'sent', 'delivered', 'read', 'failed', 'bounced')),
    
    provider_ref    TEXT,                              -- external message ID
    error_message   TEXT,
    metadata        JSONB NOT NULL DEFAULT '{}',
    
    sent_at         TIMESTAMPTZ,
    delivered_at    TIMESTAMPTZ,
    read_at         TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_messages_tenant_date ON messages(tenant_id, created_at DESC);
CREATE INDEX idx_messages_customer ON messages(customer_id);
CREATE INDEX idx_messages_campaign ON messages(campaign_id);
```

### campaigns

```sql
CREATE TABLE campaigns (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    name            TEXT NOT NULL,
    channel         TEXT NOT NULL CHECK (channel IN ('whatsapp', 'sms', 'email')),
    template_id     UUID REFERENCES message_templates(id),
    customer_group_id UUID REFERENCES customer_groups(id),
    
    status          TEXT NOT NULL DEFAULT 'draft'
                    CHECK (status IN ('draft', 'scheduled', 'sending', 'sent', 'cancelled')),
    
    total_recipients INTEGER NOT NULL DEFAULT 0,
    sent_count      INTEGER NOT NULL DEFAULT 0,
    delivered_count INTEGER NOT NULL DEFAULT 0,
    read_count      INTEGER NOT NULL DEFAULT 0,
    failed_count    INTEGER NOT NULL DEFAULT 0,
    
    scheduled_at    TIMESTAMPTZ,
    started_at      TIMESTAMPTZ,
    completed_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

## Expenses

### expenses

```sql
CREATE TABLE expenses (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    title           TEXT NOT NULL,
    category        TEXT NOT NULL,                     -- 'rent', 'utilities', 'supplies', 'marketing', 'other'
    amount          BIGINT NOT NULL,                   -- in kobo
    currency        TEXT NOT NULL DEFAULT 'NGN',
    expense_date    DATE NOT NULL DEFAULT CURRENT_DATE,
    recurrence      TEXT NOT NULL DEFAULT 'none'
                    CHECK (recurrence IN ('none', 'daily', 'weekly', 'monthly', 'yearly')),
    notes           TEXT,
    receipt_url     TEXT,                              -- uploaded receipt image
    recorded_by     UUID REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_expenses_tenant_date ON expenses(tenant_id, expense_date DESC);
CREATE INDEX idx_expenses_category ON expenses(tenant_id, category);
```

---

## Delivery & Logistics

### deliveries

```sql
CREATE TABLE deliveries (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    order_id        UUID NOT NULL REFERENCES orders(id),
    
    provider        TEXT NOT NULL
                    CHECK (provider IN ('shipbubble', 'fez', 'kwik', 'gig', 'self', 'other')),
    
    tracking_number TEXT,
    tracking_url    TEXT,
    
    status          TEXT NOT NULL DEFAULT 'pending'
                    CHECK (status IN (
                        'pending', 'booked', 'picked_up', 
                        'in_transit', 'delivered', 'failed', 'returned'
                    )),
    
    cost_price      BIGINT NOT NULL DEFAULT 0,         -- actual carrier cost billed to Punta (in kobo)
    price           BIGINT NOT NULL DEFAULT 0,         -- amount merchant/customer was charged including markup (in kobo)
    currency        TEXT NOT NULL DEFAULT 'NGN',
    
    pickup_address  JSONB,
    delivery_address JSONB,
    
    estimated_delivery TIMESTAMPTZ,
    actual_delivery    TIMESTAMPTZ,
    
    provider_metadata JSONB NOT NULL DEFAULT '{}',     -- raw provider response
    
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_deliveries_order ON deliveries(order_id);
CREATE INDEX idx_deliveries_tenant_status ON deliveries(tenant_id, status);
```

---

## Tenant Premium Features

### tenant_features

```sql
CREATE TABLE tenant_features (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    feature_key     TEXT NOT NULL,                     -- 'custom_domain', 'multi_location'
    expires_at      TIMESTAMPTZ,                       -- NULL for lifetime, or expiration date
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    
    UNIQUE(tenant_id, feature_key)
);

CREATE INDEX idx_tenant_features_tenant ON tenant_features(tenant_id);
```

### merchant_kyc

```sql
CREATE TABLE merchant_kyc (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL UNIQUE REFERENCES tenants(id) ON DELETE CASCADE,
    kyc_level       TEXT NOT NULL DEFAULT 'tier_1' CHECK (kyc_level IN ('tier_1', 'tier_2')),
    status          TEXT NOT NULL DEFAULT 'pending' CHECK (status IN ('pending', 'verified', 'rejected')),
    
    -- Tier 1 data (encrypted at rest / hashed)
    bvn_hash        TEXT,                              -- Hashed BVN for identity checks
    nin_hash        TEXT,                              -- Hashed NIN for identity checks
    
    -- Tier 2 data
    cac_number      TEXT,                              -- Corporate Affairs Commission reg number
    business_type   TEXT,                              -- 'sole_proprietorship', 'ltd', etc.
    document_urls   JSONB NOT NULL DEFAULT '[]',       -- CAC certificate, utility bills, director IDs
    
    verified_at     TIMESTAMPTZ,
    verified_by     UUID REFERENCES users(id),         -- Admin user who approved it
    rejection_reason TEXT,
    
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_merchant_kyc_tenant ON merchant_kyc(tenant_id);
```

---

## Analytics (Materialized / Aggregated)

### daily_sales_stats

```sql
-- Pre-aggregated daily sales stats (populated by background job)
CREATE TABLE daily_sales_stats (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    date            DATE NOT NULL,
    
    total_revenue   BIGINT NOT NULL DEFAULT 0,
    total_orders    INTEGER NOT NULL DEFAULT 0,
    total_items     INTEGER NOT NULL DEFAULT 0,
    new_customers   INTEGER NOT NULL DEFAULT 0,
    
    revenue_by_channel JSONB NOT NULL DEFAULT '{}',    -- {"web": 50000, "whatsapp": 30000}
    top_products       JSONB NOT NULL DEFAULT '[]',    -- [{"product_id": "...", "name": "...", "qty": 5, "revenue": 25000}]
    
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    
    UNIQUE(tenant_id, date)
);

CREATE INDEX idx_daily_stats_tenant_date ON daily_sales_stats(tenant_id, date DESC);
```

---

## Migration Order

Migrations should be applied in this order (respecting foreign key dependencies):

```
001_create_extensions.sql         -- uuid-ossp, pg_trgm, etc.
002_create_tenants.sql            -- Global tenant table
003_create_tenant_features.sql    -- Premium feature unlocks
004_create_users.sql              -- Users with RLS
005_create_refresh_tokens.sql
006_create_stores.sql
007_create_categories.sql
008_create_products.sql
009_create_product_variants.sql
010_create_product_images.sql
011_create_locations.sql
012_create_inventory.sql
013_create_inventory_movements.sql
014_create_customers.sql
015_create_customer_groups.sql
016_create_customer_group_members.sql
017_create_orders.sql
018_create_order_items.sql
019_create_payments.sql
020_create_merchant_wallets.sql
021_create_wallet_transactions.sql
022_create_message_templates.sql
023_create_messages.sql
024_create_campaigns.sql
025_create_expenses.sql
026_create_deliveries.sql
027_create_merchant_kyc.sql       -- KYC verification tracking
029_create_daily_sales_stats.sql
```
