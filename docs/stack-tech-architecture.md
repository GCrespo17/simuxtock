# SimuXTock — Finalized Tech Stack & Architecture

## Environment & Tooling
* **OS:** Fedora 44 Linux
* **Runtime:** Node.js `22.23` (managed via `dnf`)
* **Package Manager:** `npm 10.9.8` (managed via `dnf`, native `npm` workspaces)
* **Task Runner:** Optional Turborepo (can be added on top of `npm` workspaces)
* **Containerization:** Docker & Docker Compose (local PostgreSQL + independent Dockerfiles for `web` and `server`)

---

## Tech Stack Overview

| Layer | Choice | Notes |
| :--- | :--- | :--- |
| **Monorepo** | `simuxtock` — Native `npm` workspaces | Manual setup (no Better-T-Stack CLI); optional Turborepo for task orchestration |
| **Backend (`apps/server`)** | **Fastify** + TypeScript | Modular Monolith (DDD: Trading, Accounts, Market Data, Identity) |
| **API & Real-Time** | **REST + WebSockets** (`@fastify/websocket`) | No tRPC — Open Host Service so web UI, Python bots, and a future C#/.NET backend share the exact same language-agnostic contract |
| **Frontend (`apps/web`)** | **React + Vite + TanStack Router** | Pure SPA (no SSR); pairs with TanStack Query + WebSockets |
| **Shared Contracts (`packages/contracts`)** | **TypeScript + Zod** | Published Language for request/response DTOs and WebSocket payloads |
| **Database & ORM** | **PostgreSQL + Drizzle ORM** | Local Postgres via `docker-compose.yml` |
| **Authentication** | **Better Auth** (`username` + `bearer` plugins) | Generic Subdomain; `trading_accounts` links to Better Auth's `user.id` |
| **Logging & Audit** | **Pino** + `audit_events` table | Structured logging with redaction rules |
| **Trading Bots *(later)*** | **Python** (`apps/bots/`) | Connects externally via REST + Bearer token + WebSockets |

---

## Target Folder Layout

```text
simuxtock/
├── apps/
│   ├── server/          # Fastify monolith (DDD Bounded Contexts + Drizzle + Better Auth)
│   ├── web/             # Vite + React + TanStack Router (SPA)
│   └── bots/            # (Later) Python autonomous trading bots
├── packages/
│   └── contracts/       # Shared Zod schemas & API/WS types
├── docker-compose.yml   # Local PostgreSQL
└── package.json         # Root npm workspaces
```

---

## Core Architectural Rules

1. **Strict Decoupling & Migration Readiness:** `apps/web` never imports from `apps/server` or Drizzle schemas. It only imports shared types/schemas from `packages/contracts` and communicates with the server strictly over HTTP REST and WebSockets. This ensures `apps/server` can be migrated to C#/.NET in the future without rewriting the frontend.
2. **Clean Architecture / DDD Boundaries:** Drizzle ORM and Better Auth stay strictly in the **Infrastructure** layer—never inside `domain/`.
3. **Identity & Treasury Integration:** Better Auth's `databaseHooks.user.create.after` hook triggers `TraderRegistered` to fund new `trading_accounts` from `TREASURY`.
4. **In-Memory Matching Engine:** The `OrderBook` lives in RAM inside the Trading bounded context, streaming live book and tape updates over WebSockets while persisting settled state to PostgreSQL.

---

## Manual Bootstrap Checklist

1. Initialize root workspace:
   ```bash
   mkdir -p apps/server packages/contracts
   npm init -y
   ```
2. Add `"workspaces": ["apps/*", "packages/*"]` to the root `package.json`.
3. Scaffold the frontend inside `apps/`:
   ```bash
   cd apps && npm create vite@latest web -- --template react-ts
   ```
4. Set up `docker-compose.yml` for local PostgreSQL.
5. Configure `apps/server` with Fastify, Drizzle ORM, Better Auth, and the DDD bounded context directories.
