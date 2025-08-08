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


## How to Run
Deploy these services on Google Cloud GKE Cluster.
