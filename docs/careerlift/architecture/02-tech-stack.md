# 02 — Technology Stack Selection & Rationale

This document lists the tools, languages, and frameworks selected for the **CareerLift** platform and provides the architectural justification for each choice.

---

## Architecture Stack Overview

```
┌──────────────────────────────────────────────────────────────────┐
│                           Frontend Client                        │
│   • React 19 SPA                                                 │
│   • Vite Bundler                                                 │
│   • TailwindCSS & Shadcn UI (Visual Style)                       │
│   • Zustand (UI State) & TanStack Query (Server Cache)           │
└────────────────────────────────┬─────────────────────────────────┘
                                 │
                                 ▼ (REST API over HTTPS)
┌──────────────────────────────────────────────────────────────────┐
│                           Backend Service                        │
│   • Node.js + Express (TypeScript)                               │
│   • Drizzle ORM (Type-safe SQL Generation)                       │
│   • Argon2 (Password Cryptography)                               │
└────────────────────────────────┬─────────────────────────────────┘
                                 │
                     ┌───────────┴───────────┐
                     ▼                       ▼
        ┌────────────────────────┐  ┌────────────────────────┐
        │        Database        │  │     Caching & Locks    │
        │   • PostgreSQL 16      │  │   • Redis (Upstash)    │
        │   • Neon Serverless    │  │   • Timezone state     │
        └────────────────────────┘  └────────────────────────┘
```

---

## Frontend Tech Stack

| Technology | Rationale | Alternatives Considered |
| :--- | :--- | :--- |
| **React 19** | Built-in concurrent features, support for native Document Metadata, Actions API for cleaner form transitions, and a stable ecosystem. | Next.js (Overkill for a pure SPA dashboard; static site generation is not needed for private practice routes). |
| **TypeScript** | Static typing prevents runtime bugs, enforces strict contracts between modules, and coordinates with Zod schemas. | Plain JavaScript (Prone to structural bugs as state models grow). |
| **Vite** | Extremely fast cold-starts and hot module replacement (HMR). Excellent plugin ecosystem. | Webpack / Create React App (Slow builds, deprecated config setups). |
| **Zustand** | Highly performant, extremely small bundle footprint, minimal boilerplate. Ideal for client-only settings like themes and active drafts. | Redux Toolkit (High boilerplate), React Context (Causes unnecessary re-renders on rapid updates). |
| **TanStack Query** | Handles caching, pagination pre-fetching, query status monitoring, and auto-retry errors out-of-the-box. | Custom fetch wrappers (Tiresome to implement, buggy cache management). |
| **TailwindCSS** | Encourages consistent layout styling using utility tokens. Easy responsive design (`md:grid-cols-2`). | Styled Components, CSS Modules (Slower layout iteration, larger runtime overhead). |
| **Shadcn UI** | Unstyled accessible Radix primitives. You have full code ownership since files are directly copy-pasted into your project. | Material UI / Ant Design (Heavy bundles, difficult to customize visual themes). |
| **React Hook Form** | Uncontrolled input architecture ensures that key presses do not cause high-frequency re-renders on large forms. | Formik (Higher bundle size, performance bottlenecks on large text areas). |
| **Zod** | Direct schema validation of runtime JSON. Generates TypeScript types directly from validation rules. | Yup / Joi (Yup has less clean TS integration; Joi is typically node-only). |

---

## Backend Tech Stack

| Technology | Rationale | Alternatives Considered |
| :--- | :--- | :--- |
| **Node.js (TypeScript)** | Unified code language across frontend and backend. Enables sharing validation models (Zod schemas) between API and Form fields. | Rust (Axum) - Great performance, but Node.js allows for faster iteration in a frontend-focused team. |
| **Express** | Minimalist web server. Flexible routing, wide adoption, and quick middleware integration. | NestJS (High structure, but steep learning curve and heavy architecture for an MVP). |
| **Drizzle ORM** | Type-safe SQL builder. Unlike other ORMs, Drizzle translates directly to SQL structures without massive execution overhead. | Prisma (Excellent DX, but has a larger engine bundle and higher query latency on serverless cold starts). |
| **PostgreSQL 16** | Strong relational integrity. Supports JSONB columns for flexible data (e.g., question tags, metadata). | MongoDB (Lack of strict relational constraints for streaks and bookmarks). |
| **Redis** | In-memory speed. Indispensable for tracking high-frequency operations like daily streak clocks and api rate limiting. | Database polling (Too slow and resource-heavy for simple flag checks). |
