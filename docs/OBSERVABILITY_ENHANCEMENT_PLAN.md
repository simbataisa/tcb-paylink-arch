# Implementation Plan: Enhanced Observability and Monitoring

## Table of Contents

- [Overview](#overview)
- [Current State](#current-state)
- [Phase 1: Correlation ID Propagation](#phase-1-correlation-id-propagation)
- [Phase 2: Cloud Resource Metrics](#phase-2-cloud-resource-metrics)
- [Phase 3: Business Metrics](#phase-3-business-metrics)
- [Phase 4: Dashboards & Alerting](#phase-4-dashboards--alerting)
- [Files Summary](#files-summary)
- [Verification](#verification)
- [Implementation Order](#implementation-order)

---

## Overview

Enhance observability across the payment-saga platform with:
1. **End-to-end correlation tracking** across sync and async boundaries
2. **Cloud resource metrics** (JVM, DB pools, Kafka, Temporal)
3. **Business metrics** (success/failure rates by payment channel, funnel stage)

---

## Current State

| Component | Status | Gap |
|-----------|--------|-----|
| Correlation ID Filter | Exists | Not propagated to Feign, Kafka, Temporal, async pools |
| Prometheus/Micrometer | Configured | Limited business metrics |
| Grafana Dashboard | 6 panels | No funnel/channel breakdown |
| MDC Logging | Configured | Pattern uses non-existent keys |

---

## Phase 1: Correlation ID Propagation

### 1.1 MdcTaskDecorator for Async Pools
**Create:** `payment-saga-orchestrator/src/main/java/com/payment/saga/observability/MdcTaskDecorator.java`

### 1.2 Feign Client Interceptor
**Create:** `payment-saga-orchestrator/src/main/java/com/payment/saga/config/FeignCorrelationIdInterceptor.java`

### 1.3 Kafka Header Propagation
**Modify:** `OutboxPoller.java` - Inject correlation ID header
**Modify:** `WebhookEventConsumer.java` - Extract and set MDC

### 1.4 Temporal Context Propagator
**Create:** `payment-saga-orchestrator/src/main/java/com/payment/saga/observability/CorrelationIdContextPropagator.java`
**Modify:** `TemporalConfiguration.java` - Register context propagator

### 1.5 Fix Logging Pattern
**Modify:** `application.yml` - Update logging pattern to use correct MDC keys

---

## Phase 2: Cloud Resource Metrics

### 2.1 Files to Create/Modify

| File | Action | Purpose |
|------|--------|---------|
| `observability/KafkaMetricsConfig.java` | CREATE | Consumer lag, processing time |
| `observability/TemporalMetricsConfig.java` | CREATE | Task queue depth polling |
| `AsyncConfiguration.java` | MODIFY | ExecutorServiceMetrics binding |
| `application.yml` | MODIFY | HikariCP metrics enabled |

### 2.2 Metrics to Expose

| Metric | Source | Tags |
|--------|--------|------|
| `kafka.consumer.messages.consumed` | KafkaMetricsConfig | topic, partition |
| `kafka.consumer.processing.time` | KafkaMetricsConfig | topic |
| `temporal.task_queue.pollers` | TemporalMetricsConfig | queue |
| `executor.pool.size` | Micrometer auto | pool_name |
| `hikaricp.connections.*` | HikariCP | pool |

---

## Phase 3: Business Metrics

### 3.1 Enhanced MetricsService Interface
**Modify:** `MetricsService.java` - Add business metrics methods

### 3.2 Key Business Metrics

| Metric | Tags | Purpose |
|--------|------|---------|
| `payment.funnel.initiated` | payment_method | Funnel entry |
| `payment.funnel.completed` | payment_method | Success count |
| `payment.funnel.failed` | payment_method, error_code, error_category | Failure breakdown |
| `payment.funnel.stage` | stage, payment_method, outcome | Stage conversion |
| `payment.funnel.duration` | payment_method, outcome | E2E latency |
| `webhook.received` | channel, event_type | Webhook volume |
| `webhook.processing.duration` | channel, event_type | Webhook latency |

### 3.3 Integration Points

| Component | Metrics Added |
|-----------|---------------|
| `PaymentActivitiesImpl` | Funnel stage metrics per activity |
| `WebhookEventConsumer` | Webhook received/processed metrics |
| `PaymentSagaWorkflowImpl` (via activities) | Payment initiated/completed/failed |
| `CompensationActivitiesImpl` | Compensation started/completed |

---

## Phase 4: Dashboards & Alerting

### 4.1 New Grafana Dashboards

**Create:** `observability/grafana/provisioning/dashboards/json/payment-business-metrics.json`
- Payment Success Rate by Channel (pie chart)
- Funnel Stage Conversion (stacked bar)
- Error Distribution by Category (pie chart)
- Processing Time P50/P95/P99 (line chart)

**Create:** `observability/grafana/provisioning/dashboards/json/payment-infrastructure.json`
- JVM Heap Usage (all services)
- HikariCP Pool Utilization
- Kafka Consumer Lag
- Thread Pool Metrics

### 4.2 Alerting Rules

**Create:** `observability/prometheus/alerting-rules.yml`

---

## Files Summary

| Phase | Action | File |
|-------|--------|------|
| 1 | CREATE | `observability/MdcTaskDecorator.java` |
| 1 | CREATE | `config/FeignCorrelationIdInterceptor.java` |
| 1 | CREATE | `observability/CorrelationIdContextPropagator.java` |
| 1 | MODIFY | `config/AsyncConfiguration.java` |
| 1 | MODIFY | `outbox/OutboxPoller.java` |
| 1 | MODIFY | `consumer/WebhookEventConsumer.java` |
| 1 | MODIFY | `config/TemporalConfiguration.java` |
| 1 | MODIFY | `resources/application.yml` |
| 2 | CREATE | `observability/KafkaMetricsConfig.java` |
| 2 | CREATE | `observability/TemporalMetricsConfig.java` |
| 3 | MODIFY | `service/MetricsService.java` |
| 3 | MODIFY | `service/impl/MetricsServiceImpl.java` |
| 3 | MODIFY | `activity/PaymentActivitiesImpl.java` |
| 3 | MODIFY | `consumer/WebhookEventConsumer.java` |
| 4 | CREATE | `observability/grafana/.../payment-business-metrics.json` |
| 4 | CREATE | `observability/grafana/.../payment-infrastructure.json` |
| 4 | CREATE | `observability/prometheus/alerting-rules.yml` |

---

## Verification

### Unit Tests
```bash
# Test correlation propagation
mvn test -Dtest="MdcTaskDecoratorTest,FeignCorrelationIdInterceptorTest"

# Test metrics recording
mvn test -Dtest="MetricsServiceImplTest"
```

### Integration Tests
```bash
# Full E2E correlation test
mvn test -Dtest="CorrelationPropagationIntegrationTest"

# Kafka header propagation
mvn test -Dtest="KafkaHeaderPropagationTest"
```

### Manual Verification
```bash
# Start infrastructure
docker compose up -d

# Start application
mvn spring-boot:run -pl payment-saga-orchestrator

# Make a payment request
curl -X POST http://localhost:9090/api/v1/payments \
  -H "Content-Type: application/json" \
  -H "X-Correlation-ID: test-correlation-123" \
  -d '{"orderId":"ORD-001","customerId":"CUST-001","amount":100}'

# Verify correlation in logs
docker compose logs payment-saga-orchestrator | grep "test-correlation-123"

# Check Prometheus metrics
curl http://localhost:9090/actuator/prometheus | grep "payment.funnel"

# View Grafana dashboards
open http://localhost:3000
```

---

## Implementation Order

| # | Component | Depends On |
|---|-----------|------------|
| 1 | MdcTaskDecorator | - |
| 2 | FeignCorrelationIdInterceptor | - |
| 3 | Kafka header injection/extraction | - |
| 4 | CorrelationIdContextPropagator | - |
| 5 | Fix logging pattern | 1-4 |
| 6 | Enhanced MetricsService | - |
| 7 | Integrate metrics in activities | 6 |
| 8 | Cloud resource metrics configs | - |
| 9 | Business dashboard | 6-7 |
| 10 | Infrastructure dashboard | 8 |
| 11 | Alerting rules | 9-10 |
