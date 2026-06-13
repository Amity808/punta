# Week 4 — Core Commerce: Product Catalog

> **Goal:** Build the complete product catalog system — CRUD, variants, categories, images, and search.

---

## Monday — Product Database & Backend CRUD

- [ ] Write migrations: `008_create_products.sql`, `009_create_product_variants.sql`, `010_create_product_images.sql`, `007_create_categories.sql`
- [ ] Run migrations, verify tables and RLS policies
- [ ] Create `punta-catalog` crate
- [ ] Implement `product.rs` — `ProductService`
  - `create_product()` — with variants in a single transaction
  - `get_product()` — with variants and images joined
  - `list_products()` — paginated, filterable (status, category, search)
  - `update_product()` — partial update
  - `delete_product()` — soft delete (set `deleted_at`)
- [ ] Implement product validation (name required, price > 0, unique slug per tenant)
- [ ] Wire up handlers: `POST/GET/PATCH/DELETE /v1/products`, `GET /v1/products/:id`

**Deliverable:** Product CRUD API working, tested with curl.

---

## Tuesday — Variants, Categories & Search

- [ ] Implement `variant.rs` — `VariantService`
  - `add_variant()`, `update_variant()`, `delete_variant()`
  - Variant price override logic (use variant price if set, else product price)
- [ ] Implement `category.rs` — `CategoryService`
  - `create_category()` — with parent_id for tree structure
  - `list_categories()` — return as tree (recursive CTE query)
  - `update_category()`, `delete_category()`
- [ ] Implement `search.rs` — product full-text search
  - `tsvector` search with ranking
  - Trigram fuzzy matching for typo tolerance
- [ ] Handlers: `POST/PATCH/DELETE /v1/products/:id/variants`
- [ ] Handlers: `GET/POST/PATCH/DELETE /v1/categories`
- [ ] Write integration tests for product search (exact match, fuzzy match, by category)

**Deliverable:** Variants, categories, and search working.

---

## Wednesday — Image Upload System

- [ ] Implement `media.rs` — `MediaService`
  - `generate_presigned_url()` — create S3/R2 presigned upload URL
  - `confirm_upload()` — validate uploaded file, store reference in DB
  - `delete_image()` — remove from S3 and DB
  - `reorder_images()` — update sort_order
- [ ] Set up MinIO bucket in Docker for local development
- [ ] Implement image size validation (max 5MB, image/* content types only)
- [ ] Handlers: `POST /v1/products/:id/images` (get upload URL)
- [ ] Handlers: `POST /v1/products/:id/images/confirm` (confirm upload)
- [ ] Handlers: `DELETE /v1/products/:id/images/:image_id`
- [ ] Test upload flow: get presigned URL → upload to MinIO → confirm → verify URL stored

**Deliverable:** Image upload pipeline working end-to-end.

---

## Thursday — Frontend: Product List & Creation

- [ ] Create `lib/hooks/use-products.ts` — React Query hooks
  - `useProducts()` — paginated product list
  - `useProduct(id)` — single product
  - `useCreateProduct()` — mutation
  - `useUpdateProduct()` — mutation
  - `useDeleteProduct()` — mutation
- [ ] Create `(dashboard)/products/page.tsx` — Product list page
  - Table view with columns: Image, Name, Price, Stock, Status, Actions
  - Search bar with debounced search
  - Filter by category, status (active/inactive)
  - Sort by name, price, date created
  - Empty state for no products
  - "Add Product" button
- [ ] Create `(dashboard)/products/new/page.tsx` — Create product form
  - Form fields: name, description (rich text), price, compare price, cost, SKU
  - Category selector (dropdown with tree)
  - Tags input (multi-select/chips)
  - Active/Featured toggles
  - Variant builder (add rows with name, SKU, price, attributes)
  - Form validation with zod

**Deliverable:** Product list page and creation form working in browser.

---

## Friday — Frontend: Product Detail, Edit & Images

- [ ] Create `(dashboard)/products/[id]/page.tsx` — Product detail/edit page
  - Pre-filled form with existing data
  - Save changes with optimistic update
  - Delete product (with confirmation modal)
- [ ] Implement image upload UI
  - Drag-and-drop zone
  - Upload progress indicator
  - Image preview grid with reorder (drag)
  - Set primary image
  - Delete image
- [ ] Create category management UI (modal or side panel)
  - Tree view of categories
  - Create/edit/delete inline
- [ ] Connect all frontend to real API endpoints
- [ ] Test complete flow: create product with variants and images → see in list → edit → delete

**Deliverable:** Complete product management experience in the dashboard.

---

## Week 4 Checklist

```
✅ Product CRUD API (create, read, update, soft-delete)
✅ Product variants (add, edit, delete)
✅ Category tree (hierarchical, CRUD)
✅ Full-text search with fuzzy matching
✅ Image upload via presigned URLs (S3/R2)
✅ Product list page with search, filter, sort
✅ Product creation form with variants and images
✅ Product edit page
✅ Category management UI
✅ Integration tests for all product operations
```
