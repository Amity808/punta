# 07 — Infrastructure & Deployment

This document outlines the cloud infrastructure, database scaling properties, static asset hosting, and CI/CD deployment pipelines for **CareerLift**.

---

## Architecture Diagram

```
┌──────────────────────────────────────────────────────────────┐
│                       Cloudflare DNS                         │
└──────────────────────────────┬───────────────────────────────┘
                               │
               ┌───────────────┴───────────────┐
               ▼                               ▼
    ┌────────────────────┐          ┌────────────────────┐
    │  Cloudflare Pages   │          │    Fly.io Proxy    │
    │  (React 19 SPA)    │          │    (Routing Layer) │
    └────────────────────┘          └──────────┬─────────┘
                                               │
                                               ▼
                                    ┌────────────────────┐
                                    │    Express API     │
                                    │    (Fly.io VPS)    │
                                    └──────────┬─────────┘
                                               │
                              ┌────────────────┴────────────────┐
                              ▼                                 ▼
                   ┌────────────────────┐            ┌────────────────────┐
                   │   Neon Postgres    │            │   Upstash Redis    │
                   │   (Database)       │            │   (Streak Caches)  │
                   └────────────────────┘            └────────────────────┘
```

---

## Production Services

| Layer | Provider | Configuration | Rationale |
| :--- | :--- | :--- | :--- |
| **Frontend UI Client** | Cloudflare Pages | Git-connected automatic builds (Vite production compiler) | Fast static asset delivery via Cloudflare's global edge network, with free SSL certificates. |
| **Express API Service** | Fly.io | 1-2 Shared-CPU virtual machines (Docker containers) | Global routing, fast startup, simple config files, and automatic health check updates. |
| **Database** | Neon Database | Serverless PostgreSQL 16 instance with autoscaling | Auto-suspends when inactive to reduce costs; instant branching allows for safe database testing. |
| **Caching Store** | Upstash Redis | Serverless Redis instance | Pay-per-request pricing, microsecond response times, and built-in REST API support. |

---

## Docker Configuration (API Engine)

The backend Express application runs inside a Docker container:

```dockerfile
# Stage 1: Build TypeScript
FROM node:20-alpine AS builder
WORKDIR /app
COPY package*.json tsconfig.json ./
RUN npm ci
COPY src/ src/
RUN npm run build
RUN npm prune --production

# Stage 2: Runtime
FROM node:20-alpine
WORKDIR /app
COPY --from=builder /app/package*.json ./
COPY --from=builder /app/node_modules/ ./node_modules/
COPY --from=builder /app/dist/ ./dist/

EXPOSE 3000
ENV NODE_ENV=production
CMD ["node", "dist/index.js"]
```

---

## Fly.io API Settings (`fly.toml`)

```toml
app = "careerlift-api"
primary_region = "los" # Lagos, Nigeria (or nearest active region)

[http_service]
  internal_port = 3000
  force_https = true
  auto_start_machines = true
  auto_stop_machines = true
  min_machines_running = 1
  processes = ["app"]

[[http_service.checks]]
  grace_period = "10s"
  interval = "15s"
  method = "GET"
  path = "/api/v1/health"
  protocol = "http"
  timeout = "2s"
```

---

## CI/CD Pipeline (GitHub Actions)

This pipeline builds, tests, and deploys the frontend and backend whenever changes are merged into the `main` branch.

```yaml
name: Deploy pipeline

on:
  push:
    branches: [ main ]

jobs:
  validate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      - name: Run lint
        run: npm run lint

      - name: Run unit tests
        run: npm test

  deploy-api:
    needs: validate
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup Flyctl
        uses: superfly/flyctl-actions/setup-flyctl@master
        
      - name: Deploy Container
        run: flyctl deploy --remote-only
        env:
          FLY_API_TOKEN: ${{ secrets.FLY_API_TOKEN }}

  deploy-client:
    needs: validate
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Deploy to Cloudflare Pages
        uses: cloudflare/wrangler-action@v3
        with:
          apiToken: ${{ secrets.CLOUDFLARE_API_TOKEN }}
          accountId: ${{ secrets.CLOUDFLARE_ACCOUNT_ID }}
          projectName: "careerlift-web"
          directory: "dist"
          gitBranch: "main"
```
