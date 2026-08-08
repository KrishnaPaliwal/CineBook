# CineBook Ticketing Platform: High-Level Design (HLD) Specification

This High-Level Design (HLD) document describes the macro-architectural structure, service containers, deployment topology, and integration data flows of the CineBook ticketing system.

---

## 1. C4 Model - Level 1: System Context

The System Context diagram illustrates the system boundaries of CineBook, detailing external actors and integrations.

```mermaid
graph TD
    %% Actors
    User([Moviegoer]) -->|Browses & Books| CineBook[CineBook System]
    Admin([Theater Admin]) -->|Manages Shows & Layouts| CineBook
    
    %% External Integrations
    CineBook -->|Authenticates Users| Keycloak[Keycloak IDP]
    CineBook -->|Charges Cards| Razorpay[Razorpay Payment Gateway]
    CineBook -->|Sends Alerts| SMTP[Mail / SMS Gateway]
```

---

## 2. C4 Model - Level 2: Container Diagram

The Container diagram decomposes the CineBook system into individual runtime containers, databases, and message brokers, showing their communication protocols.

```mermaid
graph TD
    %% Edge Layer
    User([Browser / Client]) -->|HTTPS| Gateway[API Gateway / Netty Proxy]
    User -->|OAuth2 OIDC Auth| Keycloak[Keycloak IDP]

    %% Gateway Routing Paths
    Gateway -->|REST / HTTP| Cinema[cinema-service]
    Gateway -->|REST / HTTP| Booking[booking-service]
    Gateway -->|REST / HTTP| UserMgmt[user-management-service]
    Gateway -->|REST / HTTP| Location[location-service]

    %% Internal Communication & Event Stores
    Booking <-->|gRPC| Axon[Axon Server Event Store]
    Booking -->|Kafka Avro| Kafka[Kafka Broker]
    Booking <-->|Signal / Start| Temporal[Temporal Workflow Engine]
    Booking <-->|Kafka Events| Kafka
    
    Payment[payment-service] <-->|Kafka Events| Kafka
    Notification[notification-service] <-->|Kafka Events| Kafka

    %% Storage Layer
    UserMgmt -->|Session Cache| Redis[(Redis Profile Cache)]
    UserMgmt -->|JPA| UserDB[(User PostgreSQL)]
    
    Cinema -->|JPA| CinemaDB[(Cinema PostgreSQL)]
    
    Location -->|JPA| LocationDB[(Location PostgreSQL)]
    Booking -->|JPA Projections| BookingDB[(Booking PostgreSQL)]
    Payment -->|JPA| PaymentDB[(Payment PostgreSQL)]
    Notification -->|JPA| NotificationDB[(Notification PostgreSQL)]
```

---

## 3. High-Level Checkout Integration Sequence

This sequence diagram illustrates how separate services coordinate using Temporal to fulfill a ticket purchase transaction.

```mermaid
sequenceDiagram
    autonumber
    actor User as Client Browser
    participant Gateway as API Gateway
    participant Booking as booking-service
    participant Temporal as Temporal Engine
    participant Cinema as cinema-service
    participant Payment as payment-service
    participant Razorpay as Razorpay

    User->>Gateway: POST /api/v1/bookings (Seats A1, A2)
    Gateway->>Booking: Forward Request (User JWT + Tenant Header)
    Booking->>Booking: Validate Seats in local Projection
    Booking->>Booking: Commit CreateBookingCommand to Axon Event Store
    Booking->>Kafka: Publish event to booking-workflow-start
    Booking-->>User: Return 201 Created (Booking ID: PENDING)
    
    Kafka->>Booking: BookingWorkflowStartListener consumes start event
    Booking->>Temporal: Start BookingWorkflow (Workflow ID = Booking ID)
    
    %% Temporal Step 1: Lock Seats
    Temporal->>Booking: Activity: reserveSeat (via Kafka: seat-commands)
    Booking->>Booking: SeatCommandListener reserves seats in DB
    Booking->>Temporal: Signal: seatReservationResult (via Kafka: seat-events)
    
    %% Temporal Step 2: Payment
    Temporal->>Payment: Activity: chargePayment (via Kafka: payment-commands)
    Payment->>Razorpay: Charge Card (Idempotent call)
    Razorpay-->>Payment: Success
    Payment->>Temporal: Signal: paymentResult (via Kafka: payment-events)
    
    %% Temporal Step 3: Confirmation
    Temporal->>Booking: Activity: confirmBooking (via Kafka: booking-commands)
    Booking->>Booking: Update Aggregate state to CONFIRMED
    Temporal->>Notification: Activity: sendNotification (via Kafka: notification-commands)
```

---

## 4. Infrastructure & Deployment Architecture

CineBook runs on a **local Docker Desktop Kubernetes cluster**. Microservices are deployed via Helm charts, while all infrastructure components (Kafka, PostgreSQL, Keycloak, Temporal, etc.) run in Docker Compose and are accessible to Kubernetes pods via `host.docker.internal`.

```mermaid
graph TD
    Frontend[Frontend - npm run dev] -->|port-forward| K8s

    subgraph Docker Desktop Kubernetes
        K8s[Kubernetes Cluster]
        K8s --> UMS[user-management-service :8086]
        K8s --> CS[cinema-service :8083]
        K8s --> BS[booking-service :8085]
        K8s --> PS[payment-service :7085]
        K8s --> NS[notification-service :7083]
        K8s --> LS[location-service :7086]
    end

    subgraph Docker Compose Infra - host.docker.internal
        PG[(PostgreSQL :5432)]
        Kafka[Kafka :9092]
        KC[Keycloak :8080]
        Temporal2[Temporal :7233]
        Axon[Axon Server :8124]
        Redis2[Redis :6379]
        Apicurio[Apicurio Schema Registry :8081]
    end

    K8s -->|host.docker.internal| PG
    K8s -->|host.docker.internal| Kafka
    K8s -->|host.docker.internal| KC
    K8s -->|host.docker.internal| Temporal2
    K8s -->|host.docker.internal| Axon
    K8s -->|host.docker.internal| Redis2
    K8s -->|host.docker.internal| Apicurio
```

---

## 5. Security & Boundary Architecture

1. **Identity Verification**:
   * **Keycloak** (v24) is the OIDC identity provider, realm: `cinebook`.
   * Every microservice validates JWTs via Spring OAuth2 Resource Server (`spring.security.oauth2.resourceserver.jwt.*`).
   * `tenant-id` and `x-tenant-id` headers are propagated on every request and Kafka message for multi-tenancy isolation.

2. **Internal Network**:
   * Microservices communicate internally via Kubernetes ClusterIP services.
   * Services access the infra stack (Kafka, Postgres, etc.) via `host.docker.internal`.

3. **Secrets**:
   * Database passwords are stored in Kubernetes Secret `db-credentials`.
   * Keycloak admin credentials are configured in Docker Compose environment variables.
