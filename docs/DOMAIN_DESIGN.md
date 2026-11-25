# SimuXTock — Domain Design

## 1. Domain

**SimuXTock** is a deterministic, real-time simulated financial exchange. Its core domain is **order-driven equity trading**: accepting buy and sell orders from human traders and autonomous market bots, matching them under strict price-time priority, and atomically settling cash and share transfers under zero-sum conservation.

* **Core Invariants:** Integer-only arithmetic (cents), strict asset isolation, uncrossed resting order books (`Best Bid < Best Ask`), no negative balances or short-selling, and zero-sum conservation of cash and shares across the entire system (`TRADER` + `BOT` + `TREASURY`).

* **Architectural Style:** Modular Monolith with in-memory hot-path execution backed by a pre-acknowledgment Write-Ahead Log (WAL) for deterministic replay, paired with PostgreSQL for identity, read projections, and event auditing. *(See `CROSSCUTTING_CONCERNS.md` for logging, event envelopes, and audit persistence.)*

## 2. Subdomains (Problem Space)

SimuXTock is decomposed into six subdomains classified by business criticality:

| Subdomain | Classification | Responsibility | 
| ----- | ----- | ----- | 
| **Trading (Order Execution)** | **Core** | Order validation, FIFO price-time matching, Self-Trade Prevention, deterministic per-asset sequencing, and Order Book state. | 
| **Balance Accounting (Custody & Settlement)** | **Core** | Dual-bucket `available` / `reserved` locking, atomic trade settlement, Price Improvement refunds, and strict Zero-Sum Conservation. | 
| **Market Data** | **Supporting** | Public read-only market views: Level 2 aggregated Order Book, Recent Trades Feed (Tape), and `1m` / `5m` / `1h` OHLCV candlesticks. | 
| **Portfolio & Analytics** | **Supporting** | Per-trader private views: Weighted Average Cost Basis, Realized/Unrealized PnL, Total Equity, Total Return %, and order/trade history. | 
| **Market Simulation** | **Supporting** | Autonomous Market Bots (Market Maker, Noise, Trend-Follower, Mean-Reversion) providing 24/7 liquidity and realistic price discovery. | 
| **Identity & Access** | **Generic** | Trader registration, credential verification, and session/refresh-token lifecycle. | 

## 3. Bounded Contexts (Solution Space)

The problem space maps into **five Bounded Contexts**—four internal modules inside the TypeScript monolith and one external Python service:

1. **Trading** *(Core)* — Owns order validation, order book state, matching rules, and trade generation.

2. **Accounts / Portfolio** *(Core + Supporting)* — Owns cash and share custody, balance reservations, atomic settlement, cost basis, PnL, and 24-hour account resets. Uses internal CQRS (RAM + WAL on the write path; PostgreSQL projections for read-heavy history).

3. **Market Data** *(Supporting)* — Pure read-side projection context that listens to Trading events to build public L2 order books, trade tapes, and OHLCV candlesticks.

4. **Identity & Access** *(Generic — Delegated to Better Auth)* — Off-the-shelf authentication and session management powered by **Better Auth**, keeping custom auth plumbing out of the domain while bridging user registration into the exchange via lifecycle hooks.

5. **Market Simulation** *(Supporting — External)* — Standalone Python service running autonomous bots that interact with the exchange through the exact same public API and rules as human traders.

## 4. Tactical Design by Bounded Context

### 4.1 Trading

* **Aggregate Root**

  * **`OrderBook`** *(one isolated instance per `Ticker`)*

    * *Structure:* Sorted price levels with doubly-linked FIFO queues and an `O(1)` order lookup map.

    * *Invariants:* Asset isolation (`BR-02`), uncrossed book at rest (`Best Bid < Best Ask`, `BR-04`), strict FIFO per price level (`BR-05`), monotonic per-asset `SequenceNumber` (`BR-08`), and no resting Market Orders (`BR-09`).

* **Child Entities**

  * **`Order`** (`OrderId`) — Tracks `accountId`, `ticker`, `side`, `type`, `price` *(Limit only)*, `originalQuantity`, `filledQuantity`, `remainingQuantity`, `state`, `sequenceNumber`, and `createdAt`. Lifecycle: `OPEN` → `PARTIALLY_FILLED` → `FILLED` | `CANCELLED` (or immediate `REJECTED`).

  * **`Trade`** (`TradeId`, *Immutable*) — Tracks `ticker`, `makerOrderId`, `takerOrderId`, `buyerAccountId`, `sellerAccountId`, `aggressorSide`, `price` *(Maker's price)*, `quantity`, `sequenceNumber`, and `executedAt`. Never mutated or reversed (`BR-09`).

* **Value Objects**

  * **`Ticker`** — Validated uppercase asset symbol (e.g., `AAPL`, `NOVA`).

  * **`Price`** — Exact positive integer in cents (`>= 1`, tick size `1` cent).

  * **`Quantity`** — Whole integer share count (`1 <= qty <= 10,000`).

  * **`Side`** — `BUY` | `SELL`.

  * **`OrderType`** — `LIMIT` | `MARKET`.

  * **`OrderState`** — `OPEN` | `PARTIALLY_FILLED` | `FILLED` | `CANCELLED` | `REJECTED`.

  * **`SequenceNumber`** — Monotonically increasing non-negative integer per Asset.

* **Domain Service**

  * **`MatchingEngine`** — Stateless coordinator that processes `PlaceOrder`, `CancelOrder`, and `ModifyOrder` against the target `OrderBook`, enforcing Price-Time Priority, Maker Price Execution, and Self-Trade Prevention while invoking `SettlementService` atomically.

### 4.2 Accounts / Portfolio

* **Aggregate Root**

  * **`Account`** (`AccountId`, *persisted as `trading_accounts` in PostgreSQL to avoid colliding with Better Auth's `account` table*)

    * *Attributes:* `userId` *(optional reference to Better Auth `user.id`; `null` for `TREASURY`)*, `accountType`, `cashBalance`, `holdings` (`Map<Ticker, Holding>`), `realizedPnl`, and `lastResetAt`.

    * *Invariants:* No negative balances or short-selling (`available >= 0`, `reserved >= 0`, `BR-10`), 24-hour reset cooldown for `TRADER` accounts (`BR-15`), and exact Weighted Average Cost Basis tracking (`BR-12`).

* **Child Entity**

  * **`Holding`** *(keyed by `Ticker` inside `Account`)* — Tracks `ticker`, `availableShares`, `reservedShares`, and `averageCostBasis`. Derives `totalShares`, `holdingsValue`, and `unrealizedPnl`.

* **Value Objects**

  * **`Money`** — Exact integer cents (signed for PnL; non-negative enforced on balances).

  * **`CashBalance`** — Composite VO of `{ available: Money, reserved: Money }` with derived `total`.

  * **`ShareQuantity`** — Non-negative whole integer (`>= 0`).

  * **`AccountType`** — `TRADER` | `BOT` | `TREASURY`.

* **Domain Service**

  * **`SettlementService`** — Coordinates atomic multi-account transfers to preserve **Zero-Sum Conservation** (`BR-11`):

    * *Trade Settlement (Buyer ↔ Seller):* Moves execution cash, refunds Price Improvement, transfers shares, and updates `averageCostBasis` & `realizedPnl`.

    * *Account Funding (Treasury → Trader):* Transfers `$10,000.00` on new account creation.

    * *Account Reset (Trader ↔ Treasury):* Sweeps remaining cash/shares to Treasury, resets PnL/cost basis, and re-funds `$10,000.00`.

### 4.3 Market Data

Market Data has **no complex Aggregate Roots or write invariants**. It maintains public read models updated purely from Trading events.

* **Read Models / Projections**

  * **`MarketTickerSummary`** — `ticker`, `companyName`, `initialReferencePrice`, `lastTradedPrice`, `rollingVolume`.

  * **`OrderBookSnapshot`** — Aggregated Level 2 Bids (descending), Asks (ascending), and derived `spread`.

  * **`TapeEntry`** — Anonymized public trade record (`price`, `quantity`, `aggressorSide`, `executedAt`).

  * **`CandleSeries`** — Active and closed OHLCV bars per `(ticker, timeframe)`, generating flat carry-forward bars on zero-trade intervals (`BR-13`).

* **Value Objects**

  * **`Candle`** — `{ open: Price, high: Price, low: Price, close: Price, volume: ShareQuantity, intervalStart }`.

  * **`Timeframe`** — `1m` | `5m` | `1h`.

  * **`PriceLevelSnapshot`** — `{ price: Price, totalQuantity: ShareQuantity }`.

### 4.4 Identity & Access (Delegated to Better Auth)

As a **Generic Subdomain**, Identity & Access has no hand-coded Aggregate Roots, Entities, or password-hashing Value Objects. It is delegated to **Better Auth**:

* **Managed Persistence:** Better Auth owns and manages the `user`, `session`, `account` *(credential/OAuth provider records)*, and `verification` tables in PostgreSQL.

* **Enabled Plugins:**
  * **`username` plugin:** Supports username-based trader login alongside standard credentials.
  * **`bearer` plugin:** Allows non-browser clients (such as the external Python Market Simulation bots) to authenticate via `Authorization: Bearer <token>` headers.

* **Boundary Adapter / Lifecycle Hook:** Better Auth's `databaseHooks.user.create.after` hook acts as a thin boundary adapter that translates new user creation into the exchange's `TraderRegistered` event, keeping the rest of the monolith decoupled from Better Auth internals.

### 4.5 Market Simulation (External)

* Runs out-of-process in Python as an external client to the Exchange API.

* Implements four bot archetypes (**Market Maker**, **Noise**, **Trend-Follower**, **Mean-Reversion**), each authenticated via Better Auth's `bearer` plugin as a seeded `BOT` account with finite cash and share inventories.

* Holds no privileged access to internal domain models (`BR-14`).

## 5. Domain Events

All domain objects emit **pure, metadata-free `DomainEvent`s**. *(Tracing metadata such as `eventId`, `occurredAt`, `actorId`, `correlationId`, and `causationId` is attached by the Application layer via `EventEnvelope<T>` — see `CROSSCUTTING_CONCERNS.md`.)*

### 5.1 Trading Events

| Event | Emitted When | Key Consumers | 
| ----- | ----- | ----- | 
| **`OrderOpened`** | A validated Limit order rests on the `OrderBook`. | Market Data (L2 Book), WebSocket Gateway, Audit Store | 
| **`OrderRejected`** | An order fails validation, balance checks, or Self-Trade Prevention. | WebSocket Gateway, Audit Store | 
| **`OrderPartiallyFilled`** | An order executes for part of its quantity and remains active. | Market Data (L2 Book), WebSocket Gateway, Audit Store | 
| **`OrderFilled`** | An order's `remainingQuantity` reaches `0`. | Market Data (L2 Book), WebSocket Gateway, Audit Store | 
| **`OrderCancelled`** | An open/partially filled order is cancelled by the trader, STP, or Market Order remainder cleanup. | Market Data (L2 Book), WebSocket Gateway, Audit Store | 
| **`TradeExecuted`** | Two orders match on the `OrderBook` at the Maker's price. | Market Data (Tape, Candles, Last Price), Audit Store | 

### 5.2 Accounts / Portfolio Events

| Event | Emitted When | Key Consumers | 
| ----- | ----- | ----- | 
| **`AccountCreated`** | A new `Account` is created and funded with `$10,000.00` from the Exchange Treasury. | WebSocket Gateway, Audit Store | 
| **`AccountSettled`** | Cash, shares, cost basis, or PnL change due to a reservation, trade settlement, or cancellation release. | WebSocket Gateway (live balance/PnL), Audit Store | 
| **`AccountReset`** | A `TRADER` account completes a 24-hour reset back to `$10,000.00` and zero holdings. | WebSocket Gateway, Audit Store | 

### 5.3 Identity & Access Events

| Event | Emitted When | Key Consumers | 
| ----- | ----- | ----- | 
| **`TraderRegistered`** | Emitted by Better Auth's `databaseHooks.user.create.after` hook when a new user completes registration. | Accounts / Portfolio (triggers `AccountCreated`), Audit Store | 

## 6. Context Map

### 6.1 Integration Patterns

| Upstream → Downstream | Pattern | Mechanism | Purpose | 
| ----- | ----- | ----- | ----- | 
| **Trading → Accounts** | Customer / Supplier (Explicit Port) | Synchronous in-process call (`SettlementService`) | Atomic pre-match reservation, immediate trade settlement, Price Improvement refunds, and cancel releases. | 
| **Trading → Market Data** | Published Language | In-process Event Bus | Market Data subscribes to `TradeExecuted` and order lifecycle events to update L2 Book, Tape, and OHLCV candles without coupling Trading to views. | 
| **Identity (Better Auth) → Accounts** | Anti-Corruption Hook / Published Language | `databaseHooks.user.create.after` → Event Bus / App Orchestration | Translates Better Auth user creation into `TraderRegistered`, triggering `Account` creation and `$10,000.00` funding from the Exchange Treasury. | 
| **Accounts → WebSocket Gateway** | Published Language | In-process Event Bus | `AccountSettled` and `AccountReset` push live private balance/PnL updates to connected Traders. | 
| **Exchange API → Market Simulation** | Open Host Service (OHS) | Public REST + WebSocket API (Bearer Auth) | Bots authenticate via Better Auth's `bearer` plugin as `BOT` accounts and submit/cancel orders under the exact same rules as human Traders. | 

### 6.2 Context Map Diagram

```text
+-----------------------+   databaseHooks    +-----------------------+
|   Identity & Access   | -----------------> |  Accounts / Portfolio |
| (Better Auth: user,   |  TraderRegistered  | (Account, Holding,    |
|  session, bearer)     |                    |  SettlementService)   |
+-----------------------+                    +-----------------------+
                                                        ^
                                                        | Sync Port (Atomic Reserve & Settle)
                                                        |
+-----------------------+  Public REST / WS  +-----------------------+   Domain Events    +-------------------+
|   Market Simulation   | -----------------> |        Trading        | -----------------> |    Market Data    |
|    (Python Bots)      | <----------------- | (OrderBook, Order,    |                    | (L2 Book, Tape,   |
+-----------------------+    L2 / Tape / WS  |  Trade, MatchingEng.) |                    |  OHLCV Candles)   |
                                             +-----------------------+                    +-------------------+
```
