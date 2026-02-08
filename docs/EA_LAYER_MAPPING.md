# Enterprise Architecture Layer Mapping

This document maps the Payment SAGA Platform components to the bank-wide Enterprise Architecture layers.

## EA Layer Overview

| EA Layer | Description | Maven Modules | K8s Component Label |
|---|---|---|---|
| **Payment Product** | Customer-facing products (Open Banking, TPP) | `open-banking-api` | `payment-product` |
| **Payment Orchestrator** | SAGA workflow orchestration (Temporal + State Machine) | `payment-saga-orchestrator` | `payment-orchestrator` |
| **Payment Core** | Domain services: order validation, inventory, payment logic | `order-service`, `inventory-service` | `payment-core` |
| **Financial Gateways** | External PSP integration, webhook processing | `payment-gateway-service` | `financial-gateway` |

## Shared Infrastructure

| Module | Purpose |
|---|---|
| `payment-saga-common` | Shared utilities, domain events, outbox infrastructure |
| `order-service-api` | Order service API contracts (DTOs, interfaces) |
| `inventory-service-api` | Inventory service API contracts |
| `payment-gateway-service-api` | Payment gateway API contracts |

## K8s Directory Structure

```
k8s/base/
  platform/                      # Shared infra (namespace, config, secrets, RBAC, service accounts)
  payment-product/               # EA Layer: Payment Product
  payment-orchestrator/          # EA Layer: Payment Orchestrator
  payment-core/                  # EA Layer: Payment Core
  financial-gateways/            # EA Layer: Financial Gateways
  kong/                          # API Gateway
  istio/                         # Service Mesh
  kafka-connect/                 # CDC Connectors
```

## K8s Label Convention

All deployments and service accounts use the `app.kubernetes.io/component` label aligned to EA layers:

| Service | `app.kubernetes.io/component` |
|---|---|
| `payment-saga-orchestrator` | `payment-orchestrator` |
| `order-service` | `payment-core` |
| `inventory-service` | `payment-core` |
| `payment-gateway-service` | `financial-gateway` |
| `open-banking-api` | `payment-product` |

## Component Status

| EA Component | Status | Module/Package |
|---|---|---|
| Open Banking API (SBV Circular 64) | Implemented | `open-banking-api` |
| SAGA Orchestrator (Temporal + State Machine) | Implemented | `payment-saga-orchestrator` |
| Order Validation | Implemented | `order-service` |
| Inventory Reservation | Implemented | `inventory-service` |
| PSP Gateway (Stripe, PayPal, Adyen, Square) | Implemented | `payment-gateway-service` |
| Webhook → Kafka Pipeline (CDC Outbox) | Implemented | `payment-gateway-service` |
| Fraud Detection | Placeholder | `com.payment.saga.fraud` |
| FX Conversion | Placeholder | `com.payment.saga.fx` |
| Clearing | Placeholder | `com.payment.saga.clearing` |
| Network Management | Placeholder | `com.payment.gateway.network` |
| Clearing Limit | Placeholder | `com.payment.gateway.clearing` |

## Design Notes

### Dual WebhookIdempotencyService

Both `payment-gateway-service` and `payment-saga-orchestrator` contain a `WebhookIdempotencyService`. This is intentional **defense-in-depth**, not redundancy:

- **Payment Gateway** (`webhook/kafka/`): Ensures each webhook is written to the outbox exactly once (producer-side dedup)
- **Orchestrator** (`consumer/service/`): Ensures each Kafka event signals a workflow exactly once (consumer-side dedup)

These operate at different layers with different deduplication keys and serve complementary purposes.
