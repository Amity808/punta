# 05 — API Design

## API Conventions

| Convention | Value |
|:-----------|:------|
| Base URL | `https://api.punta.shop/v1` |
| Format | JSON |
| Auth | Bearer JWT in `Authorization` header |
| Tenant Resolution | Via subdomain (Host header) or JWT `tenant_id` claim |
| Pagination | Cursor-based (`?cursor=xxx&limit=20`) |
| Filtering | Query params (`?status=active&category_id=xxx`) |
| Sorting | `?sort=created_at&order=desc` |
| Errors | Consistent error envelope (see below) |
| Money | All amounts in **kobo** (integers) |
| Dates | ISO 8601 (`2025-01-15T10:30:00Z`) |

### Error Response Format

```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Request validation failed",
    "details": [
      {
        "field": "price",
        "message": "Price must be greater than 0"
      }
    ]
  }
}
```

### Error Codes

| HTTP Status | Code | Description |
|:------------|:-----|:------------|
| 400 | `VALIDATION_ERROR` | Invalid request body |
| 401 | `UNAUTHORIZED` | Missing or invalid token |
| 403 | `FORBIDDEN` | Insufficient permissions |
| 403 | `FEATURE_LOCKED` | Feature is locked (requires add-on purchase) |
| 404 | `NOT_FOUND` | Resource not found |
| 409 | `CONFLICT` | Duplicate resource (e.g., slug taken) |
| 429 | `RATE_LIMITED` | Too many requests |
| 500 | `INTERNAL_ERROR` | Server error |

### Pagination Response Format

```json
{
  "data": [...],
  "pagination": {
    "total": 150,
    "limit": 20,
    "cursor": "eyJpZCI6Ijk5OTk5...",
    "has_more": true
  }
}
```

---

## Auth Endpoints

### POST `/v1/auth/register`
Register a new merchant and create their tenant.

```json
// Request
{
  "business_name": "Acme Clothing",
  "slug": "acme",
  "email": "hello@acmeclothing.ng",
  "password": "SecurePass123!",
  "phone": "+2348012345678",
  "full_name": "John Doe"
}

// Response 201
{
  "data": {
    "user": { "id": "...", "email": "...", "full_name": "...", "role": "owner" },
    "tenant": { "id": "...", "slug": "acme", "business_name": "Acme Clothing" },
    "access_token": "eyJ...",
    "refresh_token": "dGhpcyBpcyBhIHJlZnJlc2g...",
    "expires_in": 900
  }
}
```

### POST `/v1/auth/login`

```json
// Request
{ "email": "hello@acmeclothing.ng", "password": "SecurePass123!" }

// Response 200
{
  "data": {
    "user": { "id": "...", "email": "...", "full_name": "...", "role": "owner" },
    "tenant": { "id": "...", "slug": "acme", "is_active": true },
    "access_token": "eyJ...",
    "refresh_token": "...",
    "expires_in": 900
  }
}
```

### POST `/v1/auth/refresh`

```json
// Request
{ "refresh_token": "dGhpcyBpcyBhIHJlZnJlc2g..." }

// Response 200
{ "data": { "access_token": "eyJ...", "expires_in": 900 } }
```

### POST `/v1/auth/forgot-password`

```json
{ "email": "hello@acmeclothing.ng" }
// Response 200 (always, to prevent email enumeration)
{ "data": { "message": "If an account exists, a reset link has been sent." } }
```

### POST `/v1/auth/reset-password`

```json
{ "token": "reset_token_from_email", "password": "NewSecurePass456!" }
```

---

## Tenant & Settings

### GET `/v1/tenant`
Returns current tenant details, unified wallet balance, and list of unlocked features.

```json
// Response 200
{
  "data": {
    "id": "t_abc123",
    "slug": "acme",
    "business_name": "Acme Clothing",
    "email": "hello@acmeclothing.ng",
    "phone": "+2348012345678",
    "wallet_balance": 1250000,
    "unlocked_features": ["custom_domain"],
    "is_active": true
  }
}
```

### PATCH `/v1/tenant`
Update business settings.

```json
{
  "business_name": "Acme Fashion",
  "phone": "+2348012345678",
  "settings": {
    "default_tax_rate": 7.5,
    "order_prefix": "ACM"
  }
}
```

### POST `/v1/tenant/staff`
Invite a staff member.

```json
{
  "email": "staff@acme.ng",
  "full_name": "Jane Smith",
  "role": "staff",
  "permissions": ["manage_products", "manage_orders", "view_analytics"]
}
```

---

## Products

### GET `/v1/products`
List products with filtering.

```
GET /v1/products?status=active&category_id=xxx&search=shirt&sort=created_at&order=desc&limit=20
```

### POST `/v1/products`

```json
{
  "name": "Classic White T-Shirt",
  "description": "Premium cotton t-shirt, available in multiple sizes",
  "price": 500000,
  "compare_at_price": 750000,
  "cost_price": 200000,
  "category_id": "cat_xxx",
  "sku": "CWT-001",
  "tags": ["cotton", "casual", "unisex"],
  "is_active": true,
  "variants": [
    { "name": "Small", "sku": "CWT-001-S", "attributes": { "size": "S" } },
    { "name": "Medium", "sku": "CWT-001-M", "attributes": { "size": "M" } },
    { "name": "Large", "sku": "CWT-001-L", "attributes": { "size": "L" } }
  ]
}
```

### POST `/v1/products/:id/images`
Upload product images (returns presigned URL for direct upload).

```json
// Request
{ "filename": "tshirt-front.jpg", "content_type": "image/jpeg" }

// Response
{
  "data": {
    "upload_url": "https://r2.punta.shop/upload?X-Amz-...",
    "object_key": "tenants/t_abc/products/p_xyz/tshirt-front.jpg",
    "image_id": "img_xxx"
  }
}
```

---

## Orders

### GET `/v1/orders`

```
GET /v1/orders?status=pending&payment_status=paid&channel=web&from=2025-01-01&to=2025-01-31
```

### POST `/v1/orders`
Create a manual order (e.g., from WhatsApp or in-person sale).

```json
{
  "customer_id": "cust_xxx",
  "channel": "whatsapp",
  "items": [
    { "product_id": "prod_xxx", "variant_id": "var_xxx", "quantity": 2 }
  ],
  "discount_amount": 100000,
  "shipping_cost": 150000,
  "shipping_address": {
    "line1": "15 Admiralty Way",
    "city": "Lekki",
    "state": "Lagos",
    "country": "NG"
  },
  "notes": "Customer wants gift wrapping"
}
```

### PATCH `/v1/orders/:id/status`

```json
{ "status": "confirmed" }
// Triggers: confirmation notification to customer
```

### POST `/v1/orders/:id/invoice`
Generate and return invoice PDF URL.

---

## Storefront (Public — No Auth Required)

These endpoints power the public-facing store at `acme.punta.shop`.

### GET `/v1/storefront/store`
Store metadata (name, logo, theme, social links).

### GET `/v1/storefront/products`
Public product listing (only active, published products).

### GET `/v1/storefront/products/:slug`
Single product detail.

### POST `/v1/storefront/cart`
Manage cart (stored in Redis, keyed by session).

```json
{ "product_id": "...", "variant_id": "...", "quantity": 1 }
```

### POST `/v1/storefront/checkout`
Initiate checkout → returns payment URL.

```json
{
  "customer": { "first_name": "...", "last_name": "...", "email": "...", "phone": "..." },
  "shipping_address": { "line1": "...", "city": "...", "state": "...", "country": "NG" },
  "payment_provider": "paystack"
}

// Response
{
  "data": {
    "order_id": "...",
    "order_number": "PNT-00042",
    "payment_url": "https://checkout.paystack.com/xxx",
    "total": 650000
  }
}
```

---

## Payments

### POST `/v1/payments/initialize`
Initialize payment for an order.

### POST `/v1/webhooks/paystack`
Paystack webhook (verify signature, update payment status).

### POST `/v1/webhooks/flutterwave`
Flutterwave webhook.

### GET `/v1/wallet`
Merchant wallet balance and recent transactions.

### POST `/v1/wallet/withdraw`

```json
{
  "amount": 5000000,
  "bank_code": "058",
  "account_number": "0123456789"
}
```

### POST `/v1/wallet/topup`
Initialize a manual topup payment for the wallet balance (returns payment initialization details).

```json
// Request
{
  "amount": 1000000,
  "payment_provider": "paystack"
}

// Response 200
{
  "data": {
    "reference": "topup_ref_99283",
    "payment_url": "https://checkout.paystack.com/xxx"
  }
}
```

### GET `/v1/wallet/kyc`
Get the current merchant's KYC status and submission details.

```json
// Response 200
{
  "data": {
    "kyc_level": "tier_1",
    "status": "verified",
    "bvn_submitted": true,
    "nin_submitted": false,
    "cac_number": null,
    "document_urls": [],
    "rejection_reason": null,
    "verified_at": "2026-06-05T12:00:00Z"
  }
}
```

### POST `/v1/wallet/kyc`
Submit KYC identification details for verification.

```json
// Request (Tier 1 submit)
{
  "kyc_level": "tier_1",
  "bvn": "22233344455",
  "nin": "11122233344"
}

// Response 200
{
  "data": {
    "kyc_level": "tier_1",
    "status": "pending",
    "bvn_submitted": true,
    "nin_submitted": true,
    "verified_at": null
  }
}
```

### POST `/v1/wallet/kyc/business`
Submit business documents for Tier 2 verification.

```json
// Request (Tier 2 submit)
{
  "cac_number": "RC1234567",
  "business_type": "ltd",
  "document_urls": [
    "https://r2.punta.shop/tenants/t_abc/kyc/cac_cert.pdf",
    "https://r2.punta.shop/tenants/t_abc/kyc/utility_bill.pdf"
  ]
}

// Response 200
{
  "data": {
    "kyc_level": "tier_2",
    "status": "pending",
    "cac_number": "RC1234567",
    "document_urls": [
      "https://r2.punta.shop/tenants/t_abc/kyc/cac_cert.pdf",
      "https://r2.punta.shop/tenants/t_abc/kyc/utility_bill.pdf"
    ],
    "verified_at": null
  }
}
```

---

## Customers

### GET `/v1/customers`

```
GET /v1/customers?search=john&tags=vip&sort=total_spent&order=desc
```

### GET `/v1/customers/:id`
Full customer profile including order history.

### POST `/v1/customers`
### PATCH `/v1/customers/:id`
### GET `/v1/customer-groups`
### POST `/v1/customer-groups`

---

## Messaging

### POST `/v1/messages/send`
Send a single message to a customer.

```json
{
  "customer_id": "cust_xxx",
  "channel": "whatsapp",
  "template_id": "tmpl_xxx",
  "variables": { "order_number": "PNT-00042", "status": "shipped" }
}
```

### POST `/v1/campaigns`
Create and schedule a bulk campaign.

```json
{
  "name": "January Sale Announcement",
  "channel": "whatsapp",
  "template_id": "tmpl_xxx",
  "customer_group_id": "grp_xxx",
  "scheduled_at": "2025-01-15T09:00:00Z"
}
```

---

## Inventory

### GET `/v1/inventory`

```
GET /v1/inventory?product_id=xxx&location_id=xxx&low_stock=true
```

### POST `/v1/inventory/adjust`

```json
{
  "product_id": "prod_xxx",
  "variant_id": "var_xxx",
  "location_id": "loc_xxx",
  "adjustment": 50,
  "reason": "restock"
}
```

### POST `/v1/inventory/transfer`

```json
{
  "product_id": "prod_xxx",
  "from_location_id": "loc_warehouse",
  "to_location_id": "loc_store",
  "quantity": 20
}
```

---

## Analytics

### GET `/v1/analytics/dashboard`
Overview metrics for the dashboard.

```json
// Response
{
  "data": {
    "period": "today",
    "revenue": 2500000,
    "orders": 12,
    "new_customers": 3,
    "conversion_rate": 4.2,
    "comparison": {
      "revenue_change": 15.3,
      "orders_change": -5.0
    }
  }
}
```

### GET `/v1/analytics/sales`

```
GET /v1/analytics/sales?period=30d&group_by=day
```

### GET `/v1/analytics/products`
Top performing products.

### GET `/v1/analytics/customers`
Customer insights (new vs returning, top spenders, etc.).

---

## Expenses

### GET `/v1/expenses`
### POST `/v1/expenses`
### GET `/v1/expenses/summary`

```
GET /v1/expenses/summary?period=month&year=2025&month=1
```

---

## Delivery

### POST `/v1/deliveries/rates`
Get shipping quotes from logistics providers.

```json
{
  "pickup_address": { "city": "Ikeja", "state": "Lagos" },
  "delivery_address": { "city": "Lekki", "state": "Lagos" },
  "weight_grams": 500,
  "items_count": 2
}

// Response
{
  "data": {
    "quotes": [
      { "provider": "shipbubble", "service": "Standard", "cost": 150000, "eta_days": 2 },
      { "provider": "fez", "service": "Express", "cost": 250000, "eta_days": 1 }
    ]
  }
}
```

### POST `/v1/deliveries`
Book a delivery.

### GET `/v1/deliveries/:id/track`
Get tracking info.

---

## Tenant Add-ons

### GET `/v1/tenant/addons`
Get the status of all optional premium add-ons for the tenant.

```json
// Response 200
{
  "data": [
    {
      "feature_key": "custom_domain",
      "display_name": "Custom Domain Mapping",
      "price": 1000000,
      "status": "unlocked",
      "expires_at": "2027-06-05T17:00:00Z"
    },
    {
      "feature_key": "multi_location",
      "display_name": "Multi-Location Support",
      "price": 500000,
      "status": "locked",
      "expires_at": null
    }
  ]
}
```

### POST `/v1/tenant/addons/purchase`
Purchase/unlock an add-on using funds from the merchant wallet balance.

```json
// Request
{
  "feature_key": "multi_location"
}

// Response 200
{
  "data": {
    "feature_key": "multi_location",
    "status": "unlocked",
    "expires_at": "2027-06-05T17:00:00Z",
    "wallet_balance": 750000
  }
}
```
