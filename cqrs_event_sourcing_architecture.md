# CQRS & Event Sourcing Architecture in CineBook

This document provides a comprehensive technical breakdown of the **Command Query Responsibility Segregation (CQRS)** and **Event Sourcing** architectural pattern implemented in CineBook's `booking-service` using **Axon Framework 4.x** and **PostgreSQL**.

---

## 1. Executive Summary & Core Concepts

Traditional database architectures rely on CRUD (Create, Read, Update, Delete) operations, where the database table state is directly mutated (e.g., `UPDATE bookings SET status = 'CONFIRMED'`). In high-concurrency event-driven systems like CineBook, direct mutations lead to database row locking, contention during flash ticket sales, and loss of historical audit trails.

To solve this, CineBook decouples **Write Operations (Commands)** from **Read Operations (Queries)** using CQRS and Event Sourcing:

* **Event Sourcing**: State changes are stored as an immutable sequence of domain events rather than mutating rows in a table.
* **CQRS**: The write pipeline processes transactional commands and records events, while an asynchronous projection engine updates a read-optimized schema for UI queries.

### 💡 Real-World Analogy: The Bank Passbook
* **Traditional CRUD:** Imagine a bank account where every transaction overwrites a single column `balance = $500`. You lose the historical record of how that balance was reached.
* **Event Sourcing (`domain_event_entry`):** Instead of storing only the current balance, you store an immutable ledger of every transaction:
  1. `AccountCreated`
  2. `MoneyDeposited ($1,000)`
  3. `MoneyWithdrawn ($500)`

In CineBook, **`domain_event_entry`** acts as that immutable ledger, capturing every event (`BookingCreatedEvent`, `BookingConfirmedEvent`, `BookingCancelledEvent`) that ever occurred.

---

## 2. Architectural Overview: Dual-Schema Model

CineBook separates concerns at the database level by utilizing two distinct PostgreSQL schemas within `cinebook_booking_db`:

| Schema | Role | Key Tables | Purpose & Operations |
| :--- | :--- | :--- | :--- |
| **`public`** | **Write Model** (Command Side) | `domain_event_entry`<br/>`token_entry`<br/>`saga_entry`<br/>`seats` | Handles command validation (seat locking, price calculation) and stores immutable domain events. |
| **`booking_read`** | **Read Model** (Query Side) | `booking_read.booking` | Dedicated exclusively to ultra-fast read queries (User Booking History, Receipt Views) without row locking. |

### ❓ Why `public.bookings` is Empty
`public.bookings` was an early legacy CRUD table created during initial setup. Under full Event Sourcing, state is maintained in `domain_event_entry` (Write) and projected into `booking_read.booking` (Read). The legacy `public.bookings` table is not written to and can safely remain empty.

---

## 3. End-to-End CQRS & Event Sourcing Data Flow

```
                            [ USER ACTION ]
                                   │
                      (HTTP POST /api/v1/bookings)
                                   │
                                   ▼
        ┌─────────────────────────────────────────────────────┐
        │  WRITE SIDE (Command Pipeline)                      │
        │  1. Validates seats and lock status                 │
        │  2. BookingAggregate emits BookingCreatedEvent      │
        │  3. Appends event to domain_event_entry (EventStore)│
        └──────────────────────────┬──────────────────────────┘
                                   │
             (Axon TrackingEventProcessor in Background)
                                   │
                                   ▼
        ┌─────────────────────────────────────────────────────┐
        │  READ SIDE (Projection Pipeline)                    │
        │  1. BookingProjection.java receives event           │
        │  2. Binds tenant context & SET ROLE cinebook_app    │
        │  3. Writes formatted record to booking_read.booking │
        └──────────────────────────┬──────────────────────────┘
                                   │
                  (HTTP GET /api/v1/bookings/history)
                                   │
                                   ▼
                       [ USER VIEWS HISTORY ]
```

### Detailed Pipeline Mechanics

#### Step 1: Write Operations (Command Handling)
1. The user initiates a booking (`POST /api/v1/bookings`).
2. `BookingService` verifies seat locks in `seats` table and calculates total pricing.
3. `BookingService` dispatches a `CreateBookingCommand` via Axon's `CommandGateway`.
4. `BookingAggregate` receives the command, validates business constraints, and executes `AggregateLifecycle.apply(new BookingCreatedEvent(...))`.
5. Axon Framework stores the event in `public.domain_event_entry`.

#### Step 2: Event Storage Infrastructure
Axon Framework relies on the following infrastructure tables in the `public` schema:
* `domain_event_entry`: Stores domain event payloads, metadata, aggregate identifiers, and sequence numbers.
* `snapshot_event_entry`: Stores aggregate state snapshots for fast rehydration.
* `token_entry`: Tracks the event processing position (segment tokens) of asynchronous background processors.
* `saga_entry`: Manages long-running saga workflows.

> [!NOTE]
> In Spring Boot 3 / Hibernate 6 for PostgreSQL, `@Lob` columns (`meta_data`, `payload`, `token`, `serialized_saga`) map to PostgreSQL `OID` (Large Object references). Tables must be created using `OID` types rather than `BYTEA` to prevent SQL expression type mismatch errors (`SQLState 42804`).

#### Step 3: Asynchronous Projection Engine
1. Axon's `TrackingEventProcessor` runs in the background within `booking-service`.
2. It fetches progress tokens from `public.token_entry` and reads unprocessed events from `public.domain_event_entry`.
3. `BookingProjection.java` receives the event (e.g., `@EventHandler public void on(BookingCreatedEvent event)`).
4. `BookingProjection` initializes the tenant session context (`SET ROLE cinebook_app` and `set_config('app.tenant_id', ...)`).
5. It saves or updates the read entity into **`booking_read.booking`**.
6. If the event is a confirmation (`BookingConfirmedEvent`), it updates the status in `booking_read.booking` to `CONFIRMED` and sets seat status to `BOOKED`.

#### Step 4: Read Operations (Queries)
1. When the UI requests user booking history (`GET /api/v1/bookings/history`), `BookingController` delegates directly to `BookingReadRepository`.
2. The query reads pre-aggregated records from `booking_read.booking`.
3. The response is returned instantly without joining transactional write tables or calculating aggregate state from scratch.

---

## 4. Enterprise Architecture Advantages

1. **Zero Database Row Contention**:
   High-volume user read queries (millions of users browsing booking histories) hit `booking_read.booking` without acquiring locks on `seats` or transactional write tables needed for live seat booking.

2. **Complete Auditability & Replayability**:
   `domain_event_entry` contains an immutable record of every state transition. If a projection model changes or data gets corrupted, projections can be completely rebuilt by replaying the event stream from index 0.

3. **Multi-Tenant Row-Level Security (RLS)**:
   Both `public` and `booking_read` schemas enforce PostgreSQL Row-Level Security (`tenant_id = current_setting('app.tenant_id', true)`), guaranteeing strict isolation across cinema tenants.

4. **Independent Scalability**:
   The read model can be scaled, cached (e.g., Redis), or migrated to a different database engine independently of the command write model.
