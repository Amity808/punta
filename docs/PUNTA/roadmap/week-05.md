# Week 5 — Core Commerce: Orders & Checkout

> **Goal:** Build the order management system — manual orders, cart, checkout flow, and invoice generation.

---

## Monday — Order Database & Service

- [ ] Write migrations: `017_create_orders.sql` (with order number trigger), `018_create_order_items.sql`
- [ ] Run migrations, verify RLS and order number generation
- [ ] Create `punta-orders` crate
- [ ] Implement `order.rs` — `OrderService`
  - `create_order()` — create order with items in a transaction
  - `get_order()` — with items, customer, payment info joined
  - `list_orders()` — paginated, filterable (status, payment_status, channel, date range)
  - `update_order_status()` — validate state transitions (pending → confirmed → processing → shipped → delivered)
  - `cancel_order()` — set cancelled_at, release inventory
- [ ] Implement `number.rs` — auto-generate order numbers (PNT-00001)
- [ ] Handlers: `GET/POST /v1/orders`, `GET/PATCH /v1/orders/:id`, `PATCH /v1/orders/:id/status`
- [ ] Test order creation and status lifecycle

**Deliverable:** Order CRUD API with status management working.

---

## Tuesday — Cart & Checkout Backend

- [ ] Implement `cart.rs` — `CartService` (Redis-backed)
  - `add_to_cart()` — store cart items in Redis hash (keyed by session)
  - `get_cart()` — return items with current prices resolved from DB
  - `update_cart_item()` — change quantity
  - `remove_from_cart()` — remove item
  - `clear_cart()` — empty cart
  - Cart TTL: 7 days of inactivity
- [ ] Implement `checkout.rs` — `CheckoutService`
  - `initiate_checkout()` → validates cart → creates customer (if new) → reserves inventory → creates order → returns order details
  - Validate stock availability before proceeding
  - Calculate totals: subtotal + shipping - discount + tax = total
  - Handle checkout failure (release reserved inventory)
- [ ] Storefront handlers:
  - `POST /v1/storefront/cart` — add item
  - `GET /v1/storefront/cart` — view cart
  - `PATCH /v1/storefront/cart/:item_id` — update quantity
  - `DELETE /v1/storefront/cart/:item_id` — remove item
  - `POST /v1/storefront/checkout` — initiate checkout

**Deliverable:** Cart ↔ Checkout pipeline working end-to-end.

---

## Wednesday — Invoice & Receipt Generation

- [ ] Implement `invoice.rs` — `InvoiceService`
  - Generate invoice data from order (items, totals, tax breakdown)
  - Format as JSON (PDF generation can be done client-side or via a future service)
  - Include merchant info, customer info, order items, totals
  - Invoice numbering (INV-PNT-00042)
- [ ] Handler: `POST /v1/orders/:id/invoice` — generate and return invoice data
- [ ] Handler: `GET /v1/orders/:id/receipt` — simplified receipt format
- [ ] Implement order status change notifications (log for now, email in Week 8):
  - Order placed → log confirmation
  - Order confirmed → log
  - Order shipped → log
  - Order delivered → log
- [ ] Write integration tests for complete order lifecycle

**Deliverable:** Invoice generation working, order lifecycle fully tested.

---

## Thursday — Frontend: Order List & Detail

- [ ] Create `lib/hooks/use-orders.ts` — React Query hooks
- [ ] Create `(dashboard)/orders/page.tsx` — Order list page
  - Table: Order #, Customer, Date, Channel, Status, Payment, Total, Actions
  - Status filter tabs (All, Pending, Processing, Shipped, Delivered, Cancelled)
  - Payment status filter
  - Channel filter (Web, WhatsApp, Manual)
  - Date range picker
  - Search by order number or customer name
  - Status badge colors (yellow=pending, blue=processing, green=delivered, red=cancelled)
- [ ] Create `(dashboard)/orders/[id]/page.tsx` — Order detail page
  - Order summary header (number, date, channel, status)
  - Customer info card
  - Order items table (product, variant, qty, price, total)
  - Order totals (subtotal, discount, shipping, tax, total)
  - Payment info
  - Order timeline (status history with timestamps)
  - Action buttons: Confirm, Process, Ship, Deliver, Cancel

**Deliverable:** Order management pages working with real data.

---

## Friday — Frontend: Manual Order Creation

- [ ] Create `(dashboard)/orders/new/page.tsx` — Manual order form
  - Customer search/select (or create new inline)
  - Product search and add items (with variant selection)
  - Quantity editing with live price calculation
  - Discount input (fixed amount or percentage)
  - Shipping cost input
  - Order totals auto-calculated
  - Channel selector (manual, whatsapp, instagram, pos)
  - Notes field
  - "Create Order" → creates order via API
- [ ] Create `components/dashboard/order-timeline.tsx`
  - Visual timeline of status changes
  - Each step with timestamp and user who made the change
- [ ] Implement order status change from detail page (confirm, ship, deliver, cancel)
- [ ] Add toast notifications for successful actions
- [ ] Test complete flow: create manual order → confirm → mark shipped → mark delivered

**Deliverable:** Complete order management experience in dashboard.

---

## Week 5 Checklist

```
✅ Order CRUD API with status lifecycle
✅ Auto-generated order numbers (PNT-XXXXX)
✅ Redis-backed cart for storefront
✅ Checkout flow (validate → reserve stock → create order)
✅ Invoice/receipt generation
✅ Order list page with filters and search
✅ Order detail page with timeline
✅ Manual order creation form
✅ Status transitions with validation
✅ Integration tests for order lifecycle
```
