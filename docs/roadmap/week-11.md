# Week 11 — Public Storefront & Store Management

> **Goal:** Build the customer-facing storefront (SSR), store theming, and multi-location management.

---

## Monday — Storefront Backend & SSR Setup

- [ ] Write migration: `006_create_stores.sql` (if not done in Week 2)
- [ ] Create `punta-store` crate
- [ ] Implement `service.rs` — `StoreService`
  - `get_store()` — return store settings for a given subdomain
  - `update_store()` — update store name, description, theme, SEO
  - `set_custom_domain()` — configure custom domain (requires custom_domain add-on unlocked)
- [ ] Implement `storefront.rs` — public storefront API (no auth required)
  - `GET /v1/storefront/store` — store metadata (name, logo, theme, social)
  - `GET /v1/storefront/products` — active published products
  - `GET /v1/storefront/products/:slug` — single product with variants and images
  - `GET /v1/storefront/categories` — category tree
- [ ] Configure Next.js middleware to detect storefront subdomain vs dashboard:
  - `app.punta.shop` → dashboard routes
  - `{store}.punta.shop` → storefront routes
- [ ] Set up storefront layout with dynamic theme loading

**Deliverable:** Storefront API and SSR routing working.

---

## Tuesday — Storefront Homepage & Product Listing

- [ ] Create `(storefront)/layout.tsx`
  - Fetch store theme and apply CSS variables dynamically
  - Store header with logo, name, navigation
  - Store footer with social links
- [ ] Create `(storefront)/page.tsx` — Store homepage
  - Hero section (configurable image + text)
  - Featured products grid
  - Category navigation
  - SEO meta tags from store settings
- [ ] Create `components/storefront/product-card.tsx`
  - Product image with hover zoom
  - Name, price (with compare_at_price strikethrough)
  - "Add to Cart" button
  - Quick view modal
- [ ] Create `(storefront)/products/page.tsx` — Product listing
  - Grid/list toggle
  - Category filter sidebar
  - Sort by: Newest, Price Low→High, Price High→Low
  - Pagination
  - Search within store
- [ ] Create `(storefront)/products/[slug]/page.tsx` — Product detail
  - Image gallery with thumbnails
  - Product info (name, price, description)
  - Variant selector (size, color, etc.)
  - Quantity picker
  - "Add to Cart" button
  - Related products

**Deliverable:** Storefront homepage and product pages rendering with SSR.

---

## Wednesday — Cart & Checkout (Storefront)

- [ ] Create `stores/cart-store.ts` — Zustand cart store
  - Cart items with product details, variant, quantity
  - Persist to localStorage
  - Sync with backend Redis cart (if logged in)
- [ ] Create `components/storefront/cart-drawer.tsx`
  - Slide-out drawer showing cart contents
  - Item: Image, Name, Variant, Qty (editable), Price, Remove
  - Subtotal at bottom
  - "Checkout" button
- [ ] Create `(storefront)/cart/page.tsx` — Full cart page
  - Cart items table
  - Quantity editing
  - Remove items
  - Cart summary (subtotal, estimated shipping, total)
  - "Proceed to Checkout" button
- [ ] Create `(storefront)/checkout/page.tsx` — Checkout page
  - Customer info form (name, email, phone)
  - Shipping address form
  - Order summary
  - Payment method selection (Paystack / Flutterwave)
  - "Place Order" → API call → redirect to payment
- [ ] Create `(storefront)/orders/[id]/page.tsx` — Order confirmation / tracking
  - "Thank you" page with order details
  - Order status with visual timeline
  - Tracking info (if shipped)

**Deliverable:** Complete storefront shopping experience from browse to buy.

---

## Thursday — Store Theming & Customization

- [ ] Create `(dashboard)/store/page.tsx` — Store settings
  - Business name, description
  - Logo upload
  - Favicon upload
  - Social links (Instagram, Twitter, Facebook, WhatsApp)
  - Store visibility toggle (published/unpublished)
- [ ] Create `(dashboard)/store/theme/page.tsx` — Theme customizer
  - Color pickers: primary color, secondary color, accent color
  - Font selector (from a curated list)
  - Layout options: Grid (2/3/4 columns)
  - Hero image upload
  - Hero text editor
  - **Live preview** panel showing changes in real-time
- [ ] Create `(dashboard)/store/domain/page.tsx` — Domain management
  - Show current subdomain (`store.punta.shop`)
  - Custom domain input (requires custom_domain add-on unlocked)
  - DNS configuration instructions
  - Domain verification status
- [ ] Implement theme application in storefront layout:
  - CSS custom properties set from store.theme JSONB
  - Dynamic font loading

**Deliverable:** Merchants can customize their store appearance.

---

## Friday — Multi-Location & Store Polish

- [ ] Enhance `(dashboard)/inventory/locations/page.tsx`
  - Location cards: Name, Type, Address, Stock count, Is Default
  - Create location form
  - Edit location
  - Set default location
- [ ] Implement location-aware storefront:
  - Products show stock from default location (or all locations)
  - Stock display: "In Stock", "Low Stock (3 left)", "Out of Stock"
- [ ] Storefront SEO optimization:
  - Dynamic `<title>`, `<meta description>` from store settings
  - Open Graph tags for social sharing
  - Structured data (JSON-LD) for products
  - `robots.txt` and `sitemap.xml` generation
- [ ] Performance optimization:
  - Image lazy loading
  - Product listing pagination (avoid loading all products)
  - React Suspense boundaries with loading skeletons
- [ ] Test complete storefront flow:
  - Visit store → browse products → add to cart → checkout → pay → order confirmation
- [ ] Cross-browser testing (Chrome, Safari, Firefox, mobile)

**Deliverable:** Polished, SEO-optimized storefront ready for real customers.

---

## Week 11 Checklist

```
✅ Public storefront API (store metadata, products, categories)
✅ Next.js subdomain routing (dashboard vs storefront)
✅ SSR storefront homepage with hero and featured products
✅ Product listing page with filters and search
✅ Product detail page with image gallery and variants
✅ Shopping cart (drawer + full page)
✅ Checkout flow with payment redirect
✅ Order confirmation and tracking page
✅ Store settings management (logo, description, social)
✅ Theme customizer with live preview
✅ Custom domain configuration (DNS setup)
✅ Multi-location stock display
✅ SEO optimization (meta tags, structured data, sitemap)
✅ Mobile-responsive storefront
```
