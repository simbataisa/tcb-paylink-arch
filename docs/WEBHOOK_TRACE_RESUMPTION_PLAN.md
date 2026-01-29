# Webhook Trace Resumption Plan

## Table of Contents

- [Problem Statement](#problem-statement)
- [Solution: Correlation Registry Pattern](#solution-correlation-registry-pattern)
- [Implementation](#implementation)
  - [Phase 1: Database Schema Change](#phase-1-database-schema-change)
  - [Phase 2: Correlation Registry (Redis)](#phase-2-correlation-registry-redis)
  - [Phase 3: Store Correlation at Payment Initiation](#phase-3-store-correlation-at-payment-initiation)
  - [Phase 4: Add Reference Mappings After Gateway Calls](#phase-4-add-reference-mappings-after-gateway-calls)
  - [Phase 5: Propagate Correlation in Webhook Kafka Headers](#phase-5-propagate-correlation-in-webhook-kafka-headers)
  - [Phase 6: Resume Trace in Consumer](#phase-6-resume-trace-in-consumer)
- [Files Summary](#files-summary)
- [Verification](#verification)
  - [Unit Tests](#unit-tests)
  - [Integration Test](#integration-test)
  - [Manual E2E Verification](#manual-e2e-verification)
- [Design Decisions](#design-decisions)

---

## Problem Statement

When async webhook responses arrive from external payment gateways (Stripe, PayPal, etc.), the distributed trace is broken:

```
Original Request (cid=abc-123) → Payment Gateway → [hours/days pass] → Webhook arrives → NEW cid generated
```

**Root Cause**: Correlation ID exists only in thread-local MDC, not persisted. Webhooks arrive without original correlation context.

---

## Solution: Correlation Registry Pattern

```
┌─────────────────────────────────────────────────────────────────────────────┐
│ PAYMENT INITIATION                                                          │
│ 1. Request arrives with X-Correlation-ID: abc-123                          │
│ 2. Store in PaymentRequestEntity.correlationId                             │
│ 3. Store in Redis: correlation:order:ORD-001 → {correlationId, traceId}    │
│ 4. When auth/capture completes, add reference mappings                      │
└─────────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│ WEBHOOK ARRIVAL (hours/days later)                                          │
│ 1. Webhook contains orderId/authId/captureId                               │
│ 2. Lookup correlation from Redis (fast) or DB (fallback)                   │
│ 3. Restore MDC context with original correlation ID                         │
│ 4. Add Kafka headers: X-Correlation-ID, X-B3-TraceId, X-B3-ParentSpanId    │
│ 5. Consumer extracts headers, resumes trace                                 │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Implementation

### Phase 1: Database Schema Change

**Create:** `payment-saga-orchestrator/src/main/resources/db/migration/V9__correlation_id_tracking.sql`

```sql
ALTER TABLE payment_requests ADD COLUMN correlation_id VARCHAR(100);
ALTER TABLE payment_requests ADD COLUMN trace_id VARCHAR(32);
ALTER TABLE payment_requests ADD COLUMN parent_span_id VARCHAR(16);

CREATE INDEX idx_payment_req_correlation ON payment_requests(correlation_id);
```

### Phase 2: Correlation Registry (Redis)

**Create in `payment-saga-common`:**

| File | Purpose |
|------|---------|
| `correlation/CorrelationContext.java` | DTO with correlationId, traceId, parentSpanId, orderId |
| `correlation/CorrelationRegistry.java` | Interface for store/lookup operations |
| `correlation/RedisCorrelationRegistry.java` | Redis implementation with 7-day TTL |

```java
public interface CorrelationRegistry {
    void store(String orderId, CorrelationContext context);
    Optional<CorrelationContext> lookupByOrderId(String orderId);
    Optional<CorrelationContext> lookupByReference(String reference);
    void addReference(String orderId, String reference); // authId, captureId mapping
}
```

**Redis Key Structure:**
- `correlation:order:{orderId}` → JSON CorrelationContext
- `correlation:ref:{authId}` → orderId (pointer)
- `correlation:ref:{captureId}` → orderId (pointer)

### Phase 3: Store Correlation at Payment Initiation

**Modify:** `PaymentRequestEntity.java`
```java
@Column(name = "correlation_id", length = 100)
private String correlationId;

@Column(name = "trace_id", length = 32)
private String traceId;

@Column(name = "parent_span_id", length = 16)
private String parentSpanId;
```

**Modify:** `PaymentController.java`
```java
// Extract from current trace context
String correlationId = MDC.get(CorrelationIdFilter.CORRELATION_ID_MDC_KEY);
Span currentSpan = tracer.currentSpan();
String traceId = currentSpan != null ? currentSpan.context().traceIdString() : null;
String spanId = currentSpan != null ? currentSpan.context().spanIdString() : null;

// Store in Redis for cross-service lookup
correlationRegistry.store(orderId, CorrelationContext.builder()
    .correlationId(correlationId)
    .traceId(traceId)
    .parentSpanId(spanId)
    .orderId(orderId)
    .build());
```

### Phase 4: Add Reference Mappings After Gateway Calls

**Modify:** `PaymentActivitiesImpl.java`
```java
// After successful authorization
correlationRegistry.addReference(orderId, authorizationResult.getAuthId());

// After successful capture
correlationRegistry.addReference(orderId, captureResult.getCaptureId());
```

### Phase 5: Propagate Correlation in Webhook Kafka Headers

**Modify:** `WebhookKafkaOutboxPoller.java` (payment-gateway-service)
```java
private void publishEvent(WebhookKafkaOutboxEntity event) {
    // Lookup correlation from Redis
    Optional<CorrelationContext> ctx = correlationRegistry.lookupByOrderId(event.getOrderId());
    if (ctx.isEmpty()) {
        ctx = correlationRegistry.lookupByReference(event.getAuthId());
    }

    ProducerRecord<String, String> record = new ProducerRecord<>(topic, key, payload);

    if (ctx.isPresent()) {
        CorrelationContext c = ctx.get();
        record.headers().add("X-Correlation-ID", c.getCorrelationId().getBytes(UTF_8));
        record.headers().add("X-B3-TraceId", c.getTraceId().getBytes(UTF_8));
        record.headers().add("X-B3-ParentSpanId", c.getParentSpanId().getBytes(UTF_8));
    }

    kafkaTemplate.send(record);
}
```

### Phase 6: Resume Trace in Consumer

**Create:** `TraceContextRestorer.java`
```java
@Component
@RequiredArgsConstructor
public class TraceContextRestorer {
    private final Tracer tracer;

    public Span resumeTrace(String traceId, String parentSpanId, String operationName) {
        if (traceId == null) return tracer.nextSpan().name(operationName).start();

        TraceContext parent = TraceContext.newBuilder()
            .traceId(Long.parseUnsignedLong(traceId.substring(0, 16), 16),
                     Long.parseUnsignedLong(traceId.substring(16), 16))
            .spanId(Long.parseUnsignedLong(parentSpanId, 16))
            .sampled(true)
            .build();

        return tracer.newChild(parent).name(operationName).start();
    }
}
```

**Modify:** `WebhookEventConsumer.java`
```java
public void consume(ConsumerRecord<String, String> record, Acknowledgment ack) {
    // Extract correlation headers
    String correlationId = extractHeader(record, "X-Correlation-ID");
    String traceId = extractHeader(record, "X-B3-TraceId");
    String parentSpanId = extractHeader(record, "X-B3-ParentSpanId");

    // Fallback lookup if headers missing
    if (correlationId == null) {
        WebhookKafkaEvent event = objectMapper.readValue(record.value(), WebhookKafkaEvent.class);
        Optional<CorrelationContext> ctx = correlationRegistry.lookupByOrderId(event.getOrderId());
        if (ctx.isPresent()) {
            correlationId = ctx.get().getCorrelationId();
            traceId = ctx.get().getTraceId();
            parentSpanId = ctx.get().getParentSpanId();
        } else {
            correlationId = UUID.randomUUID().toString(); // Last resort
        }
    }

    // Set MDC
    MDC.put(CorrelationIdFilter.CORRELATION_ID_MDC_KEY, correlationId);

    // Resume Zipkin trace
    Span span = traceContextRestorer.resumeTrace(traceId, parentSpanId, "webhook-process");
    try (Tracer.SpanInScope ws = tracer.withSpanInScope(span)) {
        // Process webhook...
    } finally {
        span.finish();
    }
}
```

---

## Files Summary

| Action | File |
|--------|------|
| CREATE | `payment-saga-orchestrator/src/main/resources/db/migration/V9__correlation_id_tracking.sql` |
| CREATE | `payment-saga-common/src/main/java/.../correlation/CorrelationContext.java` |
| CREATE | `payment-saga-common/src/main/java/.../correlation/CorrelationRegistry.java` |
| CREATE | `payment-saga-common/src/main/java/.../correlation/RedisCorrelationRegistry.java` |
| CREATE | `payment-saga-orchestrator/src/main/java/.../observability/TraceContextRestorer.java` |
| MODIFY | `payment-saga-orchestrator/src/main/java/.../entity/PaymentRequestEntity.java` |
| MODIFY | `payment-saga-orchestrator/src/main/java/.../api/PaymentController.java` |
| MODIFY | `payment-saga-orchestrator/src/main/java/.../activity/impl/PaymentActivitiesImpl.java` |
| MODIFY | `payment-gateway-service/src/main/java/.../webhook/kafka/WebhookKafkaOutboxPoller.java` |
| MODIFY | `payment-saga-orchestrator/src/main/java/.../consumer/WebhookEventConsumer.java` |

---

## Verification

### Unit Tests
```bash
mvn test -Dtest="RedisCorrelationRegistryTest,TraceContextRestorerTest"
```

### Integration Test
```bash
mvn test -Dtest="WebhookTraceResumptionIntegrationTest"
```

### Manual E2E Verification
```bash
# 1. Start infrastructure
docker compose up -d

# 2. Make payment request with correlation ID
curl -X POST http://localhost:9090/api/v1/payments \
  -H "Content-Type: application/json" \
  -H "X-Correlation-ID: original-trace-abc-123" \
  -d '{"orderId":"ORD-TEST","customerId":"CUST-001","amount":100}'

# 3. Note the trace ID in Zipkin
open http://localhost:9411

# 4. Simulate webhook (externally triggered)
curl -X POST http://localhost:8083/api/webhooks/stripe \
  -H "Content-Type: application/json" \
  -d '{"id":"evt_123","type":"payment_intent.succeeded","data":{"object":{"metadata":{"order_id":"ORD-TEST"}}}}'

# 5. Verify in Zipkin: webhook span appears in SAME trace as original request
# 6. Verify in logs: grep for "original-trace-abc-123" in orchestrator logs
docker compose logs payment-saga-orchestrator | grep "original-trace-abc-123"
```

---

## Design Decisions

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Storage | Redis + DB fallback | Fast cross-service lookup, database as backup |
| TTL | 7 days | Matches outbox retention, covers async webhook delays |
| Trace linking | Same trace ID (child span) | True end-to-end visibility in Zipkin |
| Fallback | Generate new UUID | Graceful degradation when lookup fails |
