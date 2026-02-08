# II.5 Integration Detailed Design

[< Back to Index](../DAB_Payment_SAGA_Platform.md) | [← Previous: II.4 Detailed Design](05-detailed-design.md)

---

## Message Specifications

| Attribute              | Specification                                                                |
| ---------------------- | ---------------------------------------------------------------------------- |
| **Format**             | JSON over Apache Kafka                                                       |
| **Serialization**      | Jackson ObjectMapper with Java 8 Date/Time module                            |
| **Compression**        | LZ4 (Kafka producer-level)                                                   |
| **Schema Evolution**   | Backward-compatible (additive fields only)                                   |
| **Idempotency**        | Dual-layer: HTTP `Idempotency-Key` header + Kafka `eventId` deduplication    |
| **Ordering**           | Per-partition ordering, keyed by `order_id`                                  |
| **Delivery Guarantee** | At-least-once (Kafka) + application-level exactly-once (idempotency service) |

## Kafka Topics

| Topic                            | Partitions | Key      | Purpose                                   | Retention |
| -------------------------------- | ---------- | -------- | ----------------------------------------- | --------- |
| `webhook.payment.events`         | 12         | order_id | Webhook events from PSPs via CDC outbox   | 7 days    |
| `webhook.payment.events.DLT`     | 3          | order_id | Dead letter for failed webhook processing | 30 days   |
| `payment.order.validated`        | 6          | order_id | Order validation domain events            | 3 days    |
| `payment.inventory.reserved`     | 6          | order_id | Inventory reservation domain events       | 3 days    |
| `payment.payment.authorized`     | 6          | order_id | Payment authorization domain events       | 3 days    |
| `payment.payment.captured`       | 6          | order_id | Payment capture domain events             | 3 days    |
| `payment.order.completed`        | 6          | order_id | Order completion domain events            | 3 days    |
| `payment.compensation.triggered` | 6          | order_id | Compensation trigger events               | 7 days    |

## Message Body Example

**Webhook Kafka Event (CDC Outbox → Kafka):**

```json
{
  "id": "evt_3PkS2M0B5P1a2LcHmC4Qi5",
  "eventId": "evt_3PkS2M0B5P1a2LcHmC4Qi5",
  "eventType": "PAYMENT_CONFIRMED",
  "provider": "STRIPE",
  "orderId": "ORD-001",
  "authorizationId": "auth_1MqLDe2eZvKYlo2CkBQDQ5",
  "captureId": null,
  "amount": 10000,
  "currency": "usd",
  "rawPayload": {
    "id": "evt_3PkS2M0B5P1a2LcHmC4Qi5",
    "type": "payment_intent.succeeded",
    "data": {
      "object": {
        "id": "pi_1MqLDe2eZvKYlo2CkBQDQ5",
        "metadata": { "order_id": "ORD-001" },
        "amount": 10000,
        "currency": "usd"
      }
    }
  },
  "timestamp": "2026-02-08T07:00:00Z"
}
```

## REST API Specification

The platform exposes **54 REST endpoints** across 13 controllers in 5 microservices. The full specification is available in two formats:

- **[OpenAPI 3.0.3 YAML](openapi.yaml)** — Machine-readable specification for code generation, Swagger UI, and API gateway configuration
- **[REST API Reference](openapi-reference.md)** — Human-readable rendered documentation with endpoint tables, response codes, and schema summaries

### API Summary

| Service               | Port | Controller                  | Endpoints | Auth                        |
| --------------------- | ---- | --------------------------- | --------- | --------------------------- |
| **Orchestrator**      | 9090 | PaymentController           | 5         | OAuth2 / JWT                |
|                       |      | AuditController             | 5         | JWT                         |
|                       |      | ReconciliationController    | 10        | JWT                         |
| **Order Service**     | 8081 | OrderController             | 3         | Internal                    |
| **Inventory Service** | 8082 | InventoryController         | 2         | Internal                    |
| **Payment Gateway**   | 8083 | PaymentGatewayController    | 4         | Internal                    |
|                       |      | WebhookController           | 1         | Signature verification      |
| **Open Banking**      | 8084 | ConsentController           | 5         | OAuth2 / JWT                |
|                       |      | TppController               | 7         | OAuth2 / JWT                |
|                       |      | AccountInfoController       | 4         | OAuth2 / JWT (Tier 1-3)     |
|                       |      | PaymentInitiationController | 3         | OAuth2 / JWT (Tier 3 + SCA) |
|                       |      | NapasIntegrationController  | 4         | Internal                    |
|                       |      | KongAuditController         | 1         | Internal                    |

### Key Endpoints

```bash
# Initiate a payment SAGA workflow
POST /api/v1/payments

# Get payment status / result
GET  /api/v1/payments/{workflowId}
GET  /api/v1/payments/{workflowId}/result

# Cancel a payment
POST /api/v1/payments/{workflowId}/cancel

# Receive PSP webhook (Stripe, PayPal, Adyen, Square)
POST /api/webhooks/{channel}

# Open Banking — Consent flow
POST /open-banking/v1/consents
POST /open-banking/v1/consents/{consentId}/authorize

# Open Banking — Payment initiation (Tier 3)
POST /open-banking/v1/payments
POST /open-banking/v1/payments/{paymentId}/confirm
```

### Security Schemes

| Scheme       | Type                                | Usage                       |
| ------------ | ----------------------------------- | --------------------------- |
| `bearerAuth` | HTTP Bearer (JWT)                   | All authenticated endpoints |
| `oauth2`     | OAuth 2.0 Authorization Code + PKCE | External-facing APIs        |

**OAuth2 Scopes:** `payment:create`, `payment:read`, `payment:cancel`, `payment:refund`, `consent:create/read/authorize/revoke`, `tpp:register/read/admin/credentials`, `openbanking:ais`, `openbanking:pis`

---

**Previous:** [← II.4 Detailed Design](05-detailed-design.md) | **Next:** [II.6 Infrastructure Design →](07-infrastructure-design.md)
