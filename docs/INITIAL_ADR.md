---
title: SimuXTock Prototype Architecture Decision Records
status: active
updated: 2026-11-23
---

# SimuXTock: Prototype Architecture Decision Records (ADRs)

## ADR-01: Single-Process Monolith & In-Memory Matching (No Redis / MQ in V1)
- **Status:** Accepted
- **Context:** The exchange must process `>= 1,000` order actions/sec at `< 50 ms` p99 latency while guaranteeing strict per-asset serial determinism (`SRS/BR-08`) and exact Zero-Sum Conservation (`SRS/BR-11`).
- **Decision:** Run the Backend as a single-process Node.js monolith keeping active Order Books, live account balances, hot market snapshots, and rate-limit buckets in RAM. Exclude Redis and external message brokers (RabbitMQ/Kafka) in V1.
- **Consequences:** Eliminates network serialization hops, distributed locking, and operational sprawl. Vertical scaling is bounded by one process, which is well above V1's 5-asset target.

## ADR-02: Standalone Fastify + TypeScript + TypeBox (Manual Composition Root)
- **Status:** Accepted
- **Context:** The project aims to demonstrate Clean Architecture and DDD without framework abstractions hiding wiring, validation, or HTTP/WS lifecycles.
- **Decision:** Use standalone **Fastify** (v5) on Node.js 22+ LTS with **TypeBox** (`@fastify/type-provider-typebox`) for compiled JSON Schema edge validation, `@fastify/websocket` for raw WebSockets, `pino` for structured logging, and `jose` + `argon2` for in-app JWT auth. Wire dependencies manually in an explicit Composition Root; keep `domain/` 100% plain TypeScript.
- **Consequences:** Zero decorator/DI-container magic and high raw throughput, at the cost of writing manual dependency wiring and explicit Domain-to-Persistence mappers.

## ADR-03: PostgreSQL + Drizzle ORM with Logical CQRS
- **Status:** Accepted
- **Context:** Durable read models (order history, trade history, OHLCV candles), auth refresh tokens, and periodic engine snapshots require ACID relational storage, with explicit visibility into PostgreSQL SQL (`ON CONFLICT` idempotency, bulk upserts).
- **Decision:** Use **PostgreSQL** with **Drizzle ORM (`pg`)** and `drizzle-kit` migrations. Separate the hot write path (RAM + WAL) from asynchronous batched PostgreSQL projections (logical CQRS).
- **Consequences:** Direct 1:1 SQL control with full TypeScript safety and no code-generation step. Requires bounding the async projection queue to handle backpressure and draining it cleanly on `SIGTERM`.

## ADR-04: Pre-Acknowledgment Durability via Local Append-Only WAL + PostgreSQL Snapshots
- **Status:** Accepted
- **Context:** The system must survive `SIGKILL` with zero lost or duplicated acknowledged orders/trades (`SRS/NFR-04`) and recover in `< 10 seconds` (`SRS/NFR-05`) without slowing down the hot trading path with synchronous SQL writes.
- **Decision:** Append every sequenced command to a local, length-prefixed, CRC32-checksummed Write-Ahead Log (WAL) file on a persistent Docker volume (`fsync`ed before returning ACK). Periodically serialize full in-RAM engine state into a PostgreSQL Snapshot row and rotate old WAL segments. On startup, load the latest Snapshot and replay trailing valid WAL frames (discarding any torn tail frame).
- **Consequences:** Authentic exchange durability and high throughput. Requires local block/SSD storage (not object storage like S3) for the active WAL volume and custom binary framing/checksum code.

## ADR-05: Order Book Data Structure (Sorted Array + Doubly-Linked List + O(1) Cancel Index)
- **Status:** Accepted
- **Context:** TypeScript has no built-in sorted map. The Order Book needs fast Best Bid/Ask access, strict FIFO queuing per price level (`SRS/BR-05`), fast Level 2 depth snapshots, and `O(1)` cancellations.
- **Decision:** Implement each `BookSide` with a binary-searched sorted array of active prices, a `Map<Price, PriceLevel>` where each `PriceLevel` is a doubly-linked list (FIFO), and a `Map<OrderId, OrderNode>` index for `O(1)` unlinking on cancel.
- **Consequences:** Avoids the complexity of a custom self-balancing binary tree while staying cache-friendly in V8 for typical price-level counts. Encapsulated inside the Domain layer so internals can be swapped without affecting callers.

## ADR-06: Order Idempotency via Proxy Design Pattern
- **Status:** Accepted
- **Context:** Network drops after execution but before ACK can cause Traders or Market Bots to retry `POST /orders`, risking duplicate fills.
- **Decision:** Wrap the order placement use case in an `IdempotentOrderProxy` implementing the same port interface. Clients pass a `client_order_id` (UUID); the proxy checks a bounded in-RAM cache (persisted in the WAL record) and returns the original acknowledgment on duplicate retries.
- **Consequences:** Keeps idempotency concerns completely decoupled from the core Matching Engine Domain logic.

## ADR-07: Separate Python Bot Service & `yfinance` Seeding with Fictional CSV Fallback
- **Status:** Accepted
- **Context:** Market Bots must operate as first-class participants (`SRS/BR-14`), initial assets should feel realistic via `yfinance` without breaking startup on Yahoo rate limits, and the architecture should leave a clean path for a future V2 Python AI module.
- **Decision:** Run the four Market Bot archetypes in a separate **Python** service communicating with the Backend over internal HTTP/WS. Perform a one-time `yfinance` fetch on an empty database for 5 real-world tickers, automatically falling back to a local CSV of fictional assets on any fetch failure. Never call external price APIs during live trading or crash restarts.
- **Consequences:** Guarantees offline/rate-limit-proof startup and isolates bot CPU from the Matching Engine, while introducing a second runtime (Python) for the Bot Service.

## ADR-08: Lean Testing Strategy (Vitest + `fast-check` + k6)
- **Status:** Accepted
- **Context:** Testing effort should focus on proving financial invariants, matching correctness, WAL recovery, and core UI behavior without heavy E2E or container-orchestrated test overhead.
- **Decision:** Use **Vitest** (v5.1+) across both TypeScript codebases: unit + **`fast-check`** property-based fuzzing for the Backend Domain and WAL CRC32 recovery, plus essential Frontend component tests with React Testing Library. Rely on **k6** (`k6/websockets`) for live throughput, WebSocket concurrency, rate-limit, and soak verification.
- **Consequences:** Zero-config native TypeScript/ESM testing with fast watch-mode feedback, trading off automated browser E2E coverage for higher development velocity.

## ADR-09: Independent Service Directories & Frontend State Isolation
- **Status:** Accepted
- **Context:** Keeping tooling simple while learning TypeScript/Fastify/React, and preventing `100–250 ms` live market feeds from causing React full-page re-renders.
- **Decision:** Structure the repository with independent `backend/`, `frontend/`, and `bots/` directories (each with its own dependencies and tests) rather than a workspace monorepo. On the frontend, isolate live Order Book/Tape state in **Zustand** with granular selectors and push live candlestick ticks to TradingView `lightweight-charts` imperatively via `.update()`.
- **Consequences:** Clean Dockerfiles and zero workspace build friction, with manual sync of REST/WS DTOs between `backend/` and `frontend/`.
