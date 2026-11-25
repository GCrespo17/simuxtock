# Stock Exchange Simulator — Domain Model & Business Rules

## 1. The Exchange, Assets, and Price Discovery

### Core Concept
An exchange is a matchmaker and ledger, **not** a store. The exchange never buys or sells shares itself, and it never dictates prices. Every trade occurs strictly between two market participants: a **Buyer** and a **Seller**.

### Tradable Assets (Instruments)
- **Ticker / Symbol:** Each fictional company is identified by a short unique symbol (e.g., `NOVA`, `ACME`).
- **Isolation:** Each asset has its own independent market (Order Book). Orders for `NOVA` never interact with orders for `ACME`.
- **Whole Shares:** Shares are traded in whole integer quantities (1, 2, 50 shares) to avoid fractional-share complexity in the core engine.

### Price Discovery & The Bootstrap Problem
- **Last Traded Price (Last Price):** The headline "price" of a stock is simply the price at which the most recent trade occurred. It is a historical fact, not a guarantee of what you can buy or sell at right now.
- **Initial Reference Price:** When an Admin creates a new asset before any trades have occurred, an Initial Reference Price (e.g., `$100.00`) is defined so traders and bots have an initial valuation anchor.
- **Initial Share Circulation (Bootstrapping):** Because human users start only with cash (`$`), **Market Bots** are seeded with both an initial inventory of shares and cash when an asset is created. These bots place the initial buy and sell orders so human traders immediately have counterparties to trade against.


## 2. The Order Book, Bids, Asks & The Spread

### Two Sides of the Book
Each tradable asset maintains its own **Order Book** of resting (unfilled) orders split into two sides:
- **Bids (Buyers):** Orders to buy shares up to a maximum price, sorted **highest price first**. The highest waiting buy price is the **Best Bid**.
- **Asks / Offers (Sellers):** Orders to sell shares down to a minimum price, sorted **lowest price first**. The lowest waiting sell price is the **Best Ask**.

### The Spread & Book Invariant
- **Bid-Ask Spread:** The price gap between the Best Ask and Best Bid (`Best Ask - Best Bid`).
- **Crossing Invariant:** In a resting order book, `Best Bid < Best Ask` is always true. If an incoming order arrives at a price that meets or crosses the opposite side (e.g., a Bid at or above the Best Ask), it matches immediately into a trade rather than resting in the book.
- **Mid-Price:** The midpoint between Best Bid and Best Ask is a reference indicator only; no trader can execute instantaneously at the mid-price because no counterparty is resting there. Placing a new order inside the spread does not execute immediately—it rests in the book and narrows the spread.

### Individual Orders vs. Price Levels
- **Engine View:** Internally, the matching engine tracks every individual order separately so arrival order (time priority) is preserved.
- **Display View (Aggregation):** For traders viewing the book, orders sharing the same price are summed into a single **Price Level** (price + total resting quantity).


## 3. Order Types & The Lifecycle of an Order

### Supported Order Types (V1)
- **Limit Order (Price + Quantity):**
  - Specifies a maximum purchase price (Limit Buy) or minimum sale price (Limit Sell).
  - Guarantees **price** (never executes at a worse price than specified), but does **not** guarantee immediate execution.
  - Any unfilled portion rests in the Order Book until matched or cancelled.
- **Market Order (Quantity Only):**
  - Specifies quantity with no price limit; executes immediately against the best available resting orders on the opposite side of the book.
  - Guarantees **speed** (subject to available liquidity), but does **not** guarantee price—large orders sweep through multiple price levels, causing **Slippage**.
  - **Never rests in the book:** If the opposite side of the book is completely exhausted before the Market Order finishes, any leftover quantity is immediately cancelled.

### Order Quantities & States
Every order tracks `original_quantity`, `filled_quantity`, and `remaining_quantity` (where `filled + remaining <= original`, and for active orders `filled + remaining == original`).

Valid lifecycle states:
- **`REJECTED`:** Fails validation before entering the engine (e.g., invalid quantity/price, insufficient funds/shares). Terminal state.
- **`OPEN`:** Accepted and resting in the book with `filled_quantity == 0`.
- **`PARTIALLY_FILLED`:** At least 1 share has traded (`0 < filled_quantity < original_quantity`), and the remaining shares are still resting in the book.
- **`FILLED`:** All shares traded (`filled_quantity == original_quantity`). Terminal state.
- **`CANCELLED`:** Removed from the book before filling completely (either requested by the user or automatically for an exhausted Market Order). Terminal state. Already-filled shares are **immutable** and never reversed; `filled_quantity` remains recorded for history.

### Order Modification Rule
Modifying a resting order's price or quantity is modeled as **Cancel + New Order** so traders cannot alter an order while unfairly keeping an earlier spot in line.


## 4. The Matching Engine & Price-Time Priority

### Maker vs. Taker
- **Maker (Resting Order):** An order already sitting in the Order Book providing liquidity.
- **Taker (Incoming Order):** A newly arriving order that crosses the spread and triggers one or more trades.

### Matching Rules
1. **Price-Time Priority (FIFO):**
   - Incoming orders always match against the **best price** on the opposite side first (lowest Ask for Buys; highest Bid for Sells).
   - When multiple resting orders share the same price level, they match in strict **First-In, First-Out (FIFO)** arrival order. Price always beats time.
2. **Maker Price Execution (Price Improvement):**
   - Every trade executes at the **resting (Maker) order's price**, never the incoming (Taker) order's price.
   - If a Taker's limit price is more aggressive than the Maker's price, the Taker receives **Price Improvement** (pays less or sells for more than their limit).
   - One incoming order sweeping multiple resting orders produces **one distinct Trade record per Maker order matched**, and the asset's **Last Traded Price** updates to the price of the final trade executed.
3. **Self-Trade Prevention (STP — Cancel Resting):**
   - A user is never allowed to trade against their own resting order (prevents wash trading and fake price/volume manipulation).
   - **Rule:** If an incoming order would match a resting order belonging to the same user, the engine **auto-cancels the old resting order** (releasing its reserved funds/shares) and continues matching the incoming order against the next resting order in line.
4. **Strict Serial Ordering per Asset:**
   - Every incoming command (new order or cancel) for a given asset is assigned a monotonically increasing **Sequence Number** and processed **strictly one at a time** so two orders can never double-match the same resting shares.
   - Different assets are independent and may be processed concurrently.


## 5. Balances, Holdings & Reservations

### Accounting Buckets
To prevent double-spending while orders are waiting in the book, every account's cash and per-asset share holdings are split into **Available** and **Reserved** buckets:
- **Cash:** `total_cash = available_cash + reserved_cash`
- **Shares (per asset):** `total_shares = available_shares + reserved_shares`
- **No Short-Selling (V1):** Users may only sell shares they currently own (`available_shares >= order_quantity`).

### Reservation & Settlement Rules
1. **Place Limit Buy (`qty × limit_price`):**
   - Requires `available_cash >= qty × limit_price`; otherwise `REJECTED`.
   - Moves `qty × limit_price` from `available_cash` → `reserved_cash`.
2. **Place Sell Order (Limit or Market):**
   - Requires `available_shares >= qty`; otherwise `REJECTED`.
   - Moves `qty` from `available_shares` → `reserved_shares`.
3. **Place Market Buy (No Limit Price):**
   - Never rests in the book. Walks the Ask side spending directly from `available_cash` on each fill.
   - Stops and cancels any remaining unfilled quantity if either the Ask side is exhausted or `available_cash` is insufficient to buy the next share.
4. **Trade Settlement (with Price Improvement Refund):**
   - For a trade of `fill_qty` at `execution_price`:
     - **Seller:** `reserved_shares -= fill_qty`; `available_cash += fill_qty × execution_price`.
     - **Buyer (Limit Order):** `reserved_cash -= fill_qty × buyer_limit_price`; `available_shares += fill_qty`; and any **Price Improvement** (`fill_qty × (buyer_limit_price - execution_price)`) is refunded directly back into the buyer's `available_cash`.
     - **Buyer (Market Order):** `available_cash -= fill_qty × execution_price`; `available_shares += fill_qty`.
5. **Cancellation:**
   - Releases remaining unfilled `remaining_qty × limit_price` from `reserved_cash` → `available_cash` (Buy) or `remaining_qty` from `reserved_shares` → `available_shares` (Sell).

### Conservation & Precision Invariants
- **Exact Integer Arithmetic:** All money values and prices are stored as exact integers in minor units (e.g., **cents**, where `$55.00 = 5500`)—never floating-point types.
- **Zero-Sum Conservation:** Trading never creates or destroys cash or shares. The sum of `total_cash` across all accounts (humans + bots) and the sum of `total_shares` per asset across all accounts must remain strictly constant across every trade and cancellation.


## 6. Portfolios, Cost Basis & PnL (Profit and Loss)

### Cost Basis (Weighted Average Cost)
Each user's holding in an asset tracks `average_cost_basis`:
- **On Buy (`bought_qty` at `buy_price`):**
  - `new_avg_cost = ((old_total_shares × old_avg_cost) + (bought_qty × buy_price)) / (old_total_shares + bought_qty)`
- **On Sell (`sold_qty` at `sell_price`):**
  - `average_cost_basis` **does not change** for the remaining shares.
  - When `total_shares` reaches `0`, `average_cost_basis` resets to `0`.

### Realized vs. Unrealized PnL
- **Realized PnL (Locked-in):** Calculated and permanently accumulated whenever shares are sold:
  - `trade_realized_pnl = (sell_price - average_cost_basis) × sold_qty`
- **Unrealized PnL (Paper Gain/Loss):** Computed live on currently held shares (`total_shares = available_shares + reserved_shares`) using the asset's current market price (`Last Traded Price`, falling back to `Initial Reference Price` if no trades exist):
  - `unrealized_pnl = (current_price - average_cost_basis) × total_shares`

### Portfolio Valuation & Accounting Invariant
- **Holdings Value:** Sum across all owned assets of `(total_shares × current_price)`.
- **Total Equity (Net Worth):** `total_cash + Holdings Value` (where `total_cash = available_cash + reserved_cash`, since reserved cash and shares still belong to the user until settled).
- **Total Return %:** `((Total Equity - Starting Cash) / Starting Cash) × 100%` (used for leaderboard rankings).
- **PnL Invariant:** At every point in time (with no deposits/withdrawals), `Total Equity - Starting Cash == Total Realized PnL + Total Unrealized PnL`.


## 7. Market Data, Volume & Candlesticks

All public market data is strictly **derived** from Order Book state and executed Trades; resting orders that do not match never alter Last Price, Volume, or Candlesticks.

### Live Feeds
- **Order Book Depth (Level 2):** Aggregated resting Bid and Ask price levels.
- **Recent Trades Feed (Tape):** Chronological log of executed trades including `price`, `quantity`, `timestamp`, and `aggressor_side` (`BUY` or `SELL`, identifying which side was the incoming Taker).
- **Volume:** Total number of shares exchanged in a given window, counted **once per trade** (never double-counted for buyer and seller).

### Candlesticks (OHLCV)
Trades within fixed time intervals (e.g., `1m`, `5m`, `1h`) aggregate into OHLCV bars:
- **Open (O):** First trade price in the interval.
- **High (H):** Highest trade price in the interval.
- **Low (L):** Lowest trade price in the interval.
- **Close (C):** Last trade price in the interval (`Close >= Open` = bullish/green; `Close < Open` = bearish/red).
- **Volume (V):** Sum of traded share quantities in the interval.
- **Empty Intervals:** If zero trades occur in an interval, Volume is `0` and price does not move (either omitted or represented as a flat carry-forward bar `O = H = L = C = previous_close`, `V = 0`).

### Broadcast Conflation (Throttling)
While trade execution and account settlement must process every event with zero loss, **public market data broadcasts** (Order Book snapshots and live-forming candle updates) may be batched/throttled (e.g., every 100–250ms) so high-frequency trading spikes do not overwhelm connected clients.


## 8. Liquidity & Market Bots

### Liquidity
- **Liquidity** measures how easily a trader can buy or sell shares without causing large price jumps (slippage).
- A healthy simulated market maintains a **tight Bid-Ask spread** and **sufficient depth** on both sides of the Order Book, while also producing steady trade execution so charts and feeds feel alive even with few human users online.

### First-Class Citizen Rule
- **Same Engine & Accounting:** Market Bots are regular accounts (`is_bot = true`) seeded with finite initial cash and share inventories. Every bot order passes through the exact same Matching Engine, Order Book, and Cash/Share Reservation rules as human traders.
- **Why:** Preserves the **Zero-Sum Conservation** invariant across the entire exchange (enabling exact end-to-end accounting proofs under load) and keeps supply/demand realistic.
- **Leaderboard Exclusion:** Bot accounts are excluded from human competition rankings and leaderboards.

### Bot Archetypes (Makers + Takers)
A realistic market requires **both** passive liquidity providers (Makers) and active spread-crossers (Takers)—passive Makers alone fill the Order Book but produce zero trades and frozen charts:
1. **Market Maker Bot (Maker — Liquidity Provider):**
   - Continuously maintains resting Bids below and resting Asks above the current reference/last price to keep the spread tight and the book deep.
   - **Inventory Skew:** Adjusts its quotes upward as its share inventory drops (and downward as it accumulates excess shares), organically reflecting supply and demand.
2. **Noise / Retail Trader Bot (Taker — Activity Generator):**
   - Periodically submits small Market or crossing Limit orders around the current price so trades execute continuously, keeping the Tape, Volume, and Candlesticks active.
3. **Trend-Follower & Value Anchor Bots (Waves + Guardrails):**
   - **Trend-Follower (Momentum):** Buys into short-term upward price moves and sells into downward moves, creating realistic market swings.
   - **Mean-Reversion / Value Bot:** Tracks a slow-drifting fair-value anchor—buying when market price drops far below fair value and selling when it spikes far above—preventing runaway crashes to `$0.01` or infinite bubbles during unattended demo runs.
