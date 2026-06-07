# Week 12 — Wallet, Pay-As-You-Go Billing & Launch Preparation

> **Goal:** Build the unified merchant wallet infrastructure, transactional commission split logic, logistics delivery markup quoting, pay-as-you-go messaging deductions, and launch preparation.

---

## Monday — Wallet Ledger & Add-on Backend

- [ ] Write migrations: `027_create_tenant_features.sql`, `028_update_payments_and_transactions.sql`, `031_create_merchant_kyc.sql`
- [ ] Create `punta-wallet` crate
- [ ] Implement `service.rs` — `WalletService`
  - `get_balance()` — retrieve current wallet balance in kobo
  - `credit_wallet()` — add funds (storefront payouts, manual topups, refunds)
  - `debit_wallet()` — subtract funds (withdrawals, campaign charges, addon purchases, transaction fees)
  - Write raw transactional ledger entries (`wallet_transactions`) with unique references
  - Enforce tiered KYC limits on withdrawal requests (block payouts if Tier 1 daily limit is exceeded or KYC is unverified)
- [ ] Implement `addons.rs` — `AddonService`
  - `list_addons()` — return premium add-on features and their pricing (e.g., Custom Domain: ₦10,000/year, Multi-Location: ₦5,000/year)
  - `unlock_addon()` — buy/unlock add-on features, debiting from the merchant's wallet balance
  - `has_active_addon()` — resolve if a tenant currently has a premium feature unlocked
- [ ] Handlers: 
  - `GET /v1/wallet` — retrieve balance and recent ledger history
  - `POST /v1/wallet/topup` — initiate card/transfer payment to top up wallet balance
  - `POST /v1/wallet/withdraw` — request withdrawal to merchant's bank account
  - `GET /v1/wallet/kyc` — retrieve merchant KYC verification status
  - `POST /v1/wallet/kyc` — submit Tier 1 (BVN/NIN) details
  - `POST /v1/wallet/kyc/business` — submit Tier 2 (CAC details/documents)
  - `GET /v1/tenant/addons` — list optional features and statuses
  - `POST /v1/tenant/addons/purchase` — buy/renew a premium add-on feature

**Deliverable:** Wallet ledger and premium add-on purchase APIs are fully functional.

---

## Tuesday — Transactional Commissions & Pay-As-You-Go Deductions

- [ ] Implement `enforcement.rs` — `FeatureEnforcement` middleware
  - Enforce limits and block operations based on wallet balance or addon status:
    - Resolving custom domain → verify tenant has active `custom_domain` feature
    - Creating additional locations → verify tenant has active `multi_location` feature
    - Launching bulk campaign → verify tenant's wallet balance is positive and covers the campaign size
  - Return `403 FORBIDDEN` or `402 PAYMENT_REQUIRED` with appropriate errors
- [ ] Implement checkout commission split hooks in payment webhooks (Paystack/Flutterwave):
  - Upon successful payment verification webhook:
    1. Calculate gateway fee (e.g., 1.5% + ₦100)
    2. Calculate platform commission (e.g., 1.0% platform fee)
    3. Calculate transactional notification costs (e.g., ₦18 for WhatsApp order confirmation)
    4. Debit gateway fee, platform commission, and notification fee from the gross checkout value
    5. Record platform_fee in `payments`
    6. Credit the remaining net amount into `merchant_wallets.balance` and record the transaction ledger
- [ ] Implement logistics rate markup in `punta-delivery`:
  - When querying courier rates (Shipbubble, FEZ, Kwik), add a platform markup (e.g. flat ₦150) before displaying quotes to storefront customers or dashboard merchants
  - Collect this markup as platform revenue when booking shipments
- [ ] Implement transactional messaging checks:
  - For offline checkout flows (e.g. Cash on Delivery), check if the tenant wallet balance permits sending notification messages, then debit the messaging fee from the unified wallet balance

**Deliverable:** Platform commissions, messaging deductions, and logistics markups are calculated and processed.

---

## Wednesday — Frontend: Wallet & Add-ons

- [ ] Create `(dashboard)/wallet/page.tsx` — Merchant Wallet Dashboard
  - Balance display: display balance in Naira with clear visual layout
  - Quick action buttons: "Top Up Wallet" (modal with Paystack checkout) and "Request Withdrawal" (bank transfer form)
  - Ledger list: table of recent transactions including date, description, type (credit/debit), amount, and status
- [ ] Create `(dashboard)/wallet/kyc/page.tsx` — KYC Verification Dashboard
  - Display current KYC tier and verification status (pending, verified, rejected)
  - Standard form for Tier 1 inputs (BVN, NIN)
  - File upload drag-and-drop form for Tier 2 inputs (CAC certificates, utility bills)
- [ ] Create `(dashboard)/wallet/addons/page.tsx` — Add-on Marketplace
  - Add-on feature cards (Custom Domain, Multi-Location) showing pricing and unlock status
  - "Activate Add-on" action buttons that trigger wallet debit unlocks
- [ ] Create `(dashboard)/settings/page.tsx` — General settings
  - Business info (name, email, phone)
  - Timezone selector
  - Currency display
  - Tax settings (VAT rate)
- [ ] Create `(dashboard)/settings/team/page.tsx` — Staff management
  - Staff list: Name, Email, Role, Status, Last Login
  - Invite staff form and permissions editor
- [ ] Create `(dashboard)/settings/domains/page.tsx` — Domain Settings
  - Configure custom domains, mapped to custom domain add-on unlock state

**Deliverable:** Wallet dashboard, add-on marketplace, and settings screens complete.

---

## Thursday — End-to-End Polish & Bug Fixes

- [ ] Full end-to-end testing of all transaction and pay-as-you-grow flows:
  1. Register a merchant -> wallet is initialized at ₦0, default features active
  2. Add product -> verify catalog is fully functional on free plan
  3. Customer visits storefront, buys product, pays ₦10,000 via Paystack
  4. Webhook executes, splits gateway fee (₦150), platform fee (₦100), and notification SMS cost (₦5)
  5. Net amount (₦9,745) credited to merchant wallet, SMS is sent
  6. Merchant views wallet ledger -> sees the transaction payout breakdown
  7. Merchant attempts withdrawal without KYC -> verify request rejected
  8. Merchant submits Tier 1 KYC (BVN/NIN) -> verify verified, withdrawal unlocked with limit
  9. Merchant tests withdrawing ₦5,000 -> verify transfer initiated, ₦100 withdrawal fee debited
  10. Merchant attempts to withdraw ₦200,000 (exceeding Tier 1 daily limit) -> verify blocked
  11. Merchant submits Tier 2 CAC documents -> verify verified, withdrawal limits removed
  12. Merchant purchases Custom Domain addon -> verify wallet debited ₦10,000, domain mapping unlocked
  13. Merchant attempts bulk marketing campaign -> checks wallet balance, runs or rejects based on balance
- [ ] Fix all discovered layout, synchronization, and ledger calculation bugs
- [ ] Polish UI/UX with loading skeletons, toast notifications, responsive viewport adjustments, and custom domain setup guidelines

**Deliverable:** End-to-end payment split, delivery markup, and pay-as-you-grow campaign launching verified.

---

## Friday — Deployment & Launch Preparation

- [ ] Set up production infrastructure:
  - Fly.io app for Rust API (2 instances minimum)
  - Vercel project for Next.js
  - Neon PostgreSQL production database
  - Upstash Redis production instance
  - Cloudflare R2 production bucket
  - Cloudflare DNS: *.punta.shop wildcard
- [ ] Configure environment variables for production (Paystack/Flutterwave api keys, Termii credentials, Resend keys, default platform fee rates)
- [ ] Run all migrations on production database
- [ ] Seed global parameters (e.g. system messaging unit fees, default addon cost rules)
- [ ] Set up GitHub Actions CI/CD:
  - Run tests on PR
  - Deploy to staging on merge to develop
  - Deploy to production on merge to main
- [ ] Set up monitoring, uptime alerts, Sentry error tracking, database backups
- [ ] Security checklist audit (RLS isolation verify, webhook signature validations, SSL certs)
- [ ] Smoke test production deployment (register a test merchant, make payment, verify wallet credit and WhatsApp alert)

**Deliverable:** Production environment live, CI/CD working, monitoring active.

---

## Week 12 Checklist

```
✅ Unified wallet ledger bookkeeping (deposits, payouts, withdrawals)
✅ Platform transaction commissions (1.0% cut on checkout payments)
✅ Dynamic checkout payment webhook commission splitting
✅ Transactional message fee deduction from order settlement value
✅ Logistics delivery markup added to shipping quotes
✅ Premium add-on unlocking system (custom domain, multi-location)
✅ KYC verification tracking (BVN/NIN checks & CAC document uploads)
✅ Wallet ledger, KYC submission, and withdrawal dashboards
✅ Marketplace UI for purchasing feature add-ons
✅ Complete E2E testing of payment splits, KYC limits, and withdrawal fees
✅ Production infrastructure deployed and verified
✅ CI/CD pipelines active and monitoring alerts green
✅ Security audit passed (RLS policies validated)
```

---

## 🎉 Launch Readiness

After Week 12, Punta should be ready for beta launch with:

- **Full merchant onboarding** (register → set up store → add products)
- **Complete e-commerce flow** (browse → cart → checkout → pay → fulfill)
- **Multi-channel communications** (WhatsApp, SMS, email)
- **Business intelligence** (analytics, P&L reports)
- **Pay-as-you-grow billing** (commissions, logistics markup, utility messaging)
- **Production infrastructure** with monitoring
