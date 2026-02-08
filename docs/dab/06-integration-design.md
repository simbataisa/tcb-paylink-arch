# II.5 Integration Detailed Design

[< Back to Index](../DAB_Payment_SAGA_Platform.md) | [← Previous: II.4 Detailed Design](05-detailed-design.md)

---

## Message Specifications

| Attribute | Specification |
|---|---|
| **Format** | JSON over Apache Kafka |
| **Serialization** | Jackson ObjectMapper with Java 8 Date/Time module |
| **Compression** | LZ4 (Kafka producer-level) |
| **Schema Evolution** | Backward-compatible (additive fields only) |
| **Idempotency** | Dual-layer: HTTP `Idempotency-Key` header + Kafka `eventId` deduplication |
| **Ordering** | Per-partition ordering, keyed by `order_id` |
| **Delivery Guarantee** | At-least-once (Kafka) + application-level exactly-once (idempotency service) |

## Kafka Topics

| Topic | Partitions | Key | Purpose | Retention |
|---|---|---|---|---|
| `webhook.payment.events` | 12 | order_id | Webhook events from PSPs via CDC outbox | 7 days |
| `webhook.payment.events.DLT` | 3 | order_id | Dead letter for failed webhook processing | 30 days |
| `payment.order.validated` | 6 | order_id | Order validation domain events | 3 days |
| `payment.inventory.reserved` | 6 | order_id | Inventory reservation domain events | 3 days |
| `payment.payment.authorized` | 6 | order_id | Payment authorization domain events | 3 days |
| `payment.payment.captured` | 6 | order_id | Payment capture domain events | 3 days |
| `payment.order.completed` | 6 | order_id | Order completion domain events | 3 days |
| `payment.compensation.triggered` | 6 | order_id | Compensation trigger events | 7 days |

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
        "metadata": {"order_id": "ORD-001"},
        "amount": 10000,
        "currency": "usd"
      }
    }
  },
  "timestamp": "2026-02-08T07:00:00Z"
}
```

## API Specification

| # | Service | Method | Endpoint | Description |
|---|---|---|---|---|
| 1 | Orchestrator | POST | `/api/v1/payments` | Initiate payment SAGA workflow |
| 2 | Orchestrator | GET | `/api/v1/payments/{orderId}` | Query payment status |
| 3 | Orchestrator | POST | `/api/v1/payments/{orderId}/cancel` | Cancel running workflow |
| 4 | Order Service | POST | `/api/v1/orders/validate` | Validate order request |
| 5 | Inventory Service | POST | `/api/v1/inventory/reserve` | Reserve inventory |
| 6 | Payment Gateway | POST | `/api/v1/payments/authorize` | Authorize payment |
| 7 | Payment Gateway | POST | `/api/webhooks/{provider}` | Receive PSP webhooks |
| 8 | Open Banking | POST | `/api/v1/open-banking/payments` | Initiate payment (TPP) |

---

**Previous:** [← II.4 Detailed Design](05-detailed-design.md) | **Next:** [II.6 Infrastructure Design →](07-infrastructure-design.md)
