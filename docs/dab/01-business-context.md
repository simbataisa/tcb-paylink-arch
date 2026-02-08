# I. BUSINESS CONTEXT

[< Back to Index](../DAB_Payment_SAGA_Platform.md)

---

## 1.1 Introduction / Overview

The **Payment SAGA Platform** is an enterprise-grade distributed payment processing system designed for bank-wide payment orchestration. The platform addresses the fundamental challenges of distributed transactions in financial systems by implementing a **hybrid SAGA pattern** that combines:

- **Temporal** for durable workflow orchestration — managing step ordering, retries, timeouts, and compensation flows
- **Spring State Machine** for fine-grained business state modeling — enforcing state transition guards, publishing domain events, and maintaining audit trails

The platform is deployed as **true microservices** with database-per-service isolation:

| Service | Responsibility | Port |
|---|---|---|
| **Order Service** | Order lifecycle management, validation | 8081 |
| **Inventory Service** | Stock management, reservation | 8082 |
| **Payment Gateway Service** | Payment processing, webhook handling | 8083 |
| **SAGA Orchestrator** | Workflow orchestration, state machine | 9090 |
| **Open Banking API** | SBV Circular 64 compliance, TPP access | 8085 |

The system integrates with external Payment Service Providers (Stripe, PayPal, Adyen, Square) via verified webhooks, processes events through a CDC-based outbox pattern with sub-10ms latency, and provides Open Banking APIs compliant with SBV Circular 64/2024/TT-NHNN.

## 1.2 Problem Statements

| # | Problem | Impact |
|---|---|---|
| **PS-1** | Monolithic payment systems cannot scale beyond single-digit thousands TPS; vertical scaling hits hardware limits | Bank-wide payment processing requires horizontal scalability across multiple payment channels and products |
| **PS-2** | Distributed transactions across order, inventory, and payment domains need guaranteed compensation to maintain data consistency | Failed partial transactions without proper rollback lead to financial discrepancies and reconciliation overhead |
| **PS-3** | External payment webhooks (Stripe, PayPal, Adyen, Square) require exactly-once processing to prevent duplicate charges or missed confirmations | Webhook delivery is at-least-once by design; without idempotency and deduplication, customers may be charged multiple times |
| **PS-4** | SBV Circular 64/2024/TT-NHNN mandates Open Banking APIs with TPP tiering, consent management, and audit trails by March 2027 | Non-compliance results in regulatory penalties; banks must expose payment initiation and account information APIs to licensed TPPs |
| **PS-5** | Multi-tenant payment processing requires data isolation at the database level to prevent cross-tenant data leakage | Shared database schemas without row-level isolation create compliance risk for PCI-DSS and data sovereignty requirements |

## 1.3 Objectives & Goals

| # | Objective | Target | Timeline |
|---|---|---|---|
| **OBJ-1** | Achieve 500 TPS sustained throughput (Phase 1) with a scaling path to 10K+ TPS | 500 TPS → 10K+ TPS | Phase 1: Month 1-3; Phase 5: Month 6+ |
| **OBJ-2** | Implement SAGA pattern with automatic LIFO compensation across all distributed steps | 100% compensation coverage, <5s compensation time | Month 1 |
| **OBJ-3** | Integrate 4 payment gateway providers with webhook signature verification | Stripe, PayPal, Adyen, Square | Month 2 |
| **OBJ-4** | Achieve 99.95% availability with zero-downtime deployments | <26.3 min downtime/year | Ongoing |
| **OBJ-5** | Achieve PCI-DSS 4.0.1 compliance and SBV Circular 64 compliance | Full audit trail, data encryption, TPP tiering | Month 3-6 |
| **OBJ-6** | Sub-10ms CDC event delivery latency from outbox to Kafka | <10ms P99 | Month 1 |
| **OBJ-7** | 413+ automated tests with >80% code coverage | 413 tests across 32+ test classes | Ongoing |

## 1.4 Scopes (In/Out)

### In Scope

| # | Scope Item | Description |
|---|---|---|
| 1 | SAGA Orchestration | Temporal-based workflow orchestration with Spring State Machine for business state |
| 2 | 4 Microservices | Order, Inventory, Payment Gateway, Orchestrator with database-per-service |
| 3 | CDC Outbox Pattern | Debezium-based Change Data Capture for reliable event publishing (<10ms latency) |
| 4 | Webhook Integration | Stripe, PayPal, Adyen, Square webhook processing with signature verification |
| 5 | Open Banking Module | SBV Circular 64 compliant APIs with TPP tiering, consent management, SCA |
| 6 | K8s/EKS Deployment | Kubernetes manifests with Kong Ingress, Istio Service Mesh, HPA auto-scaling |
| 7 | Observability Stack | Prometheus metrics, Grafana dashboards, Zipkin distributed tracing |
| 8 | Priority-based Sharding | Customer-hash sharding across 36 task queues with 4 priority levels |

### Out of Scope

| # | Scope Item | Rationale |
|---|---|---|
| 1 | Core Banking Integration (T24) | Separate integration project; will consume Payment SAGA APIs |
| 2 | Mobile Frontend | Separate frontend project; will use REST APIs |
| 3 | Customer IdP | Existing enterprise IAM system will be integrated via OAuth2 |
| 4 | Card Tokenization Vault | Will use PSP-provided tokenization (Stripe, Adyen) |
| 5 | ISO 20022 Messaging | Phase 2 scope; requires core banking integration |
| 6 | Multi-Region Active-Active | Phase 3 scope; requires Cassandra persistence migration |

## 1.5 Requirements

### Functional Requirements

| ID | Requirement | Priority |
|---|---|---|
| **FR-01** | Process payment transactions through a 5-step SAGA workflow (Validate → Reserve → Authorize → Capture → Complete) | CRITICAL |
| **FR-02** | Automatically compensate failed transactions in LIFO order (Refund → Void → Release → Cancel) | CRITICAL |
| **FR-03** | Receive and process webhooks from 4 PSPs with signature verification (HMAC-SHA256, RSA-SHA256) | HIGH |
| **FR-04** | Publish domain events via CDC outbox pattern to Kafka for downstream consumers | HIGH |
| **FR-05** | Route payments to priority-based sharded task queues based on customer ID hash and transaction amount | HIGH |
| **FR-06** | Expose Open Banking APIs for payment initiation and account information per SBV Circular 64 | HIGH |
| **FR-07** | Maintain immutable audit trail of all state transitions and compensation actions | HIGH |
| **FR-08** | Support idempotent request processing with dual-layer deduplication (HTTP + Kafka) | MEDIUM |

### Non-Functional Requirements

| ID | Requirement | Target |
|---|---|---|
| **NFR-01** | Throughput | 500 TPS sustained (Phase 1), 10K+ TPS (Phase 5) |
| **NFR-02** | Latency | P99 <500ms for payment processing, P99 <10ms for CDC event delivery |
| **NFR-03** | Availability | 99.95% uptime (26.3 min downtime/year) |
| **NFR-04** | Security — Transport | mTLS STRICT mode across all service-to-service communication |
| **NFR-05** | Test Coverage | >80% code coverage, 413+ automated tests |
| **NFR-06** | Deployment | Zero-downtime rolling updates with PodDisruptionBudget |
| **NFR-07** | Audit Retention | 7-year retention for financial transactions, trigger-protected immutability |
| **NFR-08** | PCI-DSS Compliance | PCI-DSS 4.0.1 compliance for payment data handling |
| **NFR-09** | Regulatory Compliance | SBV Circular 64/2024/TT-NHNN Open Banking compliance by March 2027 |

---

**Next:** [II.1 Key Design Concerns →](02-key-design-concerns.md)
