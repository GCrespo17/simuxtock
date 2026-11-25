# SimuXTock — Cross-Cutting Concerns

This document defines how observability, event tracing, structured logging, and audit persistence are implemented across all Bounded Contexts without compromising Domain purity or hot-path matching performance.

---

## 1. Core Architectural Principles

* **Domain Purity:** Domain Aggregates (`OrderBook`, `Account`, `Trader`), Entities, Value Objects, and Domain Services (`MatchingEngine`, `SettlementService`) never import loggers, never call system clocks (`Date.now()`), and never attach tracing metadata to Domain Events.
* **Determinism for WAL Replay:** All non-deterministic inputs (timestamps, correlation IDs, actor identity) are captured **once** at the system boundary inside an `ExecutionContext` and persisted in the Write-Ahead Log (WAL). Replaying the WAL feeds the exact recorded context back into the application layer.
* **Non-Blocking Hot Path:** Logging and database audit persistence are decoupled from the synchronous matching and settlement loop via the in-process Event Bus and asynchronous batching.

---

## 2. Deterministic Execution Context

Every mutating command (`PlaceOrder`, `CancelOrder`, `ModifyOrder`, `FundAccount`, `ResetAccount`) receives an `ExecutionContext` created at the HTTP/API or system entrypoint and recorded alongside the command in the WAL.

```ts
// shared/application/ExecutionContext.ts
export type ActorType = 'TRADER' | 'BOT' | 'SYSTEM';

export interface ExecutionContext {
  readonly correlationId: string; // Root request or command chain ID
  readonly actorId: string;       // AccountId / TraderId, or 'SYSTEM' for seeding/recovery
  readonly actorType: ActorType;
  readonly timestamp: number;     // UTC epoch ms captured once at the boundary
}
```

* **Live Execution:** Fastify captures `timestamp = Date.now()` and `correlationId = req.id` at the boundary and writes them into the WAL entry before executing the command.
* **Crash Recovery (WAL Replay):** The WAL replayer reconstructs the original `ExecutionContext` from each persisted WAL record, guaranteeing identical timestamps and causal metadata across replays.

---

## 3. Event Tracing: Pure `DomainEvent` vs. `EventEnvelope<T>`

To keep observability metadata completely out of the Domain layer, events are split into two distinct layers.

### 3.1 Domain Layer: Pure `DomainEvent`
Domain models emit plain, metadata-free domain facts. Unit tests assert directly on these objects without constructing fake correlation or tracing IDs.

```ts
// shared/domain/DomainEvent.ts
export interface DomainEvent {
  readonly eventType: string;
}

// Example: trading/domain/events/TradeExecuted.ts
export interface TradeExecuted extends DomainEvent {
  readonly eventType: 'TradeExecuted';
  readonly tradeId: string;
  readonly ticker: string;
  readonly price: number;
  readonly quantity: number;
  readonly aggressorSide: 'BUY' | 'SELL';
  readonly buyerAccountId: string;
  readonly sellerAccountId: string;
  readonly sequenceNumber: number;
}
```

### 3.2 Application Layer: `EventEnvelope<T>`
Before publishing to the in-process Event Bus, the Application layer wraps each pure `DomainEvent` in an `EventEnvelope<T>` stamped with `EventMetadata` derived from the `ExecutionContext`.

```ts
// shared/application/EventEnvelope.ts
import { DomainEvent } from '../domain/DomainEvent';
import { ActorType } from './ExecutionContext';

export interface EventMetadata {
  readonly eventId: string;       // Deterministic or unique ID for this specific event
  readonly occurredAt: number;    // Derived from ExecutionContext.timestamp (never Date.now())
  readonly actorId: string;       // Who triggered the action
  readonly actorType: ActorType;  // 'TRADER' | 'BOT' | 'SYSTEM'
  readonly correlationId: string; // Ties the entire causal chain together
  readonly causationId: string;   // Immediate command or event ID that caused this event
}

export interface EventEnvelope<T extends DomainEvent = DomainEvent> {
  readonly metadata: EventMetadata;
  readonly event: T;
}
```

### 3.3 Causal Chain Example
When `alice` submits a crossing Limit Buy (`cmd-901`, `correlationId: req-42`), every resulting event shares `correlationId` while `causationId` captures the exact parent-child link:

| `eventType` | `actor` | `correlationId` | `causationId` | `occurredAt` |
| :--- | :--- | :--- | :--- | :--- |
| `OrderOpened` | `TRADER:alice` | `req-42` | `cmd-901` | `1763949000100` |
| `TradeExecuted` | `TRADER:alice` | `req-42` | `evt-order-opened-1` | `1763949000100` |
| `AccountSettled` | `TRADER:alice` | `req-42` | `evt-trade-1` | `1763949000100` |
| `OrderFilled` | `TRADER:alice` | `req-42` | `evt-trade-1` | `1763949000100` |

---

## 4. Structured Logging

Structured JSON logging is powered by **Pino** (native to Fastify) and isolated behind a thin application port so domain code never depends on a logging library.

### 4.1 `Logger` Port
* Define a minimal `Logger` interface (`info`, `warn`, `error`, `child(bindings)`) in `shared/application/Logger.ts`.
* Provide a thin Pino adapter in Infrastructure for production/dev, and a no-op stub logger for unit tests and WAL replay.

### 4.2 Automated Event Tracing (`EventAuditLogger`)
* A single `EventAuditLogger` subscriber attaches to `*` on the in-process Event Bus.
* Every published `EventEnvelope` is logged as a single structured JSON record containing both `metadata` and `event`, giving 100% visibility across all Bounded Contexts without littering `MatchingEngine` or `SettlementService` with log statements.

### 4.3 Command Boundary Logging (Decorator / Proxy Pattern)
* Application Use Cases (`PlaceOrderHandler`, `CancelOrderHandler`, `ResetAccountHandler`, etc.) are wrapped in a `LoggingCommandHandler` decorator.
* Logs command start, execution duration (ms), `correlationId`, `actorId`, and structured rejection/error outcomes at the boundary.

### 4.4 Sensitive Data Redaction
Pino redaction rules must strictly strip sensitive fields before serialization:
* `password`
* `passwordHash`
* `authorization`
* `refreshToken`
* `refreshTokenHash`

### 4.5 WAL Replay Mode
During crash recovery, the WAL replayer injects a **no-op `Logger`** and disables `EventAuditLogger` so hundreds of thousands of replayed commands recover at full RAM speed without flooding stdout.

---

## 5. Persistent Event Audit Store (`audit_events`)

All domain events are persisted to an append-only `audit_events` table in PostgreSQL so historical traces and causal chains can be queried via SQL.

### 5.1 Drizzle ORM Schema

```ts
// shared/infrastructure/db/schema/auditEvents.ts
import { pgTable, text, bigint, jsonb, index } from 'drizzle-orm/pg-core';

export const auditEvents = pgTable(
  'audit_events',
  {
    eventId: text('event_id').primaryKey(),
    eventType: text('event_type').notNull(),
    occurredAt: bigint('occurred_at', { mode: 'number' }).notNull(),
    actorId: text('actor_id').notNull(),
    actorType: text('actor_type').notNull(), // 'TRADER' | 'BOT' | 'SYSTEM'
    correlationId: text('correlation_id').notNull(),
    causationId: text('causation_id').notNull(),
    payload: jsonb('payload').notNull(),     // Pure DomainEvent
  },
  (table) => [
    index('idx_audit_events_correlation').on(table.correlationId),
    index('idx_audit_events_actor_time').on(table.actorId, table.occurredAt),
    index('idx_audit_events_type_time').on(table.eventType, table.occurredAt),
  ]
);
```

### 5.2 Hot-Path Safety & Replay Idempotency
* **Asynchronous Batched Writes:** An `AuditEventPersister` subscriber listens to `*` on the Event Bus, buffers incoming `EventEnvelope`s in memory, and flushes them to PostgreSQL in batches (e.g., every `100` events or every `250ms`). Because the WAL already provides pre-acknowledgment durability, `audit_events` never sits on the synchronous critical path.
* **Replay-Safe Inserts:** All batch inserts use `ON CONFLICT (event_id) DO NOTHING` (`.onConflictDoNothing({ target: auditEvents.eventId })`). If the process crashes and replays the WAL, previously flushed events are safely skipped with zero duplicate-key errors.
* **Graceful Shutdown:** On `SIGINT`/`SIGTERM`, the persister flushes any remaining buffered events before closing the PostgreSQL pool.
