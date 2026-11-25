---
title: SimuXTock Glossary
status: ready
updated: 2026-11-23
---

# SimuXTock: Domain Glossary

### Account
A financial entity on the exchange holding cash and per-asset share balances partitioned into Available and Reserved buckets. Accounts belong to Traders, Market Bots, or the Exchange Treasury.
- **Avoid:** wallet, user profile
- **Related:** Available Balance, Reserved Balance, Exchange Treasury

### Aggressor Side
The side (`BUY` or `SELL`) of the incoming Taker order that crossed the Order Book and initiated a Trade.
- **Related:** Maker, Taker, Trade

### Ask
A resting Sell Limit Order on the Order Book representing an offer to sell shares at a specified minimum price.
- **Avoid:** offer
- **Related:** Best Ask, Bid, Bid-Ask Spread, Order Book

### Asset
A tradable company listed on the exchange, identified by a unique Ticker and traded in whole shares within its own isolated Order Book.
- **Avoid:** instrument, stock, security, symbol
- **Related:** Initial Reference Price, Order Book, Ticker

### Available Balance
The portion of an Account's cash or shares that is free to be committed to new orders.
- **Related:** Account, Reserved Balance

### Average Cost Basis
The weighted average purchase price per share of an Account's current holding in an Asset, recalculated on buys, unchanged on partial sells, and reset to zero when total shares reach zero.
- **Related:** Realized PnL, Unrealized PnL

### Best Ask
The lowest Ask price currently resting on an Asset's Order Book.
- **Related:** Ask, Best Bid, Bid-Ask Spread

### Best Bid
The highest Bid price currently resting on an Asset's Order Book.
- **Related:** Bid, Best Ask, Bid-Ask Spread

### Bid
A resting Buy Limit Order on the Order Book representing an offer to buy shares at a specified maximum price.
- **Related:** Ask, Best Bid, Bid-Ask Spread, Order Book

### Bid-Ask Spread
The difference between the Best Ask and the Best Bid (`Best Ask - Best Bid`). Always strictly positive in a resting Order Book.
- **Related:** Best Ask, Best Bid, Order Book

### Candlestick (OHLCV)
A time-bucketed summary (`1m`, `5m`, or `1h`) of executed Trades for an Asset, recording Open, High, Low, Close prices and total Volume. Intervals with zero trades emit a flat carry-forward bar using the previous close and zero volume.
- **Related:** Last Traded Price, Trade, Volume

### Exchange Treasury
A dedicated, non-trading Account seeded with initial funds that transfers starting cash to newly registered Traders and absorbs/re-credits balances during off-book Account Resets to preserve Zero-Sum Conservation.
- **Related:** Account, Zero-Sum Conservation

### Initial Reference Price
The starting anchor price assigned to an Asset at initial seeding, used for portfolio valuation and Market Bot quoting prior to the Asset's first executed Trade.
- **Related:** Asset, Last Traded Price

### Last Traded Price
The execution price of the most recent Trade for an Asset. Unmatched resting orders never alter the Last Traded Price.
- **Avoid:** mark price, mid price
- **Related:** Initial Reference Price, Trade

### Limit Order
An order to buy or sell a whole-share quantity at a specified limit price or better. Any unfilled portion rests on the Order Book until filled or cancelled.
- **Related:** Market Order, Order Book, Price Improvement

### Maker
The resting order already sitting on the Order Book when an incoming order matches against it. Every Trade executes at the Maker's price.
- **Related:** Aggressor Side, Price Improvement, Taker

### Market Bot
An automated first-class Account (`is_bot = true`) operating one of four strategies (Market Maker, Noise Trader, Trend-Follower, Mean-Reversion) under the exact same matching and accounting rules as human Traders.
- **Related:** Account, Trader

### Market Order
An order to buy or sell immediately against the best available resting liquidity on the opposite side of the Order Book. Market Orders never rest on the book and auto-cancel any unfilled remainder.
- **Related:** Limit Order, Order Book

### Matching Engine
The deterministic core component that sequences incoming commands per Asset, enforces price-time priority (FIFO), executes Trades, and enforces Self-Trade Prevention.
- **Related:** Order Book, Price-Time Priority, Self-Trade Prevention, Sequence Number

### Order Book
The per-Asset ledger of all active resting Bids (ordered highest price first) and Asks (ordered lowest price first), grouped into Price Levels.
- **Related:** Ask, Asset, Bid, Price Level

### Price Improvement
The cash savings a Limit Buyer receives when their incoming order executes against a resting Ask priced lower than their limit price, immediately refunded from Reserved Cash to Available Cash.
- **Related:** Limit Order, Maker, Reserved Balance

### Price Level
An aggregation of all resting orders on one side of an Order Book sharing the exact same price, queued in First-In, First-Out (FIFO) arrival order.
- **Related:** Order Book, Price-Time Priority

### Price-Time Priority (FIFO)
The matching rule that fills better prices first (lowest Ask for Buys, highest Bid for Sells) and breaks ties at the same Price Level strictly by arrival order.
- **Related:** Matching Engine, Price Level

### Realized PnL
The cumulative profit or loss locked in permanently from completed sell Trades, calculated as `(sell_price - average_cost_basis) × sold_qty`.
- **Related:** Average Cost Basis, Total Equity, Unrealized PnL

### Recent Trades Feed (Tape)
The chronological stream of executed Trades for an Asset, showing execution price, quantity, timestamp, and Aggressor Side.
- **Related:** Aggressor Side, Trade

### Reserved Balance
The portion of an Account's cash or shares currently locked to back active resting orders, preventing double-spending and short-selling.
- **Related:** Account, Available Balance

### Self-Trade Prevention (Cancel Resting)
The rule preventing an Account from trading against its own resting order by automatically cancelling the resting order (and releasing its reservation) before continuing to match.
- **Related:** Matching Engine, Order Book

### Sequence Number
A monotonically increasing integer assigned to every accepted command per Asset, defining the single canonical order of execution and crash recovery.
- **Related:** Matching Engine

### Taker
The incoming order that crosses the Bid-Ask Spread and immediately matches against one or more resting Maker orders.
- **Related:** Aggressor Side, Maker

### Ticker
The short, unique uppercase identifier representing a single Asset on the exchange.
- **Avoid:** symbol, code, stock ID
- **Related:** Asset
- **Example:** `AAPL`, `NVDA`, `NOVA`, `ACME`

### Total Equity
An Account's total simulated net worth, equal to `total_cash + Holdings Value`.
- **Related:** Realized PnL, Unrealized PnL

### Trade
An immutable record of an executed match between one resting Maker order and one incoming Taker order for a specific quantity at the Maker's price.
- **Avoid:** fill, execution
- **Related:** Aggressor Side, Last Traded Price, Maker, Taker, Volume

### Trader
An authenticated human user who submits, modifies, and cancels orders and tracks personal portfolio performance.
- **Related:** Account, Market Bot

### Unrealized PnL
The paper profit or loss on currently held shares, calculated as `(current_price - average_cost_basis) × total_shares` using the Last Traded Price (or Initial Reference Price before the first trade).
- **Related:** Average Cost Basis, Realized PnL, Total Equity

### Volume
The total number of shares traded for an Asset over a given period, counted once per Trade.
- **Related:** Candlestick (OHLCV), Trade

### Zero-Sum Conservation
The invariant that total cash and total shares per Asset summed across all Accounts (Traders, Market Bots, and the Exchange Treasury) remain strictly constant across all trades, cancellations, and account resets.
- **Related:** Account, Exchange Treasury
