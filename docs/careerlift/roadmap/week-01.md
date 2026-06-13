# Week 1 — Foundation: Authentication & Onboarding

This week focuses on initializing the frontend client and backend server repositories, configuring database models, and building the complete authentication and profile onboarding flows.

---

## Daily Schedule

### Day 1 — Repository Bootstrapping & DB Migrations
* **Focus:** Setup Vite project, configure Express server routing, run initial PostgreSQL database tables creation.
* **Tasks:**
  1. Initialize React 19 + Vite + TS frontend. Create directories for modules, assets, and shared files.
  2. Setup Node.js Express server workspace in TS with basic error handling middlewares.
  3. Formulate database models using Drizzle schema; generate SQL migration files and apply to Neon Database.
  4. Write backend connection pools and run quick health checks.
* **Duration:** 8 hours
* **Verification:** Run `npm run dev` locally to verify client bootstraps. Check Neon Console to verify tables were created.

### Day 2 — Backend Auth Engine
* **Focus:** Build endpoints for signup, signin, and JWT issuance.
* **Tasks:**
  1. Write argon2 encryption wrapper helper in Node.js to hash user passwords.
  2. Code register and login router handlers in Express (`/api/v1/auth/register`, `/api/v1/auth/login`).
  3. Implement token sign and verify utilities using JSON Web Tokens.
  4. Set up an authentication check middleware to guard future API requests.
* **Duration:** 7 hours
* **Verification:** Test login and registration payloads using HTTP clients to verify JWTs are successfully returned.

### Day 3 — Frontend Auth UI Pages
* **Focus:** Build Register, Login, and Forgot Password UI pages using Tailwind CSS and Radix/Shadcn primitives.
* **Tasks:**
  1. Integrate TailwindCSS with Vite compiler. Set up colors and default themes.
  2. Build shared Shadcn components (`Button`, `Input`, `Form`).
  3. Setup React Router v7 structure and define routes for `/login`, `/register`, and `/forgot-password`.
  4. Write forms with React Hook Form, linking inputs with client-side Zod validation rules.
* **Duration:** 8 hours
* **Verification:** Check client pages locally to verify validation error labels display on invalid email or password formats.

### Day 4 — State Managers Integration
* **Focus:** Code Zustand Auth Store and integrate client API call triggers.
* **Tasks:**
  1. Set up Zustand `useAuthStore` with persistent storage.
  2. Configure an Axios interceptor to automatically attach `Authorization: Bearer <token>` to request headers.
  3. Wire frontend forms to auth api endpoints using TanStack Query.
  4. Build React Router guards to redirect unauthenticated users to `/login`.
* **Duration:** 7 hours
* **Verification:** Log in with a valid account, check `localStorage` to verify the JWT is saved, and confirm redirection to the onboarding route works.

### Day 5 — Profile Onboarding Multi-Step Flow
* **Focus:** Implement the step-based onboarding wizard to collect career details.
* **Tasks:**
  1. Build step component wizard under `/onboarding`:
     * Step 1: Career track selector (PM, Frontend, Backend, etc.).
     * Step 2: Experience level check (Entry, Mid, Senior, Lead).
     * Step 3: Goal definition (Job hunt, Skill growth, etc.).
     * Step 4: Target daily practice volume (5, 10, or 20 questions).
  2. Build backend Express handler `/api/v1/onboarding` (saves selections to `onboarding_profiles` table).
  3. Connect onboarding form submission to the backend API via TanStack Query mutation.
* **Duration:** 8 hours
* **Verification:** Complete onboarding steps and verify database profile registers correctly. On completion, confirm routing to `/dashboard` succeeds.
