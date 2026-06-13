# Week 6 — Polish, QA, Accessibility & Launch

This week focuses on optimization, responsive layout checks, accessibility audits, and deploying the CareerLift MVP to production.

---

## Daily Schedule

### Day 1 — Responsive Audit & Mobile-First Checks
* **Focus:** Audit all views on mobile devices and optimize touch interactions.
* **Tasks:**
  1. Test all pages on mobile viewports using Chrome DevTools device simulators.
  2. Optimize touch target areas (ensuring interactive components are at least 44x44px).
  3. Implement responsive scroll layouts for code snippet blocks and data tables.
  4. Fix layout shifts (CLS) on the dashboard and question grid feeds.
* **Duration:** 8 hours
* **Verification:** Verify that all pages render correctly without horizontal overflow on mobile viewports down to 320px.

### Day 2 — Keyboard Navigation & Screen Reader Access (a11y)
* **Focus:** Implement keyboard navigation and screen reader support to meet WCAG AA standards.
* **Tasks:**
  1. Add visible focus indicators (`focus-visible:ring-2`) to all interactive elements.
  2. Audit semantic HTML structures (verify pages have single `<h1>` headers and follow correct heading hierarchies).
  3. Add explicit screen reader labels (`aria-label`) to icon buttons (e.g. Bookmark, Share).
  4. Verify that the onboarding wizard and modals can be navigated entirely using the keyboard.
* **Duration:** 7 hours
* **Verification:** Navigate the onboarding flow using only the `Tab` and `Enter` keys to verify accessibility support.

### Day 3 — Performance Tuning & Code-Splitting
* **Focus:** Optimize bundle sizes and load times.
* **Tasks:**
  1. Implement route-level code splitting using React Lazy (`React.lazy`) and suspense boundaries.
  2. Configure compression middlewares in Express (e.g. gzip) to compress API responses.
  3. Optimize static images and configure cache-control headers on static assets.
  4. Audit Largest Contentful Paint (LCP) and Interaction to Next Paint (INP) metrics.
* **Duration:** 8 hours
* **Verification:** Run a Lighthouse audit on the production bundle and confirm a Performance score of 90+ is achieved.

### Day 4 — Production Build & Deployment
* **Focus:** Compile production bundles and deploy the client and server applications.
* **Tasks:**
  1. Set up production environment variables in the Cloudflare Pages and Fly.io dashboards.
  2. Run the Vite production build (`npm run build`) and deploy the static assets to Cloudflare Pages.
  3. Deploy the Express API server container to Fly.io.
  4. Run production database migrations on the Neon PostgreSQL database.
* **Duration:** 8 hours
* **Verification:** Access the live production URL and verify that the application loads and runs correctly in production.

### Day 5 — Verification & Handover
* **Focus:** Perform final system verification checks and compile documentation.
* **Tasks:**
  1. Verify the complete user journey in the production environment:
     * User registration and onboarding.
     * Completing daily questions and verifying streak updates.
     * Submitting a company report.
  2. Perform security checks: verify that private API endpoints reject requests with invalid or missing tokens.
  3. Finalize all architecture and roadmap documentation, ensuring all cross-references are valid.
* **Duration:** 8 hours
* **Verification:** Confirm that all database records are correctly created and verified during the production run.
