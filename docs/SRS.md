---
title: SimuXTock
status: ready
updated: 2026-11-23
---

# SimuXTock: Software Requirements Specification

## 1. Overview

### Problem
People learning how financial markets operate must either risk real money or rely on simplified simulators that merely replay external price feeds, where user orders have zero impact on the market. As a result, learners never observe how an Order Book queues competing orders, how trades drive price discovery, or how sudden buying and selling pressure shifts liquidity. From a systems engineering standpoint, building a live exchange requires solving strict event ordering, crash-safe durability, exact financial conservation, and real-time data distribution under load.

### Users
- **Guest:** Unauthenticated visitor who views live market data across all tradable assets in read-only mode.
- **Trader:** Authenticated human participant who places, cancels, and modifies orders, tracks portfolio performance, and reviews personal order and trade history.
- **Market Bot:** Automated first-class market participant executing trading strategies to maintain liquidity and continuous market activity.
- **Admin:** System operator who configures initial assets, fallback seed data, bot inventories, and system parameters via seed configuration prior to startup.

### Scope
- Trader registration and authentication via an in-app JWT authentication stack, starting with `$10,000.00` in simulated cash and supporting cooldown-limited account resets.
- Read-only live market browsing for Guests.
- 5 seeded tradable assets operating 24/7—bootstrapped from real-world tickers via `yfinance` for realistic company names and Initial Reference Prices, with a local fictional-asset CSV fallback—each maintaining an isolated Order Book and whole-share integer trading where all post-seed price movement is driven strictly by SimuXTock trades.
- Limit Orders, Market Orders, order cancellation, and order modification (modeled as Cancel + New Order).
- Deterministic Matching Engine implementing price-time priority (FIFO), Maker-price execution, Price Improvement, Self-Trade Prevention (Cancel Resting), and strict per-asset serial ordering.
- Dual-bucket accounting (Available and Reserved cash and shares) with exact integer minor-unit arithmetic and Zero-Sum Conservation.
- Trader portfolio valuation (weighted average cost basis, Realized PnL, Unrealized PnL, Total Equity, Total Return %) and complete personal order and trade history.
- Real-time public market data: aggregated Level 2 Order Book depth, Last Traded Price, Recent Trades Feed (Tape), Volume, and OHLCV candlesticks (`1m` default, `5m`, `1h`) with flat carry-forward empty intervals.
- Four Market Bot archetypes (Market Maker with inventory skew, Noise Trader, Trend-Follower, Mean-Reversion) running in a separate Bot Service and trading through the same API, engine, and accounting rules as Traders.
- Seed-based Admin configuration for initial assets, fallback CSV, Market Bot inventories, Exchange Treasury balances, and risk/rate limits.

### Out of scope
- Live external market-data replay after initial seeding (`yfinance` is strictly a one-time seed anchor, never a live price feed).
- Leaderboards, competitions, and time-limited seasons.
- Market hours, opening/closing auctions, and trading pauses (V1 operates 24/7).
- Advanced order types (stop orders, time-in-force, end-of-day expiration).
- Short-selling and fractional shares.
- Per-trade fees or commissions.
- Admin web UI or runtime asset-management endpoints.
- External payment processor integration (deposits and withdrawals).
- AI-generated trade explanations, portfolio summaries, and AI-driven bots (deferred to V2).

## 2. Objectives
Deliver a deterministic, crash-recoverable, real-time stock exchange simulator that demonstrates production-grade matching mechanics, exact financial accounting, and believable continuous market activity under verifiable load.

- **OBJ-01:** Maintain a continuously active, two-sided simulated market immediately visible to visitors (measured by automated 24-hour unattended market-health audit across all seeded assets, target 100% of 5-minute windows containing `>= 1` executed trade and non-empty Bid and Ask sides).
- **OBJ-02:** Process concurrent order traffic with low latency (measured by k6 sustained load test across seeded assets, target `>= 1,000` order actions/second with p99 acknowledgment latency `< 50 ms`).
- **OBJ-03:** Preserve exact financial conservation under concurrent load (measured by automated post-load-test ledger reconciliation across all Traders, Market Bots, and the Exchange Treasury, target `0` cents cash discrepancy and `0` shares discrepancy per asset).
- **OBJ-04:** Recover exact exchange state after an abrupt process crash without losing or duplicating transactions (measured by automated kill-and-restart fault-injection tests under active load, target `0` lost or duplicated acknowledged orders or trades and full recovery in `< 10 seconds`).
- **OBJ-05:** Stream live market updates reliably to concurrent viewers without overwhelming clients (measured by k6 WebSocket concurrency test using `k6/websockets`, target `>= 500` simultaneous active connections sustained for 5 minutes with `< 0.1%` unexpected disconnects).

## 3. Business rules
- **BR-01 (Exchange as Matchmaker & Whole Shares):** The exchange never acts as a buyer or seller; every trade occurs strictly between two distinct accounts in whole integer share quantities (`>= 1`).
- **BR-02 (Asset Isolation):** Each tradable asset maintains an independent Order Book; orders for one asset never match against or alter another asset's Order Book.
- **BR-03 (Price Tick & Order Bounds):** All order prices must be `>= $0.01` (`1` cent) in `$0.01` (`1`-cent) increments; per-order share quantity must be `>= 1` and cannot exceed the configured maximum per-order cap (default `10,000` shares).
- **BR-04 (Crossing Invariant):** In every resting Order Book, `Best Bid < Best Ask` always holds; any incoming order whose price meets or crosses the opposite side executes immediately rather than resting.
- **BR-05 (Price-Time Priority — FIFO):** Incoming orders match against the best opposite-side price first (lowest Ask for Buys; highest Bid for Sells), breaking ties at the same price level strictly in First-In, First-Out arrival order.
- **BR-06 (Maker Price Execution & Price Improvement):** Every trade executes at the resting Maker order's price, granting Price Improvement when a Taker's limit price is more aggressive; an incoming order matching multiple resting orders produces one distinct Trade record per Maker matched, and Last Traded Price updates to the final trade's price.
- **BR-07 (Self-Trade Prevention — Cancel Resting):** An account cannot trade against its own resting order; if an incoming order would match a resting order owned by the same account, the engine auto-cancels the resting order (releasing its reservation) and continues matching against the next resting order.
- **BR-08 (Strict Per-Asset Serial Ordering):** Every incoming command (new order, cancel, or modify) for a given asset is assigned a monotonically increasing Sequence Number and processed strictly one at a time; distinct assets may process concurrently.
- **BR-09 (Order Lifecycle & Immutability):** Every order tracks `original_quantity`, `filled_quantity`, and `remaining_quantity` across valid states `REJECTED`, `OPEN`, `PARTIALLY_FILLED`, `FILLED`, and `CANCELLED`. Filled quantities are immutable and never reversed. Market Orders never rest in the book and auto-cancel any unfilled remainder upon liquidity or cash exhaustion. Modifying an active Limit Order executes atomically as Cancel + New Order, forfeiting original time priority.
- **BR-10 (No Short-Selling, Reservations & Settlement):** Every account partitions cash and per-asset shares into `available` and `reserved` buckets (`total = available + reserved`), with no short-selling permitted. Limit Buys require and reserve `qty × limit_price`; Sell orders (Limit or Market) require and reserve `qty`; insufficient available balances result in `REJECTED`. Market Buys draw directly from `available_cash` on each fill. On trade settlement, the Seller's `reserved_shares` decrease and `available_cash` increases by `fill_qty × execution_price`; the Limit Buyer's `reserved_cash` decreases by `fill_qty × buyer_limit_price`, `available_shares` increase by `fill_qty`, and any Price Improvement (`fill_qty × (buyer_limit_price - execution_price)`) refunds directly to `available_cash`. Cancellations release all remaining unfilled reservations back to `available`.
- **BR-11 (Exact Integer Precision & Zero-Sum Conservation):** All prices, balances, and PnL values are represented as exact integers in minor units (cents), never floating-point types. Total cash and total shares per asset summed across all accounts (Traders, Market Bots, and the Exchange Treasury) remain strictly constant across all trades, cancellations, and account resets.
- **BR-12 (Cost Basis & PnL Accounting):** Each account's holding per asset tracks weighted `average_cost_basis` (recalculated on buys, unchanged on partial sells, reset to `0` when `total_shares == 0`). Realized PnL permanently accumulates `(sell_price - average_cost_basis) × sold_qty` on every sell; Unrealized PnL equals `(current_price - average_cost_basis) × total_shares` using Last Traded Price (falling back to Initial Reference Price before the first trade). Total Equity equals `total_cash + Holdings Value`, and `Total Equity - Starting Cash == Total Realized PnL + Total Unrealized PnL` holds at every point between resets.
- **BR-13 (Derived Market Data & OHLCV Candlesticks):** Public market data derives strictly from Order Book state and executed Trades; unmatched resting orders never alter Last Traded Price, Volume, or Candlesticks. Volume counts traded shares once per trade. OHLCV candles aggregate executed trades into `1m`, `5m`, and `1h` intervals, emitting a flat carry-forward bar (`O = H = L = C = previous_close`, `V = 0`) when zero trades occur during an interval.
- **BR-14 (Market Bot First-Class Parity):** Market Bots operate as standard accounts (`is_bot = true`) seeded with finite cash and share inventories, subject to the exact same Matching Engine, reservation, and accounting rules as Traders.
- **BR-15 (Starting Balance & Off-Book Account Reset):** Each newly registered Trader receives `$10,000.00` transferred from a dedicated, non-trading Exchange Treasury account. A Trader may reset their account at most once every 24 hours; resetting auto-cancels all active orders, transfers all remaining cash and shares back to the Exchange Treasury off-book (without executing against the Order Book), resets realized PnL and cost basis, and credits `$10,000.00` from the Exchange Treasury.
- **BR-16 (Rate Limits & Active Order Cap):** Each Trader is capped at 10 order actions/second sustained (burst of 20) and at most 50 active resting orders (`OPEN` or `PARTIALLY_FILLED`) simultaneously; public REST endpoints are rate-limited to 60 requests/minute per IP.

## 4. Functional requirements
- **FR-01:** A Guest can view the list of all seeded tradable assets with their Ticker, company name, Last Traded Price (or Initial Reference Price), and rolling Volume.
- **FR-02:** A Guest can view a live aggregated Level 2 Order Book (Bid and Ask Price Levels with total resting quantities and Bid-Ask Spread) for any selected asset.
- **FR-03:** A Guest can view a live Recent Trades Feed (Tape) displaying `price`, `quantity`, `timestamp`, and `aggressor_side` (`BUY` or `SELL`) for any selected asset.
- **FR-04:** A Guest can view live-updating OHLCV candlestick charts for any selected asset across `1m` (default), `5m`, and `1h` timeframes.
- **FR-05:** A Guest can register a new Trader account with unique credentials and immediately receive the `$10,000.00` starting balance per `SRS/BR-15`.
- **FR-06:** A Trader can sign in, refresh their session, and sign out of their account.
- **FR-07:** A Trader can access all live public market views available to Guests (`SRS/FR-01` through `SRS/FR-04`).
- **FR-08:** A Trader can submit Limit Buy and Limit Sell orders specifying asset, side, price, and whole-share quantity, subject to `SRS/BR-03`, `SRS/BR-10`, and `SRS/BR-16`.
- **FR-09:** A Trader can submit Market Buy and Market Sell orders specifying asset, side, and whole-share quantity, subject to `SRS/BR-03`, `SRS/BR-09`, `SRS/BR-10`, and `SRS/BR-16`.
- **FR-10:** A Trader can cancel any of their active (`OPEN` or `PARTIALLY_FILLED`) orders, immediately releasing unfilled reservations per `SRS/BR-10`.
- **FR-11:** A Trader can modify the price or quantity of any active Limit Order, executed atomically as Cancel + New Order per `SRS/BR-09`.
- **FR-12:** A Trader can view their real-time portfolio showing Available Cash, Reserved Cash, per-asset Available and Reserved Shares, weighted `average_cost_basis`, Holdings Value, Realized PnL, Unrealized PnL, Total Equity, and Total Return % per `SRS/BR-12`.
- **FR-13:** A Trader can view their personal order history (filterable by asset and lifecycle state) and personal executed trade history.
- **FR-14:** A Trader receives real-time private updates whenever their order status, fills, or cash/share balances change.
- **FR-15:** A Trader can trigger a full account reset back to `$10,000.00` once every 24 hours per `SRS/BR-15`.
- **FR-16:** A Market Bot can autonomously submit and cancel Limit and Market orders across four archetypes (Market Maker with inventory skew, Noise Trader, Trend-Follower, and Mean-Reversion) through the same API, engine, and accounting rules as Traders per `SRS/BR-14`.
- **FR-17:** An Admin can configure the target `yfinance` seed ticker list, the local fictional-asset fallback CSV, Market Bot accounts and starting cash/share inventories, Exchange Treasury balances, and risk/rate limits via startup seed configuration (`SRS/TR-09`).

## 5. Non-functional requirements

### Performance
- **NFR-01:** The Matching Engine shall sustain `>= 1,000` order actions/second (placements and cancellations combined) across seeded assets with p99 order acknowledgment latency `< 50 ms`. **Verified by:** Automated k6 load test suite (`SRS/TR-10`).
- **NFR-02:** Public real-time market data broadcasts (Order Book snapshots and live-forming candle updates) shall be conflated/throttled to intervals between `100 ms` and `250 ms` per asset while sustaining `>= 500` simultaneous active WebSocket connections for 5 minutes with `< 0.1%` unexpected disconnects. **Verified by:** Automated k6 WebSocket concurrency test using `k6/websockets` (`SRS/TR-10`).
- **NFR-03:** Portfolio and trade-history read endpoints shall respond with p95 latency `< 200 ms` while the engine operates at `1,000` order actions/second. **Verified by:** Automated k6 mixed-workload benchmark (`SRS/TR-10`).

### Reliability & Durability
- **NFR-04:** Every accepted order, cancellation, and executed trade shall be durably persisted prior to client acknowledgment via the checksummed WAL (`SRS/TR-08`), resulting in `0` lost or duplicated acknowledged orders or trades after an unexpected process termination (`SIGKILL`). **Verified by:** Automated WAL replay/CRC32 unit tests and fault-injection kill-and-restart verification.
- **NFR-05:** Upon restart after an unexpected crash, the system shall reconstruct the exact Order Book, Sequence Numbers, and account balances from the latest PostgreSQL snapshot plus trailing WAL replay and resume accepting orders in `< 10 seconds`. **Verified by:** Crash-recovery timing test comparing pre-crash acknowledged state against post-restart snapshots.
- **NFR-06:** Market Bots and the Matching Engine shall operate continuously 24/7 without memory leaks (resident memory growth `< 5%` over a 2-hour steady-state soak test) or runaway price collapse/explosion (asset prices remain `>= $0.01` and bounded by Mean-Reversion bots). **Verified by:** Automated 2-hour k6/soak test with memory telemetry and price-bound assertions.

### Data Integrity
- **NFR-07:** Under concurrent order placement, partial fills, self-trade prevention, cancellations racing with matches, and account resets, `Total Cash` and per-asset `Total Shares` across all accounts (including Exchange Treasury and Market Bots) shall have `0` cent/share drift, and `Total Equity - Starting Cash == Total Realized PnL + Total Unrealized PnL` shall hold for every non-reset Trader. **Verified by:** Vitest + `fast-check` property-based domain tests (`SRS/TR-11`) and post-k6 ledger reconciliation script.
- **NFR-08:** All monetary and share calculations shall use exact integer arithmetic (`Number.isSafeInteger`-guarded minor-unit Value Objects) with `0` floating-point rounding discrepancies. **Verified by:** Static code review and Vitest unit tests covering sub-dollar ticks, large multi-level sweeps, and weighted cost-basis edge cases.

### Security
- **NFR-09:** Trader passwords shall be hashed using Argon2id (`argon2`) offloaded to the native thread pool away from the main matching event loop; short-lived JWT access tokens (`jose`) and hashed PostgreSQL refresh tokens shall use `HttpOnly; Secure; SameSite=Lax` cookies for browsers; all authenticated endpoints and private real-time channels shall reject unauthenticated or cross-account requests with HTTP `401`/`403` (or WebSocket close code). **Verified by:** Automated API security and authorization tests.
- **NFR-10:** Rate limits (`SRS/BR-16`) shall reject excess trading commands and REST requests with HTTP `429` without degrading engine correctness or blocking other accounts. **Verified by:** Automated k6 rate-limit enforcement test.

### Observability & Operability
- **NFR-11:** The system shall expose structured JSON logs (`pino`), live container telemetry (`Dozzle`), exception alerting (`Sentry`), and built-in JSON `/health` and `/metrics` endpoints reporting per-asset throughput, projection queue depth, active WebSocket connections, and last processed Sequence Number (`SRS/TR-10`). **Verified by:** Integration check polling health/metrics endpoints during load execution.

## 6. Technological requirements
- **TR-01:** The Backend shall be implemented as a **single-process TypeScript (Node.js 22+ LTS) monolith** using **standalone Fastify** and **TypeBox** (`@fastify/type-provider-typebox`) for edge schema validation, structured with **Clean Architecture and DDD** (explicit manual Composition Root, zero framework or ORM coupling in the Domain layer) and keeping bounded active working state (Order Books, live account balances, hot market snapshots, and per-account rate-limit buckets) in local RAM. **Why:** Chosen by project owner to guarantee lock-free single-writer serial determinism (`SRS/BR-08`), eliminate framework magic, and avoid inter-process network overhead in V1.
- **TR-02:** **Nginx** shall act as the public reverse proxy, serving static Frontend assets, terminating TLS, routing `/api` and `/ws` under a single origin, and enforcing per-IP public REST rate limits (`SRS/BR-16`). **Why:** Chosen by project owner for unified edge routing and clean client security boundaries.
- **TR-03:** The web client Frontend shall be built with **React + TypeScript (Vite)**, **Tailwind CSS**, **Zustand** (isolating high-frequency market state from UI re-renders), and **TradingView `lightweight-charts`** (updated imperatively for live OHLCV candlestick rendering, `SRS/FR-04`). **Why:** Chosen by project owner for responsive real-time charting and static-bundle compatibility behind Nginx.
- **TR-04:** Primary durable relational storage shall use **PostgreSQL** accessed via **Drizzle ORM (`pg`)** with versioned SQL migrations (`drizzle-kit`), following a logical **CQRS** separation between projected read/audit tables, periodic snapshots, and auth storage (persistence schemas and mappers remain strictly outside the Domain layer and in-memory Matching Engine). **Why:** Chosen by project owner for portable ACID persistence, explicit SQL visibility, and clean DDD persistence boundaries without vendor lock-in.
- **TR-05:** User authentication shall use an in-app JWT stack (**`jose` + `argon2`** with `@fastify/cookie`) issuing short-lived JWT access tokens verified in memory and hashed refresh tokens persisted in PostgreSQL, accepting `HttpOnly` cookies for browsers and `Authorization: Bearer` headers for automated clients. **Why:** Chosen by project owner to avoid external auth dependencies and third-party load-test rate limits while keeping hot-path auth DB-free and non-blocking.
- **TR-06:** Market Bots shall run as a **separate Python Bot Service** communicating directly with the Backend's HTTP/WS API over the internal network. **Why:** Enforces first-class participant parity (`SRS/BR-14`), isolates bot CPU from the Matching Engine, bypasses public Nginx per-IP rate limits without bypassing engine validation, and establishes the Python runtime for future V2 AI extensions.
- **TR-07:** Real-time public market feeds and private account updates shall use raw **WebSockets (`@fastify/websocket`)** supporting broadcast conflation (`100–250 ms`) and server-side `ping`/`pong` heartbeats. **Why:** Required to satisfy live Order Book, Tape, and Candlestick streaming (`SRS/NFR-02`) without HTTP polling or protocol-wrapper overhead.
- **TR-08:** External message brokers (RabbitMQ/Kafka) and external caches (Redis) shall be **excluded in V1**; pre-acknowledgment durability shall use a **local append-only Write-Ahead Log (WAL) file** (length-prefixed, CRC32-checksummed, `fsync`ed prior to ACK) on a persistent Docker volume paired with **periodic engine snapshots in PostgreSQL** and deterministic replay on restart (`SRS/NFR-04`, `SRS/NFR-05`). **Why:** Chosen by project owner to maximize hot-path throughput without broker/cache overhead while delivering authentic exchange crash recovery.
- **TR-09:** Initial database seeding (on an empty database only) shall fetch 5 real-world tickers and their Initial Reference Prices via **`yfinance`**, automatically falling back to a **local CSV file of fictional assets** if the external fetch fails or is rate-limited; crash restarts and live trading shall never invoke external price APIs. **Why:** Chosen by project owner to combine recognizable market realism with guaranteed offline/rate-limit-proof startup reliability.
- **TR-10:** Performance, WebSocket concurrency, and rate-limit benchmarks shall run on **k6** (`k6/websockets`); the full stack shall deploy hosting-agnostically via a single **Docker Compose** command; runtime observability shall use **`pino`** (built-in Fastify JSON logging), **Dozzle**, **Sentry (free tier)**, and built-in JSON `/health` and `/metrics` endpoints (`SRS/NFR-11`). **Why:** Chosen by project owner for reproducible portfolio deployment, verifiable benchmarks, and minimal observability footprint.
- **TR-11:** Automated testing shall use **Vitest** across both TypeScript codebases (`backend/` and `frontend/`), focused on **Backend Domain unit and property-based tests (`fast-check`)** (verifying matching rules, Zero-Sum Conservation, and WAL replay/CRC32 recovery) plus **essential Frontend component tests** (React Testing Library), complemented by **k6** for live load and WebSocket benchmarks (`SRS/TR-10`). **Why:** Concentrates test effort on financial correctness and core UI behavior without heavy E2E or container-orchestrated test overhead.

## Notes
- **Custodial `T+0` Exchange Model (Binance-like vs. Traditional Equities Exchanges):** Structurally, SimuXTock operates closer to a centralized cryptocurrency exchange (such as Binance or Coinbase) than a traditional stock exchange (such as NYSE or Nasdaq). Traditional equities venues act solely as ticker-sharded matching engines while external brokers custody balances and clearinghouses settle trades on `T+1`. SimuXTock combines exchange, broker, and custodian in a single system—holding user balances directly, enforcing pre-trade `available`/`reserved` locks, settling every trade instantaneously (`T+0`), serving retail Traders directly, and operating 24/7.
- **V2 Python AI Extensibility:** Because the exchange exposes clean HTTP and WebSocket boundaries, a future V2 Python AI service (for trade explanations, portfolio analysis, or AI-driven bots) can integrate directly as an external client without modifying the TypeScript Matching Engine.
