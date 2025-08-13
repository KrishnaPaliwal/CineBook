# CineBook Application

This is the central manifest for CineBook Application, a distributed system built with microservices.

## Microservices Architecture

Here are the primary services that make up the application. See each repository for specific details.

* **[auth-service](https://github.com/KrishnaPaliwal/auth-service)**: Manages user accounts, authentication, and profiles.
* **[cinema-service](https://github.com/KrishnaPaliwal/cinema-service)**: Manages movies and its related functionality.
* **[booking-service](https://github.com/KrishnaPaliwal/booking-service)**: Manages business functionality of booking flow.
* **[notification-service](https://github.com/KrishnaPaliwal/notification-service)**: Sends emails, messages and push notifications.
* **[api-gateway](https://github.com/KrishnaPaliwal/api-gateway)**: The single entry point for all client requests.
* **[cinebook-infra](https://github.com/KrishnaPaliwal/cinebook-infra)**: Configuration files for CineBook Application.
* **[cinebook-frontend](https://github.com/KrishnaPaliwal/cinebook-frontend)**: Frontend GUI project.

## High-Level Architecture
Architecture: This is a well-defined microservice architecture. Each service has a clear responsibility, and they communicate effectively through REST APIs (for synchronous calls) and RabbitMQ (for asynchronous notifications).

Authentication & Authorization: The auth-service handles user registration (with OTP) and login, issuing JWTs. The other services (cinema-service in particular) correctly use a JWT filter to validate these tokens and enforce role-based access (ROLE_ADMIN vs. ROLE_USER).

Booking Flow: The booking process is robust, following a "lock-then-pay" model. It correctly interacts with the cinema-service for show details and the payment-service for transactions.

Notifications: The notification-service is properly decoupled and handles both email and SMS notifications for OTP and booking confirmations.

Frontend: The React application uses a modern stack with Vite, Material UI, and a component-based structure. Global state for authentication is managed well with a Context.

## How to Run
Deploy these services on Google Cloud GKE Cluster.
