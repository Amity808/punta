# 07 — Frontend Architecture (Next.js 15)

## App Structure

The Next.js app serves two distinct experiences:

1. **Merchant Dashboard** — authenticated SPA at `app.punta.shop`
2. **Public Storefront** — SSR pages at `{store}.punta.shop`

```
web/
├── package.json
├── next.config.ts
├── tsconfig.json
├── .env.local
│
├── public/
│   ├── favicon.ico
│   └── images/
│
├── src/
│   ├── app/
│   │   ├── layout.tsx                 # Root layout (fonts, providers)
│   │   ├── globals.css                # Design system tokens
│   │   │
│   │   ├── (auth)/                    # Auth pages (unauthenticated)
│   │   │   ├── login/page.tsx
│   │   │   ├── register/page.tsx
│   │   │   ├── forgot-password/page.tsx
│   │   │   ├── reset-password/page.tsx
│   │   │   └── layout.tsx             # Centered auth layout
│   │   │
│   │   ├── (dashboard)/               # Merchant dashboard (authenticated)
│   │   │   ├── layout.tsx             # Sidebar + header layout
│   │   │   ├── page.tsx               # Dashboard overview
│   │   │   │
│   │   │   ├── products/
│   │   │   │   ├── page.tsx           # Product list
│   │   │   │   ├── new/page.tsx       # Create product
│   │   │   │   └── [id]/
│   │   │   │       ├── page.tsx       # Product detail/edit
│   │   │   │       └── variants/page.tsx
│   │   │   │
│   │   │   ├── orders/
│   │   │   │   ├── page.tsx           # Order list
│   │   │   │   ├── new/page.tsx       # Create manual order
│   │   │   │   └── [id]/page.tsx      # Order detail
│   │   │   │
│   │   │   ├── customers/
│   │   │   │   ├── page.tsx           # Customer list
│   │   │   │   ├── new/page.tsx       # Add customer
│   │   │   │   ├── [id]/page.tsx      # Customer profile
│   │   │   │   └── groups/
│   │   │   │       ├── page.tsx       # Customer groups
│   │   │   │       └── [id]/page.tsx  # Group detail
│   │   │   │
│   │   │   ├── inventory/
│   │   │   │   ├── page.tsx           # Stock levels
│   │   │   │   ├── adjustments/page.tsx
│   │   │   │   ├── transfers/page.tsx
│   │   │   │   └── locations/
│   │   │   │       ├── page.tsx       # Locations list
│   │   │   │       └── new/page.tsx
│   │   │   │
│   │   │   ├── messaging/
│   │   │   │   ├── page.tsx           # Message inbox / overview
│   │   │   │   ├── campaigns/
│   │   │   │   │   ├── page.tsx       # Campaign list
│   │   │   │   │   └── new/page.tsx   # Create campaign
│   │   │   │   └── templates/
│   │   │   │       ├── page.tsx       # Template list
│   │   │   │       └── new/page.tsx   # Create template
│   │   │   │
│   │   │   ├── analytics/
│   │   │   │   ├── page.tsx           # Analytics overview
│   │   │   │   ├── sales/page.tsx
│   │   │   │   ├── products/page.tsx
│   │   │   │   └── customers/page.tsx
│   │   │   │
│   │   │   ├── expenses/
│   │   │   │   ├── page.tsx           # Expense list
│   │   │   │   └── new/page.tsx       # Add expense
│   │   │   │
│   │   │   ├── deliveries/
│   │   │   │   ├── page.tsx           # Delivery list
│   │   │   │   └── [id]/page.tsx      # Tracking detail
│   │   │   │
│   │   │   ├── store/
│   │   │   │   ├── page.tsx           # Store settings
│   │   │   │   ├── theme/page.tsx     # Theme customization
│   │   │   │   └── domain/page.tsx    # Domain management
│   │   │   │
│   │   │   ├── wallet/
│   │   │   │   ├── page.tsx           # Current balance + ledger transactions
│   │   │   │   └── addons/
│   │   │   │       └── page.tsx       # Premium add-ons marketplace
│   │   │   │
│   │   │   └── settings/
│   │   │       ├── page.tsx           # General settings
│   │   │       ├── team/page.tsx      # Staff management
│   │   │       ├── payments/page.tsx  # Payment gateway config
│   │   │       └── notifications/page.tsx
│   │   │
│   │   └── (storefront)/              # Public store (SSR)
│   │       ├── layout.tsx             # Store layout (header, footer)
│   │       ├── page.tsx               # Store homepage
│   │       ├── products/
│   │       │   ├── page.tsx           # Product listing
│   │       │   └── [slug]/page.tsx    # Product detail
│   │       ├── cart/page.tsx          # Shopping cart
│   │       ├── checkout/page.tsx      # Checkout flow
│   │       └── orders/
│   │           └── [id]/page.tsx      # Order tracking (public)
│   │
│   ├── components/
│   │   ├── ui/                        # Design system primitives
│   │   │   ├── button.tsx
│   │   │   ├── input.tsx
│   │   │   ├── select.tsx
│   │   │   ├── textarea.tsx
│   │   │   ├── modal.tsx
│   │   │   ├── dropdown.tsx
│   │   │   ├── table.tsx
│   │   │   ├── badge.tsx
│   │   │   ├── card.tsx
│   │   │   ├── avatar.tsx
│   │   │   ├── skeleton.tsx
│   │   │   ├── toast.tsx
│   │   │   ├── tabs.tsx
│   │   │   ├── pagination.tsx
│   │   │   ├── empty-state.tsx
│   │   │   ├── file-upload.tsx
│   │   │   ├── date-picker.tsx
│   │   │   ├── search-input.tsx
│   │   │   └── stat-card.tsx
│   │   │
│   │   ├── layout/
│   │   │   ├── sidebar.tsx            # Dashboard sidebar navigation
│   │   │   ├── header.tsx             # Dashboard top bar
│   │   │   ├── breadcrumb.tsx
│   │   │   └── page-header.tsx        # Title + actions bar
│   │   │
│   │   ├── dashboard/                 # Dashboard-specific components
│   │   │   ├── overview-stats.tsx
│   │   │   ├── recent-orders.tsx
│   │   │   ├── sales-chart.tsx
│   │   │   ├── top-products.tsx
│   │   │   ├── product-form.tsx
│   │   │   ├── order-timeline.tsx
│   │   │   ├── customer-card.tsx
│   │   │   ├── inventory-table.tsx
│   │   │   └── campaign-builder.tsx
│   │   │
│   │   └── storefront/               # Public store components
│   │       ├── store-header.tsx
│   │       ├── store-footer.tsx
│   │       ├── product-card.tsx
│   │       ├── product-gallery.tsx
│   │       ├── cart-drawer.tsx
│   │       ├── checkout-form.tsx
│   │       └── order-status.tsx
│   │
│   ├── lib/
│   │   ├── api.ts                     # Fetch wrapper (base URL, auth headers)
│   │   ├── auth.ts                    # Auth state, token management
│   │   ├── hooks/
│   │   │   ├── use-auth.ts            # Auth context hook
│   │   │   ├── use-products.ts        # React Query hooks for products
│   │   │   ├── use-orders.ts
│   │   │   ├── use-customers.ts
│   │   │   ├── use-inventory.ts
│   │   │   └── use-analytics.ts
│   │   ├── utils.ts                   # Formatting, helpers
│   │   ├── money.ts                   # formatMoney(kobo) → "₦5,000.00"
│   │   ├── constants.ts              # Enums, status labels, colors
│   │   └── validations.ts            # Zod schemas for forms
│   │
│   ├── stores/                        # Zustand stores
│   │   ├── cart-store.ts              # Shopping cart state
│   │   └── ui-store.ts                # Sidebar open, modals, etc.
│   │
│   └── types/
│       ├── api.ts                     # API response types
│       ├── product.ts
│       ├── order.ts
│       ├── customer.ts
│       ├── inventory.ts
│       └── analytics.ts
│
└── middleware.ts                       # Next.js middleware for auth redirect
```

---

## State Management Strategy

| State Type | Solution | Example |
|:-----------|:---------|:--------|
| Server state | `@tanstack/react-query` | Products, orders, customers, analytics |
| Auth state | React Context + cookies | JWT tokens, user info |
| UI state | `zustand` | Sidebar toggle, active modals |
| Cart state | `zustand` + localStorage | Shopping cart items |
| Form state | `react-hook-form` + `zod` | Product creation form |

---

## Key Design Patterns

### API Client

```typescript
// lib/api.ts
class ApiClient {
  private baseUrl: string;
  
  async fetch<T>(endpoint: string, options?: RequestInit): Promise<T> {
    const token = getAccessToken();
    const res = await fetch(`${this.baseUrl}${endpoint}`, {
      ...options,
      headers: {
        'Content-Type': 'application/json',
        ...(token ? { Authorization: `Bearer ${token}` } : {}),
        ...options?.headers,
      },
    });
    
    if (res.status === 401) {
      // Attempt token refresh
      const refreshed = await this.refreshToken();
      if (refreshed) return this.fetch(endpoint, options); // Retry
      redirect('/login');
    }
    
    if (!res.ok) {
      const error = await res.json();
      throw new ApiError(error);
    }
    
    return res.json();
  }
}
```

### Money Formatting

```typescript
// lib/money.ts
export function formatMoney(kobo: number, currency = 'NGN'): string {
  const naira = kobo / 100;
  return new Intl.NumberFormat('en-NG', {
    style: 'currency',
    currency,
  }).format(naira);
}

// Usage: formatMoney(500000) → "₦5,000.00"
```

---

## Responsive Design

| Breakpoint | Target | Layout |
|:-----------|:-------|:-------|
| `< 640px` | Mobile | Single column, bottom nav |
| `640-1024px` | Tablet | Collapsible sidebar |
| `> 1024px` | Desktop | Fixed sidebar + content area |

The dashboard will be mobile-responsive (since no native app is planned), ensuring merchants can manage their business from any device.
