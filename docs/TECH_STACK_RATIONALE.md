# Tech Stack Selection Rationale & Scalability Architecture

This document explains the technology selection decisions for the Paylink Payment Platform, including scalability considerations and migration paths for each architectural layer.

---

## Table of Contents

1. [Executive Summary](#executive-summary)
2. [Architecture Overview](#architecture-overview)
3. [Application Layer](#application-layer)
4. [Database Layer](#database-layer)
5. [Messaging Layer](#messaging-layer)
6. [Caching Layer](#caching-layer)
7. [Workflow Orchestration](#workflow-orchestration)
8. [Observability](#observability)
9. [Scalability Tiers](#scalability-tiers)
10. [Decision Matrix](#decision-matrix)

---

## Executive Summary

### Design Principles

1. **Future-Proof**: Choose technologies with strong community support and clear roadmaps
2. **High Performance**: Optimize for throughput and latency at each layer
3. **Operational Simplicity**: Prefer managed services where possible
4. **Incremental Scalability**: Design for horizontal scaling without major rewrites

### Performance Targets

| Metric       | Current | Phase 1 Target | Phase 2 Target |
| ------------ | ------- | -------------- | -------------- |
| Throughput   | 500 TPS | 2,000 TPS      | 10,000+ TPS    |
| P99 Latency  | <500ms  | <200ms         | <100ms         |
| Availability | 99.9%   | 99.95%         | 99.99%         |

---

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           PAYLINK PAYMENT PLATFORM                          │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌─────────────┐   ┌─────────────┐   ┌─────────────┐   ┌─────────────┐      │
│  │   Kong      │   │   Order     │   │  Inventory  │   │  Payment    │      │
│  │   Gateway   │──▶│   Service   │   │   Service   │   │  Gateway    │      │
│  │   (Ingress) │   └──────┬──────┘   └──────┬──────┘   └──────┬──────┘      │
│  └─────────────┘          │                 │                 │             │
│                           ▼                 ▼                 ▼             │
│                    ┌────────────────────────────────────────────┐           │
│                    │         SAGA ORCHESTRATOR                  │           │
│                    │  ┌──────────┐  ┌──────────┐  ┌──────────┐  │           │
│                    │  │ Temporal │  │  State   │  │  Outbox  │  │           │
│                    │  │ Workflow │  │ Machine  │  │ Pattern  │  │           │
│                    │  └──────────┘  └──────────┘  └──────────┘  │           │
│                    └────────────────────────────────────────────┘           │
│                                      │                                      │
│         ┌────────────────────────────┼────────────────────────────┐         │
│         ▼                            ▼                            ▼         │
│  ┌─────────────┐           ┌─────────────┐              ┌─────────────┐     │
│  │ PostgreSQL  │           │    Kafka    │              │    Redis    │     │
│  │  (Aurora)   │           │  (Debezium) │              │ (ElastiCache│     │
│  │             │           │             │              │  + Caffeine)│     │
│  └─────────────┘           └─────────────┘              └─────────────┘     │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Application Layer

### Language: Java 21 LTS

**Selection Rationale:**

| Factor          | Decision      | Rationale                                                          |
| --------------- | ------------- | ------------------------------------------------------------------ |
| LTS Support     | Java 21       | 8+ years of support (until 2031)                                   |
| Virtual Threads | Project Loom  | 10x throughput for I/O-bound workloads without reactive complexity |
| Ecosystem       | Mature        | Best-in-class libraries for payments (Spring, Temporal, etc.)      |
| Talent Pool     | Large         | Easier to hire and maintain                                        |
| Performance     | JIT Optimized | GraalVM native optional for sidecars                               |

**Why NOT Other Languages:**

| Alternative | Reason for Not Choosing                             |
| ----------- | --------------------------------------------------- |
| Kotlin      | Team expertise, marginal benefits over Java 21      |
| Go          | Smaller ecosystem for enterprise payment processing |
| Rust        | Higher learning curve, slower development velocity  |
| Node.js     | Type safety concerns for financial applications     |

**Virtual Threads Configuration:**

```yaml
spring:
  threads:
    virtual:
      enabled: true # Enables Project Loom
```

**Benefits:**

- Blocking code performs like non-blocking
- Simpler programming model than reactive
- 30-50% throughput improvement for Feign/Kafka operations
- No code rewrites needed

**Scalability Path:**

1. **Current**: Virtual Threads for I/O operations
2. **Future**: GraalVM native compilation for sidecar services (metrics exporters, cleanup jobs)

---

### Framework: Spring Boot 3.2

**Selection Rationale:**

| Factor         | Spring Boot | Quarkus   | Micronaut |
| -------------- | ----------- | --------- | --------- |
| Startup Time   | 3-5s        | 0.5-1s    | 1-2s      |
| Memory         | ~300MB      | ~100MB    | ~150MB    |
| Ecosystem      | ★★★★★       | ★★★☆☆     | ★★★☆☆     |
| Temporal SDK   | ★★★★★       | ★★☆☆☆     | ★★☆☆☆     |
| State Machine  | ★★★★★       | ☆☆☆☆☆     | ☆☆☆☆☆     |
| Migration Cost | N/A         | Very High | Very High |

**Why Spring Boot:**

- Spring State Machine 4.0 only available for Spring
- Temporal SDK has excellent Spring Boot integration
- Spring Cloud ecosystem (Feign, Kubernetes, Resilience4j)
- Spring Security OAuth2 for JWT validation
- Mature testing framework (MockMvc, TestContainers)

**Scalability Path:**

1. **Current**: Spring Boot with AOT (Ahead-of-Time) compilation
2. **Future**: Consider Quarkus for new greenfield microservices only

---

## Database Layer

### Primary: PostgreSQL (Aurora)

**Selection Rationale:**

| Factor         | PostgreSQL      | MySQL       | SQL Server  |
| -------------- | --------------- | ----------- | ----------- |
| ACID           | ★★★★★           | ★★★★☆       | ★★★★★       |
| JSON Support   | ★★★★★ (JSONB)   | ★★★☆☆       | ★★★☆☆       |
| Replication    | ★★★★★ (Logical) | ★★★★☆       | ★★★★☆       |
| CDC (Debezium) | ★★★★★           | ★★★★☆       | ★★★☆☆       |
| Extensions     | ★★★★★           | ★★☆☆☆       | ★☆☆☆☆       |
| Cost           | Open Source     | Open Source | License $$$ |

**Why Aurora PostgreSQL:**

```
┌─────────────────────────────────────────────────────────────────┐
│                    AURORA POSTGRESQL BENEFITS                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ✓ 5x throughput vs standard PostgreSQL                         │
│  ✓ Auto-scaling storage (0 → 128TB)                             │
│  ✓ Up to 15 read replicas with <10ms lag                        │
│  ✓ Multi-AZ automatic failover (<30s)                           │
│  ✓ Point-in-time recovery (up to 35 days)                       │
│  ✓ Performance Insights for query optimization                  │
│  ✓ Global Database for multi-region (<1s replication)           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Aurora Serverless v2 Configuration:**

```yaml
# terraform/aurora.tf
resource "aws_rds_cluster" "saga" {
  engine_mode = "provisioned"  # Required for Serverless v2

  serverlessv2_scaling_configuration {
    min_capacity = 2    # 4GB RAM minimum (cost-efficient idle)
    max_capacity = 64   # 128GB RAM for peaks (burst handling)
  }
}
```

**Use Cases by Database Type:**

| Use Case            | Database          | Rationale                                 |
| ------------------- | ----------------- | ----------------------------------------- |
| Transactional Core  | Aurora PostgreSQL | ACID, complex queries, JPA compatibility  |
| Audit Logs (7yr)    | DynamoDB + S3     | Append-only, TTL, cost-effective archival |
| Idempotency/Session | ElastiCache Redis | TTL auto-expiry, sub-ms latency           |
| Configuration       | Aurora + Caffeine | Cacheable, rarely changes                 |
| Outbox Pattern      | Aurora + Debezium | CDC < 10ms, transactional guarantees      |

---

### High-Volume Audit: DynamoDB

**Selection Rationale:**

| Factor               | DynamoDB  | ScyllaDB   | Cassandra |
| -------------------- | --------- | ---------- | --------- |
| Managed              | ★★★★★     | ★★★☆☆      | ★★☆☆☆     |
| Latency              | 1-5ms     | <1ms       | 5-15ms    |
| Scalability          | Auto      | Manual     | Manual    |
| TTL Support          | ★★★★★     | ★★★★★      | ★★★★★     |
| Cost (1M writes/day) | $500-3000 | $1000-2000 | $800-1500 |
| Ops Overhead         | Zero      | Medium     | High      |

**DynamoDB Schema for Audit Logs:**

```json
{
  "TableName": "PaymentAuditLogs",
  "KeySchema": [
    { "AttributeName": "PK", "KeyType": "HASH" }, // "SAGA#saga-123"
    { "AttributeName": "SK", "KeyType": "RANGE" } // "EVENT#2024-01-15T10:30:00Z"
  ],
  "GlobalSecondaryIndexes": [
    {
      "IndexName": "ByDate",
      "KeySchema": [
        { "AttributeName": "GSI1PK", "KeyType": "HASH" }, // "DATE#2024-01-15"
        { "AttributeName": "GSI1SK", "KeyType": "RANGE" } // "TIME#saga-id"
      ]
    }
  ],
  "TimeToLiveSpecification": {
    "AttributeName": "ttl",
    "Enabled": true // 7-year expiry
  }
}
```

**Why NOT ScyllaDB:**

- Operational complexity (self-managed clusters)
- DynamoDB sufficient for 1K-10K TPS
- Consider ScyllaDB at >10K TPS for cost optimization

---

### Financial Ledger: Future TigerBeetle Evaluation

**TigerBeetle Characteristics:**

| Factor            | TigerBeetle         | PostgreSQL             |
| ----------------- | ------------------- | ---------------------- |
| TPS (single node) | 1,000,000+          | 10,000-50,000          |
| Double-entry      | Built-in            | Manual implementation  |
| Consistency       | Strict serializable | Configurable           |
| Maturity          | Beta (2024)         | Production (25+ years) |

**Current Decision:** Keep PostgreSQL for ledger until TigerBeetle reaches 1.0 GA

**Future Migration Path:**

1. PostgreSQL handles balances with optimistic locking
2. When approaching 10K+ TPS, evaluate TigerBeetle for account balances
3. Keep PostgreSQL for metadata/audit, use TigerBeetle for real-time balance tracking

---

## Messaging Layer

### Primary: Apache Kafka (MSK)

**Selection Rationale:**

| Factor        | Kafka    | Redpanda  | Pulsar     | NATS       |
| ------------- | -------- | --------- | ---------- | ---------- |
| Latency (P99) | 5-10ms   | 1-2ms     | 5-15ms     | <1ms       |
| Throughput    | 1M msg/s | 1M+ msg/s | 500K msg/s | 10M+ msg/s |
| Durability    | ★★★★★    | ★★★★★     | ★★★★★      | ★★☆☆☆      |
| Debezium      | ★★★★★    | ★★★★★     | ★★★☆☆      | ☆☆☆☆☆      |
| AWS Managed   | MSK      | No        | No         | No         |
| Ecosystem     | ★★★★★    | ★★★★☆     | ★★★☆☆      | ★★★☆☆      |

**Why Kafka:**

- Debezium CDC requires Kafka or Kafka-compatible API
- MSK provides managed Kafka with minimal ops
- Mature ecosystem, extensive documentation
- Consistent partitioning for ordering guarantees

**Kafka Producer Optimization:**

```yaml
spring:
  kafka:
    producer:
      acks: all # Durability guarantee
      properties:
        batch.size: 65536 # 64KB batches (2x default)
        linger.ms: 10 # Wait for batch fill
        compression.type: lz4 # Fast compression
        enable.idempotence: true # Exactly-once semantics
```

**Why These Settings:**

- `batch.size: 65536`: Larger batches = fewer network round trips = 30% higher throughput
- `linger.ms: 10`: Small delay to fill batches without impacting latency significantly
- `compression.type: lz4`: Best balance of speed and compression ratio
- `enable.idempotence: true`: Prevents duplicate messages on retries

**Future Evaluation: Redpanda**

Consider Redpanda when:

- Latency requirements tighten (<2ms P99)
- Operational simplicity becomes priority (no ZooKeeper)
- Cost optimization needed at scale

---

### CDC: Debezium

**Selection Rationale:**

| Factor       | Debezium     | Kafka Connect JDBC | AWS DMS |
| ------------ | ------------ | ------------------ | ------- |
| Latency      | <10ms        | 1s+ (polling)      | <100ms  |
| WAL-based    | ✓            | ✗                  | ✓       |
| Exactly-once | ✓ (Kafka TX) | ✗                  | ✗       |
| Self-hosted  | ✓            | ✓                  | ✗       |
| Cost         | Free         | Free               | $$$     |

**Why Debezium:**

- WAL-based capture (no polling overhead)
- <10ms latency for outbox pattern
- EventRouter SMT for topic routing
- Kafka transactions for exactly-once

**Debezium Configuration:**

```json
{
  "connector.class": "io.debezium.connector.postgresql.PostgresConnector",
  "plugin.name": "pgoutput",
  "slot.name": "outbox_slot",
  "publication.name": "outbox_publication",

  "transforms": "outbox",
  "transforms.outbox.type": "io.debezium.transforms.outbox.EventRouter",
  "transforms.outbox.route.by.field": "topic",

  "errors.tolerance": "all",
  "errors.deadletterqueue.topic.name": "outbox-dlq"
}
```

---

## Caching Layer

### L1 Cache: Caffeine

**Selection Rationale:**

| Factor             | Caffeine | Guava | Ehcache |
| ------------------ | -------- | ----- | ------- |
| Performance        | ★★★★★    | ★★★☆☆ | ★★★★☆   |
| Memory Efficiency  | ★★★★★    | ★★★☆☆ | ★★★★☆   |
| Spring Integration | ★★★★★    | ★★★★☆ | ★★★★★   |
| TTL/Eviction       | ★★★★★    | ★★★★☆ | ★★★★★   |

**Why Caffeine:**

- Near-optimal hit rate with W-TinyLFU eviction
- Sub-microsecond access time
- Native Spring Cache integration
- Lower memory footprint than alternatives

**Configuration:**

```java
@Bean
public CacheManager caffeineCacheManager() {
    CaffeineCacheManager manager = new CaffeineCacheManager();
    manager.setCaffeine(Caffeine.newBuilder()
        .maximumSize(10_000)           // Max entries
        .expireAfterWrite(Duration.ofMinutes(5))  // TTL
        .recordStats());               // For monitoring
    return manager;
}
```

**Cache Hierarchy:**

```
┌─────────────────────────────────────────────────────────────────┐
│                      CACHE HIERARCHY                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  REQUEST                                                        │
│     │                                                           │
│     ▼                                                           │
│  ┌─────────────────┐                                            │
│  │ L1: Caffeine    │  Hit: 0.1ms (80-90% hit rate)              │
│  │ (Local JVM)     │                                            │
│  └────────┬────────┘                                            │
│           │ Miss                                                │
│           ▼                                                     │
│  ┌─────────────────┐                                            │
│  │ L2: Redis       │  Hit: 1-5ms                                │
│  │ (Distributed)   │                                            │
│  └────────┬────────┘                                            │
│           │ Miss                                                │
│           ▼                                                     │
│  ┌─────────────────┐                                            │
│  │ L3: Database    │  5-20ms                                    │
│  │ (PostgreSQL)    │                                            │
│  └─────────────────┘                                            │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

### L2 Cache: Redis (ElastiCache) / Valkey

**Selection Rationale:**

| Factor            | Redis       | Valkey      | KeyDB | Dragonfly |
| ----------------- | ----------- | ----------- | ----- | --------- |
| License           | SSPL        | BSD-3 ✓     | BSD-3 | BSD-3     |
| API Compatibility | N/A         | 100%        | 100%  | 99%       |
| Multi-threaded    | ✗           | ✗           | ✓     | ✓         |
| AWS Managed       | ElastiCache | ElastiCache | ✗     | ✗         |
| Performance       | Baseline    | Same        | 5x    | 25x       |

**Current Decision:** ElastiCache Redis

**Future Migration:** Valkey (when available on ElastiCache)

- License certainty (BSD-3 vs Redis SSPL)
- Same API, zero code changes
- Backed by Linux Foundation (AWS, Google, Oracle)

**ElastiCache Configuration:**

```yaml
# terraform/elasticache.tf
resource "aws_elasticache_replication_group" "cache" {
  node_type            = "cache.r6g.large"   # 13GB RAM
  num_cache_clusters   = 3                    # Multi-AZ

  automatic_failover_enabled = true
  multi_az_enabled          = true

  engine         = "redis"
  engine_version = "7.0"
}
```

---

## Workflow Orchestration

### Primary: Temporal

**Selection Rationale:**

| Factor       | Temporal | Cadence | Step Functions | Airflow |
| ------------ | -------- | ------- | -------------- | ------- |
| Latency      | <100ms   | <100ms  | ~1s            | Minutes |
| SDK Quality  | ★★★★★    | ★★★★☆   | ★★☆☆☆          | ★★★☆☆   |
| Durability   | ★★★★★    | ★★★★★   | ★★★★★          | ★★★☆☆   |
| Scalability  | ★★★★★    | ★★★★★   | Auto           | ★★★☆☆   |
| Self-hosted  | ✓        | ✓       | ✗              | ✓       |
| Long-running | ★★★★★    | ★★★★★   | ★★★☆☆          | ★★☆☆☆   |

**Why Temporal:**

- Purpose-built for payment workflows
- Automatic retry and compensation
- Workflow versioning for zero-downtime deployments
- Signal-based external event handling (webhooks)
- Strong consistency guarantees

**Sharding Configuration:**

```yaml
temporal:
  sharding:
    enabled: true
    algorithm: consistent-hash
    priorities:
      critical:
        shard-count: 4
        max-concurrent-workflows: 50
      high:
        shard-count: 8
        max-concurrent-workflows: 100
      normal:
        shard-count: 16
        max-concurrent-workflows: 200
      low:
        shard-count: 8
        max-concurrent-workflows: 300
```

**Scalability Path:**

| TPS    | Persistence                | Shards | Notes                                |
| ------ | -------------------------- | ------ | ------------------------------------ |
| <1K    | PostgreSQL                 | 256    | Current setup                        |
| 1K-5K  | PostgreSQL                 | 512    | Increase history shards              |
| 5K-10K | PostgreSQL + Read replicas | 512    | Add Elasticsearch visibility         |
| 10K+   | Cassandra/ScyllaDB         | 1024+  | See TEMPORAL_SCALING_ARCHITECTURE.md |

---

## Observability

### Tracing: OpenTelemetry (Migration Path)

**Current:** Micrometer + Zipkin (Brave)

**Target:** OpenTelemetry

**Migration Rationale:**

| Factor              | Brave/Zipkin | OpenTelemetry      |
| ------------------- | ------------ | ------------------ |
| Standard            | Proprietary  | CNCF Graduated     |
| Vendor Lock-in      | Medium       | None               |
| Metrics+Traces+Logs | Separate     | Unified            |
| Future Support      | Maintenance  | Active Development |

**Migration Steps:**

1. Add OTel bridge (coexistence):

```xml
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-tracing-bridge-otel</artifactId>
</dependency>
```

2. Deploy OTel Collector:

```yaml
receivers:
  otlp:
    protocols:
      grpc:
      http:

exporters:
  prometheus:
  jaeger:

processors:
  batch:
```

3. Switch propagation format:

```yaml
management:
  tracing:
    propagation:
      type: w3c # From b3_multi
```

---

## Scalability Tiers

### Tier 1: 500-2,000 TPS (Current + Phase 1)

**Infrastructure:**

- Aurora PostgreSQL (2-64 ACU)
- ElastiCache Redis (3 nodes)
- MSK Kafka (3 brokers)
- Temporal with 256 history shards

**Optimizations Applied:**

- Virtual Threads enabled
- Caffeine L1 caching
- Temporal sharding enabled
- Kafka batch optimization

**Expected Improvement:** 4x throughput

---

### Tier 2: 2,000-10,000 TPS (Phase 2)

**Infrastructure Changes:**

- Aurora with 15 read replicas
- DynamoDB for audit logs
- PgBouncer connection pooling
- Temporal with 512 history shards

**Additional Optimizations:**

- Outbox table partitioning
- Connection pooling (PgBouncer)
- Read/write splitting
- Elasticsearch for Temporal visibility

---

### Tier 3: 10,000-100,000+ TPS (Phase 3)

**Infrastructure Changes:**

- Cassandra/ScyllaDB for Temporal
- Multi-region Aurora Global Database
- DynamoDB Global Tables
- Consider Redpanda for latency

**Architecture Changes:**

- Cell-based architecture
- Regional isolation
- Active-active multi-region

---

## Decision Matrix

### Quick Reference

| Layer          | Current         | Phase 1         | Phase 2            | Phase 3            |
| -------------- | --------------- | --------------- | ------------------ | ------------------ |
| **Language**   | Java 21         | Virtual Threads | -                  | GraalVM (sidecars) |
| **Framework**  | Spring Boot 3.2 | -               | -                  | -                  |
| **Primary DB** | PostgreSQL      | Aurora          | Aurora + PgBouncer | Aurora Global      |
| **Audit DB**   | PostgreSQL      | -               | DynamoDB           | DynamoDB Global    |
| **Cache L1**   | -               | Caffeine        | -                  | -                  |
| **Cache L2**   | Redis           | ElastiCache     | Valkey             | -                  |
| **Messaging**  | Kafka           | Optimized       | MSK                | Redpanda (eval)    |
| **CDC**        | Debezium        | -               | -                  | -                  |
| **Workflow**   | Temporal        | Sharding        | 512 shards         | Cassandra backend  |
| **Tracing**    | Zipkin          | -               | OpenTelemetry      | -                  |

### Cost Estimation (Monthly, 1M transactions/day)

| Component   | Phase 1          | Phase 2          | Phase 3            |
| ----------- | ---------------- | ---------------- | ------------------ |
| Aurora      | $500-1,500       | $2,000-4,000     | $5,000-10,000      |
| DynamoDB    | -                | $500-1,500       | $1,000-3,000       |
| ElastiCache | $300-600         | $600-1,200       | $1,200-2,400       |
| MSK         | $500-1,000       | $1,000-2,000     | $2,000-4,000       |
| Temporal    | Self-hosted      | Self-hosted      | $2,000-5,000       |
| **Total**   | **$1,300-3,100** | **$4,100-8,700** | **$11,200-24,400** |

---

## References

- [TEMPORAL_SCALING_ARCHITECTURE.md](./TEMPORAL_SCALING_ARCHITECTURE.md) - Detailed Temporal scaling guide
- [CDC_OUTBOX_ARCHITECTURE.md](./CDC_OUTBOX_ARCHITECTURE.md) - Outbox pattern implementation
- [API_GATEWAY_ARCHITECTURE.md](./API_GATEWAY_ARCHITECTURE.md) - Kong + Istio layered gateway

---

## Revision History

| Version | Date       | Author        | Changes                     |
| ------- | ---------- | ------------- | --------------------------- |
| 1.0     | 2024-01-15 | Platform Team | Initial version             |
| 1.1     | 2024-02-02 | Platform Team | Added Phase 1 optimizations |
