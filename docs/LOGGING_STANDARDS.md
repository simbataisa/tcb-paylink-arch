# Logging Standards

This document defines the logging conventions for the Payment SAGA Platform, ensuring consistent log formatting across all services for effective debugging, monitoring, and log aggregation.

## Table of Contents

- [Log Pattern Specification](#log-pattern-specification)
- [MDC Key Definitions](#mdc-key-definitions)
- [Log Prefix Conventions](#log-prefix-conventions)
- [Error Logging Guidelines](#error-logging-guidelines)
- [Sensitive Data Handling](#sensitive-data-handling)
- [Example Log Statements](#example-log-statements)
- [Log Aggregation](#log-aggregation)

---

## Log Pattern Specification

All services use the following unified log pattern:

```
%d{yyyy-MM-dd HH:mm:ss.SSS} [%thread] [svc=%X{service:-}] [cid=%X{correlationId:-}] [tid=%X{traceId:-}] %-5level %logger{36} - %msg%n
```

### Pattern Components

| Component | Description | Example |
|-----------|-------------|---------|
| `%d{...}` | Timestamp in ISO format | `2024-01-15 10:30:45.123` |
| `[%thread]` | Thread name | `[http-nio-8081-exec-1]` |
| `[svc=%X{service:-}]` | Service name from MDC | `[svc=order-service]` |
| `[cid=%X{correlationId:-}]` | Correlation ID from MDC | `[cid=abc-123-def]` |
| `[tid=%X{traceId:-}]` | Trace ID from MDC | `[tid=xyz-789]` |
| `%-5level` | Log level (padded) | `INFO `, `DEBUG`, `ERROR` |
| `%logger{36}` | Logger name (truncated) | `c.p.order.service.OrderServiceImpl` |
| `%msg%n` | Log message with newline | `[ORDER-SVC] Order validated...` |

### Orchestrator Extended Pattern

The orchestrator includes an additional workflow ID for SAGA tracking:

```
[wf=%X{workflowId:-}]
```

---

## MDC Key Definitions

The following MDC (Mapped Diagnostic Context) keys are used across all services:

| Key | Description | Set By | Propagation |
|-----|-------------|--------|-------------|
| `correlationId` | Unique ID for request tracing across services | `CorrelationIdFilter` | HTTP Header: `X-Correlation-ID` |
| `requestId` | Unique ID for individual requests | `CorrelationIdFilter` | Internal only |
| `service` | Current service name | `CorrelationIdFilter` | N/A (set per service) |
| `traceId` | Distributed trace ID (B3) | Micrometer Tracing | B3 headers |
| `workflowId` | Temporal workflow ID (orchestrator only) | Workflow activities | Internal only |

### Key Management

```java
// Access MDC keys via LoggingConstants
import static com.payment.saga.common.observability.LoggingConstants.*;

MDC.put(CORRELATION_ID_MDC_KEY, correlationId);
MDC.put(SERVICE_MDC_KEY, serviceName);
```

### Correlation ID Propagation

The `CorrelationIdFilter` (from `payment-saga-common`) automatically:
1. Extracts `X-Correlation-ID` from incoming requests
2. Generates a new UUID if not present
3. Sets MDC keys for logging
4. Adds `X-Correlation-ID` to response headers

Feign clients propagate correlation IDs via `FeignCorrelationIdInterceptor`.

---

## Log Prefix Conventions

Use consistent prefixes to categorize log messages by component layer:

### Layer Prefixes

| Layer | Prefix | Usage |
|-------|--------|-------|
| Controller/API | `[API]` | REST endpoint entry/exit, request validation |
| Service | `[{SERVICE}-SVC]` | Business logic operations |
| Activity (Temporal) | `[ACTIVITY-*]` | Temporal activity execution |
| State Machine | `[SM-*]` | State transitions, guards, actions |
| Consumer (Kafka) | `[*-CONSUMER]` | Message consumption, processing |
| Client (Feign) | `[CLIENT]` or `[RESILIENT-CLIENT]` | External service calls |
| Fallback | `[FALLBACK-*]` | Circuit breaker fallbacks |
| Repository | `[REPO]` | Database operations (if logged) |

### Service-Specific Prefixes

| Service | Prefix | Example |
|---------|--------|---------|
| Order Service | `[ORDER-SVC]` | `[ORDER-SVC] Validating order: orderId=ORD-123` |
| Inventory Service | `[INVENTORY-SVC]` | `[INVENTORY-SVC] Reserving inventory: orderId=ORD-123` |
| Payment Gateway | `[PAYMENT-SVC]` | `[PAYMENT-SVC] Authorizing payment: orderId=ORD-123` |
| Webhook Processing | `[WEBHOOK]` | `[WEBHOOK] Processing webhook: channel=stripe` |

### Activity Prefixes (Orchestrator)

| Prefix | Usage |
|--------|-------|
| `[ACTIVITY-START]` | Activity execution started |
| `[ACTIVITY-END]` | Activity execution completed |
| `[ACTIVITY-FAILED]` | Activity execution failed |
| `[ACTIVITY-IDEMPOTENT]` | Idempotent activity skipped (already executed) |
| `[DATA-ACTIVITY]` | Data access activities |
| `[DEBIT-ACTIVITY]` | Debit/credit operations |

### State Machine Prefixes

| Prefix | Usage |
|--------|-------|
| `[SM-INIT]` | State machine initialization |
| `[SM-TRANSITION]` | State transitions |
| `[SM-QUERY]` | State queries |

### Error Prefixes

| Prefix | Usage |
|--------|-------|
| `[BUSINESS-ERROR]` | Business rule violations |
| `[SYSTEM-ERROR]` | System/infrastructure errors |
| `[VALIDATION-ERROR]` | Input validation failures |
| `[FEIGN-ERROR]` | External service call failures |
| `[TEMPORAL-FAILURE]` | Temporal workflow failures |

---

## Error Logging Guidelines

### Log Levels

| Level | Usage | Examples |
|-------|-------|----------|
| `ERROR` | System failures requiring attention | Database connection lost, external service down |
| `WARN` | Unexpected but recoverable situations | Retry attempts, fallback activation |
| `INFO` | Business operations, state changes | Order created, payment captured |
| `DEBUG` | Detailed diagnostic information | Method parameters, query results |
| `TRACE` | Very detailed debugging (dev only) | Full request/response bodies |

### Error Logging Best Practices

```java
// DO: Include contextual information
log.error("[PAYMENT-SVC] Payment authorization failed: orderId={} amount={} error={}",
    orderId, amount, e.getMessage(), e);

// DON'T: Vague error messages
log.error("Error occurred", e);

// DO: Log at appropriate level for retries
log.warn("[RESILIENT-CLIENT] Retry attempt {}: service={} error={}",
    attempt, serviceName, e.getMessage());

// DO: Include correlation context (automatic via MDC)
log.error("[FALLBACK-ORDER] validateOrder: orderId={} error={}",
    orderId, e.getMessage());
```

### Exception Logging

- Always include the exception as the last argument to preserve stack traces
- Include relevant business context (IDs, states, amounts)
- Use `WARN` for expected failures, `ERROR` for unexpected ones

---

## Sensitive Data Handling

### Never Log

- Full credit card numbers (use last 4 digits only)
- CVV/CVC codes
- Passwords or API keys
- Full Social Security Numbers
- OAuth tokens or session IDs

### Mask or Truncate

```java
// Credit card masking
String maskedCard = "**** **** **** " + cardNumber.substring(cardNumber.length() - 4);
log.info("[PAYMENT-SVC] Processing card: {}", maskedCard);

// Email masking
String maskedEmail = email.replaceAll("(?<=.{2}).(?=.*@)", "*");
log.info("[ORDER-SVC] Customer email: {}", maskedEmail);
```

### Safe to Log

- Order IDs, transaction IDs, workflow IDs
- Amounts and currencies (monetary values)
- Status codes and states
- Timestamps
- Service names and endpoints

---

## Example Log Statements

### API Layer
```java
log.info("[API] Processing payment: orderId={}", request.getOrderId());
log.debug("[API] Payment request stored: workflowId={} correlationId={}",
    workflowId, correlationId);
log.error("[API] Failed to start payment workflow: orderId={} error={}",
    request.getOrderId(), e.getMessage(), e);
```

### Service Layer
```java
log.info("[ORDER-SVC] Validating order: orderId={}", orderId);
log.info("[ORDER-SVC] Order status updated: orderId={} {}->{}",
    orderId, previousStatus, newStatus);
log.warn("[INVENTORY-SVC] Insufficient stock: sku={} requested={} available={}",
    sku, requested, available);
```

### Temporal Activities
```java
log.info("[ACTIVITY-START] validateOrder: id={}", orderId);
log.info("[ACTIVITY-END] validateOrder: id={} duration={}ms", orderId, duration);
log.info("[ACTIVITY-IDEMPOTENT] Order already validated: orderId={}", orderId);
```

### Kafka Consumer
```java
log.info("[WEBHOOK-CONSUMER] Received message: partition={} offset={} key={}",
    partition, offset, key);
log.info("[WEBHOOK-CONSUMER] Successfully processed event: eventId={} workflowId={}",
    eventId, workflowId);
log.error("[WEBHOOK-CONSUMER] Error processing message: partition={} offset={} error={}",
    partition, offset, e.getMessage(), e);
```

### State Machine
```java
log.info("[SM-INIT] Initializing state machine: workflowId={} orderId={}",
    workflowId, orderId);
log.info("[SM-TRANSITION] State changed: workflowId={} {}->{}",
    workflowId, previousState, newState);
```

### Full Trace Example
```
# Request flow across services (same correlationId)

2024-01-15 10:30:45.123 [http-nio-9090-exec-1] [svc=payment-saga-orchestrator] [cid=abc-123-def] [tid=xyz-789] INFO PaymentController - [API] Processing payment: orderId=ORD-001

2024-01-15 10:30:45.150 [http-nio-8081-exec-1] [svc=order-service] [cid=abc-123-def] [tid=xyz-789] INFO OrderServiceImpl - [ORDER-SVC] Validating order: orderId=ORD-001

2024-01-15 10:30:45.200 [http-nio-8082-exec-1] [svc=inventory-service] [cid=abc-123-def] [tid=xyz-789] INFO InventoryServiceImpl - [INVENTORY-SVC] Reserving inventory: orderId=ORD-001

2024-01-15 10:30:45.250 [http-nio-8083-exec-1] [svc=payment-gateway-service] [cid=abc-123-def] [tid=xyz-789] INFO PaymentGatewayServiceImpl - [PAYMENT-SVC] Authorizing payment: orderId=ORD-001
```

---

## Log Aggregation

### Filtering by Correlation ID

```bash
# Find all logs for a specific request
grep "cid=abc-123-def" /var/log/payment-saga/*.log

# Docker compose
docker compose logs | grep "cid=abc-123-def"
```

### ELK Stack Queries

```json
// Kibana query for correlation ID
{
  "query": {
    "match": {
      "correlationId": "abc-123-def"
    }
  }
}
```

### Grafana Loki

```logql
{service=~"order-service|inventory-service|payment-gateway-service"} |= "cid=abc-123-def"
```

---

## Configuration Reference

### application.yml Template

```yaml
logging:
  level:
    root: INFO
    com.payment: DEBUG
    com.inventory: DEBUG
  pattern:
    console: "%d{yyyy-MM-dd HH:mm:ss.SSS} [%thread] [svc=%X{service:-}] [cid=%X{correlationId:-}] [tid=%X{traceId:-}] %-5level %logger{36} - %msg%n"
```

### LoggingConstants Reference

```java
// MDC Keys
public static final String CORRELATION_ID_MDC_KEY = "correlationId";
public static final String REQUEST_ID_MDC_KEY = "requestId";
public static final String SERVICE_MDC_KEY = "service";
public static final String WORKFLOW_ID_MDC_KEY = "workflowId";
public static final String TRACE_ID_MDC_KEY = "traceId";

// HTTP Headers
public static final String CORRELATION_ID_HEADER = "X-Correlation-ID";
public static final String REQUEST_ID_HEADER = "X-Request-ID";

// Log Prefixes
public static final String PREFIX_API = "[API]";
public static final String PREFIX_SVC = "[SVC]";
public static final String PREFIX_ORDER_SVC = "[ORDER-SVC]";
public static final String PREFIX_INVENTORY_SVC = "[INVENTORY-SVC]";
public static final String PREFIX_PAYMENT_SVC = "[PAYMENT-SVC]";
```

---

## Related Documentation

- [Observability.md](Observability.md) - Overall observability strategy
- [CLAUDE.md](../CLAUDE.md) - Project coding conventions
