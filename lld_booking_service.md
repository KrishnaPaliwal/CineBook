# CineBook: Low-Level Design — booking-service

## Overview

`booking-service` is the most complex service in CineBook. It is responsible for:
- Accepting booking requests via REST API
- Managing seats and pricing
- Running the **Temporal saga orchestrator** (embedded — no separate service)
- Using **Axon Framework** for CQRS + Event Sourcing on the `Booking` aggregate

Port: **8085** (local Kubernetes, via port-forward)

---

## Package Structure

```
com.cinebook.bookingservice
├── config/
│   ├── KafkaAvroConfig.java          # Kafka Avro producer/consumer config
│   ├── RateLimitingInterceptor.java  # Rate limiting on API endpoints
│   ├── RedisConfig.java              # Redis connection config
│   ├── SecurityConfig.java           # Spring Security / OAuth2 Resource Server
│   ├── TemporalConfig.java           # Temporal WorkflowClient + Worker setup
│   ├── TenantDataSourceConfig.java   # Multi-tenant datasource routing
│   └── WebMvcConfig.java             # CORS, interceptors
│
├── controller/
│   ├── BookingController.java        # POST/GET /api/v1/bookings
│   ├── SeatController.java           # GET /api/v1/seats
│   └── PricingController.java        # GET /api/v1/pricing
│
├── dto/
│   ├── BookingRequestDTO.java
│   ├── BookingResponseDTO.java
│   ├── BookingNotification.java
│   ├── LockRequestDTO.java
│   ├── LockResponseDTO.java
│   ├── PricingTierDTO.java
│   ├── RowConfig.java
│   ├── SeatDTO.java
│   ├── SeatInitRequest.java
│   ├── UserRequestDTO.java
│   └── UserResponseDTO.java
│
├── exception/
│   ├── GlobalExceptionHandler.java
│   ├── SeatNotAvailableException.java
│   ├── SeatNotFoundException.java
│   └── SeatNotLockedException.java
│
├── listener/                         # Kafka consumers (saga participant role)
│   ├── BookingCommandListener.java   # Consumes 'booking-commands' → ConfirmBookingCommand to Axon
│   ├── BookingEventsListener.java    # Consumes 'booking-events' (Avro)
│   └── SeatCommandListener.java      # Consumes 'seat-commands' → reserve/release seats
│
├── messaging/
│   ├── KafkaAvroProducer.java        # Publishes Avro events to 'booking-events'
│   ├── SagaMessageParser.java        # Parses incoming Kafka saga messages
│   └── SagaMessageProducer.java      # Publishes saga command/event messages to Kafka
│
├── model/
│   ├── Booking.java                  # Axon aggregate root
│   ├── BookingRead.java              # CQRS read model (JPA entity)
│   ├── PricingTier.java
│   ├── Seat.java
│   └── User.java
│
├── projection/
│   └── BookingProjection.java        # Axon @EventHandler — builds read model, publishes to Kafka
│
├── repository/
│   ├── BookingReadRepository.java    # JPA repo for BookingRead
│   ├── BookingRepository.java        # JPA repo for Booking
│   ├── PricingTierRepository.java
│   └── SeatRepository.java
│
├── saga/
│   ├── activity/
│   │   └── BookingActivitiesImpl.java  # Temporal activities — publish Kafka commands
│   ├── listener/
│   │   ├── BookingWorkflowStartListener.java  # Kafka: booking-workflow-start → start Temporal workflow
│   │   └── ParticipantEventsListener.java     # Kafka: seat-events, payment-events → signal Temporal
│   ├── messaging/
│   │   └── SagaMessageCodec.java     # Encode/decode saga messages
│   └── workflow/
│       ├── BookingActivities.java    # Temporal activity interface
│       ├── BookingWorkflow.java      # Temporal workflow interface
│       └── BookingWorkflowImpl.java  # Temporal workflow — orchestrates saga steps
│
└── service/
    ├── BookingService.java           # Core booking business logic
    ├── CinemaServiceDelegate.java    # Feign client to cinema-service
    ├── PricingService.java
    └── SeatService.java              # Seat reservation/release logic
```

---

## CQRS / Event Sourcing (Axon)

### Command Side
- `BookingService` sends `CreateBookingCommand` via `CommandGateway`
- `BookingAggregate` (Axon `@Aggregate`) handles the command and emits `BookingCreatedEvent`
- Axon stores the event in `domain_event_entry` table in PostgreSQL

### Projection (Read Side)
- `BookingProjection` (`@EventHandler`) catches `BookingCreatedEvent`
- Writes `BookingRead` entity (read model) to `booking_read.booking` table
- Publishes Avro `BookingCreatedEvent` to Kafka topic `booking-events`
- Publishes `booking-workflow-start` string message to trigger the saga

---

## Embedded Saga (Temporal)

The Temporal worker is registered inside `booking-service` at startup. There is **no separate saga-orchestrator service**.

### Saga Flow

```
1. BookingWorkflowStartListener
   - Listens: Kafka topic 'booking-workflow-start'
   - Action: Calls WorkflowClient.start(BookingWorkflow::runBookingSaga, tenantId, bookingId, correlationId)

2. BookingWorkflowImpl.runBookingSaga()
   - Step 1: activities.reserveSeat() → publishes to 'seat-commands'
             Workflow.await(30s) for seatReservationResult signal
             On failure/timeout: releaseSeat() + markBookingFailed()
   - Step 2: activities.chargePayment() → publishes to 'payment-commands'
             Workflow.await(90s) for paymentResult signal
             On failure/timeout: releaseSeat() + markBookingFailed()
   - Step 3: activities.confirmBooking() → publishes to 'booking-commands'
   - Step 4: activities.sendNotification() → publishes to 'notification-commands'

3. ParticipantEventsListener
   - Listens: 'seat-events' → calls workflow.seatReservationResult(success)
   - Listens: 'payment-events' → calls workflow.paymentResult(success)
```

### Kafka Topics (booking-service produces/consumes)

| Topic | Role |
|---|---|
| `booking-workflow-start` | Produces (projection) / Consumes (saga start listener) |
| `seat-commands` | Produces (saga activity) / Consumes (SeatCommandListener) |
| `seat-events` | Produces (SeatCommandListener) / Consumes (saga signal) |
| `booking-commands` | Produces (saga activity) / Consumes (BookingCommandListener) |
| `booking-events` | Produces (projection, Avro) |
| `notification-commands` | Produces (saga activity) |
| `booking-failure-events` | Produces (saga, on failure) |

---

## REST API Endpoints

| Method | Path | Description |
|---|---|---|
| `POST` | `/api/v1/bookings` | Create a new booking |
| `GET` | `/api/v1/bookings/{id}` | Get booking by ID |
| `GET` | `/api/v1/seats` | Get seats for a show |
| `POST` | `/api/v1/seats/init` | Initialize seats for a screen |
| `GET` | `/api/v1/pricing` | Get pricing tiers |

---

## Database Schema

**PostgreSQL DB**: `cinebook_booking_db`

### Axon tables (auto-created)
- `domain_event_entry` — event store
- `snapshot_event_entry` — aggregate snapshots

### Application tables
- `booking` — Axon aggregate state
- `booking_read` — CQRS read model
- `seat` — seat data per show/screen
- `pricing_tier` — pricing configuration

---

## External Dependencies

| Dependency | Purpose |
|---|---|
| `cinema-service` | Feign client (`CinemaServiceDelegate`) for show/seat validation |
| Axon Server | Event store (gRPC `:8124`) |
| Temporal | Workflow engine (gRPC `:7233`) |
| Kafka | Event bus (`:9092`) |
| PostgreSQL | Persistence |
| Redis | (Config present — seat locking support) |
| Keycloak | JWT validation |
