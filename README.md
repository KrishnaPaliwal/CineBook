# 'CineBook' Movie Ticket Booking Application

Welcome to the CineBook application repository! CineBook is a modern, highly scalable, and distributed movie ticket booking system built from the ground up using microservices architecture. It aims to provide a seamless and robust experience for users to browse movies, select seats, and book tickets, while offering an administrative dashboard for managing content.

This repository serves as the central manifest and documentation hub for the entire system, detailing the architecture, technology stack, and technical design patterns implemented across all services.

## What is CineBook?

CineBook is a **multi-tenant movie ticketing platform** built with a microservices architecture. It supports browsing movies, selecting seats, completing payments via Razorpay, and receiving booking confirmations via email/SMS.


---

## 🏗️ Microservices Architecture

The system is decomposed into several independent microservices, strictly enforcing Domain-Driven Design (DDD) principles. This ensures loose coupling, independent scalability, and deployment autonomy. 

* **[user-management-service](https://github.com/KrishnaPaliwal/user-management-service)**: 
  - **Domain:** Identity and Access Management (IAM), User Profiles.
  - **Technical Role:** Integrates with **Keycloak** (acting as an OAuth2 Authorization Server) to securely manage identities, roles (User/Admin), and issue JSON Web Tokens (JWT) for secure downstream service communication.

* **[cinema-service](https://github.com/KrishnaPaliwal/cinema-service)**: 
  - **Domain:** Catalog and Master Data Management.
  - **Technical Role:** Manages the hierarchical data model of Movies -> Cinemas -> Screens -> Showtimes. It exposes optimized REST APIs (Query models) for the frontend to quickly fetch available inventory based on location and date constraints.

* **[booking-service](https://github.com/KrishnaPaliwal/booking-service)**: 
  - **Domain:** Core Transactional Engine.
  - **Technical Role:** The most complex service in the ecosystem. It orchestrates the entire ticket booking distributed transaction (Saga). It is built entirely on the **CQRS (Command Query Responsibility Segregation)** and **Event Sourcing** patterns using the **Axon Framework**.

* **[notification-service](https://github.com/KrishnaPaliwal/notification-service)**: 
  - **Domain:** Outbound Communications.
  - **Technical Role:** A decoupled, asynchronous consumer. It listens to Domain Events via Kafka (e.g., `UserRegisteredEvent`, `BookingConfirmedEvent`) and dispatches emails (via SMTP) and SMS (via Twilio APIs). 

* **[payment-service](https://github.com/KrishnaPaliwal/payment-service)**: 
  - **Domain:** Financial Processing.
  - **Technical Role:** Acts as an anti-corruption layer against third-party payment gateways (**Razorpay**). It manages payment intents, verifies cryptographically signed webhooks to confirm transactions, and publishes `PaymentCompletedEvent`s to the message broker.

* **[location-service](https://github.com/KrishnaPaliwal/location-service)**: 
  - **Domain:** Geolocation mapping.
  - **Technical Role:** Utilizes the **OpenCage Geocoder API** for reverse geocoding (GPS to City). It acts as a backend-for-frontend (BFF) proxy to hide API keys and enforce caching policies to minimize third-party API costs.

* **[cinebook-infra](https://github.com/KrishnaPaliwal/cinebook-infra)**: 
  - **Domain:** Infrastructure-as-Code (IaC).
  - **Technical Role:** Contains Kubernetes deployment manifests (`Deployment`, `Service`, `Ingress`, `ConfigMap`, `Secret`) and Helm charts necessary for provisioning the cluster topology.

* **[cinebook-frontend](https://github.com/KrishnaPaliwal/cinebook-frontend)**: 
  - **Domain:** User Interface.
  - **Technical Role:** A highly responsive Single Page Application (SPA) built with **React 18** and **Vite**. It utilizes **Material UI (MUI)** for the component library and React Context API for global state management.

* **Shared Libraries (cinebook-core-common & cinebook-common-messaging)**: 
  - **Technical Role:** Internal Maven dependencies housing shared Data Transfer Objects (DTOs), cross-cutting concern configurations (like Global Exception Handlers), and the **Avro schemas** required for strictly typed Kafka messaging.

---

## ⚙️ Deep Dive: Technical Design & Architecture

The CineBook backend is built using a modern enterprise Java stack: **Java 21** (utilizing virtual threads and new language features), **Spring Boot 3.3**, and **Spring Cloud**.

### 1. Distributed Data Management & Event-Driven Architecture
- **CQRS & Event Sourcing (Axon):** The `booking-service` separates write operations (Commands) from read operations (Queries). Every change to a booking's state (e.g., `CreateBookingCommand`, `ConfirmPaymentCommand`) generates an immutable Domain Event (`BookingCreatedEvent`, `PaymentConfirmedEvent`). These events are persisted in an Event Store. The current state is derived by replaying these events, guaranteeing a flawless audit trail.
- **Saga Pattern (Temporal & Kafka):** Booking a ticket involves multiple services (Lock seats in Cinema Service -> Wait for Payment in Payment Service -> Confirm in Booking Service -> Send Notification). **Temporal.io** is used alongside Kafka to orchestrate this Saga, handling distributed compensation logic (rollbacks/refunds) if any step fails (e.g., payment timeout).
- **Asynchronous Messaging:** **Apache Kafka** is the central nervous system. Services consume and publish events via `spring-cloud-stream-binder-kafka`. **Avro serialization** combined with Confluent Schema Registry ensures backward compatibility and strictly typed message contracts across service boundaries.

### 2. Security Architecture
- **OAuth2 & OIDC Authentication:** Keycloak acts as the OpenID Connect (OIDC) Provider. The frontend authenticates directly via PKCE flow, receiving a JWT.
- **Stateless Authorization:** Spring Boot services are configured as OAuth2 Resource Servers. They intercept requests, parse the Bearer JWT, validate its signature against the Keycloak JWKS endpoint, and map custom claims into Spring Security `GrantedAuthority` objects (e.g., `ROLE_USER`, `ROLE_ADMIN`) for method-level security (e.g., `@PreAuthorize`).

### 3. Database Isolation & Optimization
- **Database-per-Service:** Each microservice owns its independent PostgreSQL database schema to prevent integration database coupling. Schema migrations are strictly versioned and executed at startup using **Flyway**.
- **Distributed Caching:** **Redis** is utilized as a distributed cache. For example, `cinema-service` caches the master catalog of movies per city to bypass database queries for read-heavy operations, invalidating cache entries only upon admin updates.

### 4. API Resilience & Traffic Control
- **Circuit Breaking:** Synchronous OpenFeign HTTP calls between services are wrapped with **Resilience4j** Circuit Breakers and Timeouts. This prevents a cascading failure (e.g., if `location-service` is down, `cinema-service` fails fast instead of exhausting its connection pool).
- **Rate Limiting:** Public-facing APIs are protected by **Bucket4j**, backed by Redis. This implements a token bucket algorithm to throttle excessive requests and prevent brute-force attacks or API abuse.

### 5. Observability & Telemetry Stack
To diagnose issues in a distributed system, comprehensive observability is mandatory.
- **Distributed Tracing:** **OpenTelemetry (OTel)** java-agents auto-instrument incoming HTTP requests, outgoing REST calls, database queries, and Kafka message publishing/consuming. Trace IDs and Span IDs propagate across all network boundaries, allowing end-to-end visualization of a single user request.
- **Structured Logging:** The `Logstash-Logback-Encoder` outputs application logs in JSON format, automatically injecting OTel Trace IDs. This allows logs to be aggregated and correlated perfectly with traces.
- **Metrics Dashboarding:** **Micrometer** exposes JVM, database connection pool, and application-specific metrics to a `/actuator/prometheus` endpoint, which is scraped by a Prometheus server for Grafana dashboarding.
- **API Documentation:** Interactive API contracts are automatically generated using **Springdoc OpenAPI 3 (Swagger UI)**, accessible at the `/swagger-ui.html` endpoint on each service.


---

## 🚀 Implemented Functional Features

### Core Booking Flow & Concurrency Control
* **Distributed Seat Locking:** To prevent the classic "double-booking" race condition, seats are locked transactionally when a user selects them. This lock has a Time-To-Live (TTL). If the payment webhook is not received within the TTL, Temporal orchestrates a compensation transaction to release the lock.
* **Idempotent Payment Webhooks:** The `payment-service` webhook endpoint is designed to be idempotent. It cryptographically verifies the Razorpay signature and ensures that duplicate webhook deliveries do not result in double state changes.

### Administration & RBAC
* **Dynamic Master Data Updates:** Admins (`ROLE_ADMIN`) can dynamically add Movies, Cinemas, and schedule Showtimes via protected REST APIs, immediately invalidating the relevant Redis caches to reflect changes system-wide.

### Frontend Application (SPA)
* **Vite HMR & Build Optimization:** Leveraging Vite for Lightning-fast Hot Module Replacement (HMR) during development and highly optimized Rollup builds for production.
* **Context API & React Router:** Efficient client-side routing combined with React Context to manage global authentication tokens and the user's currently selected geolocation without prop-drilling.

---

## Tech Stack

| Layer | Technology |
|---|---|
| Language | Java 21 |
| Framework | Spring Boot 3.x |
| Build | Maven |
| Event Sourcing | Axon Framework |
| Saga Orchestration | Temporal |
| Messaging | Apache Kafka (Confluent) + Avro (Apicurio Schema Registry) |
| Identity | Keycloak 24 |
| Database | PostgreSQL 15 |
| Cache | Redis |
| Observability | OpenTelemetry → Prometheus + Grafana + OpenSearch |
| Containers | Docker |
| Orchestration | Kubernetes (Docker Desktop) |
| Deployment | Helm charts |
| Payment | Razorpay |

---

## Local Development Setup

### Prerequisites

- Docker Desktop with Kubernetes enabled
- `kubectl` configured to use Docker Desktop context
- `helm` installed
- Java 21 + Maven
- Node.js (for frontend)

### Step 1 — Start the Infrastructure

```powershell
docker-compose -f cinebook-infra/local/docker-compose.dev.yml up -d
```

This starts: PostgreSQL, Kafka, Zookeeper, Apicurio Schema Registry, Temporal, Axon Server, Keycloak, Redis, Adminer, Kafka UI, OpenTelemetry Collector, Prometheus, Grafana, OpenSearch, OpenSearch Dashboards.

Wait ~30 seconds for all services to be healthy.

### Step 2 — Build Docker Images

```powershell
# Example: build booking-service
cd booking-service
docker build -t booking-service:latest .

# Repeat for each service
cd ../cinema-service && docker build -t cinema-service:latest .
cd ../user-management-service && docker build -t user-management-service:latest .
cd ../payment-service && docker build -t payment-service:latest .
cd ../notification-service && docker build -t notification-service:latest .
cd ../location-service && docker build -t location-service:latest .
```

### Step 3 — Deploy to Kubernetes via Helm

```powershell
cd cinebook-infra

helm upgrade --install user-management-service ./helm/cinebook-microservice -f ./helm/values-user-management-local.yaml
helm upgrade --install cinema-service          ./helm/cinebook-microservice -f ./helm/values-cinema-local.yaml
helm upgrade --install booking-service         ./helm/cinebook-microservice -f ./helm/values-booking-local.yaml
helm upgrade --install payment-service         ./helm/cinebook-microservice -f ./helm/values-payment-local.yaml
helm upgrade --install notification-service    ./helm/cinebook-microservice -f ./helm/values-notification-local.yaml
helm upgrade --install location-service        ./helm/cinebook-microservice -f ./helm/values-location-local.yaml
```

### Step 4 — Port Forward Services

```powershell
Start-Job -ScriptBlock { kubectl port-forward -n cinebook svc/user-management-service-svc 8086:8086 }
Start-Job -ScriptBlock { kubectl port-forward -n cinebook svc/cinema-service-svc 8083:8083 }
Start-Job -ScriptBlock { kubectl port-forward -n cinebook svc/booking-service-svc 8085:8085 }
Start-Job -ScriptBlock { kubectl port-forward -n cinebook svc/payment-service-svc 7085:7085 }
Start-Job -ScriptBlock { kubectl port-forward -n cinebook svc/notification-service-svc 7083:7083 }
Start-Job -ScriptBlock { kubectl port-forward -n cinebook svc/location-service-svc 7086:7086 }
```

### Step 5 — Start Frontend

```powershell
cd cinebook-frontend
npm install
npm run dev
```

### Stop Port Forwarding

```powershell
Get-Job | Stop-Job | Remove-Job
```

---

## Service Ports (Local)

| Service | Port |
|---|---|
| user-management-service | 8086 |
| cinema-service | 8083 |
| booking-service | 8085 |
| payment-service | 7085 |
| notification-service | 7083 |
| location-service | 7086 |

## Infrastructure Services & Dashboards (Docker Compose)

### 📊 Web UI & Observability Dashboards

| Service | Web UI / Interface URL | Credentials / Details |
| :--- | :--- | :--- |
| **Axon Server** | [http://localhost:8024](http://localhost:8024) | Axon Event Store & Command Routing Dashboard |
| **Grafana** | [http://localhost:3000](http://localhost:3000) | Observability Dashboard (Default: `admin` / `admin`) |
| **OpenSearch Dashboards** | [http://localhost:5601](http://localhost:5601) | Log & Search Analytics (User: `admin`, Password: `admin`) |
| **Prometheus** | [http://localhost:9090](http://localhost:9090) | Metrics & Alerting Engine |
| **Temporal UI** | [http://localhost:8089](http://localhost:8089) | Saga Workflow Orchestration UI |
| **Keycloak (OIDC Auth)** | [http://localhost:8080](http://localhost:8080) | Realm: `cinebook` \| Admin: `admin` / `admin` |
| **Kafka UI** | [http://localhost:9091](http://localhost:9091) | Kafka Topic & Message Inspector |
| **Apicurio (Schema Registry)** | [http://localhost:8081](http://localhost:8081) | Avro Schema Repository |
| **Adminer (DB Web Client)** | [http://localhost:8082](http://localhost:8082) | PostgreSQL Web Client |

### 🔌 Backend Infrastructure Connection Endpoints

| Service | Host & Port | Description |
| :--- | :--- | :--- |
| **Axon Server (gRPC)** | `localhost:8124` | Application gRPC Connection |
| **OpenSearch API** | `https://localhost:9200` | OpenSearch Engine API (Basic Auth: `admin` / `admin`) |
| **OTEL Collector (OTLP gRPC)** | `localhost:4317` | OpenTelemetry Traces/Metrics gRPC |
| **OTEL Collector (OTLP HTTP)** | `localhost:4318` | OpenTelemetry Traces/Metrics HTTP |
| **Kafka Broker** | `localhost:9092` / `localhost:29092` | Kafka Bootstrap Server |
| **Temporal Server** | `localhost:7233` | Temporal gRPC Endpoint |
| **PostgreSQL** | `localhost:5432` | DB: `cinebook_booking_db` \| User: `postgres`, Password: `password` |
| **Redis** | `localhost:6379` | User Profile & Caching Layer |



