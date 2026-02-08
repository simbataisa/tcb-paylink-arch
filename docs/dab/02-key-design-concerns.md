# II. PROPOSED SOLUTION

[< Back to Index](../DAB_Payment_SAGA_Platform.md)

---

## II.1 Key Design Concerns

### Concern 1: Workflow vs State Consistency

**Challenge:** Maintaining consistency between Temporal workflow state (which step are we executing?) and business domain state (what business state is this payment in?).

**Decision:** Hybrid architecture separating concerns:
- **Temporal** manages workflow orchestration: step ordering, durability, retries, timeouts, compensation flow
- **Spring State Machine** manages business state: `PENDING → VALIDATING → VALIDATED → ... → COMPLETED`

**Rationale:** Temporal's replay mechanism can re-execute activities, but business state transitions must be idempotent and auditable. The state machine provides guard conditions and publishes domain events on transitions.

### Concern 2: Event Delivery Guarantee

**Challenge:** Ensuring exactly-once event delivery from application database to Kafka.

**Decision:** Transactional Outbox + Debezium CDC (< 10ms latency), replacing traditional polling.

**Rationale:** CDC captures events directly from PostgreSQL WAL, eliminating polling overhead and achieving sub-10ms latency. The outbox write is part of the business transaction, guaranteeing atomicity.

### Concern 3: Compensation Reliability

**Challenge:** Ensuring all compensation actions execute even when individual compensations fail.

**Decision:** LIFO compensation stack with continue-on-failure semantics. Failed compensations log errors but do not prevent remaining compensations. Dead Letter Topic (DLT) captures unrecoverable failures for manual intervention.

### Concern 4: Horizontal Scaling

**Challenge:** Scaling beyond single-worker throughput limits.

**Decision:** Customer-hash sharding with priority-based routing across 36 task queues (4 CRITICAL + 8 HIGH + 16 NORMAL + 8 LOW). Formula: `shard_id = abs(customerId.hashCode()) % shardCount`.

### Concern 5: Multi-Tenancy Data Isolation

**Challenge:** Preventing cross-tenant data access in a shared infrastructure.

**Decision:** PostgreSQL Row-Level Security (RLS) policies with `tenant_id` column and `SET LOCAL` session variables. RLS is enforced at the database level, making it transparent to the application layer.

### Concern 6: Gateway Layering

**Challenge:** Separating external (north-south) and internal (east-west) traffic concerns.

**Decision:** Layered gateway architecture:
- **Kong** handles external traffic: JWT authentication, rate limiting, security headers, correlation ID injection
- **Istio** handles internal traffic: mTLS encryption, authorization policies, circuit breaking, distributed tracing

---

**Previous:** [← I. Business Context](01-business-context.md) | **Next:** [II.2 High-level Architecture →](03-high-level-architecture.md)
