# Database Selection Guide for Payment Platforms

This document provides detailed guidance on when and how to use different database technologies for payment microservices and event-driven architectures.

---

## Table of Contents

1. [Selection Framework](#selection-framework)
2. [Database Comparison Matrix](#database-comparison-matrix)
3. [PostgreSQL / Aurora](#postgresql--aurora)
4. [DynamoDB](#dynamodb)
5. [ScyllaDB](#scylladb)
6. [MongoDB](#mongodb)
7. [CockroachDB](#cockroachdb)
8. [TigerBeetle](#tigerbeetle)
9. [Use Case Mapping](#use-case-mapping)
10. [Migration Strategies](#migration-strategies)

---

## Selection Framework

### Decision Criteria

When selecting a database for payment workloads, evaluate these factors:

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    DATABASE SELECTION CRITERIA                              │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  1. CONSISTENCY MODEL                                                       │
│     ├── Strong (ACID) → Financial transactions, balances                    │
│     └── Eventual → Audit logs, analytics, non-critical reads                │
│                                                                             │
│  2. ACCESS PATTERNS                                                         │
│     ├── Point queries → Key-value stores (DynamoDB, Redis)                  │
│     ├── Range queries → Time-series (ScyllaDB, TimescaleDB)                 │
│     ├── Complex joins → Relational (PostgreSQL, CockroachDB)                │
│     └── Full-text search → Elasticsearch, OpenSearch                        │
│                                                                             │
│  3. WRITE PATTERNS                                                          │
│     ├── Insert-heavy → Append-only stores (ScyllaDB, DynamoDB)              │
│     ├── Update-heavy → MVCC databases (PostgreSQL, CockroachDB)             │
│     └── Mixed → Depends on transaction requirements                         │
│                                                                             │
│  4. SCALE REQUIREMENTS                                                      │
│     ├── <10K TPS → Single-node PostgreSQL sufficient                        │
│     ├── 10K-100K TPS → Sharded PostgreSQL or distributed DBs                │
│     └── >100K TPS → Purpose-built systems (TigerBeetle, ScyllaDB)           │
│                                                                             │
│  5. OPERATIONAL MODEL                                                       │
│     ├── Managed (AWS) → Aurora, DynamoDB, ElastiCache                       │
│     ├── Self-hosted → PostgreSQL, ScyllaDB, TigerBeetle                     │
│     └── Hybrid → Mix based on criticality                                   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Database Comparison Matrix

### Performance Characteristics

| Database          | Read Latency | Write Latency | Max TPS     | Consistency |
| ----------------- | ------------ | ------------- | ----------- | ----------- |
| PostgreSQL        | 1-5ms        | 1-10ms        | 50K         | Strong      |
| Aurora PostgreSQL | 1-5ms        | 1-5ms         | 200K        | Strong      |
| DynamoDB          | 1-5ms        | 1-5ms         | Unlimited\* | Tunable     |
| ScyllaDB          | <1ms         | <1ms          | 1M+         | Tunable     |
| MongoDB           | 2-10ms       | 2-10ms        | 100K        | Tunable     |
| CockroachDB       | 5-50ms       | 10-100ms      | 100K        | Strong      |
| TigerBeetle       | <1ms         | <1ms          | 1M+         | Strong      |

\*DynamoDB scales automatically with on-demand mode

### Feature Comparison

| Feature           | PostgreSQL    | DynamoDB           | ScyllaDB       | MongoDB        | CockroachDB | TigerBeetle |
| ----------------- | ------------- | ------------------ | -------------- | -------------- | ----------- | ----------- |
| ACID Transactions | ✓             | Limited (25 items) | ✗              | ✓ (4.0+)       | ✓           | ✓           |
| Multi-region      | Aurora Global | Global Tables      | ✓              | ✓              | ✓ (native)  | ✗ (planned) |
| SQL Support       | ✓             | PartiQL            | CQL            | MQL            | ✓           | ✗           |
| JSON/Document     | JSONB         | Native             | ✗              | Native         | JSONB       | ✗           |
| Time-to-Live      | Manual        | Native             | Native         | Native         | Manual      | ✗           |
| Change Capture    | Debezium      | Streams            | CDC            | Change Streams | Changefeeds | ✗           |
| Managed (AWS)     | Aurora/RDS    | Native             | ScyllaDB Cloud | Atlas          | Serverless  | ✗           |

---

## PostgreSQL / Aurora

### When to Use

**IDEAL FOR:**

- ✅ Transactional core (payments, orders, inventory)
- ✅ Complex queries with JOINs
- ✅ ACID compliance requirements
- ✅ Outbox pattern with CDC (Debezium)
- ✅ State machine context storage
- ✅ Multi-table transactions

**NOT IDEAL FOR:**

- ❌ Massive write throughput (>100K TPS)
- ❌ Multi-region active-active writes
- ❌ Time-series at extreme scale
- ❌ Document-heavy workloads

### Aurora vs Standard PostgreSQL

| Factor              | Standard RDS | Aurora              |
| ------------------- | ------------ | ------------------- |
| Throughput          | 1x           | 5x                  |
| Storage Scaling     | Manual       | Automatic (0-128TB) |
| Replication Lag     | 100ms+       | <10ms               |
| Failover Time       | 1-2 min      | <30s                |
| Read Replicas       | 5            | 15                  |
| Global Distribution | ✗            | Aurora Global       |
| Serverless          | ✗            | Serverless v2       |

### Configuration for Payment Workloads

```yaml
# Aurora Serverless v2 sizing for 1K-10K TPS
serverlessv2_scaling_configuration:
  min_capacity: 2 # 4GB RAM - cost-efficient idle
  max_capacity: 64 # 128GB RAM - burst handling

# Connection pooling (HikariCP)
hikari:
  maximum-pool-size: 50 # Match Aurora max connections
  minimum-idle: 20 # Warm connection pool
  connection-timeout: 10000 # Fast fail
  leak-detection-threshold: 60000

# For higher scale, add PgBouncer
# Application → PgBouncer (transaction pooling) → Aurora
```

### Outbox Pattern with Debezium

PostgreSQL is the **best choice** for the transactional outbox pattern because:

1. **WAL-based CDC**: Debezium captures from write-ahead log
2. **Transactional Guarantees**: Business data + outbox event in same transaction
3. **<10ms Latency**: WAL streaming is near real-time
4. **Exactly-once**: Combined with Kafka transactions

```sql
-- Outbox table optimized for CDC
CREATE TABLE outbox_events (
    id BIGSERIAL PRIMARY KEY,
    event_id VARCHAR(36) NOT NULL UNIQUE,
    event_type VARCHAR(100) NOT NULL,
    aggregate_id VARCHAR(100) NOT NULL,
    topic VARCHAR(255) NOT NULL,
    partition_key VARCHAR(100),
    payload JSONB NOT NULL,
    created_at TIMESTAMP NOT NULL DEFAULT NOW()
);

-- BRIN index for time-based cleanup (compact, efficient for append-only)
CREATE INDEX idx_outbox_created_brin ON outbox_events USING BRIN (created_at);
```

---

## DynamoDB

### When to Use

**IDEAL FOR:**

- ✅ High-volume audit logs (append-only)
- ✅ Session/idempotency storage (TTL auto-expiry)
- ✅ Key-value lookups with predictable latency
- ✅ Serverless architectures (pay-per-request)
- ✅ Multi-region with Global Tables
- ✅ Event sourcing (when eventual consistency acceptable)

**NOT IDEAL FOR:**

- ❌ Complex transactions (>25 items)
- ❌ Ad-hoc queries and analytics
- ❌ Strong consistency requirements
- ❌ Relational data with JOINs

### DynamoDB Access Patterns

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    DYNAMODB ACCESS PATTERN DESIGN                           │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  PRIMARY KEY DESIGN:                                                        │
│  ┌─────────────┬─────────────────────────────────────────────────────────┐  │
│  │ Pattern     │ Key Structure                                           │  │
│  ├─────────────┼─────────────────────────────────────────────────────────┤  │
│  │ By Saga     │ PK: "SAGA#saga-123"  SK: "EVENT#2024-01-15T10:30:00Z"   │  │
│  │ By Order    │ PK: "ORDER#order-456" SK: "EVENT#timestamp"             │  │
│  │ By Date     │ PK: "DATE#2024-01-15" SK: "saga-123#timestamp"          │  │
│  └─────────────┴─────────────────────────────────────────────────────────┘  │
│                                                                             │
│  GSI FOR SECONDARY ACCESS:                                                  │
│  ┌─────────────┬─────────────────────────────────────────────────────────┐  │
│  │ GSI1        │ PK: customerId       SK: createdAt                      │  │
│  │ GSI2        │ PK: eventType        SK: createdAt                      │  │
│  └─────────────┴─────────────────────────────────────────────────────────┘  │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Audit Log Schema

```java
@DynamoDbBean
public class AuditLogEntry {
    @DynamoDbPartitionKey
    private String pk;  // "SAGA#saga-123"

    @DynamoDbSortKey
    private String sk;  // "EVENT#2024-01-15T10:30:00Z"

    // GSI for time-range queries
    @DynamoDbSecondaryPartitionKey(indexNames = "ByDate")
    private String gsi1pk;  // "DATE#2024-01-15"

    @DynamoDbSecondarySortKey(indexNames = "ByDate")
    private String gsi1sk;  // "TIME#10:30:00#saga-123"

    private String eventType;
    private String fromState;
    private String toState;
    private Map<String, Object> details;

    // TTL for 7-year expiry
    private Long ttl;  // Unix timestamp
}
```

### Cost Optimization

```yaml
# On-demand mode for variable workloads
billing_mode: PAY_PER_REQUEST

# Provisioned mode for predictable workloads (30-50% cheaper)
billing_mode: PROVISIONED
read_capacity: 1000   # RCU
write_capacity: 500   # WCU
autoscaling:
  min_capacity: 100
  max_capacity: 2000
  target_utilization: 70%
```

---

## ScyllaDB

### When to Use

**IDEAL FOR:**

- ✅ Extreme write throughput (>100K TPS)
- ✅ Time-series data (metrics, events)
- ✅ Audit logs at massive scale
- ✅ Low-latency reads (<1ms P99)
- ✅ Temporal workflow persistence (at scale)

**NOT IDEAL FOR:**

- ❌ ACID transactions
- ❌ Complex queries with JOINs
- ❌ Small-scale deployments (operational overhead)
- ❌ Strong consistency requirements

### ScyllaDB vs Cassandra

| Factor            | ScyllaDB   | Cassandra          |
| ----------------- | ---------- | ------------------ |
| Language          | C++        | Java               |
| Latency (P99)     | <1ms       | 5-15ms             |
| Throughput        | 10x faster | Baseline           |
| Memory Efficiency | Better     | Higher GC overhead |
| Compatibility     | 100% CQL   | Native             |

### Schema Design for Audit Logs

```cql
-- Partition by month for efficient time-range queries
CREATE TABLE audit_logs (
    year_month TEXT,           -- Partition key (YYYY-MM)
    saga_id TEXT,              -- Clustering key
    event_time TIMESTAMP,      -- Clustering key
    event_type TEXT,
    from_state TEXT,
    to_state TEXT,
    payload BLOB,
    PRIMARY KEY ((year_month), saga_id, event_time)
) WITH CLUSTERING ORDER BY (saga_id ASC, event_time DESC)
  AND default_time_to_live = 220752000  -- 7 years
  AND compaction = {'class': 'TimeWindowCompactionStrategy',
                    'compaction_window_size': 1,
                    'compaction_window_unit': 'DAYS'};
```

### When to Migrate from PostgreSQL

Consider ScyllaDB when:

1. Audit log table exceeds 1TB
2. Write throughput exceeds 50K TPS
3. Latency requirements are <5ms P99
4. Operational team can handle distributed systems

---

## MongoDB

### When to Use

**IDEAL FOR:**

- ✅ Document-oriented data (flexible schema)
- ✅ Rapid prototyping and iteration
- ✅ Product catalogs with varying attributes
- ✅ Customer preferences and profiles
- ✅ Content management systems

**NOT IDEAL FOR:**

- ❌ Financial transactions (weak ACID history)
- ❌ Strict schema requirements
- ❌ Regulatory compliance (audit trails)
- ❌ High-volume payment processing

### Why NOT for Payment Core

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    MONGODB LIMITATIONS FOR PAYMENTS                         │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  1. TRANSACTION LIMITATIONS                                                 │
│     • Multi-document transactions added in 4.0 (2018)                       │
│     • Higher latency than PostgreSQL for transactions                       │
│     • Shard key restrictions for distributed transactions                   │
│                                                                             │
│  2. CONSISTENCY CONCERNS                                                    │
│     • Default is eventual consistency                                       │
│     • Write concern configuration complexity                                │
│     • Read preference affects consistency                                   │
│                                                                             │
│  3. AUDIT COMPLIANCE                                                        │
│     • Schema flexibility is a liability for audits                          │
│     • Document versioning not native                                        │
│     • Harder to prove data integrity                                        │
│                                                                             │
│  4. ECOSYSTEM                                                               │
│     • Weaker CDC support (Change Streams vs Debezium)                       │
│     • Limited Spring State Machine integration                              │
│     • Temporal prefers SQL databases                                        │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Acceptable Use Cases in Payment Platform

```yaml
# MongoDB for non-critical, document-oriented data
collections:
  - product_catalog: # Flexible product attributes
      use: "varying product schemas"
  - customer_preferences: # User settings, UI preferences
      use: "nested documents, frequent schema changes"
  - merchant_profiles: # Rich merchant data
      use: "complex nested structures"
```

---

## CockroachDB

### When to Use

**IDEAL FOR:**

- ✅ Multi-region active-active writes
- ✅ Regulatory data residency requirements
- ✅ Global distribution with strong consistency
- ✅ PostgreSQL compatibility needed
- ✅ Zero-downtime schema changes

**NOT IDEAL FOR:**

- ❌ Single-region deployments (overkill)
- ❌ Cost-sensitive environments
- ❌ Extreme low-latency requirements (<5ms)
- ❌ Simple read-heavy workloads

### CockroachDB vs Aurora Global

| Factor                   | CockroachDB   | Aurora Global           |
| ------------------------ | ------------- | ----------------------- |
| Multi-region writes      | Active-Active | Active-Passive          |
| Cross-region latency     | 50-100ms      | <1s replication         |
| Consistency              | Serializable  | Eventual (cross-region) |
| Failover                 | Automatic     | Manual promotion        |
| PostgreSQL Compatibility | 95%           | 100%                    |
| Cost                     | Higher        | Lower                   |

### When to Consider CockroachDB

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    COCKROACHDB DECISION TREE                                │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  Need multi-region active-active?                                           │
│  ├── NO → Use Aurora Global Database                                        │
│  └── YES ↓                                                                  │
│                                                                             │
│      Need strong consistency across regions?                                │
│      ├── NO → Use DynamoDB Global Tables                                    │
│      └── YES ↓                                                              │
│                                                                             │
│          Need SQL and PostgreSQL compatibility?                             │
│          ├── NO → Consider ScyllaDB or custom solution                      │
│          └── YES → CockroachDB is a good fit                                │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Geo-Partitioning Example

```sql
-- Partition data by region for data residency
CREATE TABLE payments (
    id UUID PRIMARY KEY,
    region STRING NOT NULL,
    customer_id STRING,
    amount DECIMAL,
    created_at TIMESTAMP
) PARTITION BY LIST (region) (
    PARTITION eu VALUES IN ('eu-west-1', 'eu-central-1'),
    PARTITION us VALUES IN ('us-east-1', 'us-west-2'),
    PARTITION apac VALUES IN ('ap-southeast-1', 'ap-northeast-1')
);

-- Pin partitions to regions for data residency
ALTER PARTITION eu OF TABLE payments CONFIGURE ZONE USING
    constraints = '[+region=eu]';
```

---

## TigerBeetle

### When to Use

**IDEAL FOR:**

- ✅ Financial ledger with double-entry accounting
- ✅ Extreme throughput (1M+ TPS)
- ✅ Account balance tracking
- ✅ High-frequency trading settlement
- ✅ Mission-critical money movement

**NOT IDEAL FOR:**

- ❌ General-purpose data storage
- ❌ Complex queries and reporting
- ❌ Document storage
- ❌ Production systems (still in beta as of 2024)

### TigerBeetle Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    TIGERBEETLE ARCHITECTURE                                 │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  DESIGN PRINCIPLES:                                                         │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │ • Purpose-built for financial transactions                          │    │
│  │ • Double-entry accounting is native (not bolted on)                 │    │
│  │ • Designed to NEVER lose money                                      │    │
│  │ • Strict serializability (no eventual consistency)                  │    │
│  │ • Deterministic simulation testing (finds edge cases)               │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
│                                                                             │
│  PERFORMANCE:                                                               │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │ • 1,000,000+ transactions per second (single cluster)               │    │
│  │ • Sub-millisecond latency                                           │    │
│  │ • Batch processing for throughput                                   │    │
│  │ • Zero-copy I/O                                                     │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Data Model

```zig
// TigerBeetle Account (built-in structure)
Account {
    id: u128,              // Unique account ID
    debits_pending: u64,   // Pending debit amount
    debits_posted: u64,    // Posted debit amount
    credits_pending: u64,  // Pending credit amount
    credits_posted: u64,   // Posted credit amount
    user_data: [16]u8,     // Custom data (link to metadata DB)
    ledger: u32,           // Ledger/currency identifier
    code: u16,             // Account type code
    flags: AccountFlags,   // Configuration flags
}

// TigerBeetle Transfer (built-in structure)
Transfer {
    id: u128,              // Unique transfer ID
    debit_account_id: u128,
    credit_account_id: u128,
    amount: u64,           // Transfer amount
    pending_id: u128,      // For two-phase commits
    user_data: [16]u8,     // Custom data
    timeout: u32,          // For pending transfers
    ledger: u32,
    code: u16,
    flags: TransferFlags,
}
```

### Integration Pattern with PostgreSQL

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    HYBRID ARCHITECTURE                                      │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌─────────────────┐          ┌─────────────────┐                           │
│  │   PostgreSQL    │          │   TigerBeetle   │                           │
│  │   (Metadata)    │          │   (Ledger)      │                           │
│  │                 │          │                 │                           │
│  │ • Payment       │   Sync   │ • Account       │                           │
│  │   requests      │◄────────►│   balances      │                           │
│  │ • Audit trails  │  (Kafka) │ • Transfers     │                           │
│  │ • Customer data │          │                 │                           │
│  │ • Configurations│          │                 │                           │
│  └─────────────────┘          └─────────────────┘                           │
│                                                                             │
│  Flow:                                                                      │
│  1. Payment request → PostgreSQL (metadata)                                 │
│  2. Balance check → TigerBeetle                                             │
│  3. Create transfer → TigerBeetle                                           │
│  4. Update status → PostgreSQL                                              │
│  5. Audit log → PostgreSQL/DynamoDB                                         │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### When to Adopt TigerBeetle

| Stage              | Recommendation                        |
| ------------------ | ------------------------------------- |
| Current (<10K TPS) | Wait - PostgreSQL is sufficient       |
| 10K-50K TPS        | Evaluate in staging environment       |
| 50K+ TPS           | Strong candidate for balance tracking |
| TigerBeetle 1.0 GA | Safe for production adoption          |

---

## Use Case Mapping

### Payment Platform Data Model

| Data Type              | Database               | Rationale                    |
| ---------------------- | ---------------------- | ---------------------------- |
| **Payment Requests**   | Aurora PostgreSQL      | ACID, complex queries, JPA   |
| **Order State**        | Aurora PostgreSQL      | Transactions, state machine  |
| **Inventory**          | Aurora PostgreSQL      | ACID for stock levels        |
| **Outbox Events**      | Aurora + Debezium      | Transactional outbox pattern |
| **Audit Logs**         | DynamoDB               | Append-only, TTL, scale      |
| **Idempotency Keys**   | ElastiCache Redis      | TTL auto-expiry, fast lookup |
| **Session Data**       | ElastiCache Redis      | Ephemeral, fast access       |
| **Configuration**      | Aurora + Caffeine      | Rarely changes, cacheable    |
| **Account Balances**   | Aurora (→ TigerBeetle) | ACID now, scale later        |
| **Temporal Workflows** | Aurora (→ Cassandra)   | SQL now, scale later         |

### Decision Flowchart

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    DATABASE SELECTION FLOWCHART                             │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  START: What type of data?                                                  │
│  │                                                                          │
│  ├── Financial Transaction ────────────────────────► PostgreSQL/Aurora      │
│  │                                                                          │
│  ├── Audit Log ─────► Volume? ─── <1M/day ──────────► PostgreSQL            │
│  │                          └─── >1M/day ──────────► DynamoDB               │
│  │                                                                          │
│  ├── Session/Cache ─────────────────────────────────► Redis/ElastiCache     │
│  │                                                                          │
│  ├── Configuration ─────────────────────────────────► PostgreSQL + Cache    │
│  │                                                                          │
│  ├── Time-Series Metrics ───► Scale? ─ <100K TPS ──► TimescaleDB            │
│  │                                  └─ >100K TPS ──► ScyllaDB               │
│  │                                                                          │
│  ├── Document/Flexible Schema ──────────────────────► MongoDB (non-critical)│
│  │                                                                          │
│  └── Multi-Region Active-Active ────────────────────► CockroachDB           │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Migration Strategies

### PostgreSQL → Aurora

**Approach:** AWS DMS (Database Migration Service)

```yaml
Steps: 1. Create Aurora cluster
  2. Configure DMS replication instance
  3. Create source (PostgreSQL) and target (Aurora) endpoints
  4. Create replication task (full load + CDC)
  5. Monitor replication lag
  6. Cutover during low-traffic window

Timeline: 1-2 weeks
Downtime: <5 minutes (with proper planning)
```

### PostgreSQL → DynamoDB (Audit Logs)

**Approach:** Dual-write with gradual cutover

```java
// Phase 1: Dual-write (PostgreSQL primary)
@Transactional
public void saveAuditLog(AuditLog log) {
    postgresRepository.save(log);      // Primary
    dynamoDbRepository.save(log);       // Secondary (async)
}

// Phase 2: Dual-write (DynamoDB primary)
@Transactional
public void saveAuditLog(AuditLog log) {
    dynamoDbRepository.save(log);       // Primary
    postgresRepository.save(log);       // Secondary (for rollback)
}

// Phase 3: DynamoDB only
public void saveAuditLog(AuditLog log) {
    dynamoDbRepository.save(log);       // Only destination
}
```

### Aurora → CockroachDB (Multi-Region)

**Approach:** Application-level migration with feature flags

```yaml
Steps: 1. Deploy CockroachDB in target regions
  2. Implement repository abstraction layer
  3. Enable dual-write with feature flag
  4. Validate data consistency
  5. Gradually shift read traffic (10% → 50% → 100%)
  6. Shift write traffic after read validation
  7. Decommission Aurora after stabilization

Timeline: 2-3 months
Risk: Medium (requires extensive testing)
```

---

## References

- [TECH_STACK_RATIONALE.md](./TECH_STACK_RATIONALE.md) - Overall tech stack decisions
- [TEMPORAL_SCALING_ARCHITECTURE.md](./TEMPORAL_SCALING_ARCHITECTURE.md) - Temporal persistence scaling
- [CDC_OUTBOX_ARCHITECTURE.md](./CDC_OUTBOX_ARCHITECTURE.md) - Outbox pattern details

---

## Revision History

| Version | Date       | Author        | Changes         |
| ------- | ---------- | ------------- | --------------- |
| 1.0     | 2024-02-02 | Platform Team | Initial version |
