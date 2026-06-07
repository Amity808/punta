# Week 9 — Analytics & Reports

> **Goal:** Build the analytics aggregation system and dashboard — sales, product performance, customer insights.

---

## Monday — Analytics Backend & Aggregation

- [ ] Write migration: `029_create_daily_sales_stats.sql`
- [ ] Create `punta-analytics` crate
- [ ] Implement `aggregator.rs` — background aggregation job
  - Run nightly (or on-demand): aggregate previous day's orders into `daily_sales_stats`
  - Calculate: total_revenue, total_orders, total_items, new_customers
  - Break down by channel (revenue_by_channel JSONB)
  - Top products of the day (top_products JSONB)
  - Schedule via `punta-jobs` cron scheduler
- [ ] Implement `dashboard.rs` — `DashboardService`
  - `get_overview()` — today's stats with comparison to yesterday/last week
  - Real-time: query live from `orders` table for today
  - Historical: query from `daily_sales_stats` for past days
  - Return: revenue, orders, new_customers, conversion_rate, % changes
- [ ] Handler: `GET /v1/analytics/dashboard?period=today|7d|30d|90d`

**Deliverable:** Dashboard overview API returning real analytics data.

---

## Tuesday — Sales Analytics

- [ ] Implement `sales.rs` — `SalesAnalytics`
  - `get_sales_over_time()` — revenue & orders grouped by day/week/month
  - `get_sales_by_channel()` — breakdown by channel (web, whatsapp, manual, etc.)
  - `get_sales_by_payment_method()` — card vs transfer vs cash
  - `get_sales_by_location()` — per-location revenue (multi-store)
  - `get_average_order_value()` — AOV trend over time
- [ ] Implement comparison periods:
  - "vs previous period" — e.g., last 30 days vs 30 days before that
  - Calculate % change for each metric
- [ ] Handler: `GET /v1/analytics/sales?period=30d&group_by=day&channel=all`
- [ ] Write tests: seed orders across dates/channels, verify aggregation accuracy

**Deliverable:** Sales analytics API with time-series data.

---

## Wednesday — Product & Customer Analytics

- [ ] Implement `products.rs` — `ProductAnalytics`
  - `get_top_products()` — by revenue, by quantity sold (configurable period)
  - `get_slow_movers()` — products with zero or low sales in period
  - `get_product_performance()` — individual product: revenue trend, units sold trend
  - `get_category_performance()` — sales breakdown by category
- [ ] Implement `customers.rs` — `CustomerAnalytics`
  - `get_customer_growth()` — new customers over time
  - `get_new_vs_returning()` — ratio of first-time vs repeat buyers
  - `get_top_customers()` — by total spent (lifetime)
  - `get_customer_lifetime_value()` — average CLV
  - `get_customer_acquisition_by_channel()` — where customers come from
- [ ] Handlers:
  - `GET /v1/analytics/products?period=30d&sort=revenue&limit=10`
  - `GET /v1/analytics/customers?period=30d`
- [ ] Implement Redis caching for analytics queries (TTL: 5 min for dashboard, 1 hour for reports)

**Deliverable:** Product and customer analytics APIs complete.

---

## Thursday — Frontend: Analytics Dashboard

- [ ] Update `(dashboard)/page.tsx` — wire overview stats to real API
  - Replace mock data with live API calls
  - Add loading skeletons
  - Add period selector (Today, 7 days, 30 days, 90 days)
  - Comparison chips showing % change
- [ ] Create `(dashboard)/analytics/page.tsx` — Analytics overview
  - Revenue chart (line/area chart, 30-day default)
  - Orders chart
  - Channel breakdown (pie/donut chart)
  - Quick stats row
- [ ] Create `(dashboard)/analytics/sales/page.tsx` — Sales deep dive
  - Time-series chart with period selector
  - Group by: Day / Week / Month toggle
  - Channel filter
  - Sales table: Date, Orders, Revenue, AOV
  - Export to CSV button
- [ ] Create `components/dashboard/sales-chart.tsx` — enhanced with recharts
  - Responsive, animated line/bar chart
  - Tooltip with formatted values (₦)
  - Legend for multiple series

**Deliverable:** Analytics dashboard with real data and beautiful charts.

---

## Friday — Frontend: Product & Customer Analytics + Reports

- [ ] Create `(dashboard)/analytics/products/page.tsx`
  - Top 10 products table: Product, Units Sold, Revenue, Trend
  - Category performance bar chart
  - Slow movers alert list
- [ ] Create `(dashboard)/analytics/customers/page.tsx`
  - Customer growth line chart
  - New vs returning donut chart
  - Top customers table
  - Acquisition channel chart
- [ ] Add summary stats to relevant pages:
  - Products page header: Total products, Active, Out of stock
  - Customers page header: Total, New this month, Revenue from top 10%
  - Orders page header: Total orders, Pending, Revenue
- [ ] Implement CSV export for analytics data
- [ ] Polish all analytics pages: loading states, empty states, error handling
- [ ] Verify dashboard loads fast (Redis caching working)

**Deliverable:** Complete analytics experience across the dashboard.

---

## Week 9 Checklist

```
✅ Daily sales stats aggregation (background job)
✅ Dashboard overview API with real metrics
✅ Sales analytics (time-series, by channel, by method)
✅ Product analytics (top products, slow movers, categories)
✅ Customer analytics (growth, new vs returning, CLV)
✅ Period comparison (% changes)
✅ Redis caching for analytics queries
✅ Analytics overview page with charts
✅ Sales deep-dive page
✅ Product performance page
✅ Customer insights page
✅ CSV export for reports
✅ Live data on dashboard overview
```
