# Week 6 — Core Commerce: Payments & Inventory

> **Goal:** Integrate Paystack + Flutterwave payment gateways and build the complete inventory management system.

---

## Monday — Payment Gateway Abstraction & Paystack

- [ ] Write migrations: `019_create_payments.sql`, `020_create_merchant_wallets.sql`, `021_create_wallet_transactions.sql`
- [ ] Create `punta-payments` crate
- [ ] Implement `gateway.rs` — `PaymentGateway` trait (initialize, verify, transfer, verify_webhook)
- [ ] Implement `paystack.rs` — `PaystackGateway`
  - `initialize()` — POST to Paystack transaction/initialize
  - `verify()` — GET Paystack transaction/verify/:reference
  - `verify_webhook_signature()` — HMAC SHA512 verification
  - `transfer()` — initiate bank transfer for withdrawals
- [ ] Implement `webhook.rs` — Paystack webhook handler
  - Parse `charge.success` event
  - Verify signature
  - Idempotency check (Redis: store processed event IDs)
  - Update payment status
  - Update order payment_status
  - Credit merchant wallet
- [ ] Handler: `POST /v1/webhooks/paystack`
- [ ] Test with Paystack test keys: initialize → mock payment → verify webhook

**Deliverable:** Paystack payment flow working with test keys.

---

## Tuesday — Flutterwave & Merchant Wallet

- [ ] Implement `flutterwave.rs` — `FlutterwaveGateway`
  - Same trait implementation for Flutterwave endpoints
  - Webhook verification with `verifi-hash` header
- [ ] Handler: `POST /v1/webhooks/flutterwave`
- [ ] Implement `wallet.rs` — `WalletService`
  - `get_balance()` — current wallet balance
  - `credit()` — add funds (on successful payment, minus platform fee)
  - `request_withdrawal()` — initiate payout to merchant's bank
  - `get_transactions()` — wallet transaction history
- [ ] Implement platform fee calculation (configurable %, e.g., 1.5%)
- [ ] Handlers: `GET /v1/wallet`, `GET /v1/wallet/transactions`, `POST /v1/wallet/withdraw`
- [ ] Implement `POST /v1/payments/initialize` — initialize payment for an order
  - Select gateway based on merchant preference or customer choice
  - Generate unique reference
  - Call gateway.initialize()
  - Return checkout URL

**Deliverable:** Both payment gateways working, merchant wallet system operational.

---

## Wednesday — Inventory Backend

- [ ] Write migrations: `011_create_locations.sql`, `012_create_inventory.sql`, `013_create_inventory_movements.sql`
- [ ] Create `punta-inventory` crate
- [ ] Implement `stock.rs` — `StockService`
  - `get_stock_levels()` — list inventory with filters (product, location, low_stock)
  - `adjust_stock()` — manual adjustment (with movement record)
  - `reserve_stock()` — reserve for pending order (atomic)
  - `release_stock()` — release reservation (order cancelled)
  - `confirm_stock()` — deduct reserved stock (order fulfilled)
- [ ] Implement `location.rs` — `LocationService`
  - CRUD for locations (store, warehouse)
  - Default location per tenant
- [ ] Implement `transfer.rs` — inter-location stock transfer
  - Create `transfer_out` movement at source
  - Create `transfer_in` movement at destination
  - Atomic transaction
- [ ] Implement `alerts.rs` — low stock detection
  - Query: `WHERE quantity - reserved <= reorder_level`
  - Return list of products below reorder threshold

**Deliverable:** Complete inventory backend with stock operations and multi-location.

---

## Thursday — Inventory Integration & Frontend

- [ ] Wire inventory into order flow:
  - On order creation → reserve stock
  - On order cancellation → release stock
  - On order delivered → confirm stock (deduct)
- [ ] Implement `inventory_movements` logging for all stock changes
- [ ] Handlers: `GET /v1/inventory`, `POST /v1/inventory/adjust`, `POST /v1/inventory/transfer`, `GET /v1/inventory/alerts`
- [ ] Handlers: `GET/POST/PATCH/DELETE /v1/locations`
- [ ] Create `(dashboard)/inventory/page.tsx` — Stock levels page
  - Table: Product, Variant, Location, In Stock, Reserved, Available, Reorder Level
  - Color coding: green (healthy), yellow (low), red (out of stock)
  - Filter by location, category, stock status
  - Search by product name/SKU
- [ ] Create `(dashboard)/inventory/locations/page.tsx` — Locations management

**Deliverable:** Inventory UI showing stock levels, locations manageable.

---

## Friday — Stock Adjustments, Transfers & Payment UI

- [ ] Create `(dashboard)/inventory/adjustments/page.tsx`
  - Select product/variant/location
  - Enter adjustment quantity (+ or -)
  - Select reason (restock, damage, return, correction)
  - Add notes
  - Movement history log
- [ ] Create `(dashboard)/inventory/transfers/page.tsx`
  - Select product, source location, destination, quantity
  - Transfer confirmation
  - Transfer history
- [ ] Add payment settings to dashboard: `(dashboard)/settings/payments/page.tsx`
  - Enter Paystack API keys (public + secret)
  - Enter Flutterwave API keys
  - Bank account details for withdrawals
  - Default payment gateway selection
- [ ] Wire checkout flow to payment initialization
- [ ] Test complete flow: add product → add stock → customer checkout → payment → stock reserved → order confirmed → stock deducted

**Deliverable:** Complete payment + inventory flow tested end-to-end.

---

## Week 6 Checklist

```
✅ Paystack payment integration (initialize, verify, webhook)
✅ Flutterwave payment integration
✅ Payment gateway trait abstraction
✅ Webhook signature verification + idempotency
✅ Merchant wallet (balance, credit, withdrawal)
✅ Platform fee calculation
✅ Inventory stock levels (CRUD, multi-location)
✅ Stock reservation/release/confirmation lifecycle
✅ Inter-location stock transfers
✅ Low stock alerts
✅ Inventory movement audit trail
✅ Inventory dashboard pages (levels, adjustments, transfers)
✅ Payment settings page
✅ End-to-end checkout → payment → inventory flow
```
