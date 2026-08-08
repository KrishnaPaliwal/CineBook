# CineBook: Low-Level Design — notification-service

## Overview

`notification-service` is a pure consumer service. It listens to multiple Kafka topics and sends **Email** and **SMS** notifications for various system events. It also implements **idempotent processing** to prevent duplicate notifications.

Port: **7083** (local Kubernetes, via port-forward)

---

## Package Structure

```
com.cinebook.notificationservice
├── config/
│   ├── KafkaConsumerConfig.java      # Two listener container factories: Avro + String
│   ├── SecurityConfig.java           # Spring Security / OAuth2 Resource Server
│   └── TenantDataSourceConfig.java   # Multi-tenant datasource routing
│
├── dto/
│   ├── BookingNotification.java
│   ├── OtpNotification.java          # OTP email/SMS payload
│   ├── PasswordResetNotification.java
│   └── UserProfileDTO.java           # User profile fetched from user-management-service
│
├── entity/
│   └── SentNotification.java         # JPA entity — notification audit + dedup record
│
├── exception/
│   ├── GlobalExceptionHandler.java
│   ├── NotificationFailedException.java
│   └── UserProfileNotFoundException.java
│
├── listener/
│   └── NotificationConsumer.java     # All Kafka consumers (booking-events, payment-events, otp, password-reset)
│
├── repository/
│   └── SentNotificationRepository.java
│
└── service/
    ├── EmailService.java             # SMTP email sending
    ├── SmsService.java               # SMS sending
    └── UserServiceClient.java        # HTTP client to user-management-service for profile lookup
```

---

## Kafka Consumers

`NotificationConsumer` handles **four** Kafka topics:

| Topic | Consumer Factory | Event Types Handled |
|---|---|---|
| `booking-events` | `avroKafkaListenerContainerFactory` | `BookingCreatedEvent`, `BookingConfirmedEvent`, `BookingCancelledEvent` (Avro) |
| `payment-events` | `stringKafkaListenerContainerFactory` | `PAYMENT_CHARGED`, `PAYMENT_FAILED` (JSON) |
| `otp-notifications` | `stringKafkaListenerContainerFactory` | OTP delivery (JSON) |
| `password-reset-notifications` | `stringKafkaListenerContainerFactory` | Password reset link (JSON) |

---

## Notification Processing Logic

### BookingCreatedEvent
- Fetches user profile from `user-management-service` via HTTP
- Sends: *"Your booking is created, please complete payment"* email + SMS

### BookingConfirmedEvent
- Fetches user profile from `user-management-service` via HTTP
- Sends: *"Your booking is confirmed!"* email + SMS

### BookingCancelledEvent
- Resolves recipient from `sent_notifications` DB (no userId in cancellation event)
- Sends: *"Your booking has been cancelled"* email

### OTP Notification
- Sends OTP code via email + SMS (if mobile number present)

### Password Reset Notification
- Sends password reset token via email

---

## Idempotency (Deduplication)

Every notification is deduplicated using a `SentNotification` record:

1. Before processing, check if `eventId` already exists in `sent_notifications` with status `SUCCESS`
2. If duplicate → skip
3. If retry (status `FAILED` or `PENDING`) → allow reprocessing
4. On success → mark as `SUCCESS` with `processedAt` timestamp
5. On failure → mark as `FAILED`

---

## Database Schema

**Shared PostgreSQL** (no dedicated DB URL configured)

### Tables
- `sent_notification` — notification audit log
  - `id`, `event_id` (dedup key), `booking_id`, `recipient` (email), `notification_type`, `status`, `tenant_id`, `correlation_id`, `created_at`, `processed_at`

---

## External Dependencies

| Dependency | Purpose |
|---|---|
| `user-management-service` | HTTP call to fetch user profile (email, name, mobile) |
| Kafka | Consumes `booking-events` (Avro), `payment-events`, `otp-notifications`, `password-reset-notifications` |
| SMTP server | Email delivery (`EmailService`) |
| SMS gateway | SMS delivery (`SmsService`) |
| PostgreSQL | `sent_notifications` table |
| Apicurio Schema Registry | Avro schema deserialization for `booking-events` |
| Keycloak | JWT validation |

---

## Configuration

```yaml
# notification-service values-notification-local.yaml
MANAGEMENT_HEALTH_MAIL_ENABLED: "false"   # Disable mail health check in local env
```
