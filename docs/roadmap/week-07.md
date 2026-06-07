# Week 7 — CRM & Customer Management

> **Goal:** Build the customer management system with profiles, segmentation, groups, and purchase history.

---

## Monday — Customer Database & Backend

- [ ] Write migrations: `014_create_customers.sql`, `015_create_customer_groups.sql`, `016_create_customer_group_members.sql`
- [ ] Run migrations, verify RLS policies and search index
- [ ] Create `punta-crm` crate
- [ ] Implement `customer.rs` — `CustomerService`
  - `create_customer()` — with duplicate detection (email/phone)
  - `get_customer()` — with order history and group memberships
  - `list_customers()` — paginated, searchable (full-text on name/email/phone)
  - `update_customer()` — partial update
  - `delete_customer()` — soft delete
  - `import_customers()` — bulk import from CSV (validate, deduplicate)
  - `export_customers()` — export to CSV
- [ ] Handlers: `GET/POST/PATCH/DELETE /v1/customers`, `GET /v1/customers/:id`
- [ ] Wire customer creation into checkout flow (auto-create if new)
- [ ] Test customer CRUD and search

**Deliverable:** Customer CRUD API with full-text search.

---

## Tuesday — Customer Segmentation & Groups

- [ ] Implement `groups.rs` — `CustomerGroupService`
  - `create_group()` — with filter criteria (JSONB rules)
  - `list_groups()` — with cached member counts
  - `get_group_members()` — paginated
  - `update_group()` — update criteria, re-evaluate members
  - `delete_group()`
  - `add_member()` / `remove_member()` — manual group membership
- [ ] Implement dynamic segmentation engine:
  - Filter by: total_spent range, order_count range, tags, last_order_within_days
  - Build dynamic SQL from filter criteria JSONB
  - Evaluate and cache group membership (background job)
- [ ] Create default groups on tenant creation:
  - "All Customers" (default, everyone)
  - "New Customers" (order_count = 0)
  - "Returning Customers" (order_count > 1)
  - "VIP" (total_spent > threshold)
- [ ] Handlers: `GET/POST/PATCH/DELETE /v1/customer-groups`, `GET /v1/customer-groups/:id/members`

**Deliverable:** Customer groups with dynamic segmentation working.

---

## Wednesday — Customer History & Aggregates

- [ ] Implement `history.rs` — purchase history
  - `get_purchase_history()` — orders by customer (paginated, with items)
  - `get_customer_metrics()` — total spent, order count, avg order value, last order date
  - `get_communication_history()` — messages sent to this customer
- [ ] Implement denormalized aggregate updates:
  - On new order paid → update customer.total_spent, order_count, last_order_at
  - Use database trigger or background job
- [ ] Implement customer tagging:
  - `add_tags()` / `remove_tags()` — update tags array
  - Search/filter by tags
- [ ] Implement customer notes:
  - `add_note()` — append note with timestamp and author
- [ ] Write integration tests for customer lifecycle (create → order → aggregates updated → group membership recalculated)

**Deliverable:** Customer profiles enriched with purchase history and aggregated metrics.

---

## Thursday — Frontend: Customer List & Profile

- [ ] Create `lib/hooks/use-customers.ts` — React Query hooks
- [ ] Create `(dashboard)/customers/page.tsx` — Customer list page
  - Table: Name, Email, Phone, Total Spent, Orders, Last Order, Tags
  - Search bar (search across name, email, phone)
  - Filter by tags, customer group
  - Sort by name, total_spent, order_count, last_order_at
  - Bulk actions: tag, add to group, export
  - "Add Customer" button
- [ ] Create `(dashboard)/customers/new/page.tsx` — Add customer form
  - First name, last name, email, phone, WhatsApp
  - Address(es) with add/remove
  - Tags input
  - Notes textarea
  - Marketing consent checkbox
- [ ] Create `(dashboard)/customers/[id]/page.tsx` — Customer profile page
  - Contact info card (editable)
  - Stats row: Total Spent, Orders, Avg Order, First Order, Last Order
  - Tabs: Orders, Messages, Notes
  - Order history tab with mini order list
  - Tags display with edit
  - Group memberships

**Deliverable:** Customer management pages functional.

---

## Friday — Customer Groups UI & Polish

- [ ] Create `(dashboard)/customers/groups/page.tsx` — Customer groups page
  - Group cards/list: Name, Member Count, Filter Criteria Summary
  - "Create Group" button
- [ ] Create group builder modal/form:
  - Name and description
  - Filter criteria builder:
    - Min/max total spent
    - Min/max order count
    - Tags include/exclude
    - Last order within X days
  - Preview: show count of matching customers
  - Save → evaluate membership
- [ ] Create `(dashboard)/customers/groups/[id]/page.tsx` — Group detail
  - Members list (paginated)
  - Group stats
  - Edit criteria
  - "Send Campaign" shortcut (links to Week 8)
- [ ] Add customer count to sidebar navigation badge
- [ ] Polish: empty states, loading skeletons, error handling
- [ ] Test: create customer → place orders → verify aggregates → verify group membership

**Deliverable:** Complete CRM experience in dashboard.

---

## Week 7 Checklist

```
✅ Customer CRUD API with full-text search
✅ Auto-create customers from checkout
✅ Customer import/export (CSV)
✅ Customer groups with dynamic segmentation
✅ Default groups created on tenant setup
✅ Purchase history and aggregated metrics
✅ Customer tagging system
✅ Customer list page with search and filters
✅ Customer profile page with order history
✅ Customer group builder with criteria preview
✅ Integration tests for customer lifecycle
```
