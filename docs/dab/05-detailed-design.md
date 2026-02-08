# II.4 Detailed Design

[< Back to Index](../DAB_Payment_SAGA_Platform.md) | [← Previous: II.3 Data Design](04-data-design.md)

---

## Payment Happy Path Flow

The forward payment flow progresses through 10 business states managed by Spring State Machine, with 5 SAGA steps orchestrated by Temporal:

```
PENDING → VALIDATING → VALIDATED → RESERVING → RESERVED →
AUTHORIZING → AUTHORIZED → CAPTURING → CAPTURED →
COMPLETING → COMPLETED
```

```mermaid
sequenceDiagram
    participant Client
    participant Kong as Kong Gateway
    participant Orch as SAGA Orchestrator
    participant SM as State Machine
    participant Order as Order Service
    participant Inv as Inventory Service
    participant Pay as Payment Gateway
    participant Temporal as Temporal Server

    Client->>Kong: POST /api/v1/payments
    Kong->>Orch: Forward (JWT validated)
    Orch->>Temporal: Start Workflow(orderId)
    Temporal-->>Orch: WorkflowExecution

    Note over Orch,SM: Step 1: Validate Order
    Orch->>SM: transition(START_PAYMENT) → PENDING→VALIDATING
    Orch->>Order: validateOrder(orderId)
    Order-->>Orch: OrderValidation{validationId}
    Orch->>SM: transition(ORDER_VALIDATED) → VALIDATING→VALIDATED

    Note over Orch,SM: Step 2: Reserve Inventory
    Orch->>SM: transition → VALIDATED→RESERVING
    Orch->>Inv: reserveInventory(orderId)
    Inv-->>Orch: InventoryReservation{reservationId}
    Orch->>SM: transition(INVENTORY_RESERVED) → RESERVING→RESERVED

    Note over Orch,SM: Step 3: Authorize Payment
    Orch->>SM: transition → RESERVED→AUTHORIZING
    Orch->>Pay: authorizePayment(orderId)
    Pay-->>Orch: PaymentAuth{authId}
    Orch->>SM: transition(PAYMENT_AUTHORIZED) → AUTHORIZING→AUTHORIZED

    Note over Orch,SM: Step 4: Capture Payment
    Orch->>SM: transition → AUTHORIZED→CAPTURING
    Orch->>Pay: capturePayment(authId)
    Pay-->>Orch: PaymentCapture{captureId}
    Orch->>SM: transition(PAYMENT_CAPTURED) → CAPTURING→CAPTURED

    Note over Orch,SM: Step 5: Complete Order
    Orch->>SM: transition → CAPTURED→COMPLETING
    Orch->>Order: updateOrderStatus(orderId, COMPLETED)
    Order-->>Orch: OrderUpdate
    Orch->>SM: transition(ORDER_COMPLETED) → COMPLETING→COMPLETED
    Orch->>SM: transition(SAGA_COMPLETED) → COMPLETED

    Orch-->>Client: PaymentResult{SUCCESS, captureId}
```

## Compensation Flow (LIFO Rollback)

When any step fails, the compensation stack is executed in reverse order (Last-In-First-Out):

```mermaid
sequenceDiagram
    participant Orch as SAGA Orchestrator
    participant SM as State Machine
    participant Pay as Payment Gateway
    participant Inv as Inventory Service
    participant Order as Order Service

    Note over Orch: Step 4 FAILS: capturePayment() throws exception

    Orch->>SM: transition(START_COMPENSATION)
    Note over Orch: Compensation Stack (LIFO):<br/>1. VoidAuthorization (last added)<br/>2. ReleaseInventory<br/>3. CancelOrder (first added)

    rect rgb(255, 230, 230)
        Note over Orch,Pay: Compensation 1: Void Authorization
        Orch->>Pay: voidAuthorization(authId)
        Pay-->>Orch: OK

        Note over Orch,Inv: Compensation 2: Release Inventory
        Orch->>Inv: releaseInventory(reservationId)
        Inv-->>Orch: OK

        Note over Orch,Order: Compensation 3: Cancel Order
        Orch->>Order: cancelOrder(orderId, validationId)
        Order-->>Orch: OK
    end

    Orch->>SM: transition(COMPENSATION_COMPLETED)
    Note over Orch: WorkflowState = COMPENSATED
```

## Webhook → Kafka → Workflow Pipeline

```mermaid
sequenceDiagram
    participant PSP as Payment Provider<br/>(Stripe/PayPal)
    participant Kong as Kong Gateway
    participant PGW as Payment Gateway<br/>Service
    participant DB as payment_db<br/>(Outbox Table)
    participant Deb as Debezium CDC
    participant Kafka as Kafka<br/>(webhook.payment.events)
    participant Consumer as Webhook Consumer<br/>(Orchestrator)
    participant Idemp as Idempotency<br/>Service
    participant Corr as Workflow Correlation<br/>Service
    participant WF as PaymentSagaWorkflow

    PSP->>Kong: POST /api/webhooks/stripe
    Kong->>Kong: Rate limit check
    Kong->>PGW: Forward webhook
    PGW->>PGW: Verify signature (HMAC-SHA256)
    PGW->>PGW: WebhookProcessorEngine.dispatch()
    PGW->>DB: INSERT INTO webhook_kafka_outbox<br/>(within business TX)

    Note over DB,Deb: CDC captures WAL change (<10ms)
    Deb->>DB: Read PostgreSQL WAL
    Deb->>Kafka: Publish to webhook.payment.events<br/>(12 partitions, keyed by order_id)

    Kafka->>Consumer: WebhookEventConsumer.consume()
    Consumer->>Idemp: checkAndMarkProcessed(eventId)
    alt Already Processed
        Idemp-->>Consumer: DUPLICATE → Skip
    else New Event
        Idemp-->>Consumer: NEW → Process
        Consumer->>Corr: findWorkflowId(orderId)
        Corr-->>Consumer: workflowId
        Consumer->>WF: signal(externalPaymentConfirmed)
    end
```

## Error Handling

### Error Code Categories

| Category | Prefix | Examples | Handling |
|---|---|---|---|
| **Validation** | VAL | VAL_001 (Invalid amount), VAL_002 (Missing field) | Return 400, no compensation |
| **Inventory** | INV | INV_001 (Insufficient stock), INV_002 (SKU not found) | Compensate prior steps |
| **Payment** | PAY | PAY_001 (Declined), PAY_002 (Gateway timeout) | Retry with backoff, then compensate |
| **System** | SYS | SYS_001 (Database unavailable), SYS_002 (Kafka unreachable) | Circuit breaker, retry, alert |
| **SAGA** | SAGA | SAGA_001 (Compensation failed), SAGA_002 (Timeout) | DLT + manual intervention |
| **Resource** | RES | RES_001 (Concurrent modification), RES_002 (Lock timeout) | Optimistic retry |

### Resilience Patterns

| Pattern | Technology | Configuration |
|---|---|---|
| **Circuit Breaker** | Resilience4j | 50% failure threshold, 60s half-open, 10 calls in sliding window |
| **Bulkhead** | Resilience4j | 25 max concurrent calls, 10 max wait |
| **Rate Limiter** | Resilience4j + Kong | 100/min per client (Kong), 500/min internal (Resilience4j) |
| **Retry** | Temporal RetryOptions | 3 max attempts, 1s initial → 30s max, 2.0 backoff coefficient |
| **Dead Letter Topic** | Kafka DLT | `webhook.payment.events.DLT` (3 partitions), manual review queue |
| **Timeout** | Temporal ActivityOptions | 5 min start-to-close for payment activities, 30s for data activities |

---

**Previous:** [← II.3 Data Design](04-data-design.md) | **Next:** [II.5 Integration Design →](06-integration-design.md)
