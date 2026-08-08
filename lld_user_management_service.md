# CineBook: Low-Level Design — user-management-service

## Overview

`user-management-service` handles all user identity, authentication, and organization management. It integrates with **Keycloak** as the OIDC identity provider and also runs an embedded **Temporal saga** for the user registration workflow (OTP verification flow).

Port: **8086** (local Kubernetes, via port-forward)

---

## Package Structure

```
com.cinebook.usermanagementservice
├── config/
│   ├── KafkaProducerConfig.java      # Kafka producer for OTP/password-reset notifications
│   ├── KeycloakConfig.java           # Keycloak admin client config
│   ├── OpenApiConfig.java            # Swagger/OpenAPI
│   ├── RedisConfig.java              # Redis session cache config
│   ├── SecurityConfig.java           # Spring Security / OAuth2 Resource Server
│   ├── TemporalConfig.java           # Temporal WorkflowClient + Worker setup
│   └── WebMvcConfig.java             # CORS, interceptors (no auth needed for AuthController)
│
├── controller/
│   ├── AuthController.java           # Login, register, OTP verify, refresh token, forgot/reset password
│   ├── OrganizationController.java   # Organization CRUD + invite management
│   └── UserController.java           # User profile CRUD
│
├── dto/
│   ├── AuthResponse.java
│   ├── ChangePasswordRequest.java
│   ├── ErrorResponse.java
│   ├── ForgotPasswordRequest.java
│   ├── LoginRequest.java
│   ├── LoginResponse.java
│   ├── OtpNotification.java          # Kafka message payload for OTP
│   ├── PasswordResetNotification.java
│   ├── RegistrationRequest.java
│   ├── ResetPasswordRequest.java
│   ├── TokenRefreshRequest.java
│   ├── UserProfileDTO.java
│   └── VerificationRequest.java
│
├── exception/
│   ├── DuplicateResourceException.java
│   ├── InvalidCredentialsException.java
│   ├── ResourceNotFoundException.java
│   └── TokenValidationException.java
│
├── handler/
│   └── GlobalExceptionHandler.java
│
├── listener/
│   └── UserLifecycleConsumer.java    # Kafka consumer for user lifecycle events
│
├── model/
│   ├── Organization.java             # JPA entity — organization/tenant
│   ├── OrganizationInvite.java       # JPA entity — pending org invitation
│   └── User.java                     # JPA entity — user profile
│
├── repository/
│   ├── OrganizationInviteRepository.java
│   ├── OrganizationRepository.java
│   └── UserRepository.java
│
├── saga/
│   ├── activity/
│   │   ├── UserRegistrationActivities.java      # Temporal activity interface
│   │   └── UserRegistrationActivitiesImpl.java  # Sends OTP via Kafka, verifies OTP
│   └── workflow/
│       ├── UserRegistrationWorkflow.java         # Temporal workflow interface
│       └── UserRegistrationWorkflowImpl.java     # Orchestrates OTP verification flow
│
└── service/
    ├── AuthService.java                # Login, registration, OTP, token refresh logic
    ├── KeycloakAdminClientService.java # Keycloak Admin REST API client (user CRUD in Keycloak)
    └── OrganizationService.java        # Organization management, invite flow
```

---

## Authentication Flow

```
POST /api/v1/usermanagement/register
  → AuthService.register()
  → Creates user in local DB
  → Creates user in Keycloak via KeycloakAdminClientService
  → Starts UserRegistrationWorkflow (Temporal) — sends OTP via Kafka → notification-service

POST /api/v1/usermanagement/verify-otp
  → AuthService.verifyOtp()
  → Signals Temporal workflow with OTP result
  → On success: enables user in Keycloak

POST /api/v1/usermanagement/login
  → AuthService.login()
  → Authenticates via Keycloak token endpoint
  → Returns JWT access token + refresh token

POST /api/v1/usermanagement/refresh
  → Exchanges refresh token with Keycloak for new access token

POST /api/v1/usermanagement/forgot-password
  → Generates reset token → publishes to 'password-reset-notifications' Kafka topic → notification-service sends email

POST /api/v1/usermanagement/reset-password
  → Validates token → updates password in Keycloak
```

---

## REST API Endpoints

| Method | Path | Description |
|---|---|---|
| `POST` | `/api/v1/usermanagement/register` | Register a new user |
| `POST` | `/api/v1/usermanagement/verify-otp` | Verify OTP for account activation |
| `POST` | `/api/v1/usermanagement/login` | Login and get JWT |
| `POST` | `/api/v1/usermanagement/refresh` | Refresh access token |
| `POST` | `/api/v1/usermanagement/forgot-password` | Request password reset email |
| `POST` | `/api/v1/usermanagement/reset-password` | Reset password with token |
| `GET` | `/api/v1/users/{id}` | Get user profile |
| `PUT` | `/api/v1/users/{id}` | Update user profile |
| `GET` | `/api/v1/organizations` | List organizations |
| `POST` | `/api/v1/organizations` | Create organization |
| `POST` | `/api/v1/organizations/{id}/invite` | Invite user to organization |

---

## Kafka Topics Produced

| Topic | Purpose |
|---|---|
| `otp-notifications` | Send OTP code to notification-service |
| `password-reset-notifications` | Send password reset link to notification-service |

---

## Database Schema

**PostgreSQL DB**: `cinebook_user_management_db`

### Tables
- `user` — user profile (id, email, name, mobileNumber, keycloakId, tenantId, status)
- `organization` — organization/tenant (id, name, tenantId, ownerId)
- `organization_invite` — pending invites (id, organizationId, inviteeEmail, status, token)

---

## External Dependencies

| Dependency | Purpose |
|---|---|
| Keycloak | User CRUD in identity provider, JWT issuance |
| Kafka | Produces `otp-notifications`, `password-reset-notifications` |
| Redis | Session/profile caching |
| PostgreSQL | Persistence (`cinebook_user_management_db`) |
| Temporal | UserRegistrationWorkflow (OTP verification saga) |
| `notification-service` | Receives Kafka events → sends emails/SMS |
