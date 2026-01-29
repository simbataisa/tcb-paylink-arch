# Implementation Plan: Temporal-Kafka Integration Use Cases

## Table of Contents

- [Analysis Summary](#analysis-summary)
- [COMPLETED: Debit Sequencing Test Enhancement](#completed-debit-sequencing-test-enhancement)
  - [Problem Statement (Resolved)](#problem-statement-resolved)
  - [Implementation Plan](#implementation-plan)
  - [Files Created/Modified (COMPLETE)](#files-createdmodified-complete)
- [Use Case 1: Debit Transaction Sequencing](#use-case-1-debit-transaction-sequencing)
  - [Current State](#current-state)
  - [Implementation Plan](#implementation-plan-1)
- [Use Case 2: Event-Driven Architecture](#use-case-2-event-driven-architecture)
- [Use Case 3: Audit Trail & Compliance](#use-case-3-audit-trail--compliance)
  - [Current State](#current-state-1)
  - [Implementation Plan](#implementation-plan-2)

---

## Analysis Summary

Based on exploration of `temporal-kafka-integration.md` and the codebase:

| Use Case | Documentation | Implementation | Status |
|----------|---------------|----------------|--------|
| 1. Debit Transaction Sequencing | Fully documented | 100% implemented | **Complete** |
| 2. Event-Driven Architecture | Fully documented | 95% implemented | **Complete** |
| 3. Audit Trail & Compliance | Fully documented | 100% implemented | **Complete** |

---

## COMPLETED: Debit Sequencing Test Enhancement

### Problem Statement (Resolved)

The current `DebitWorkflowTest` tests the workflow in isolation by directly calling `workflow.processDebit()`. This **bypasses the critical Kafka + Single Workflow pattern** that ensures sequential processing:

```
Current Test (insufficient):
  Test → workflow.processDebit() → Workflow

Missing Test (the actual pattern):
  Kafka Message → PaymentConsistentHashPartitioner → Same Partition
                                                           ↓
  DebitRequestConsumer → getDebitWorkflowId() → Single Workflow per Account
                                                           ↓
                                      workflow.processDebit() → Sequential Processing
```

**Risk**: If parallel requests arrive for the same account without Kafka partitioning, multiple workflows could be created instead of one.

### Implementation Plan

#### T1. PaymentConsistentHashPartitioner Unit Test

**Create:** `payment-saga-orchestrator/src/test/java/com/payment/saga/kafka/PaymentConsistentHashPartitionerTest.java`

Tests:
- Same accountId always returns same partition
- Different accountIds distribute across partitions
- Consistent hashing survives partition count changes

#### T2. DebitRequestConsumer Unit Test

**Create:** `payment-saga-orchestrator/src/test/java/com/payment/saga/consumer/DebitRequestConsumerTest.java`

Tests:
- Routes to same workflow for same accountId
- Creates new workflow if not exists
- Handles "workflow already started" race condition
- Invalid requests are handled gracefully

#### T3. Debit Sequencing Integration Test

**Create:** `payment-saga-orchestrator/src/test/java/com/payment/saga/consumer/DebitSequencingIntegrationTest.java`

Tests:
- Parallel requests for same account signal same workflow (not create multiple)
- Requests for different accounts use different workflows
- Full Kafka → Consumer → Workflow pattern with EmbeddedKafka

#### T4. Update Existing DebitWorkflowTest

**Modify:** `payment-saga-orchestrator/src/test/java/com/payment/saga/workflow/DebitWorkflowTest.java`

Add clarifying comment that this tests workflow logic in isolation, while consumer tests verify the routing pattern.

### Files Created/Modified (COMPLETE)

| Action | File | Tests |
|--------|------|-------|
| EXISTS | `payment-saga-orchestrator/src/test/java/com/payment/saga/kafka/PaymentConsistentHashPartitionerTest.java` | 14 tests |
| CREATE | `payment-saga-orchestrator/src/test/java/com/payment/saga/consumer/DebitRequestConsumerTest.java` | 9 tests |
| CREATE | `payment-saga-orchestrator/src/test/java/com/payment/saga/consumer/DebitSequencingIntegrationTest.java` | 8 tests |
| MODIFY | `payment-saga-orchestrator/src/test/java/com/payment/saga/workflow/DebitWorkflowTest.java` | 5 tests |

**Total: 36 tests covering the complete debit sequencing pattern**

---

## Use Case 1: Debit Transaction Sequencing

### Current State

**Exists:**
- `PaymentConsistentHashPartitioner` - Consistent hashing for Kafka partitions
- Temporal workflow infrastructure (`PaymentSagaWorkflow`)
- Kafka consumer configuration with manual ack
- Design documentation in `temporal-kafka-integration.md` (lines 398-744)

**Missing:**
- `DebitWorkflow` interface and implementation
- Account balance entity and repository
- Balance checking activities
- Single-workflow-per-account pattern
- Debit-specific state machine states
- `debit-requests` Kafka topic

### Implementation Plan

#### 1.1 Create Debit Domain Model

**Create:** `payment-saga-orchestrator/src/main/java/com/payment/saga/domain/debit/`
- `AccountBalance.java` - Account balance value object
- `DebitRequest.java` - Debit transaction request DTO
- `DebitResult.java` - Debit operation result

**Create:** `payment-saga-orchestrator/src/main/java/com/payment/saga/entity/`
- `AccountBalanceEntity.java` - JPA entity with optimistic locking
- `DebitTransactionEntity.java` - Debit audit trail

**Create:** `payment-saga-orchestrator/src/main/java/com/payment/saga/repository/`
- `AccountBalanceRepository.java` - With `@Lock(PESSIMISTIC_WRITE)` for atomic debit

#### 1.2 Create Debit Workflow

**Create:** `payment-saga-orchestrator/src/main/java/com/payment/saga/workflow/`
- `DebitWorkflow.java` - Interface with `@SignalMethod void processDebit(DebitRequest)`
- `DebitWorkflowImpl.java` - Long-running workflow with internal queue

Pattern from temporal-kafka-integration.md:
```java
@WorkflowInterface
public interface DebitWorkflow {
    @WorkflowMethod
    void run(String accountId);

    @SignalMethod
    void processDebit(DebitRequest request);

    @QueryMethod
    BigDecimal getCurrentBalance();
}
```

#### 1.3 Create Debit Activities

**Create:** `payment-saga-orchestrator/src/main/java/com/payment/saga/activity/`
- `DebitActivities.java` - Interface with `getBalance()`, `debitAccount()`, `recordFailure()`
- `DebitActivitiesImpl.java` - Implementation calling repository

#### 1.4 Create Kafka Consumer for Debit Requests

**Create:** `payment-saga-orchestrator/src/main/java/com/payment/saga/consumer/`
- `DebitRequestConsumer.java` - Consumes from `debit-requests` topic
- Routes to workflow via signal (single workflow per account)

#### 1.5 Add Kafka Topic

**Modify:** `docker-compose.yml` - Add `debit-requests` topic (12 partitions)

#### 1.6 Database Migration

**Create:** `payment-saga-orchestrator/src/main/resources/db/migration/V4__debit_tables.sql`
```sql
CREATE TABLE account_balance (
    account_id VARCHAR(36) PRIMARY KEY,
    balance DECIMAL(19,4) NOT NULL DEFAULT 0,
    version BIGINT NOT NULL DEFAULT 0,
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE debit_transaction (
    id BIGSERIAL PRIMARY KEY,
    account_id VARCHAR(36) NOT NULL,
    request_id VARCHAR(36) UNIQUE NOT NULL,
    amount DECIMAL(19,4) NOT NULL,
    balance_before DECIMAL(19,4) NOT NULL,
    balance_after DECIMAL(19,4),
    status VARCHAR(20) NOT NULL,
    error_message TEXT,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);
```

#### 1.7 Tests

**Create:** `payment-saga-orchestrator/src/test/java/com/payment/saga/workflow/`
- `DebitWorkflowTest.java` - Temporal TestWorkflowEnvironment tests
- `DebitWorkflowConcurrencyTest.java` - Race condition prevention tests

---

## Use Case 2: Event-Driven Architecture

### Current State: **COMPLETE**

The codebase already implements event-driven architecture:

- **Loose Coupling via Kafka:**
  - `WebhookKafkaPublisher` → `webhook.payment.events` → `WebhookEventConsumer`
  - `PaymentEventPublisher` → domain event topics → external consumers

- **Transactional Outbox Pattern:**
  - `OutboxEventEntity` + `OutboxPublisher` + `OutboxPoller` (orchestrator)
  - `WebhookKafkaOutboxEntity` + `WebhookKafkaOutboxPoller` (payment-gateway)

- **Event Store for Replay:**
  - `EventStoreEntity` + `DomainEventPublisherImpl` (dual publishing)

- **Idempotency:**
  - `WebhookIdempotencyService` prevents duplicate webhook processing

**No implementation needed.** The system uses a hybrid approach:
- Loose coupling for reactive flows (external webhooks)
- Tight coupling (Feign) for orchestrated operations within workflows

---

## Use Case 3: Audit Trail & Compliance

### Current State

**Exists (60%):**
- `SagaAuditTrailEntity` - Comprehensive audit record
- `SagaAuditService` + `SagaAuditServiceImpl` - Recording methods
- `SagaAuditTrailRepository` - Query methods with indexes
- Short-term retention: 7 days (outbox), 30 days (idempotency)

**Missing:**
- 7-year retention configuration
- Audit archival to long-term storage
- Kafka `payment.audit.events` topic
- Immutability enforcement
- Compliance-specific fields
- Audit reporting endpoints

### Implementation Plan

#### 3.1 Enhance Audit Entity

**Modify:** `payment-saga-orchestrator/src/main/java/com/payment/saga/entity/SagaAuditTrailEntity.java`

Add compliance fields:
```java
@Column(name = "created_by")
private String createdBy;

@Column(name = "data_classification")
private String dataClassification;  // PII, FINANCIAL, INTERNAL

@Column(name = "regulatory_context")
private String regulatoryContext;   // PCI-DSS, SOX, GDPR
```

#### 3.2 Add Retention Configuration

**Modify:** `payment-saga-orchestrator/src/main/resources/application.yml`
```yaml
saga:
  audit:
    retention-days: 2555  # 7 years
    archival:
      enabled: true
      threshold-days: 365  # Archive after 1 year
      storage: s3  # or database
```

#### 3.3 Create Audit Archival Service

**Create:** `payment-saga-orchestrator/src/main/java/com/payment/saga/service/`
- `AuditArchivalService.java` - Interface
- `AuditArchivalServiceImpl.java` - Implementation with S3 export

Scheduled job to archive old records:
```java
@Scheduled(cron = "0 0 2 * * ?")  // Daily at 2 AM
public void archiveOldAuditRecords() {
    // Export to S3/archival storage
    // Delete from primary database
}
```

#### 3.4 Create Audit Kafka Topic

**Modify:** `docker-compose.yml`
```yaml
payment.audit.events:3:1  # 3 partitions, 1 replica
```

**Create:** `payment-saga-orchestrator/src/main/java/com/payment/saga/audit/`
- `AuditKafkaPublisher.java` - Publishes audit events to Kafka for external consumers

#### 3.5 Add Immutability Enforcement

**Create:** `payment-saga-orchestrator/src/main/resources/db/migration/V5__audit_immutability.sql`
```sql
-- Trigger to prevent updates/deletes on saga_audit_trail
CREATE OR REPLACE FUNCTION prevent_audit_modification()
RETURNS TRIGGER AS $$
BEGIN
    RAISE EXCEPTION 'Audit records are immutable';
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER audit_immutability_trigger
BEFORE UPDATE OR DELETE ON saga_audit_trail
FOR EACH ROW EXECUTE FUNCTION prevent_audit_modification();
```

#### 3.6 Create Compliance Reporting Endpoint

**Create:** `payment-saga-orchestrator/src/main/java/com/payment/saga/controller/`
- `AuditController.java` - REST endpoints for compliance queries

Endpoints:
- `GET /api/v1/audit/{sagaId}` - Full audit trail for a payment
- `GET /api/v1/audit/search` - Search by date range, event type, user
- `GET /api/v1/audit/export` - Export audit logs in compliance format (CSV/JSON)

#### 3.7 Database Migration

**Create:** `payment-saga-orchestrator/src/main/resources/db/migration/V5__audit_compliance.sql`
```sql
ALTER TABLE saga_audit_trail
ADD COLUMN created_by VARCHAR(100),
ADD COLUMN data_classification VARCHAR(20) DEFAULT 'FINANCIAL',
ADD COLUMN regulatory_context VARCHAR(50);

CREATE INDEX idx_audit_created_by ON saga_audit_trail(created_by);
CREATE INDEX idx_audit_regulatory ON saga_audit_trail(regulatory_context);
```

#### 3.8 Tests

**Create:** `payment-saga-orchestrator/src/test/java/com/payment/saga/service/`
- `AuditArchivalServiceTest.java` - Archival logic tests
- `AuditControllerTest.java` - REST endpoint tests

---

## Files Summary

### Use Case 1: Debit Transaction Sequencing

| Action | File |
|--------|------|
| CREATE | `payment-saga-orchestrator/src/main/java/com/payment/saga/domain/debit/AccountBalance.java` |
| CREATE | `payment-saga-orchestrator/src/main/java/com/payment/saga/domain/debit/DebitRequest.java` |
| CREATE | `payment-saga-orchestrator/src/main/java/com/payment/saga/domain/debit/DebitResult.java` |
| CREATE | `payment-saga-orchestrator/src/main/java/com/payment/saga/entity/AccountBalanceEntity.java` |
| CREATE | `payment-saga-orchestrator/src/main/java/com/payment/saga/entity/DebitTransactionEntity.java` |
| CREATE | `payment-saga-orchestrator/src/main/java/com/payment/saga/repository/AccountBalanceRepository.java` |
| CREATE | `payment-saga-orchestrator/src/main/java/com/payment/saga/repository/DebitTransactionRepository.java` |
| CREATE | `payment-saga-orchestrator/src/main/java/com/payment/saga/workflow/DebitWorkflow.java` |
| CREATE | `payment-saga-orchestrator/src/main/java/com/payment/saga/workflow/DebitWorkflowImpl.java` |
| CREATE | `payment-saga-orchestrator/src/main/java/com/payment/saga/activity/DebitActivities.java` |
| CREATE | `payment-saga-orchestrator/src/main/java/com/payment/saga/activity/DebitActivitiesImpl.java` |
| CREATE | `payment-saga-orchestrator/src/main/java/com/payment/saga/consumer/DebitRequestConsumer.java` |
| CREATE | `payment-saga-orchestrator/src/main/resources/db/migration/V4__debit_tables.sql` |
| CREATE | `payment-saga-orchestrator/src/test/java/com/payment/saga/workflow/DebitWorkflowTest.java` |
| MODIFY | `docker-compose.yml` - Add `debit-requests` topic |
| MODIFY | `payment-saga-orchestrator/.../config/TemporalConfig.java` - Register debit worker |

### Use Case 3: Audit Trail & Compliance

| Action | File |
|--------|------|
| MODIFY | `payment-saga-orchestrator/src/main/java/com/payment/saga/entity/SagaAuditTrailEntity.java` |
| MODIFY | `payment-saga-orchestrator/src/main/resources/application.yml` - Add audit config |
| CREATE | `payment-saga-orchestrator/src/main/java/com/payment/saga/service/AuditArchivalService.java` |
| CREATE | `payment-saga-orchestrator/src/main/java/com/payment/saga/service/impl/AuditArchivalServiceImpl.java` |
| CREATE | `payment-saga-orchestrator/src/main/java/com/payment/saga/audit/AuditKafkaPublisher.java` |
| CREATE | `payment-saga-orchestrator/src/main/java/com/payment/saga/controller/AuditController.java` |
| CREATE | `payment-saga-orchestrator/src/main/resources/db/migration/V5__audit_compliance.sql` |
| CREATE | `payment-saga-orchestrator/src/test/java/com/payment/saga/service/AuditArchivalServiceTest.java` |
| CREATE | `payment-saga-orchestrator/src/test/java/com/payment/saga/controller/AuditControllerTest.java` |
| MODIFY | `docker-compose.yml` - Add `payment.audit.events` topic |

---

## Verification

### Build and Test
```bash
# Build all modules
mvn clean compile

# Run all tests
mvn test

# Run specific test suites
mvn test -Dtest="DebitWorkflow*Test"
mvn test -Dtest="AuditArchival*Test"
```

### Integration Test - Debit Sequencing
```bash
# Start infrastructure
docker compose up -d

# Create Temporal namespace
docker exec payment-saga-temporal tctl --address temporal:7233 \
  --namespace payment-saga namespace register --retention 168h

# Start application
java -jar payment-saga-orchestrator.jar

# Test concurrent debits (should process sequentially)
curl -X POST http://localhost:9092/debit-requests \
  -H "Content-Type: application/json" \
  -d '{"accountId":"ACC-001","amount":100,"requestId":"REQ-1"}'

curl -X POST http://localhost:9092/debit-requests \
  -H "Content-Type: application/json" \
  -d '{"accountId":"ACC-001","amount":100,"requestId":"REQ-2"}'

# Query balance (should reflect sequential processing)
curl http://localhost:9090/api/v1/accounts/ACC-001/balance
```

### Integration Test - Audit Compliance
```bash
# Start payment
curl -X POST http://localhost:9090/api/v1/payments \
  -H "Content-Type: application/json" \
  -d '{"orderId":"ORD-001","customerId":"CUST-001","amount":100}'

# Query audit trail
curl http://localhost:9090/api/v1/audit/search?sagaId=<SAGA_ID>

# Export compliance report
curl http://localhost:9090/api/v1/audit/export?fromDate=2024-01-01&format=csv
```

---

## Implementation Order

1. **Phase 1: Debit Transaction Sequencing** (highest value)
   - Domain model and entities
   - Repository with locking
   - Workflow and activities
   - Kafka consumer
   - Tests

2. **Phase 2: Audit Compliance Enhancement**
   - Entity enhancements
   - Archival service
   - Immutability enforcement
   - REST endpoints
   - Tests

3. **Phase 3: Documentation Update**
   - Update CLAUDE.md with new components
   - Update temporal-kafka-integration.md with implementation status
