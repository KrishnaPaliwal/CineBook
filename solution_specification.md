# CineBook: Solution Specification

## 1. Executive Summary

CineBook is a **cloud-native, multi-tenant movie ticketing platform** built with a microservices architecture. It enables moviegoers to browse films, select seats, complete payments, and receive booking confirmations. Theater administrators can manage cinema halls, screens, shows, and pricing through the same platform.

The system is designed around **Domain-Driven Design (DDD)**, **CQRS + Event Sourcing**, and **Saga Orchestration** patterns to ensure consistency, scalability, and resilience in a distributed environment.

---

## 2. Business Requirements

| # | Requirement |
|---|---|
| BR-01 | Users can register, verify via OTP, and log in |
| BR-02 | Users can browse movies and shows by cinema and location |
| BR-03 | Users can select available seats for a show |
| BR-04 | Users can complete a payment via Razorpay to confirm a booking |
| BR-05 | Users receive email and SMS notifications for booking created, confirmed, and cancelled |
| BR-06 | Theater admins can manage movies, cinemas, screens, and shows |
| BR-07 | The platform supports multiple tenants (theater organizations) |
| BR-08 | All booking transactions must be consistent — either fully confirmed or fully rolled back |
| BR-09 | Duplicate notifications must be prevented |
| BR-10 | System must be observable — traces, metrics, and logs must be centralized |

---

## 3. System Actors

| Actor | Description |
|---|---|
| **Moviegoer** | Registers, browses shows, books seats, completes payment |
| **Theater Admin** | Manages movies, cinemas, screens, shows, and pricing |
| **Keycloak** | OIDC Identity Provider — issues and validates JWTs |
| **Razorpay** | External payment gateway — processes card charges, sends webhooks |
| **Notification Gateway** | External SMTP server and SMS provider |

---

## 4. Microservices

### 4.1 Service Registry

| Service | Port | Primary Responsibility |
|---|---|---|
| `user-management-service` | 8086 | Identity, authentication, user profiles, organizations |
| `cinema-service` | 8083 | Movies, cinemas, screens, shows |
| `booking-service` | 8085 | Bookings, seat management, saga orchestration |
| `payment-service` | 7085 | Payment processing, Razorpay webhook handling |
| `notification-service` | 7083 | Email and SMS delivery |
| `location-service` | 7086 | Geo-location lookups |
| `cinebook-frontend` | — | React SPA (runs on host via `npm run dev`) |

### 4.2 Shared Libraries

| Module | Contents |
|---|---|
| `cinebook-core-common` | `TenantContext`, `TenantContextFilter`, `TenantAwareDataSource`, `MDCFilter` |
| `cinebook-common-messaging` | Avro schemas (`BookingCreatedEvent`, `BookingConfirmedEvent`, `BookingCancelledEvent`, `UserLifecycleEvent`) and shared saga DTOs (`BookingSagaCommand`, `BookingSagaEvent`) |

---

## 5. Architecture Decisions

### 5.1 CQRS + Event Sourcing (booking-service)

**Decision**: Use Axon Framework to implement CQRS and Event Sourcing on the `Booking` aggregate.

**Rationale**:
- Booking state changes (created → confirmed → cancelled) form a natural event stream
- Event sourcing provides a full audit log of all booking transitions at no extra cost
- CQRS separates the write model (aggregate) from the read model (projection), allowing independent scaling and optimized queries

**Implementation**:
- Write side: `BookingAggregate` receives `CreateBookingCommand`, emits `BookingCreatedEvent` stored in Axon's `domain_event_entry` table
- Read side: `BookingProjection` (`@EventHandler`) builds the `BookingRead` table and publishes events to Kafka

### 5.2 Saga Orchestration with Temporal (embedded in booking-service)

**Decision**: Implement the booking saga using **Temporal** as the workflow engine, embedded inside `booking-service` — no separate orchestrator service.

**Rationale**:
- Temporal provides durable, fault-tolerant workflow execution with built-in retry, timeout, and compensation support
- Embedding the saga in `booking-service` avoids the operational overhead of a separate microservice for orchestration logic that is tightly coupled to booking semantics
- Signal-based communication over Kafka allows loose coupling with participant services

**Saga Steps (happy path)**:
1. Reserve seats → wait for `seatReservationResult` signal (30s timeout)
2. Charge payment → wait for `paymentResult` signal (90s timeout)
3. Confirm booking → send to Axon
4. Send notification

**Compensation (on failure)**:
- If seat reservation fails/times out → `releaseSeat` + `markBookingFailed`
- If payment fails/times out → `releaseSeat` + `markBookingFailed`

### 5.3 Event-Driven Communication via Kafka

**Decision**: Use Apache Kafka as the primary inter-service communication bus, with Avro serialization for domain events and JSON for saga commands/events.

**Rationale**:
- Kafka provides at-least-once delivery, replay capability, and decoupled service communication
- Avro + Apicurio Schema Registry enforces schema contracts for `booking-events` topic
- JSON used for saga command/event topics for simpler parsing without schema registry coupling

**Topic Ownership**:

| Topic | Format | Producer | Consumers |
|---|---|---|---|
| `booking-workflow-start` | JSON | booking-service | booking-service (saga) |
| `seat-commands` | JSON | booking-service (saga) | booking-service (listener) |
| `seat-events` | JSON | booking-service (listener) | booking-service (saga signal) |
| `payment-commands` | JSON | booking-service (saga) | payment-service |
| `payment-events` | JSON | payment-service | booking-service (saga), notification-service |
| `booking-commands` | JSON | booking-service (saga) | booking-service (listener) |
| `booking-events` | **Avro** | booking-service (projection) | notification-service |
| `notification-commands` | JSON | booking-service (saga) | notification-service |
| `otp-notifications` | JSON | user-management-service | notification-service |
| `password-reset-notifications` | JSON | user-management-service | notification-service |
| `booking-failure-events` | JSON | booking-service (saga) | — |

### 5.4 Multi-Tenancy

**Decision**: Implement multi-tenancy using a shared database with tenant-scoped data isolation via `tenant_id` column and `TenantContext` thread-local.

**Implementation**:
- `TenantContextFilter` extracts `tenant-id` / `x-tenant-id` from HTTP headers and stores in `TenantContext`
- `TenantAwareDataSource` routes DB queries to tenant-specific schema/connection
- All Kafka saga messages carry `tenantId` field and `tenant_id` Kafka header
- `TenantContext.setTenantId()` / `TenantContext.clear()` used around all Kafka consumer handlers

### 5.5 Identity & Security

**Decision**: Use **Keycloak 24** as the OIDC provider. All services validate JWTs locally via Spring OAuth2 Resource Server.

**Implementation**:
- Keycloak realm: `cinebook`
- JWT issuer: `http://host.docker.internal:8080/realms/cinebook`
- JWK Set URI: `http://host.docker.internal:8080/realms/cinebook/protocol/openid-connect/certs`
- Every service has `SecurityConfig` with `spring.security.oauth2.resourceserver.jwt.*`
- `user-management-service` manages Keycloak users via Admin REST API (`KeycloakAdminClientService`)
- OTP verification uses an embedded Temporal saga (`UserRegistrationWorkflow`)

### 5.6 Outbox Pattern (payment-service)

**Decision**: Use the Transactional Outbox Pattern in `payment-service` for reliable Kafka event publishing.

**Rationale**: Prevents scenarios where payment DB is updated but Kafka publish fails (or vice versa).

**Implementation**:
- `PaymentEventPublisher` writes events to `payment_outbox_event` table within the same DB transaction
- `PaymentOutboxScheduler` polls the outbox on a schedule and publishes pending events to Kafka
- Outbox records are marked as sent after successful publish

### 5.7 Idempotency (notification-service)

**Decision**: Deduplicate all notification events in `notification-service` using a `SentNotification` audit table.

**Implementation**:
- Every consumed event is assigned a `deduplicationId` (from Kafka header `event_id` or a deterministic composite key)
- Before processing, check if `event_id` exists in `sent_notification` table with status `SUCCESS`
- If duplicate → skip; if retry (FAILED/PENDING) → reprocess

### 5.8 Resilience (booking-service → cinema-service)

**Decision**: Use **Resilience4j Circuit Breaker** on the Feign client call from `booking-service` to `cinema-service`.

**Implementation**:
- `spring-cloud-starter-circuitbreaker-resilience4j` with `spring-cloud-starter-openfeign`
- `CinemaServiceDelegate` (Feign client) has circuit breaker configured
- Tests: `CinemaServiceFeignCircuitBreakerTest`

---

## 6. Technology Stack

### Backend

| Technology | Version | Usage |
|---|---|---|
| Java | 21 (with `--enable-preview`) | All services |
| Spring Boot | 3.3.2 | Application framework |
| Spring Cloud | 2023.0.2 | OpenFeign, Circuit Breaker, Stream |
| Axon Framework | 4.9.3 | CQRS + Event Sourcing (booking-service) |
| Temporal SDK | 1.31.0 | Saga workflow engine |
| Apache Kafka | Confluent 7.4.0 | Event bus |
| Apache Avro | 1.11.3 | Schema-based serialization |
| Confluent Kafka Avro Serializer | 7.5.0 | Avro Serde for Kafka |
| Apicurio Schema Registry | 2.4.14 | Avro schema registry |
| Keycloak | 24.0.5 | OIDC Identity Provider |
| PostgreSQL | 15 | Persistent storage |
| Redis | Alpine | Session cache, seat locking |
| Flyway | Spring Boot managed | DB schema migrations |
| Resilience4j | 2.2.0 | Circuit breaker |
| Bucket4j | 8.3.0 | Rate limiting |
| SpringDoc OpenAPI | 2.6.0 | API documentation (Swagger UI) |
| Jackson | 2.15.2 | JSON serialization |
| Logstash Logback Encoder | 7.4 | Structured JSON logging |

### Observability

| Technology | Version | Usage |
|---|---|---|
| OpenTelemetry SDK | 2.28.1 | Traces + metrics instrumentation |
| Micrometer Prometheus | Spring Boot managed | Metrics export |
| Micrometer Tracing OTel Bridge | Spring Boot managed | Trace bridging |
| OpenTelemetry Collector | Latest | Telemetry pipeline |
| Prometheus | Latest | Metrics storage |
| Grafana | Latest | Metrics dashboards |
| OpenSearch | 2.11.0 | Log aggregation |
| OpenSearch Dashboards | 2.11.0 | Log visualization |

### Infrastructure & Deployment

| Technology | Usage |
|---|---|
| Docker | Container runtime |
| Docker Desktop Kubernetes | Local Kubernetes cluster |
| Helm | Kubernetes deployment (shared chart `cinebook-microservice`) |
| Docker Compose | Infrastructure stack (Kafka, Postgres, Keycloak, etc.) |
| Axon Server | 4.5.15 — Event store + command/query routing |
| Temporal | `temporalio/auto-setup` — Workflow engine |

---

## 7. Data Architecture

### Databases

Each service has its own PostgreSQL database (per microservice database pattern):

| Service | Database |
|---|---|
| user-management-service | `cinebook_user_management_db` |
| cinema-service | `cinebook_cinema_db` |
| booking-service | `cinebook_booking_db` |
| payment-service | `cinebook_payment_db` |

### Key Domain Entities

**booking-service**
- `Booking` — Axon aggregate (write model)
- `BookingRead` — CQRS read model (projection)
- `Seat` — seat per show, with status (available/reserved/booked)
- `PricingTier` — pricing rules per screen/category

**cinema-service**
- `Movie` — movie catalog
- `Cinema` — cinema hall with location
- `Screen` — screen inside a cinema with capacity
- `Show` — scheduled screening (movie × screen × time)

**user-management-service**
- `User` — user profile linked to Keycloak
- `Organization` — tenant/organization entity
- `OrganizationInvite` — pending member invitations

**payment-service**
- `Payment` — payment record (amount, status, Razorpay IDs)
- `PaymentOutboxEvent` — transactional outbox for Kafka publishing

**notification-service**
- `SentNotification` — notification audit + deduplication log

---

## 8. API Design

All services expose **RESTful JSON APIs** with:
- **OpenAPI 3.0** documentation via SpringDoc (`/swagger-ui.html`)
- **JWT Bearer token** authentication on all protected endpoints
- **`tenant-id`** / **`x-tenant-id`** header for multi-tenant context
- **`/actuator/health`** endpoint on every service (used as Kubernetes liveness probe)

### Endpoint Summary

| Service | Base Path | Key Operations |
|---|---|---|
| user-management | `/api/v1/usermanagement` | register, verify-otp, login, refresh, forgot-password, reset-password |
| user-management | `/api/v1/users` | get/update user profile |
| user-management | `/api/v1/organizations` | CRUD organizations, invite members |
| cinema | `/api/v1/movies` | CRUD movies |
| cinema | `/api/v1/cinemas` | CRUD cinemas |
| cinema | `/api/v1/screens` | CRUD screens |
| cinema | `/api/v1/shows` | CRUD shows |
| booking | `/api/v1/bookings` | create booking, get booking |
| booking | `/api/v1/seats` | get seats, init seats |
| booking | `/api/v1/pricing` | get pricing tiers |
| payment | `/api/v1/payments` | initiate payment, webhook, get status |
| location | `/api/v1/locations` | geo location lookup |

---

## 9. Cross-Cutting Concerns

### Logging
- All services use **Logstash Logback Encoder** for structured JSON log output
- Logs are shipped to **OpenSearch** via **OpenTelemetry Collector**
- **MDCFilter** (`cinebook-core-common`) injects `tenant_id` and `correlation_id` into MDC for every request

### Distributed Tracing
- All services instrument with **OpenTelemetry** (`opentelemetry-spring-boot-starter`)
- Traces exported to **OpenTelemetry Collector** (gRPC `:4317`)
- Collector forwards to OpenSearch and Prometheus
- Trace context propagated via standard W3C `traceparent` headers and Kafka headers

### Metrics
- **Micrometer** with Prometheus registry exposes `/actuator/prometheus` on each service
- **Prometheus** scrapes all services
- **Grafana** provides dashboards

### Rate Limiting
- **Bucket4j** (`8.3.0`) implemented in `RateLimitingInterceptor` on `booking-service`, `cinema-service`, `payment-service`, `user-management-service`

### Database Migrations
- **Flyway** manages schema migrations in `booking-service` (and others where configured)

---

## 10. Deployment Architecture

### Local Development

```
Developer Machine
├── Docker Desktop Kubernetes (microservices via Helm)
│   ├── user-management-service  ClusterIP :8086
│   ├── cinema-service           ClusterIP :8083
│   ├── booking-service          ClusterIP :8085
│   ├── payment-service          ClusterIP :7085
│   ├── notification-service     ClusterIP :7083
│   └── location-service         ClusterIP :7086
│
├── Docker Compose (infrastructure — host.docker.internal)
│   ├── PostgreSQL          :5432
│   ├── Kafka               :9092
│   ├── Apicurio            :8081
│   ├── Temporal            :7233
│   ├── Axon Server         :8024 / :8124
│   ├── Keycloak            :8080
│   ├── Redis               :6379
│   ├── OTel Collector      :4317 / :4318
│   ├── Prometheus          :9090
│   ├── Grafana             :3000
│   ├── OpenSearch          :9200
│   └── OpenSearch Dashboards :5601
│
└── Frontend (npm run dev — talks to services via kubectl port-forward)
```

### Helm Chart

- Single shared chart: `cinebook-infra/helm/cinebook-microservice/`
- Per-service config: `values-{service}-local.yaml`
- Deploys: `Deployment` + `ClusterIP Service` + `ServiceAccount` per service
- Liveness probe: `GET /actuator/health` with 120s initial delay

---

## 11. Non-Functional Requirements

| NFR | Implementation |
|---|---|
| **Consistency** | Temporal saga with compensation ensures distributed transaction consistency |
| **Idempotency** | Notification deduplication via `SentNotification.eventId`; Payment outbox for at-least-once delivery |
| **Resilience** | Circuit breaker (Resilience4j) on Feign calls; Temporal retry policies on workflow activities |
| **Observability** | Full OTel tracing, Prometheus metrics, structured JSON logs → OpenSearch |
| **Security** | JWT validation on every service via Keycloak; tenant isolation via TenantContext |
| **Multi-Tenancy** | `tenant_id` column on all entities; TenantContextFilter on all HTTP requests; tenant headers on all Kafka messages |
| **Rate Limiting** | Bucket4j on API endpoints across services |
| **Schema Evolution** | Avro schemas managed via Apicurio Schema Registry for domain events |
| **Testability** | Payment `auto-approve` flag; WireMock for Feign client tests; H2 in-memory DB for unit tests |
