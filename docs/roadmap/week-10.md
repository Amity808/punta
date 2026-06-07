# Week 10 — Expenses & Delivery

> **Goal:** Build expense tracking with P&L reports, and integrate logistics providers for delivery management.

---

## Monday — Expense Backend

- [ ] Write migration: `025_create_expenses.sql`
- [ ] Create `punta-expenses` crate
- [ ] Implement `expense.rs` — `ExpenseService`
  - `create_expense()` — with category, amount, date, receipt upload
  - `list_expenses()` — paginated, filterable (category, date range)
  - `update_expense()` — edit expense
  - `delete_expense()` — remove expense
  - Expense categories: Rent, Utilities, Supplies, Marketing, Transport, Salaries, Logistics, Other
- [ ] Implement `recurring.rs` — recurring expense handling
  - When creating: set recurrence (daily, weekly, monthly, yearly)
  - Background job: auto-create new expense entries based on recurrence
  - Ability to stop/modify recurrence
- [ ] Handlers: `GET/POST/PATCH/DELETE /v1/expenses`

**Deliverable:** Expense CRUD API with recurring support.

---

## Tuesday — Expense Reports & P&L

- [ ] Implement `reports.rs` — `ExpenseReports`
  - `get_expense_summary()` — total expenses by period (day, week, month)
  - `get_expenses_by_category()` — breakdown by category with totals
  - `get_profit_and_loss()` — Revenue (from orders) - Expenses = Profit/Loss
    - Revenue from `daily_sales_stats`
    - Expenses from `expenses` table
    - Net profit/loss calculation
  - `get_expense_trend()` — expenses over time (line chart data)
- [ ] Handler: `GET /v1/expenses/summary?period=month&year=2025&month=6`
- [ ] Handler: `GET /v1/expenses/report/pnl?from=2025-01-01&to=2025-06-30`
- [ ] Write tests: seed expenses + orders, verify P&L calculation

**Deliverable:** Expense reports and P&L API working.

---

## Wednesday — Delivery Backend

- [ ] Write migration: `026_create_deliveries.sql`
- [ ] Create `punta-delivery` crate
- [ ] Implement `provider.rs` — `LogisticsProvider` trait
  - `get_rates()` — fetch shipping quotes
  - `book_shipment()` — create shipment booking
  - `get_tracking()` — get tracking info
  - `cancel_shipment()` — cancel booking
- [ ] Implement `shipbubble.rs` — Shipbubble API client
  - Rate fetching from multiple courier partners
  - Shipment booking
  - Tracking number retrieval
  - Webhook handling for status updates
- [ ] Implement `rates.rs` — `RateCalculator`
  - Aggregate rates from available providers
  - Sort by price, estimated delivery time
  - Cache rates for short period (5 min)
- [ ] Implement `tracking.rs` — `TrackingService`
  - Store tracking events
  - Status mapping (provider status → Punta status)
- [ ] Handlers:
  - `POST /v1/deliveries/rates` — get shipping quotes
  - `POST /v1/deliveries` — book delivery
  - `GET /v1/deliveries/:id/track` — tracking info
  - `POST /v1/webhooks/delivery` — delivery status webhooks

**Deliverable:** Logistics integration with rate comparison and booking.

---

## Thursday — Frontend: Expenses

- [ ] Create `(dashboard)/expenses/page.tsx` — Expense list
  - Table: Date, Title, Category, Amount, Recurrence, Receipt
  - Filter by category, date range
  - Category-colored badges
  - "Add Expense" button
  - Monthly total in page header
- [ ] Create `(dashboard)/expenses/new/page.tsx` — Add expense form
  - Title, category (dropdown), amount, date
  - Recurrence selector (none, daily, weekly, monthly, yearly)
  - Receipt upload (image)
  - Notes
- [ ] Add expense summary cards on expense page:
  - This month total, vs last month (% change)
  - By category donut chart
  - P&L summary: Revenue | Expenses | Net Profit
- [ ] Implement P&L report view (table format):
  - Revenue by month
  - Expenses by category by month
  - Net profit row
  - Highlight profit (green) / loss (red)

**Deliverable:** Expense management and P&L reporting in dashboard.

---

## Friday — Frontend: Deliveries & Integration

- [ ] Create `(dashboard)/deliveries/page.tsx` — Delivery list
  - Table: Order #, Provider, Tracking #, Status, Cost, ETA
  - Status badges with colors
  - Filter by status, provider
- [ ] Create `(dashboard)/deliveries/[id]/page.tsx` — Delivery tracking
  - Tracking timeline (pending → picked up → in transit → delivered)
  - Map view placeholder (future)
  - Delivery details (addresses, cost, provider info)
- [ ] Add "Ship Order" flow to order detail page:
  - Click "Ship" on confirmed order
  - Fetch shipping rates → show comparison table
  - Select provider + service
  - Confirm booking → creates delivery record
  - Order status → "shipped"
  - Customer notification triggered
- [ ] Wire delivery webhook → update delivery status → notify customer
- [ ] Test full flow: order confirmed → book delivery → tracking updates → delivered

**Deliverable:** Complete delivery management with order integration.

---

## Week 10 Checklist

```
✅ Expense CRUD API with categories
✅ Recurring expense auto-generation
✅ Expense reports (by category, by period)
✅ Profit & Loss calculation
✅ Logistics provider trait abstraction
✅ Shipbubble API integration (rates, booking, tracking)
✅ Delivery status webhooks
✅ Expense list and creation pages
✅ P&L report view
✅ Delivery list and tracking pages
✅ "Ship Order" flow from order detail
✅ Customer delivery notifications
✅ End-to-end order → delivery → tracking flow
```
