# CineBook: Low-Level Design — cinema-service

## Overview

`cinema-service` manages all cinema-related domain entities: movies, cinemas, screens, and shows. It is a standard CRUD microservice with JPA persistence and serves as the reference data provider for `booking-service`.

Port: **8083** (local Kubernetes, via port-forward)

---

## Package Structure

```
com.cinebook.cinemaservice
├── config/
│   ├── OpenApiConfig.java            # Swagger/OpenAPI config
│   ├── RateLimitingInterceptor.java  # API rate limiting
│   ├── RedisConfig.java              # Redis connection config
│   ├── SecurityConfig.java           # Spring Security / OAuth2 Resource Server
│   ├── TenantDataSourceConfig.java   # Multi-tenant datasource routing
│   └── WebMvcConfig.java             # CORS, interceptors
│
├── controller/
│   ├── CinemaController.java         # CRUD for /api/v1/cinemas
│   ├── MovieController.java          # CRUD for /api/v1/movies
│   ├── ScreenController.java         # CRUD for /api/v1/screens
│   └── ShowController.java           # CRUD for /api/v1/shows
│
├── dto/
│   └── MovieDTO.java
│
├── exception/
│   ├── GlobalExceptionHandler.java
│   └── ResourceNotFoundException.java
│
├── model/
│   ├── Cinema.java                   # JPA entity — cinema hall
│   ├── Movie.java                    # JPA entity — movie
│   ├── Screen.java                   # JPA entity — screen inside a cinema
│   └── Show.java                     # JPA entity — scheduled show
│
├── repository/
│   ├── CinemaRepository.java
│   ├── MovieRepository.java
│   ├── ScreenRepository.java
│   └── ShowRepository.java
│
└── service/
    ├── CinemaService.java
    ├── MovieService.java
    ├── ScreenService.java
    └── ShowService.java
```

---

## REST API Endpoints

| Method | Path | Description |
|---|---|---|
| `GET` | `/api/v1/movies` | List all movies |
| `POST` | `/api/v1/movies` | Create a movie |
| `GET` | `/api/v1/movies/{id}` | Get movie by ID |
| `PUT` | `/api/v1/movies/{id}` | Update movie |
| `DELETE` | `/api/v1/movies/{id}` | Delete movie |
| `GET` | `/api/v1/cinemas` | List all cinemas |
| `POST` | `/api/v1/cinemas` | Create a cinema |
| `GET` | `/api/v1/cinemas/{id}` | Get cinema by ID |
| `GET` | `/api/v1/screens` | List screens |
| `POST` | `/api/v1/screens` | Create a screen |
| `GET` | `/api/v1/screens/{id}` | Get screen by ID |
| `GET` | `/api/v1/shows` | List shows |
| `POST` | `/api/v1/shows` | Create a show |
| `GET` | `/api/v1/shows/{id}` | Get show by ID |

---

## Domain Model

```
Movie ──< Show >── Screen ──< Cinema
```

- A **Cinema** has many **Screens**
- A **Screen** hosts many **Shows**
- A **Show** is a scheduled screening of a **Movie** on a **Screen**

---

## Database Schema

**PostgreSQL DB**: `cinebook_cinema_db`

### Tables
- `movie` — movie catalog (title, description, duration, genre, language, rating)
- `cinema` — cinema hall (name, location, address, tenantId)
- `screen` — screen inside cinema (name, capacity, cinemaId)
- `show` — scheduled show (movieId, screenId, startTime, endTime, tenantId)

---

## External Dependencies

| Dependency | Purpose |
|---|---|
| PostgreSQL | Persistence (`cinebook_cinema_db`) |
| Redis | Available (config present) for caching |
| Keycloak | JWT validation |
| `booking-service` | Calls cinema-service via Feign client (`CinemaServiceDelegate`) for show/seat validation |
