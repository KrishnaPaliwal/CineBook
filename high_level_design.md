# CineBook Ticketing Platform: High-Level Design (HLD) Specification

This High-Level Design (HLD) document describes the macro-architectural structure, service containers, deployment topology, and integration data flows of the CineBook ticketing system.

---

## 1. C4 Model - Level 1: System Context Architecture Diagram

The System Context diagram illustrates the system boundaries of CineBook, detailing external actors and integrations.

```mermaid
graph TD
    %% Custom Styling
    classDef actorStyle fill:#2A3B50,stroke:#8FA1B7,stroke-width:2px,color:#FFFFFF;
    classDef systemStyle fill:#1E6091,stroke:#184E77,stroke-width:3px,color:#FFFFFF;
    classDef extStyle fill:#4A5568,stroke:#2D3748,stroke-width:2px,color:#FFFFFF;

    %% Actors
    User["👤 Moviegoer<br/>(Person)<br/>Browses shows, books seats, pays for tickets, and views history."]:::actorStyle
    TheaterAdmin["👤 Theater Admin<br/>(Person)<br/>Manages screens, show schedules, seat layouts, and ticket prices."]:::actorStyle
    SysAdmin["👤 Platform Admin<br/>(Person)<br/>Manages organizations, tenant setup, and system users."]:::actorStyle

    %% Core System Boundary
    subgraph CineBookSystem ["CineBook Platform Boundary"]
        CineBook["🎟️ CineBook System<br/>(Software System)<br/>Enables multi-tenant movie browsing, seat locking, CQRS booking management, and event-driven transaction handling."]:::systemStyle
    end

    %% External Systems
    Keycloak["🔐 Keycloak IDP<br/>(External System)<br/>Authenticates users, manages JWT tokens, roles, and tenant scopes."]:::extStyle
    Razorpay["💳 Razorpay Gateway<br/>(External System)<br/>Processes card/UPI payment charges and returns payment status."]:::extStyle
    MailSMS["✉️ Mail and SMS Gateway<br/>(External System)<br/>Delivers booking confirmations, e-tickets, and SMS alerts."]:::extStyle

    %% Relationships
    User -->|"1. Browses shows and initiates booking (HTTPS/REST)"| CineBook
    TheaterAdmin -->|"2. Configures theaters and showtimes (HTTPS/REST)"| CineBook
    SysAdmin -->|"3. Manages users and tenants (HTTPS/REST)"| CineBook

    User -->|"Authenticates and obtains JWT"| Keycloak
    TheaterAdmin -->|"Authenticates and obtains JWT"| Keycloak
    SysAdmin -->|"Authenticates and obtains JWT"| Keycloak

    CineBook -->|"Validates JWT tokens and credentials"| Keycloak
    CineBook -->|"Executes payment transactions"| Razorpay
    CineBook -->|"Sends booking receipts and alerts"| MailSMS

```

---

## 2. C4 Model - Level 2: Container Architecture Diagram

The Container diagram decomposes the CineBook system into individual runtime containers, databases, and message brokers, showing their communication protocols.

```mermaid
graph TD
    %% Clients & Security
    subgraph ClientLayer ["Client & Edge Security"]
        Client(["React + Vite UI (:5173)"])
        Gateway["API Gateway / Netty Proxy"]
        Keycloak["Keycloak IDP (:8080)<br/>Realm: cinebook"]
    end

    %% Microservices
    subgraph Services ["Spring Boot Microservices (Java 21)"]
        UMS["user-management-service (:8086)"]
        CS["cinema-service (:8083)"]
        LS["location-service (:7086)"]
        BS["booking-service (:8085)<br/>(Axon CQRS / Event Sourcing)"]
        PS["payment-service (:7085)"]
        NS["notification-service (:7083)"]
    end

    %% Eventing & Saga Orchestration
    subgraph Messaging ["Messaging & Distributed Saga Engine"]
        Kafka["Kafka / Redpanda Broker (:9092)"]
        Apicurio["Apicurio Schema Registry (:8081)<br/>(Avro Schemas)"]
        Temporal["Temporal Workflow Engine (:7233)<br/>Temporal Web UI (:8080)"]
    end

    %% Persistence Layer
    subgraph Storage ["Persistence Layer (PostgreSQL :5432 & Redis)"]
        Redis[("Redis Cache (:6379)<br/>Profiles & Sessions")]
        UserDB[("cinebook_user_db")]
        CinemaDB[("cinebook_cinema_db")]
        LocationDB[("cinebook_location_db")]
        
        subgraph BookingDBSchema ["cinebook_booking_db"]
            WriteDB[("public schema<br/>domain_event_entry (Event Store)<br/>token_entry / saga_entry / seats")]
            ReadDB[("booking_read schema<br/>booking (CQRS Read Model)")]
        end

        PaymentDB[("cinebook_payment_db")]
        NotificationDB[("cinebook_notification_db")]
    end

    %% External Systems
    subgraph External ["External Third-Party APIs"]
        Razorpay["Razorpay Payment Gateway"]
        SMTP["Mail / SMS Gateway"]
    end

    %% Client / Security Connections
    Client -->|HTTPS / OAuth2| Gateway
    Client -->|OIDC Auth| Keycloak
    Gateway --> UMS
    Gateway --> CS
    Gateway --> LS
    Gateway --> BS

    %% Service to Database Connections
    UMS --> UserDB
    UMS --> Redis
    CS --> CinemaDB
    LS --> LocationDB
    BS -->|Write Commands / Apply Events| WriteDB
    BS -->|Async Projections / Read History| ReadDB
    PS --> PaymentDB
    NS --> NotificationDB

    %% Integration & Orchestration Connections
    BS <-->|Kafka Events & Workflow Start| Kafka
    Kafka <--> Apicurio
    Kafka <-->|Trigger Workflows & Receive Signals| Temporal
    PS <-->|Kafka Events| Kafka
    NS <-->|Kafka Events| Kafka

    %% External API Connections
    PS -->|Charge Card / Idempotent API| Razorpay
    NS -->|Send Email & SMS| SMTP

```

---

## 3. High-Level Checkout Integration Sequence Diagram

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

## E2E Ticket Booking Saga Sequence Diagram
```mermaid
sequenceDiagram
    autonumber
    actor User as Client Browser
    participant Gateway as API Gateway
    participant BS as booking-service
    participant WriteDB as Postgres: public.domain_event_entry
    participant ReadDB as Postgres: booking_read.booking
    participant Kafka as Kafka / Redpanda Broker
    participant Temporal as Temporal Workflow Engine
    participant PS as payment-service
    participant Razorpay as Razorpay API
    participant NS as notification-service

    %% Phase 1: Command Execution & Event Sourcing
    rect rgb(240, 248, 255)
        note over User, BS: Phase 1: Booking Initiation & CQRS Event Sourcing
        User->>Gateway: POST /api/v1/bookings (ShowId, Seats: A1, A2)
        Gateway->>BS: Forward Request (Bearer JWT + x-tenant-id)
        BS->>BS: Validate seat locks in seats table
        BS->>BS: Axon CommandGateway dispatches CreateBookingCommand
        BS->>WriteDB: Append BookingCreatedEvent to domain_event_entry
        BS->>ReadDB: BookingProjection inserts record into booking_read.booking (PENDING)
        BS->>Kafka: Publish event to topic: booking-workflow-start
        BS-->>User: 201 Created (Booking ID: PENDING)
    end

    %% Phase 2: Temporal Saga Triggering
    rect rgb(255, 250, 240)
        note over Kafka, Temporal: Phase 2: Start Temporal Saga Workflow
        Kafka->>BS: BookingWorkflowStartListener consumes start event
        BS->>Temporal: Start BookingWorkflow (WorkflowID = BookingID)
    end

    %% Phase 3: Saga Step 1 - Seat Lock Activity
    rect rgb(245, 245, 245)
        note over Temporal, BS: Phase 3: Saga Activity 1 — Seat Lock Execution
        Temporal->>Kafka: Activity: reserveSeat -> Topic: seat-commands
        Kafka->>BS: SeatCommandListener reserves seats
        BS->>Kafka: Topic: seat-events (RESERVED)
        Kafka->>Temporal: Signal: seatReservationResult
    end

    %% Phase 4: Saga Step 2 - Payment Execution
    rect rgb(255, 240, 245)
        note over Temporal, Razorpay: Phase 4: Saga Activity 2 — Payment Processing
        Temporal->>Kafka: Activity: chargePayment -> Topic: payment-commands
        Kafka->>PS: PaymentCommandListener processes payment
        PS->>Razorpay: Charge Card (Idempotent API Call)
        Razorpay-->>PS: 200 OK (Payment Success)
        PS->>Kafka: Topic: payment-events (CHARGED)
        Kafka->>Temporal: Signal: paymentResult
    end

    %% Phase 5: Saga Step 3 & 4 - Confirmation & Notification
    rect rgb(240, 255, 240)
        note over Temporal, NS: Phase 5: Saga Activity 3 & 4 — Confirmation & Receipt
        Temporal->>Kafka: Activity: confirmBooking -> Topic: booking-commands
        Kafka->>BS: Update Aggregate state to CONFIRMED
        BS->>WriteDB: Append BookingConfirmedEvent
        BS->>ReadDB: Update status to CONFIRMED in booking_read.booking
        Temporal->>Kafka: Activity: sendNotification -> Topic: notification-commands
        Kafka->>NS: Send email/SMS ticket to User
    end

    %% Compensation Note
    note over Temporal, BS: Fallback/Compensation Note: If payment or seat lock fails, Temporal invokes compensation activities (releaseSeat command via Kafka) and sets booking status to CANCELLED.

```

---

## 4. Infrastructure & Deployment Architecture Diagram

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
