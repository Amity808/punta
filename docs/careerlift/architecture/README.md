# CareerLift — Architecture Documentation

> An interactive interview-preparation platform designed to make job candidates interview-ready every day through gamification, streaks, and community-sourced interview databases.

---

## Technical Overview

| Dimension | Specification |
| :--- | :--- |
| **Product Focus** | Job interview prep (Duolingo for Software Engineers, PMs, HR, Data Analysts) |
| **Frontend Platform** | React 19 (SPA) + Vite + TypeScript |
| **State Management** | Zustand (UI/Auth) & TanStack Query (Server Cache) |
| **CSS Framework** | TailwindCSS + Shadcn UI |
| **Backend Service** | Node.js + Express + TypeScript |
| **Database** | PostgreSQL 16 (Neon Serverless) |
| **Database ORM** | Drizzle ORM |
| **Caching / Store** | Redis (Upstash) for timezone-aware streaks & locks |
| **Security** | JSON Web Tokens (JWT) + argon2 password hashing |

---

## Documentation Index

| # | Document | Description |
|:--|:---------|:------------|
| 01 | [System Overview](./01-system-overview.md) | High-level system architecture, C4 diagrams, and component interactions. |
| 02 | [Tech Stack](./02-tech-stack.md) | Architectural justification for frontend and backend stack choices. |
| 03 | [Database Schema](./03-database-schema.md) | Detailed PostgreSQL schema definitions and Drizzle ORM models. |
| 04 | [API Design](./04-api-design.md) | Complete list of RESTful API endpoints, request bodies, and responses. |
| 05 | [Frontend Architecture](./05-frontend-architecture.md) | Module directory breakdowns, Zustand store structures, and components layout. |
| 06 | [Gamification & Streaks](./06-gamification-streaks.md) | Algorithmic logic for timezone-adjusted streaks and achievement badges. |
| 07 | [Infrastructure & Deployment](./07-infrastructure.md) | Hosting configuration, serverless scaling, monitoring, and CI/CD pipelines. |
| 08 | [Security](./08-security.md) | Session control, authentication schemes, rate limiting, and inputs validations. |

---

## Roadmap

To view the 6-week MVP delivery timeline, refer to [Roadmap Index](../roadmap/README.md).
