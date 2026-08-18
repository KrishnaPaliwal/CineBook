# Gang of Four (GoF) Design Patterns Breakdown in CineBook

This document provides a comprehensive, production-grade technical breakdown of the **Gang of Four (GoF) Design Patterns** implemented across the microservices in the **CineBook** ticketing platform.

---

## 📐 Overview of GoF Categories in CineBook

The **CineBook** platform leverages enterprise patterns (CQRS, Event Sourcing, Saga Orchestration) built on top of classic Gang of Four (GoF) patterns across three main classifications:

1. **Creational Patterns**: Object creation mechanisms that increase flexibility and reuse.
2. **Structural Patterns**: Class and object composition for building larger, decoupled structures.
3. **Behavioral Patterns**: Algorithms and assignment of responsibilities between objects.

---

## 📦 Microservice-by-Microservice Pattern Breakdown

### 1. 🎟️ `booking-service`
*Location*: [`booking-service`](file:///d:/Development/CineBook_Development/booking-service)

The `booking-service` is the core transactional hub of CineBook, implementing CQRS and Event Sourcing via the Axon Framework.

#### Behavioral Patterns
* **Command Pattern**
  * **Implementation**: [`CreateBookingCommand`](file:///d:/Development/CineBook_Development/booking-service/src/main/java/com/cinebook/bookingservice/axon/command/CreateBookingCommand.java), `ConfirmBookingCommand`, `CancelBookingCommand`.
  * **Details**: Encapsulates user booking requests into immutable command objects. Commands are dispatched via Axon's `CommandGateway` and processed by `@CommandHandler` methods inside [`BookingAggregate`](file:///d:/Development/CineBook_Development/booking-service/src/main/java/com/cinebook/bookingservice/axon/aggregate/BookingAggregate.java).
* **State Pattern**
  * **Implementation**: [`BookingAggregate.java`](file:///d:/Development/CineBook_Development/booking-service/src/main/java/com/cinebook/bookingservice/axon/aggregate/BookingAggregate.java).
  * **Details**: The aggregate maintains domain state (`status` transitions: `PENDING` $\rightarrow$ `CONFIRMED` / `FAILED`). State mutations occur exclusively within `@EventSourcingHandler` methods in response to domain events (`BookingCreatedEvent`, `BookingConfirmedEvent`, `BookingCancelledEvent`).
* **Observer / Publish-Subscribe Pattern**
  * **Implementation**: [`BookingProjection.java`](file:///d:/Development/CineBook_Development/booking-service/src/main/java/com/cinebook/bookingservice/projection/BookingProjection.java).
  * **Details**: Axon's `TrackingEventProcessor` listens to events emitted by the aggregate via `AggregateLifecycle.apply(...)`. The projection observes these events asynchronously to update the read-optimized database model (`booking_read.booking`).
* **Mediator Pattern**
  * **Implementation**: Axon `CommandGateway` & `EventGateway`.
  * **Details**: Serves as a central hub routing commands and events between controllers, aggregates, and projection handlers without direct coupling.
* **Memento Pattern**
  * **Implementation**: Axon Event Store (`public.domain_event_entry` & `public.snapshot_event_entry`).
  * **Details**: Preserves the complete historical state transitions of the `BookingAggregate`. State can be rehydrated at any time by replaying historical domain events from index 0.

#### Creational Patterns
* **Builder Pattern**
  * **Implementation**: Lombok `@Builder` on `BookingRequestDTO` and `BookingResponseDTO`.
  * **Details**: Simplifies creation of complex data transfer objects with validated optional and mandatory fields.

---

### 2. 🔑 `user-management-service`
*Location*: [`user-management-service`](file:///d:/Development/CineBook_Development/user-management-service)

This service bridges CineBook domain logic with **Keycloak Identity Provider** and manages organization tenancy.

#### Structural Patterns
* **Adapter Pattern**
  * **Implementation**: [`KeycloakAdminClientService.java`](file:///d:/Development/CineBook_Development/user-management-service/src/main/java/com/cinebook/usermanagementservice/service/KeycloakAdminClientService.java).
  * **Details**: Converts CineBook domain operations (`createUser`, `resetUserPassword`, `updateUserTenantAndRole`) into Keycloak Java REST Client SDK calls (`UsersResource`, `UserRepresentation`, `CredentialRepresentation`).
* **Facade Pattern**
  * **Implementation**: [`OrganizationService.java`](file:///d:/Development/CineBook_Development/user-management-service/src/main/java/com/cinebook/usermanagementservice/service/OrganizationService.java) & [`KeycloakAdminClientService.java`](file:///d:/Development/CineBook_Development/user-management-service/src/main/java/com/cinebook/usermanagementservice/service/KeycloakAdminClientService.java).
  * **Details**: Provides simplified high-level interface methods that orchestrate multi-step low-level operations (Keycloak user search, creation, credential assignment, role mapping, and local PostgreSQL tenant record creation).
* **Proxy & Decorator Pattern**
  * **Implementation**: `@CircuitBreaker(name = "keycloakAdmin", fallbackMethod = "createUserFallback")`.
  * **Details**: Resilience4j dynamically wraps service calls in proxy objects to provide circuit breaking, retry policies, and fallback handling against external Keycloak service outages.

#### Creational Patterns
* **Singleton Pattern**
  * **Implementation**: `Keycloak` bean defined in `KeycloakConfig.java`.
  * **Details**: Maintained as a single, thread-safe shared bean instance within the Spring IoC Container.

---

### 3. 💳 `payment-service`
*Location*: [`payment-service`](file:///d:/Development/CineBook_Development/payment-service)

`payment-service` manages payment gateways, refunds, and transactional outbox event streams.

#### Structural Patterns
* **Adapter Pattern**
  * **Implementation**: [`PaymentService.java`](file:///d:/Development/CineBook_Development/payment-service/src/main/java/com/cinebook/paymentservice/service/PaymentService.java).
  * **Details**: Wraps third-party `RazorpayClient` API SDK to adapt external JSON/Response models into internal domain structures (`PaymentResponse`, `Payment`).

#### Behavioral Patterns
* **Strategy Pattern**
  * **Implementation**: [`PaymentService.java`](file:///d:/Development/CineBook_Development/payment-service/src/main/java/com/cinebook/paymentservice/service/PaymentService.java#L72-L95).
  * **Details**: Dynamically selects execution strategy at runtime (Simulated/Mock payment path vs Live Razorpay API integration path) based on transaction properties.
* **Chain of Responsibility Pattern**
  * **Implementation**: Spring Security Filter Chain & `@RestControllerAdvice` Exception Handlers.
  * **Details**: Requests pass sequentially through authorization filters, tenant headers validation, and global exception resolvers.
* **Template Method Pattern**
  * **Implementation**: `KafkaTemplate` and `WebClient` abstractions.
  * **Details**: Defines skeleton execution flows for async message publishing and HTTP communication, while Spring handles socket connections, serialization, and resource cleanup.

---

### 4. 🍿 `cinema-service` & 📍 `location-service`
*Locations*: [`cinema-service`](file:///d:/Development/CineBook_Development/cinema-service), [`location-service`](file:///d:/Development/CineBook_Development/location-service)

#### Creational & Structural Patterns
* **Factory Method Pattern**
  * **Implementation**: `CinemaRepository`, `ShowRepository`, `SeatRepository`, `LocationRepository`.
  * **Details**: Spring Data JPA creates proxy instances of repositories dynamically based on entity interface contracts.
* **Flyweight Pattern**
  * **Implementation**: Spring Bean Instance Registry & JPA Meta-model Caching.
  * **Details**: Reuses immutable structural definitions (e.g. seat layouts, hall types) to reduce memory overhead across concurrent user requests.

---

### 5. 🔔 `notification-service` & 🔄 Shared Saga Messaging
*Locations*: [`notification-service`](file:///d:/Development/CineBook_Development/notification-service), [`cinebook-common-messaging`](file:///d:/Development/CineBook_Development/cinebook-common-messaging)

#### Behavioral & Creational Patterns
* **Observer Pattern**
  * **Implementation**: `@KafkaListener` topic consumers in `notification-service`.
  * **Details**: Listens asynchronously to Kafka topics (`notification-commands`, `booking-events`) and triggers external email/SMS dispatch routines.
* **Saga Pattern / State Machine**
  * **Implementation**: Temporal Workflow (`BookingWorkflowImpl`).
  * **Details**: Manages complex distributed state transactions (`reserveSeat` $\rightarrow$ `chargePayment` $\rightarrow$ `confirmBooking`) with automatic compensating rollback actions.
* **Builder Pattern**
  * **Implementation**: Avro auto-generated Java record builders.
  * **Details**: Constructs binary Avro event objects safely (e.g. `BookingCreatedEvent.newBuilder()...build()`).

---

## 📊 Comprehensive Summary Matrix

| GoF Pattern Category | Pattern Name | Microservice / Component | Key Target Files / Classes | Purpose & Benefit |
| :--- | :--- | :--- | :--- | :--- |
| **Creational** | **Builder** | All Microservices | `DTOs`, Avro generated models | Safe, readable construction of complex immutable objects |
| | **Singleton** | All Microservices | Spring `@Service`, `@Repository` | Single shared instance management via Spring IoC container |
| | **Factory Method** | All Microservices | Spring Data JPA Repositories | Runtime generation of database access proxies |
| **Structural** | **Adapter** | `user-management-service`, `payment-service` | [`KeycloakAdminClientService.java`](file:///d:/Development/CineBook_Development/user-management-service/src/main/java/com/cinebook/usermanagementservice/service/KeycloakAdminClientService.java), [`PaymentService.java`](file:///d:/Development/CineBook_Development/payment-service/src/main/java/com/cinebook/paymentservice/service/PaymentService.java) | Translating 3rd-party APIs (Keycloak, Razorpay) into domain models |
| | **Facade** | `user-management-service` | [`OrganizationService.java`](file:///d:/Development/CineBook_Development/user-management-service/src/main/java/com/cinebook/usermanagementservice/service/OrganizationService.java) | Providing simplified interfaces over complex multi-step routines |
| | **Proxy / Decorator** | All Microservices | `@CircuitBreaker`, `@Transactional` | Dynamic wrapping for resilience, security, and transaction boundaries |
| | **Flyweight** | `cinema-service`, `location-service` | JPA Entity Metamodel, Spring Bean Registry | Memory efficiency for static screen layouts and metadata |
| **Behavioral** | **Command** | `booking-service` | [`CreateBookingCommand.java`](file:///d:/Development/CineBook_Development/booking-service/src/main/java/com/cinebook/bookingservice/axon/command/CreateBookingCommand.java) | Encapsulating transactional requests as first-class objects |
| | **State** | `booking-service` | [`BookingAggregate.java`](file:///d:/Development/CineBook_Development/booking-service/src/main/java/com/cinebook/bookingservice/axon/aggregate/BookingAggregate.java) | Encapsulating status transitions (`PENDING` $\rightarrow$ `CONFIRMED`) |
| | **Observer** | `booking-service`, `notification-service` | [`BookingProjection.java`](file:///d:/Development/CineBook_Development/booking-service/src/main/java/com/cinebook/bookingservice/projection/BookingProjection.java), `@KafkaListener` | Asynchronous pub-sub handling across read models & services |
| | **Mediator** | `booking-service`, Temporal Saga | Axon `CommandGateway`, Temporal Engine | Decoupling senders from downstream execution handlers |
| | **Memento** | `booking-service` | Axon Event Store (`domain_event_entry`) | Full audit trail and state rehydration via event streams |
| | **Strategy** | `payment-service` | [`PaymentService.java`](file:///d:/Development/CineBook_Development/payment-service/src/main/java/com/cinebook/paymentservice/service/PaymentService.java) | Selecting execution mode (Mock vs Live Razorpay) at runtime |
| | **Chain of Resp.** | All Microservices | Spring Security `FilterChain`, `@RestControllerAdvice` | Sequential request validation and centralized exception handling |
| | **Template Method** | All Microservices | `KafkaTemplate`, `WebClient`, `JdbcTemplate` | Standardizing execution steps for async messaging and HTTP calls |
