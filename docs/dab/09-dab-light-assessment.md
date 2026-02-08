# III. DAB LIGHT ASSESSMENT

[< Back to Index](../DAB_Payment_SAGA_Platform.md) | [← Previous: II.7 Security Design](08-security-design.md)

---

## Application and Software

| Criterion | Assessment |
|---|---|
| **Architecture Pattern** | Hybrid SAGA: Temporal (workflow orchestration) + Spring State Machine (business state). 9 Maven modules with strict dependency rules. |
| **Technology Maturity** | Java 21 LTS (8+ year support), Spring Boot 3.2.1 (GA), Temporal 1.22.3 (production-proven at Uber, Netflix, Snap). All components are GA releases. |
| **Code Quality** | 703 automated tests (34 skipped) across 32+ test classes. Categories: Workflow (12), Activity (25), State Machine (20), Service (75), Repository (51), Outbox (23), Webhook Kafka (21), Integration (20), Config (24), and EA domain tests. TestContainers for real database/Kafka testing. |
| **Build & Deployment** | Maven multi-module build. Docker images per service. Kubernetes manifests with Kustomize overlays (base, EKS). Zero-downtime rolling updates with PodDisruptionBudget. |
| **State Management** | Temporal manages workflow state (durable, replay-safe). Spring State Machine manages business state (PENDING → COMPLETED, 10 states). Minimal workflow state pattern — only IDs stored in workflow, full data in database. |

## Software Integration

| Integration Type | Technology | Details |
|---|---|---|
| **Synchronous (REST)** | Spring Cloud OpenFeign | Profile-based service discovery: direct URLs (local), Docker DNS (docker), K8s DNS (k8s/eks), Mesh-aware (istio) |
| **Asynchronous (Events)** | Apache Kafka + Debezium CDC | Transactional outbox → PostgreSQL WAL → Debezium → Kafka. <10ms latency. 8 topics, 12 partitions max. |
| **Workflow (gRPC)** | Temporal Server | Durable workflow execution, signal/query/cancel. gRPC communication between workers and Temporal server. |
| **External (Webhooks)** | 4 PSPs: Stripe, PayPal, Adyen, Square | Signature verification per provider, CDC outbox for reliable processing, consumer idempotency with Redis. |

## Security Design

| Layer | Controls |
|---|---|
| **Edge** | AWS WAF (OWASP rules) + Shield (DDoS) |
| **Gateway** | Kong: JWT auth, rate limiting (100/min), security headers, correlation ID |
| **Application** | RBAC (@PreAuthorize), webhook signature verification (HMAC-SHA256/RSA-SHA256), Bean Validation |
| **Service Mesh** | Istio: mTLS STRICT, AuthorizationPolicy (service access matrix), circuit breaking |
| **Network** | Namespace isolation, pod-level network policies, B3 header propagation |
| **Data** | PostgreSQL RLS (tenant isolation), AES-256 at rest, TLS 1.2+ in transit, 90-day credential rotation |

## Data Integration

| Aspect | Details |
|---|---|
| **Database Strategy** | Database-per-service: 4 PostgreSQL databases (saga_db, order_db, inventory_db, payment_db) |
| **Event Sourcing** | Transactional outbox pattern with Debezium CDC for guaranteed event delivery |
| **Event Store** | Immutable event log with trigger-protected retention (7 years for financial events) |
| **Schema Migration** | Flyway versioned migrations: saga_db V1,V3–V17; order_db V1–V4; inventory_db V1–V5; payment_db V1–V6. Forward-only. |
| **Data Isolation** | PostgreSQL RLS with `tenant_id`, enforced via `SET LOCAL` session variables |
| **Consistency Model** | Eventual consistency across services, strong consistency within each database |

## Technology Stack and Hardware

| Aspect | Assessment |
|---|---|
| **Open Source** | All components are open source (Java, Spring Boot, Temporal, Kafka, PostgreSQL, Redis, Kong, Istio) or AWS managed services |
| **LTS Support** | Java 21 LTS (Sept 2023 – Sept 2031+), Spring Boot 3.x (commercial support available), PostgreSQL 16 (Nov 2023 – Nov 2028) |
| **Cloud Platform** | AWS EKS (managed Kubernetes), Aurora PostgreSQL Serverless v2, Amazon MSK, ElastiCache Redis |
| **Scaling** | HPA with 8-50 pods (orchestrator), customer-hash sharding across 36 task queues, Aurora ACU auto-scaling (2-64 ACU) |
| **Vendor Lock-in** | LOW — all core components are portable. AWS services (Aurora, MSK, ElastiCache) have OSS equivalents (PostgreSQL, Kafka, Redis) |

## Complexity Criteria

| Complexity | Area | Justification |
|---|---|---|
| **HIGH** | SAGA Compensation | LIFO compensation stack with continue-on-failure semantics across 4 services. Partial compensation handling with DLT escalation. |
| **HIGH** | CDC Exactly-Once Delivery | Debezium WAL capture → Kafka → Consumer with dual-layer idempotency. Requires correct PostgreSQL replication slot management. |
| **MEDIUM** | RLS Multi-Tenancy | PostgreSQL Row-Level Security policies per table. Requires careful session variable management and testing. |
| **MEDIUM** | Open Banking Compliance | SBV Circular 64 TPP tiering, consent management, SCA. Regulatory requirements still evolving. |
| **MEDIUM** | Priority-Based Sharding | Customer-hash sharding across 36 task queues. Hot-spot mitigation for VIP customers. |
| **LOW** | CRUD Operations | Standard REST CRUD within individual services. Well-established Spring Boot patterns. |

## Offline Stakeholder Alignment

| Stakeholder Team | Alignment Topic | Status |
|---|---|---|
| **Architecture** | Hybrid SAGA pattern (Temporal + State Machine), microservice boundaries, module dependency rules | PENDING |
| **Security** | 6-layer defense-in-depth, PCI-DSS 4.0.1 controls, RLS multi-tenancy, webhook signature verification | PENDING |
| **DBA** | Database-per-service strategy, Aurora Serverless v2 sizing, Flyway migration management, RLS policies | PENDING |
| **Infrastructure** | EKS cluster topology, HPA configuration, Aurora/MSK/ElastiCache sizing, Temporal cluster deployment | PENDING |
| **Compliance** | SBV Circular 64 Open Banking requirements, PCI-DSS 4.0.1 audit controls, 7-year retention policy | PENDING |
| **Operations** | Observability stack (Prometheus/Grafana/Zipkin), alerting rules, runbook for DLT/compensation failures | PENDING |

---

*End of DAB Document — Payment SAGA Platform v1.0*
