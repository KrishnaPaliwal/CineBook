# CineBook: Low-Level Design — location-service

## Overview

`location-service` provides geo-location and address lookup capabilities for CineBook. It integrates with an external location/maps API, uses Spring Cache for caching results, and exposes a simple REST API.

Port: **7086** (local Kubernetes, via port-forward)

---

## Package Structure

```
com.cinebook.locationservice
├── config/
│   ├── CacheConfig.java              # Spring Cache configuration
│   ├── OpenApiConfig.java            # Swagger/OpenAPI config
│   ├── SecurityConfig.java           # Spring Security / OAuth2 Resource Server
│   └── WebClientConfig.java          # WebClient for external location API calls
│
├── controller/
│   └── LocationController.java       # REST endpoint for location lookups
│
├── dto/
│   └── LocationResponse.java         # Response DTO for location data
│
├── exception/
│   ├── ExternalServiceException.java # External API failures
│   ├── GlobalExceptionHandler.java
│   ├── LocationServiceException.java
│   └── RateLimitExceededException.java
│
└── service/
    └── LocationService.java          # Calls external location API, applies caching
```

---

## REST API Endpoints

| Method | Path | Description |
|---|---|---|
| `GET` | `/api/v1/locations` | Lookup location / geo data |

---

## Design Notes

- Uses **WebClient** (reactive HTTP client) to call external location/maps API
- Results are **cached** via Spring Cache (`CacheConfig`) to reduce external API calls and latency
- **Rate limiting** is handled at the exception level (`RateLimitExceededException`)
- No database — location-service is stateless; all data comes from external API + in-memory cache

---

## External Dependencies

| Dependency | Purpose |
|---|---|
| External Maps/Location API | Source of geo/location data (via WebClient) |
| Spring Cache | In-memory caching of location results |
| Keycloak | JWT validation |
