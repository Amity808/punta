# Week 3 — Foundation: Frontend Setup

> **Goal:** Set up Next.js project, build the design system, implement auth pages, and create the dashboard shell layout.

---

## Monday — Next.js Project & Design System

- [ ] Initialize Next.js 15 project in `web/` directory (App Router, TypeScript)
- [ ] Install dependencies: `@tanstack/react-query`, `zustand`, `react-hook-form`, `zod`, `sonner`, `recharts`, `date-fns`
- [ ] Set up Google Fonts (Inter for body, outfit for headings)
- [ ] Create `globals.css` with full design token system:
  - Color palette (dark-first, Nigerian-inspired accents)
  - Typography scale
  - Spacing, border-radius, shadow tokens
  - CSS custom properties for theming
- [ ] Create UI components: `Button`, `Input`, `Card`, `Badge`
- [ ] Create UI components: `Select`, `Textarea`, `Skeleton`
- [ ] Set up `next.config.ts` with API proxy to Rust backend

**Deliverable:** Next.js running with design system, basic components render.

---

## Tuesday — More UI Components & API Layer

- [ ] Create UI components: `Modal` (dialog), `Dropdown`, `Table`
- [ ] Create UI components: `Tabs`, `Toast` (sonner integration), `Avatar`
- [ ] Create UI components: `EmptyState`, `StatCard`, `SearchInput`, `Pagination`
- [ ] Create UI components: `FileUpload`, `DatePicker`
- [ ] Implement `lib/api.ts` — API client class
  - Base URL configuration
  - Automatic Authorization header injection
  - Error handling with typed errors
  - Token refresh on 401
- [ ] Implement `lib/money.ts` — `formatMoney()` (kobo → "₦5,000.00")
- [ ] Implement `lib/utils.ts` — common helpers (date formatting, status labels)

**Deliverable:** Full UI component library ready, API client configured.

---

## Wednesday — Auth Pages

- [ ] Create `(auth)/layout.tsx` — centered layout with gradient background, logo
- [ ] Create `(auth)/login/page.tsx`
  - Email + password form with validation (zod)
  - Login API call, token storage
  - Redirect to dashboard on success
  - Error display (invalid credentials)
  - "Forgot password?" link
  - "Create an account" link
- [ ] Create `(auth)/register/page.tsx`
  - Business name, slug (auto-generated from name), email, password, phone
  - Real-time slug availability check
  - Store subdomain preview (`your-store.punta.shop`)
  - Registration API call
  - Redirect to dashboard on success
- [ ] Create `(auth)/forgot-password/page.tsx`
- [ ] Create `(auth)/reset-password/page.tsx`
- [ ] Implement `lib/auth.ts` — token storage (cookies), `useAuth()` hook

**Deliverable:** Full auth flow works in browser (register → login → dashboard redirect).

---

## Thursday — Dashboard Layout

- [ ] Create `(dashboard)/layout.tsx`
  - Sidebar + main content area
  - Responsive: collapsible sidebar on mobile
- [ ] Create `components/layout/sidebar.tsx`
  - Logo at top
  - Navigation items with icons:
    - Overview, Products, Orders, Customers, Inventory
    - Messaging, Analytics, Expenses, Deliveries
    - Store, Wallet, Settings
  - Active state highlighting
  - Tenant wallet indicator
  - User avatar + dropdown at bottom (profile, logout)
- [ ] Create `components/layout/header.tsx`
  - Breadcrumb navigation
  - Search bar
  - Notification bell
  - Quick actions (+ New Order, + New Product)
- [ ] Create `components/layout/page-header.tsx`
  - Page title + description
  - Action buttons area
- [ ] Create `middleware.ts` — auth guard (redirect unauthenticated to /login)

**Deliverable:** Dashboard layout renders with sidebar navigation, auth guard works.

---

## Friday — Dashboard Overview Page

- [ ] Create `(dashboard)/page.tsx` — Overview dashboard
- [ ] Create `components/dashboard/overview-stats.tsx`
  - 4 stat cards: Today's Revenue, Orders, New Customers, Conversion Rate
  - Comparison with previous period (↑12% / ↓5%)
  - Skeleton loading states
- [ ] Create `components/dashboard/sales-chart.tsx`
  - 30-day sales line chart (recharts)
  - Toggle: Revenue / Orders
- [ ] Create `components/dashboard/recent-orders.tsx`
  - Last 5 orders table with status badges
  - Click to view order detail
- [ ] Create `components/dashboard/top-products.tsx`
  - Top 5 selling products with revenue
- [ ] Wire up `useAuth()` hook to fetch real user/tenant data
- [ ] Connect overview stats to `GET /v1/analytics/dashboard` (mock data for now)
- [ ] Test full flow: register → redirected to dashboard → see overview

**Deliverable:** Beautiful dashboard overview page with stats, charts, and recent orders.

---

## Week 3 Checklist

```
✅ Next.js 15 project running
✅ Design system with tokens and 15+ UI components
✅ API client with auth header injection and error handling
✅ Login page working end-to-end
✅ Registration page with slug preview
✅ Password reset flow
✅ Dashboard layout with responsive sidebar
✅ Auth middleware (redirect to /login)
✅ Overview dashboard with stats and charts
✅ Mobile-responsive layout
```
