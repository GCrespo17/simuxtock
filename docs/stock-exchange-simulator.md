# Stock Exchange Simulator with Paper Trading

## The pitch

A simulated stock market where users trade with fake money. Buy and sell orders go into a live order book, a matching engine pairs them into trades, prices move in real time, and every user sees their portfolio, trade history and profit or loss. Bots keep the market active so it always feels alive.

## The problem

People who want to learn trading either risk real money or use simplified simulators that just replay real prices, where your orders never affect anything. They never see how a market actually works: how orders wait in line, why prices move, what happens when many people buy at once.

From an engineering perspective, exchanges and brokers are some of the most demanding systems out there: trades must be processed in strict order, quickly, without ever losing or duplicating anything, while thousands of people watch prices change live.

## Who uses it

- **Traders (users):** place orders, watch the market, manage their portfolio.
- **Bots:** simulated traders with different strategies that provide activity and liquidity.
- **Admin:** creates tradable assets, sets market hours, pauses trading, resets competitions and monitors the system.

## Core features

1. **Accounts and starting balance:** each user gets fake cash to start trading.
2. **Tradable assets:** a set of fictional companies, each with its own order book.
3. **Order types:**
   - Market orders (buy or sell immediately at the best available price).
   - Limit orders (buy or sell only at a chosen price or better).
   - Cancelling or changing an order that hasn't been filled yet.
4. **Order book:** a live view of all waiting buy and sell orders at each price level.
5. **Matching engine:** pairs buy and sell orders by best price first, then by who arrived first, and turns matches into trades.
6. **Live market data:** last price, price chart, volume, and a feed of recent trades, updating in real time.
7. **Portfolio:** cash, holdings, average purchase price, current value and profit or loss.
8. **Trade history:** every order and trade the user made, with status (open, partly filled, filled, cancelled).
9. **Market bots:** automated traders with different behaviors (cautious, aggressive, trend-following) so the market moves even with few real users.
10. **Leaderboard and competitions:** rankings by portfolio return, with time-limited competitions that reset.

## The interesting problems to solve

- **Strict ordering.** Two orders arriving at almost the same time must be processed in a fair, consistent order. The matching engine can never process orders for the same asset in two places at once.
- **Speed vs. safety.** The engine should be fast, but every accepted order and trade must be safely recorded. Decide how to keep it fast without risking losing a trade if the system crashes.
- **Recovering after a crash.** If the system restarts, the order book must be rebuilt exactly as it was, with no lost or duplicated orders.
- **Money and holdings must always add up.** A user can't buy with cash they don't have or sell shares they don't own. Funds and shares are reserved when an order is placed and released when it's cancelled.
- **Partial fills.** One large order may be matched against several smaller ones at different prices. Every piece must update both sides correctly.
- **Live updates at scale.** Sending every single price change to every connected user can overwhelm the system. Decide when to send every update and when to group them.
- **Busy moments.** Market open, big bot activity or a competition start create spikes. The system must stay correct under load, even if it slows down.
- **Bots that behave believably.** Bots should create a realistic market without dominating it or making it trivially predictable.

## First version vs. later

**First version:**

- Accounts with fake starting cash
- A few fictional assets
- Market and limit orders, with cancellation
- Matching engine with best-price-then-first-come rules
- Live order book, last price and recent trades
- Portfolio and trade history
- A few simple bots

That's already a complete, demo-ready project.

**Later:**

- More order types (stop orders, orders that expire at end of day)
- Price charts with different time ranges
- Competitions with leaderboards and seasons
- Market hours, opening and closing, and trading pauses
- Fees per trade
- Admin dashboard with system health and activity
- Deposits and withdrawals through the payment processor project (see below)

## AI as the final layer (optional)

The platform must work fully without AI. Once it does, AI can add:

- Plain-language explanations of a user's trades ("you bought at the peak of a spike; the price dropped as bots took profit").
- An AI-driven bot as one more market participant, compared against the rule-based bots.
- A weekly portfolio summary in plain words.

## Connecting it with the payment processor

Users could fund their trading account by depositing through the payment processor simulator and withdraw their balance back. That turns the two projects into separate systems communicating through APIs and webhooks, like real companies do, and raises real questions: what happens if a deposit confirmation arrives twice, arrives late, or never arrives?

## How to make it stand out

- **Load-test the matching engine** and publish the results: how many orders per second it handles, and proof that every balance still adds up afterward.
- **Write tests for the hard parts:** partial fills, simultaneous orders, cancellations racing with matches, and crash recovery.
- **Document your trade-offs** in the README: how you guarantee ordering, how you recover after a crash, how you decided when to batch live updates.
- **Deploy it with bots running**, so anyone who opens the link sees a living market immediately.
