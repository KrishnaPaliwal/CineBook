# CineBook: Low-Level Design — payment-service

## Overview

`payment-service` handles all payment processing for CineBook. It integrates with **Razorpay** as the payment gateway and participates in the booking saga via Kafka.

Port: **7085** (local Kubernetes, via port-forward)

---

## Package Structure

```
com.cinebook.paymentservice
├── config/
│   ├── AppConfig.java                # ObjectMapper, general config
│   ├── KafkaConfig.java              # Kafka consumer/producer setup
│   ├── OpenApiConfig.java            # Swagger/OpenAPI config
│   ├── RateLimitingInterceptor.java  # API rate limiting
│   ├── SecurityConfig.java           # Spring Security / OAuth2 Resource Server
│   ├── TenantDataSourceConfig.java   # Multi-tenant datasource routing
│   └── WebMvcConfig.java             # CORS config
│
├── controller/
│   └── PaymentController.java        # REST endpoints for payment initiation & webhook
│
├── dto/
│   ├── BookingStatusUpdateRequest.java
│   ├── InitiatePaymentRequest.java
│   ├── PaymentResponse.java
│   └── RazorpayWebhookRequest.java   # Webhook payload from Razorpay
│
├── entity/
│   ├── Payment.java                  # JPA entity — payment record
│   └── PaymentOutboxEvent.java       # Outbox table entity for reliable publishing
│
├── exception/
│   ├── GlobalExceptionHandler.java
│   └── PaymentNotFoundException.java
│
├── messaging/
│   ├── PaymentCommandListener.java   # Kafka: consumes 'payment-commands' from saga
│   └── PaymentEventPublisher.java    # Publishes to 'payment-events'
│
├── repository/
│   ├── PaymentRepository.java
│   └── PaymentOutboxRepository.java
│
├── scheduler/
│   └── PaymentOutboxScheduler.java   # Polls outbox table and publishes pending events
│
└── service/
    └── PaymentService.java           # Core payment business logic
```

---

## Payment Saga Flow

`payment-service` is a **saga participant** — it does not orchestrate; it reacts to commands from `booking-service`'s embedded Temporal saga.

```
Kafka: payment-commands (from booking-service saga activity)
  ↓
PaymentCommandListener.consume()
  ├── If commandType = "CHARGE_PAYMENT":
  │     ├── Look up Payment by bookingId in DB
  │     ├── If payment not found AND auto-approve=true → create mock payment → publish PAYMENT_CHARGED
  │     ├── If payment status = SUCCESS (Razorpay webhook already completed) → publish PAYMENT_CHARGED
  │     ├── If payment status = PENDING AND auto-approve=true → update to SUCCESS → publish PAYMENT_CHARGED
  │     └── If payment status = PENDING (real flow) → wait for Razorpay webhook callback
  └── On exception → publish PAYMENT_FAILED

Razorpay Webhook → PaymentController → update Payment status → publish PAYMENT_CHARGED/FAILED
  ↓
Kafka: payment-events (consumed by booking-service saga signal + notification-service)
```

---

## Outbox Pattern

To ensure at-least-once delivery of Kafka events even if the service crashes:

1. `PaymentEventPublisher` writes events to `payment_outbox_event` table (transactionally with DB update)
2. `PaymentOutboxScheduler` polls the outbox table on a schedule and publishes any pending events to Kafka
3. Once published, outbox records are marked as sent

---

## REST API Endpoints

| Method | Path | Description |
|---|---|---|
| `POST` | `/api/v1/payments/initiate` | Initiate a Razorpay order for a booking |
| `POST` | `/api/v1/payments/webhook` | Razorpay webhook callback (payment result) |
| `GET` | `/api/v1/payments/{bookingId}` | Get payment status for a booking |
| `PATCH` | `/api/v1/payments/{bookingId}/status` | Update booking payment status |

---

## Database Schema

**PostgreSQL DB**: `cinebook_payment_db`

### Tables
- `payment` — payment records (bookingId, amount, status, razorpayOrderId, razorpayPaymentId, tenantId, correlationId)
- `payment_outbox_event` — outbox for reliable Kafka publishing

---

## Configuration Flags

| Property | Default | Purpose |
|---|---|---|
| `payment.saga.auto-approve` | `false` | If `true`, auto-approves PENDING payments (for saga testing without Razorpay) |

---

## External Dependencies

| Dependency | Purpose |
|---|---|
| Razorpay | Payment gateway (webhook-based confirmation) |
| Kafka | Consumes `payment-commands`, produces `payment-events` |
| PostgreSQL | Persistence (`cinebook_payment_db`) |
| Keycloak | JWT validation |
