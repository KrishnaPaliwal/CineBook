# 'CineBook' Movie Ticket Booking Application

Welcome to the CineBook application repository! CineBook is a modern, highly scalable, and distributed movie ticket booking system built from the ground up using microservices architecture. It aims to provide a seamless and robust experience for users to browse movies, select seats, and book tickets, while offering an administrative dashboard for managing content.

This repository serves as the central manifest and documentation hub for the entire system, detailing the architecture, technology stack, and implemented features across all services.

---

## 🏗️ Microservices Architecture

The system is decomposed into several independent microservices, each bounded by its specific domain. This ensures loose coupling, independent scalability, and easier maintenance. Here is a detailed breakdown of each service:

* **[user-management-service](https://github.com/KrishnaPaliwal/user-management-service)**: 
  - **Responsibility:** Handles all user-related operations, including registration, profile management, and authentication.
  - **Key Features:** It seamlessly integrates with **Keycloak** (acting as an OAuth2 Authorization Server) to securely manage identities, roles (User/Admin), and issue JSON Web Tokens (JWT) for secure downstream service communication.

* **[cinema-service](https://github.com/KrishnaPaliwal/cinema-service)**: 
  - **Responsibility:** The core catalog service that manages all master data related to the cinema business.
  - **Key Features:** Manages the catalog of movies, geographical cinemas, physical screens within cinemas, and the scheduling of showtimes. It exposes REST APIs for the frontend to query available movies and showtimes based on location and date.

* **[booking-service](https://github.com/KrishnaPaliwal/booking-service)**: 
  - **Responsibility:** The most complex service in the ecosystem, orchestrating the entire ticket booking workflow.
  - **Key Features:** It implements **CQRS (Command Query Responsibility Segregation)** and **Event Sourcing** using the **Axon Framework** to guarantee transactional integrity and maintain an immutable audit log of booking state changes. Furthermore, it leverages **Temporal** for robust, fault-tolerant workflow orchestration (e.g., handling timeouts if a payment isn't completed within the seat-lock window).

* **[notification-service](https://github.com/KrishnaPaliwal/notification-service)**: 
  - **Responsibility:** A decoupled, asynchronous service dedicated to outbound communications.
  - **Key Features:** Listens to Domain Events via Kafka (e.g., `UserRegisteredEvent`, `BookingConfirmedEvent`) and dispatches emails (via SMTP) and SMS (via Twilio). It handles OTPs for registration and e-tickets for confirmed bookings.

* **[payment-service](https://github.com/KrishnaPaliwal/payment-service)**: 
  - **Responsibility:** Securely handles financial transactions.
  - **Key Features:** Provides a secure integration with the **Razorpay API**. It manages payment intents, verifies webhook signatures from Razorpay to confirm successful payments, and initiates refunds for cancelled bookings.

* **[location-service](https://github.com/KrishnaPaliwal/location-service)**: 
  - **Responsibility:** Handles geolocation and mapping functionalities.
  - **Key Features:** Utilizes the **OpenCage Geocoder API** to perform reverse geocoding (converting GPS coordinates from the user's browser into a city name). This dedicated service abstracts third-party API dependencies and keeps sensitive API keys secure on the backend.

* **[cinebook-infra](https://github.com/KrishnaPaliwal/cinebook-infra)**: 
  - **Responsibility:** The Infrastructure-as-Code (IaC) repository.
  - **Key Features:** Contains all Kubernetes deployment manifests, services, ingress configurations, ConfigMaps, Secrets, and Helm charts necessary to deploy the entire CineBook suite into a Kubernetes cluster (like GKE).

* **[cinebook-frontend](https://github.com/KrishnaPaliwal/cinebook-frontend)**: 
  - **Responsibility:** The user-facing web application.
  - **Key Features:** A highly responsive Single Page Application (SPA) built with **React 18** and **Vite**. It utilizes **Material UI (MUI)** for a clean, modern aesthetic and React Context API for global state management (auth and location).

* **Shared Libraries (cinebook-core-common & cinebook-common-messaging)**: 
  - **Responsibility:** DRY (Don't Repeat Yourself) principle implementation.
  - **Key Features:** These modules contain shared Data Transfer Objects (DTOs), common utility classes, global exception handlers, and the **Avro schemas** required for Kafka messaging, ensuring all services communicate using a strictly typed contract.

---

## ⚙️ High-Level System Architecture & Tech Stack

The CineBook backend is built using a modern Java stack: **Java 21**, **Spring Boot 3.3**, and **Spring Cloud**.

### 1. Inter-Service Communication
- **Synchronous:** Services communicate via REST/HTTP using Spring Cloud OpenFeign for direct queries (e.g., Booking Service querying Cinema Service for showtime details).
- **Asynchronous (Event-Driven):** **Apache Kafka** is the backbone of the system. State changes (like a completed payment or a new user registration) are published as events to Kafka topics. Services consume these events to update their local state or trigger actions (like sending an email). **Avro serialization** and Confluent Schema Registry are used to ensure schema evolution and type safety.

### 2. Security & API Gateway
- **OAuth2 & Keycloak:** The system employs a robust security model. Keycloak acts as the Identity Provider (IdP). The Spring Boot services act as OAuth2 Resource Servers, validating JWTs on every request to ensure the user is authenticated and authorized (checking for `ROLE_USER` or `ROLE_ADMIN`).

### 3. Databases & Caching
- **PostgreSQL:** Each microservice has its own isolated PostgreSQL database (Database-per-service pattern), ensuring loose coupling. Database schemas and migrations are strictly managed using **Flyway**.
- **Redis:** Used extensively for caching frequently accessed data (like movie lists per city) to reduce database load and improve response times. It is also utilized by **Bucket4j** for API rate limiting to prevent abuse.

### 4. Observability, Logging, & Resiliency
- **OpenTelemetry (OTel):** The entire application is heavily instrumented with OpenTelemetry. It provides distributed tracing across all microservices, allowing developers to track a request from the frontend, through the API, and across Kafka events.
- **Structured Logging:** Logs are formatted using the Logstash Logback encoder, making them easily searchable in centralized logging systems (like ELK/EFK stack).
- **Metrics:** Micrometer exposes application metrics to a **Prometheus** registry.
- **Resilience4j:** Protects the system from cascading failures using Circuit Breakers, Retry mechanisms, and Timeouts on synchronous inter-service calls.

---

## 🚀 Implemented Features

This section details the rich feature set available to both end-users and administrators.

### Core User & Authentication Features
* **OTP-Verified Registration:** To ensure high data quality, new user registrations require verification via a One-Time Password (OTP) sent to both email and SMS (via Twilio) before the account becomes active.
* **Secure JWT Login:** Users authenticate securely, receiving a JWT that manages their session state securely without server-side session overhead.
* **Password Management:** Complete flow for "Forgot Password" with secure, single-use email links, and the ability for logged-in users to update their passwords from their profile dashboard.
* **Profile Management:** Users can view and manage their personal information seamlessly.

### Core Booking & Payment Flow
* **Dynamic Movie Browsing:** A rich UI allowing users to browse currently showing movies, filter by genre or language, and view available showtimes for their selected date and city.
* **Interactive Seat Selection:** A visual, interactive seat map for a specific showtime. Users can see available, booked, and currently locked seats in real-time.
* **Distributed Seat Locking:** To prevent race conditions (double-booking), selected seats are temporarily locked in the backend (using Redis/Temporal) for a defined duration. If the user doesn't pay within this window, the seats are automatically released.
* **Razorpay Payment Integration:** A seamless and secure checkout experience integrated with Razorpay, supporting various payment methods (UPI, Credit/Debit Cards, NetBanking).
* **Instant Notifications:** Upon successful payment validation via Razorpay webhooks, the system instantly dispatches booking confirmation e-tickets via email and SMS.

### Admin & Content Management
* **Role-Based Access Control (RBAC):** Strict separation of concerns. Only users with the `ROLE_ADMIN` authority can access the admin dashboard.
* **Comprehensive Admin Dashboard:** A dedicated interface for managing the entire cinema catalog.
* **Content Management:** Admins can effortlessly add new Movies (with posters and metadata), create new Cinemas, define Screens within those cinemas, and schedule Showtimes with specific pricing tiers.

### UI/UX & Personalization Features
* **Smart Location Detection:** The application can automatically detect the user's city using HTML5 Geolocation (processed via the location-service) to instantly filter the movie catalog to their area. Users can also manually select their city from a dynamic dropdown.
* **Organized Booking History:** A dedicated user dashboard categorizing bookings into "Upcoming" and "Past" for easy navigation.
* **Downloadable PDF e-Tickets:** Confirmed bookings generate a professional, downloadable PDF ticket containing a scannable QR code, seat details, and cinema directions.
* **Automated Cancellation & Refunds:** Users can cancel upcoming bookings directly from the UI. This triggers an automated workflow in the backend that communicates with Razorpay to initiate a refund and instantly frees up the seats for other customers.

---

## 🛠️ How to Run & Deploy

This application is designed for cloud-native environments. 

1. **Prerequisites:** You will need a Kubernetes cluster (e.g., Google Kubernetes Engine - GKE, Minikube, or Docker Desktop with Kubernetes enabled), Helm, and `kubectl` configured.
2. **Infrastructure:** Navigate to the `cinebook-infra` repository. Ensure you have the necessary secrets configured (e.g., Database passwords, Keycloak credentials, Razorpay API keys, Twilio credentials).
3. **Deployment:** Apply the Kubernetes manifests and Helm charts provided in the `cinebook-infra` repository to spin up the databases, Kafka cluster, Keycloak instance, and all the microservices.
