# Webhook → Kafka → Workflow Integration Plan

## Table of Contents

- [Overview](#overview)
- [Design Decisions](#design-decisions)
- [Implementation Phases](#implementation-phases)
  - [Phase 1: Payment Gateway Service (Kafka Publisher)](#phase-1-payment-gateway-service-kafka-publisher)
  - [Phase 2: Payment Saga Orchestrator (Kafka Consumer)](#phase-2-payment-saga-orchestrator-kafka-consumer)
  - [Phase 3: Infrastructure & Testing](#phase-3-infrastructure--testing)
- [Key Configuration](#key-configuration)
- [Workflow Correlation Strategy](#workflow-correlation-strategy)
- [Critical Files to Modify](#critical-files-to-modify)
- [Verification](#verification)
- [Metrics to Monitor](#metrics-to-monitor)
- [Summary](#summary)

---

## Overview

Implement a resilient, high-availability integration that connects webhook events from payment gateways through Kafka to trigger actions on Temporal workflows.

```
┌─────────────────────────────────────────────────────────────────────────┐
│  PAYMENT-GATEWAY-SERVICE                                                │
│  ─────────────────────────────────────────────────────────────────────  │
│  External Webhook (Stripe/PayPal/etc)                                   │
│         │                                                               │
│         ▼                                                               │
│  WebhookController → WebhookProcessorEngine → WebhookKafkaPublisher    │
│                                                      │                  │
│                                          webhook_kafka_outbox (table)   │
│                                                      │                  │
│                                          WebhookKafkaOutboxPoller       │
│                                                      │                  │
└──────────────────────────────────────────────────────┼──────────────────┘
                                                       │
                                                       ▼
┌─────────────────────────────────────────────────────────────────────────┐
│  KAFKA CLUSTER                                                          │
│  ─────────────────────────────────────────────────────────────────────  │
│  webhook.payment.events (12 partitions, keyed by orderId)              │
│  webhook.payment.events.dlt (Dead Letter Topic)                         │
└──────────────────────────────────────────────────────┼──────────────────┘
                                                       │
                                                       ▼
┌─────────────────────────────────────────────────────────────────────────┐
│  PAYMENT-SAGA-ORCHESTRATOR                                              │
│  ─────────────────────────────────────────────────────────────────────  │
│  WebhookEventConsumer (consumer-group: webhook-saga-consumer-group)     │
│         │                                                               │
│         ▼                                                               │
│  WebhookIdempotencyService → WorkflowCorrelationService                │
│                                      │                                  │
│                                      ▼                                  │
│                            WorkflowActionDispatcher                     │
│                                      │                                  │
│                                      ▼                                  │
│                            PaymentSagaWorkflow.signal()                 │
└─────────────────────────────────────────────────────────────────────────┘
```

## Design Decisions

| Decision | Choice | Rationale |
|----------|--------|-----------|
| **Webhooks signal existing workflows** | Signal, not start new | Workflows already running; webhooks confirm async operations |
| **Kafka topic partitioning** | By orderId | Ensures ordering per order; allows parallelism |
| **Delivery guarantee** | At-least-once + idempotency | Transactional outbox pattern |
| **Consumer group** | Single group, multiple instances | HA via consumer group rebalancing |

---

## Implementation Phases

### Phase 1: Payment Gateway Service (Kafka Publisher)

#### New Files

| File | Purpose |
|------|---------|
| `webhook/kafka/WebhookKafkaEvent.java` | DTO for Kafka message payload |
| `webhook/kafka/WebhookKafkaOutboxEntity.java` | JPA entity for outbox table |
| `webhook/kafka/WebhookKafkaOutboxRepository.java` | Repository with `FOR UPDATE SKIP LOCKED` |
| `webhook/kafka/WebhookKafkaPublisher.java` | Writes events to outbox (transactional) |
| `webhook/kafka/WebhookKafkaOutboxPoller.java` | Scheduled poller publishes to Kafka |
| `db/migration/V5__webhook_kafka_outbox.sql` | Creates outbox table |

#### Modify Existing

| File | Change |
|------|--------|
| `WebhookProcessorEngine.java` (line ~64) | Inject `WebhookKafkaPublisher`, call `publish()` after successful processing |
| `pom.xml` | Add `spring-kafka` dependency |
| `application.yml` | Add Kafka producer configuration |

#### WebhookKafkaEvent Structure

```java
@Data @Builder
public class WebhookKafkaEvent {
    String eventId;           // Idempotency key
    String channel;           // STRIPE, PAYPAL, etc.
    String eventType;         // PAYMENT_AUTHORIZED, PAYMENT_CAPTURED, etc.
    String orderId;           // Partition key + correlation
    String authId;            // For correlation
    String captureId;         // For correlation
    String externalReference; // Gateway reference
    BigDecimal amount;
    String currency;
    String errorCode;
    String errorMessage;
    Instant occurredAt;
    Instant receivedAt;
    Map<String, Object> metadata;
}
```

#### Database Migration: V5__webhook_kafka_outbox.sql

```sql
CREATE TABLE webhook_kafka_outbox (
    id BIGSERIAL PRIMARY KEY,
    event_id VARCHAR(36) NOT NULL UNIQUE,
    topic VARCHAR(255) NOT NULL,
    partition_key VARCHAR(100),
    payload JSONB NOT NULL,
    status VARCHAR(20) NOT NULL DEFAULT 'PENDING',
    retry_count INTEGER NOT NULL DEFAULT 0,
    last_error TEXT,
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    published_at TIMESTAMP
);

CREATE INDEX idx_webhook_kafka_outbox_status ON webhook_kafka_outbox(status);
CREATE INDEX idx_webhook_kafka_outbox_created ON webhook_kafka_outbox(created_at);
CREATE INDEX idx_webhook_kafka_outbox_pending ON webhook_kafka_outbox(created_at)
    WHERE status = 'PENDING';
```

---

### Phase 2: Payment Saga Orchestrator (Kafka Consumer)

#### New Files

| File | Purpose |
|------|---------|
| `consumer/dto/WebhookKafkaEvent.java` | Mirror DTO for deserialization |
| `consumer/dto/PaymentConfirmation.java` | Signal payload for payment confirmed |
| `consumer/dto/PaymentFailure.java` | Signal payload for payment failed |
| `consumer/dto/DisputeInfo.java` | Signal payload for dispute opened |
| `consumer/entity/WebhookConsumedEventEntity.java` | Idempotency tracking entity |
| `consumer/repository/WebhookConsumedEventRepository.java` | Repository for idempotency |
| `consumer/service/WebhookIdempotencyService.java` | Checks/records processed events |
| `consumer/service/WorkflowCorrelationService.java` | Finds workflowId from orderId/authId/captureId |
| `consumer/service/WorkflowActionDispatcher.java` | Dispatches signals to workflows |
| `consumer/WebhookEventConsumer.java` | Main `@KafkaListener` consumer |
| `consumer/config/WebhookKafkaConsumerConfig.java` | Consumer factory with DLT handler |
| `db/migration/V6__webhook_consumed_events.sql` | Creates idempotency table |

#### Modify Existing

| File | Change |
|------|--------|
| `PaymentSagaWorkflow.java` (line ~89) | Add 6 new `@SignalMethod` declarations |
| `PaymentSagaWorkflowImpl.java` | Implement signal handlers |
| `application.yml` | Add Kafka consumer configuration |

#### New Signal Methods for PaymentSagaWorkflow

```java
/**
 * Signal: External payment authorization confirmed via webhook
 */
@SignalMethod
void externalPaymentConfirmed(PaymentConfirmation confirmation);

/**
 * Signal: External capture confirmed via webhook
 */
@SignalMethod
void externalCaptureConfirmed(CaptureConfirmation confirmation);

/**
 * Signal: External payment failed via webhook
 */
@SignalMethod
void externalPaymentFailed(PaymentFailure failure);

/**
 * Signal: External refund completed via webhook
 */
@SignalMethod
void externalRefundCompleted(RefundCompletion completion);

/**
 * Signal: Dispute opened by cardholder
 */
@SignalMethod
void disputeOpened(DisputeInfo dispute);

/**
 * Signal: Authorization voided or expired externally
 */
@SignalMethod
void authorizationInvalidated(String authId, String reason);
```

#### Database Migration: V6__webhook_consumed_events.sql

```sql
CREATE TABLE webhook_consumed_events (
    id BIGSERIAL PRIMARY KEY,
    event_id VARCHAR(36) NOT NULL UNIQUE,
    event_type VARCHAR(50) NOT NULL,
    order_id VARCHAR(36),
    workflow_id VARCHAR(100),
    action_taken VARCHAR(50),
    status VARCHAR(20) NOT NULL DEFAULT 'PROCESSED',
    error_message TEXT,
    received_at TIMESTAMP,
    processed_at TIMESTAMP NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_webhook_consumed_event_id ON webhook_consumed_events(event_id);
CREATE INDEX idx_webhook_consumed_order_id ON webhook_consumed_events(order_id);
CREATE INDEX idx_webhook_consumed_workflow_id ON webhook_consumed_events(workflow_id);
CREATE INDEX idx_webhook_consumed_status ON webhook_consumed_events(status);
```

---

### Phase 3: Infrastructure & Testing

#### Docker Compose Changes

```yaml
kafka-init:
  image: confluentinc/cp-kafka:7.5.0
  depends_on:
    kafka:
      condition: service_healthy
  entrypoint: ["/bin/sh", "-c"]
  command: |
    "
    kafka-topics --bootstrap-server kafka:9092 --create --if-not-exists \
      --topic webhook.payment.events --partitions 12 --replication-factor 1
    kafka-topics --bootstrap-server kafka:9092 --create --if-not-exists \
      --topic webhook.payment.events.dlt --partitions 3 --replication-factor 1
    "
```

#### Test Files

| File | Type |
|------|------|
| `WebhookKafkaPublisherTest.java` | Unit test - outbox creation |
| `WebhookKafkaOutboxPollerTest.java` | Unit test - polling/publishing |
| `WebhookEventConsumerTest.java` | Unit test - consumption |
| `WorkflowCorrelationServiceTest.java` | Unit test - correlation strategies |
| `WebhookToKafkaIntegrationTest.java` | Integration - gateway to Kafka |
| `KafkaToWorkflowIntegrationTest.java` | Integration - Kafka to workflow |

---

## Key Configuration

### Kafka Producer Config (payment-gateway-service)

```yaml
spring:
  kafka:
    bootstrap-servers: ${KAFKA_BOOTSTRAP_SERVERS:localhost:9092}
    producer:
      key-serializer: org.apache.kafka.common.serialization.StringSerializer
      value-serializer: org.apache.kafka.common.serialization.StringSerializer
      acks: all
      retries: 3
      properties:
        enable.idempotence: true

webhook:
  kafka:
    enabled: true
    topic: webhook.payment.events
    poller:
      batch-size: 50
      interval-ms: 100
      retry-interval-ms: 60000
      max-retries: 5
```

### Kafka Consumer Config (payment-saga-orchestrator)

```yaml
webhook:
  kafka:
    enabled: true
    topic: webhook.payment.events
    consumer:
      group-id: webhook-saga-consumer-group
      concurrency: 3
      auto-offset-reset: earliest
      enable-auto-commit: false
      max-poll-records: 10
      session-timeout-ms: 30000
      max-poll-interval-ms: 300000
```

### Error Handling

- **Retry**: Exponential backoff (1s initial, 2x multiplier, 5 max attempts)
- **After max retries**: Publish to DLT topic (`webhook.payment.events.dlt`)
- **Idempotency**: Record all consumed events in `webhook_consumed_events` table

---

## Workflow Correlation Strategy

The consumer must find the `workflowId` (sagaId) from webhook event data:

```
1. By orderId:   PaymentRequestRepository.findByOrderId() → sagaId
2. By authId:    TransactionRepository.findByGatewayReference() → sagaId
3. By captureId: TransactionRepository.findByGatewayReference() → sagaId
```

```java
public Optional<String> findWorkflowId(WebhookKafkaEvent event) {
    // Strategy 1: Direct orderId lookup (most common)
    if (event.getOrderId() != null) {
        return paymentRequestRepository.findByOrderId(event.getOrderId())
            .map(PaymentRequestEntity::getSagaId);
    }

    // Strategy 2: AuthId lookup via transactions
    if (event.getAuthId() != null) {
        return transactionRepository.findByGatewayReference(event.getAuthId())
            .stream().findFirst()
            .map(TransactionEntity::getSagaId);
    }

    // Strategy 3: CaptureId lookup
    if (event.getCaptureId() != null) {
        return transactionRepository.findByGatewayReference(event.getCaptureId())
            .stream().findFirst()
            .map(TransactionEntity::getSagaId);
    }

    return Optional.empty();
}
```

---

## Critical Files to Modify

| File | Path |
|------|------|
| WebhookProcessorEngine | `payment-gateway-service/src/main/java/com/payment/gateway/webhook/processor/WebhookProcessorEngine.java` |
| PaymentSagaWorkflow | `payment-saga-orchestrator/src/main/java/com/payment/saga/workflow/PaymentSagaWorkflow.java` |
| PaymentSagaWorkflowImpl | `payment-saga-orchestrator/src/main/java/com/payment/saga/workflow/PaymentSagaWorkflowImpl.java` |
| OutboxPoller (pattern to follow) | `payment-saga-orchestrator/src/main/java/com/payment/saga/outbox/OutboxPoller.java` |

---

## Verification

### Unit Tests
```bash
mvn test -Dtest=WebhookKafka*Test -pl payment-gateway-service
mvn test -Dtest=WebhookEventConsumer*Test -pl payment-saga-orchestrator
```

### Integration Tests
```bash
mvn test -Dtest=*IntegrationTest
```

### Manual E2E Test

1. **Start infrastructure**:
   ```bash
   docker compose up -d
   ```

2. **Start a payment workflow**:
   ```bash
   curl -X POST http://localhost:9090/api/v1/payments \
     -H "Content-Type: application/json" \
     -d '{"orderId": "ORD-001", "customerId": "CUST-001", "amount": 100.00, ...}'
   ```

3. **Simulate webhook** (Stripe payment captured):
   ```bash
   curl -X POST http://localhost:8083/api/webhooks/stripe \
     -H "Content-Type: application/json" \
     -H "Stripe-Signature: ..." \
     -d '{"type": "payment_intent.succeeded", "data": {"object": {"metadata": {"order_id": "ORD-001"}}}}'
   ```

4. **Verify Kafka message**:
   ```bash
   kafka-console-consumer --bootstrap-server localhost:9092 \
     --topic webhook.payment.events --from-beginning
   ```

5. **Verify workflow received signal**:
   ```bash
   curl http://localhost:9090/api/v1/payments/{workflowId}
   ```

6. **Test idempotency** (resend same webhook, should skip):
   ```bash
   # Same curl command as step 3
   # Check logs for "Duplicate event, skipping"
   ```

---

## Metrics to Monitor

| Metric | Alert Threshold |
|--------|-----------------|
| `webhook_kafka_outbox_pending_total` | > 1000 |
| `webhook_consumer_lag` | > 5 minutes |
| `webhook_consumer_errors_total` | Rate > 5% |
| `webhook.payment.events.dlt` message count | Any (requires investigation) |
| `webhook_consumer_duplicates_total` | Informational only |
| `webhook_consumer_correlation_failures` | > 10/minute |

---

## Summary

This integration provides:

1. **Reliability**: Transactional outbox ensures no events lost even if Kafka is down
2. **Scalability**: Partitioned topic allows horizontal scaling of consumers
3. **Resilience**: Dead letter queue captures failed events for investigation
4. **Idempotency**: Three-layer protection against duplicate processing
5. **Observability**: Metrics and logging at every stage
6. **Ordering**: Per-order message ordering via partition key
