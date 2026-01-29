# State Machine & Webhook Event Processing Documentation

## Table of Contents

- [Part 1: State Machine Architecture](#part-1-state-machine-architecture)
  - [Overview](#overview)
  - [States and Events](#states-and-events)
  - [Integration with Temporal Workflow](#integration-with-temporal-workflow)
  - [Persistence Architecture](#persistence-architecture)
  - [Recovery Flow](#recovery-flow)
- [Part 2: Webhook Event Processing](#part-2-webhook-event-processing)
  - [Overview](#overview-1)
  - [Database Schema](#database-schema)
  - [Webhook Reception](#webhook-reception)
  - [Transformer Pattern](#transformer-pattern)

---

## Part 1: State Machine Architecture

### Overview

This codebase implements a hybrid SAGA pattern combining:

- Temporal for workflow orchestration (step ordering, durability, retries)
- Spring State Machine for business state modeling (domain states, transition guards)

┌─────────────────────────────────────────────────────────────────┐
│ Temporal Workflow (Orchestration) Spring State Machine │
│ ───────────────────────────────── ─────────────────────── │
│ "What step are we on?" "What business state │
│ "Are we compensating?" is this payment in?" │
│ │
│ Persisted: Temporal server Persisted: PostgreSQL │
└─────────────────────────────────────────────────────────────────┘

### States and Events

- PaymentState (payment-saga-common/.../statemachine/PaymentState.java):
  ┌──────────────┬───────┬──────────┬────────────────────────────────────┐
  │ State │ Order │ Terminal │ Description │
  ├──────────────┼───────┼──────────┼────────────────────────────────────┤
  │ PENDING │ 0 │ No │ Initial state │
  ├──────────────┼───────┼──────────┼────────────────────────────────────┤
  │ VALIDATING │ 1 │ No │ Order validation in progress │
  ├──────────────┼───────┼──────────┼────────────────────────────────────┤
  │ VALIDATED │ 2 │ No │ Order validated │
  ├──────────────┼───────┼──────────┼────────────────────────────────────┤
  │ RESERVING │ 3 │ No │ Inventory reservation in progress │
  ├──────────────┼───────┼──────────┼────────────────────────────────────┤
  │ RESERVED │ 4 │ No │ Inventory reserved │
  ├──────────────┼───────┼──────────┼────────────────────────────────────┤
  │ AUTHORIZING │ 5 │ No │ Payment authorization in progress │
  ├──────────────┼───────┼──────────┼────────────────────────────────────┤
  │ AUTHORIZED │ 6 │ No │ Payment authorized (funds on hold) │
  ├──────────────┼───────┼──────────┼────────────────────────────────────┤
  │ CAPTURING │ 7 │ No │ Payment capture in progress │
  ├──────────────┼───────┼──────────┼────────────────────────────────────┤
  │ CAPTURED │ 8 │ No │ Payment captured │
  ├──────────────┼───────┼──────────┼────────────────────────────────────┤
  │ COMPLETING │ 9 │ No │ Order completion in progress │
  ├──────────────┼───────┼──────────┼────────────────────────────────────┤
  │ COMPLETED │ 10 │ Yes │ Success terminal state │
  ├──────────────┼───────┼──────────┼────────────────────────────────────┤
  │ COMPENSATING │ -1 │ No │ Rollback in progress │
  ├──────────────┼───────┼──────────┼────────────────────────────────────┤
  │ COMPENSATED │ -2 │ Yes │ Rollback complete │
  ├──────────────┼───────┼──────────┼────────────────────────────────────┤
  │ FAILED │ -10 │ Yes │ Unrecoverable failure │
  └──────────────┴───────┴──────────┴────────────────────────────────────┘
  PaymentEvent (payment-saga-common/.../statemachine/PaymentEvent.java):

* Forward flow: START_PAYMENT, ORDER_VALIDATED, INVENTORY_RESERVED, PAYMENT_AUTHORIZED, PAYMENT_CAPTURED, ORDER_COMPLETED
* Failure events: ORDER_VALIDATION_FAILED, INVENTORY_RESERVATION_FAILED, PAYMENT_AUTHORIZATION_FAILED, etc.
* Compensation: START_COMPENSATION, COMPENSATION_COMPLETED, COMPENSATION_FAILED

State Transitions

Configured in PaymentStateMachineConfig.java:

Forward Flow:
PENDING ──START_PAYMENT──► VALIDATING ──ORDER_VALIDATED──► VALIDATED
──ORDER_VALIDATED──► RESERVING ──INVENTORY_RESERVED──► RESERVED
──INVENTORY_RESERVED──► AUTHORIZING ──PAYMENT_AUTHORIZED──► AUTHORIZED
──PAYMENT_AUTHORIZED──► CAPTURING ──PAYMENT_CAPTURED──► CAPTURED
──PAYMENT_CAPTURED──► COMPLETING ──ORDER_COMPLETED──► COMPLETED

Compensation Flow:
Any compensatable state ──START_COMPENSATION──► COMPENSATING
COMPENSATING ──COMPENSATION_COMPLETED──► COMPENSATED
COMPENSATING ──COMPENSATION_FAILED──► COMPENSATION_FAILED

Integration with Temporal Workflow

PaymentSagaWorkflowImpl.java (lines 108-117, 140-159):

// State machine registered as LOCAL activity (fast, same worker process)
StateMachineActivities stateMachineActivities = Workflow.newLocalActivityStub(
StateMachineActivities.class,
LocalActivityOptions.newBuilder()
.setStartToCloseTimeout(Duration.ofSeconds(10))
.setRetryOptions(RetryOptions.newBuilder().setMaximumAttempts(5).build())
.build()
);

// Usage in workflow
stateMachineActivities.initializeStateMachine(workflowId, orderId);
stateMachineActivities.transitionState(workflowId, PaymentEvent.START_PAYMENT, null);

// After each step succeeds
stateMachineActivities.transitionState(workflowId, PaymentEvent.ORDER_VALIDATED, result);

Persistence Architecture

Hybrid storage in PaymentStateMachineServiceImpl.java:

// In-memory cache for active state machines (fast access)
private final Map<String, StateMachine<...>> stateMachines = new ConcurrentHashMap<>();

// PostgreSQL for durability (StateMachineContextEntity)
@Entity
@Table(name = "state_machine_context")
public class StateMachineContextEntity {
String sagaId; // Workflow ID
String currentState; // e.g., "AUTHORIZED"
Map<String, Object> contextJson; // Extended state (JSONB)
Integer version; // Optimistic locking
}

Recovery Flow

When application restarts, StateMachineRestorer.java properly restores state:

// Key fix: Uses resetStateMachine() to restore to actual state (not PENDING)
DefaultStateMachineContext<PaymentState, PaymentEvent> context =
new DefaultStateMachineContext<>(
targetState, // e.g., AUTHORIZED (from DB)
null, null,
extendedState // Restored variables
);

stateMachine.getStateMachineAccessor()
.doWithAllRegions(accessor -> accessor.resetStateMachine(context));

---

## Part 2: Webhook Event Processing

### Overview

The webhook system in payment-gateway-service handles inbound payment notifications from external providers (Stripe, PayPal, Adyen, Square).

┌──────────────────────────────────────────────────────────────────────┐
│ WEBHOOK PROCESSING FLOW │
├──────────────────────────────────────────────────────────────────────┤
│ │
│ External Provider (Stripe/PayPal/etc) │
│ │ │
│ ▼ │
│ ┌─────────────────┐ │
│ │ WebhookController│ POST /api/webhooks/{channel} │
│ └────────┬────────┘ │
│ │ │
│ ▼ │
│ ┌─────────────────┐ ┌─────────────────────┐ │
│ │ Idempotency │────►│ WebhookIdempotency │ (Duplicate check) │
│ │ Check │ │ Entity │ │
│ └────────┬────────┘ └─────────────────────┘ │
│ │ │
│ ▼ │
│ ┌─────────────────┐ ┌─────────────────────┐ │
│ │ Signature │────►│ Channel-specific │ (HMAC verification)│
│ │ Verification │ │ Verifier │ │
│ └────────┬────────┘ └─────────────────────┘ │
│ │ │
│ ▼ │
│ ┌─────────────────┐ ┌─────────────────────┐ │
│ │ Transform │────►│ StripeTransformer │ (Normalize payload)│
│ │ │ │ PayPalTransformer │ │
│ └────────┬────────┘ │ etc. │ │
│ │ └─────────────────────┘ │
│ ▼ │
│ ┌─────────────────┐ │
│ │ Store Event │────► webhook_events table (JSONB payloads) │
│ └────────┬────────┘ │
│ │ │
│ ▼ │
│ ┌─────────────────┐ ┌─────────────────────┐ │
│ │ Process Event │────►│ Strategy Pattern: │ │
│ │ (Processor │ │ - PaymentSuccess │ │
│ │ Engine) │ │ - PaymentFailed │ │
│ └─────────────────┘ │ - RefundCompleted │ │
│ │ - DisputeOpened │ │
│ └─────────────────────┘ │
└──────────────────────────────────────────────────────────────────────┘

### Database Schema

- webhook_events table stores inbound webhooks:
  ┌───────────────────────────────┬──────────────┬──────────────────────────────────────────────────┐
  │ Column │ Type │ Description │
  ├───────────────────────────────┼──────────────┼──────────────────────────────────────────────────┤
  │ event_id │ VARCHAR(100) │ External event ID (unique) │
  ├───────────────────────────────┼──────────────┼──────────────────────────────────────────────────┤
  │ internal_event_id │ UUID │ Internal tracking ID │
  ├───────────────────────────────┼──────────────┼──────────────────────────────────────────────────┤
  │ channel │ VARCHAR(50) │ STRIPE, PAYPAL, ADYEN, SQUARE │
  ├───────────────────────────────┼──────────────┼──────────────────────────────────────────────────┤
  │ event_type │ VARCHAR(100) │ External type (e.g., "payment_intent.succeeded") │
  ├───────────────────────────────┼──────────────┼──────────────────────────────────────────────────┤
  │ internal_event_type │ VARCHAR(50) │ Normalized type (e.g., "PAYMENT_CAPTURED") │
  ├───────────────────────────────┼──────────────┼──────────────────────────────────────────────────┤
  │ order_id, auth_id, capture_id │ VARCHAR │ Payment references │
  ├───────────────────────────────┼──────────────┼──────────────────────────────────────────────────┤
  │ raw_payload │ JSONB │ Original webhook payload │
  ├───────────────────────────────┼──────────────┼──────────────────────────────────────────────────┤
  │ transformed_payload │ JSONB │ Normalized internal format │
  ├───────────────────────────────┼──────────────┼──────────────────────────────────────────────────┤
  │ status │ VARCHAR(20) │ RECEIVED, PROCESSING, PROCESSED, FAILED │
  ├───────────────────────────────┼──────────────┼──────────────────────────────────────────────────┤
  │ signature_verified │ BOOLEAN │ Signature validation result │
  └───────────────────────────────┴──────────────┴──────────────────────────────────────────────────┘

- webhook_outbox table for outbound webhooks (transactional outbox pattern):
  ┌───────────────┬─────────────┬─────────────────────────────────────────────────┐
  │ Column │ Type │ Description │
  ├───────────────┼─────────────┼─────────────────────────────────────────────────┤
  │ status │ VARCHAR(20) │ PENDING, SENDING, SENT, FAILED, RETRY_SCHEDULED │
  ├───────────────┼─────────────┼─────────────────────────────────────────────────┤
  │ retry_count │ INTEGER │ Current retry attempt │
  ├───────────────┼─────────────┼─────────────────────────────────────────────────┤
  │ max_retries │ INTEGER │ Default 5 │
  ├───────────────┼─────────────┼─────────────────────────────────────────────────┤
  │ next_retry_at │ TIMESTAMP │ Exponential backoff scheduling │
  └───────────────┴─────────────┴─────────────────────────────────────────────────┘

- webhook_idempotency table prevents duplicates:
  ┌─────────────────┬───────────┬─────────────────────────────────────┐
  │ Column │ Type │ Description │
  ├─────────────────┼───────────┼─────────────────────────────────────┤
  │ idempotency_key │ VARCHAR │ "{channel}:{eventId}" composite key │
  ├─────────────────┼───────────┼─────────────────────────────────────┤
  │ expires_at │ TIMESTAMP │ TTL (default 24 hours) │
  └─────────────────┴───────────┴─────────────────────────────────────┘

### Webhook Reception

- WebhookController.java (lines 85-180):

```java
@PostMapping("/{channel}")
public ResponseEntity<WebhookResponse> receiveWebhook(
@PathVariable String channel,
@RequestBody String rawBody,
@RequestHeader Map<String, String> headers) {

      // 1. Parse payload
      Map<String, Object> payload = objectMapper.readValue(rawBody, Map.class);

      // 2. Check idempotency (prevent duplicate processing)
      String eventId = transformer.extractEventId(payload);
      if (idempotencyService.isDuplicate(channel, eventId)) {
          return ResponseEntity.ok(WebhookResponse.duplicate(eventId));
      }

      // 3. Verify signature (HMAC)
      if (!verifier.verify(rawBody, headers)) {
          return ResponseEntity.badRequest()
              .body(WebhookResponse.invalidSignature());
      }

      // 4. Transform to internal format
      InternalWebhookEvent event = transformer.transformInbound(payload);

      // 5. Store event
      WebhookEventEntity entity = webhookEventService.store(event);

      // 6. Mark as processed for idempotency
      idempotencyService.markProcessed(channel, eventId);

      // 7. Process via strategy pattern
      processorEngine.process(event);

      return ResponseEntity.ok(WebhookResponse.success(eventId));

}
```

### Transformer Pattern

- StripeWebhookTransformer.java example:

```java
@Component
public class StripeWebhookTransformer extends AbstractWebhookTransformer {

      // Event type mappings
      private static final Map<String, WebhookEventType> EVENT_MAPPINGS = Map.of(
          "payment_intent.succeeded", WebhookEventType.PAYMENT_CAPTURED,
          "payment_intent.payment_failed", WebhookEventType.PAYMENT_FAILED,
          "charge.refunded", WebhookEventType.REFUND_COMPLETED,
          "charge.dispute.created", WebhookEventType.DISPUTE_OPENED
          // ... more mappings
      );

      @Override
      protected InternalWebhookEvent buildInternalEvent(Map<String, Object> payload) {
          Map<String, Object> dataObject = getNestedValue(payload, "data.object");
          return InternalWebhookEvent.builder()
              .eventType(mapEventType(getString(payload, "type")))
              .orderId(getNestedValue(dataObject, "metadata.order_id"))
              .amount(getAmountFromCents(dataObject, "amount"))
              .currency(getString(dataObject, "currency"))
              .build();
      }

}
```

### Processing Strategies

- WebhookProcessorEngine.java dispatches to strategies:

```java
@Component
public class WebhookProcessorEngine {
private final Map<WebhookEventType, List<WebhookProcessor>> processorIndex;

      public WebhookProcessingResult process(InternalWebhookEvent event) {
          List<WebhookProcessor> processors = processorIndex.get(event.getEventType());
          if (processors == null || processors.isEmpty()) {
              return WebhookProcessingResult.skipped(event.getEventId(), "No processor");
          }
          // Use highest priority processor
          return processors.get(0).process(event);
      }

}
```

### Payment Success Strategy

- PaymentSuccessStrategy.java (lines 44-80):

```java
@Component
@Transactional
public class PaymentSuccessStrategy implements WebhookProcessor {

      @Override
      public WebhookEventType[] getSupportedEventTypes() {
          return new WebhookEventType[] {
              WebhookEventType.PAYMENT_AUTHORIZED,
              WebhookEventType.PAYMENT_CAPTURED
          };
      }

      @Override
      public WebhookProcessingResult process(InternalWebhookEvent event) {
          if (event.getEventType() == WebhookEventType.PAYMENT_CAPTURED) {
              // Update capture status to COMPLETED
              captureRepository.findByOrderId(event.getOrderId())
                  .ifPresent(capture -> {
                      capture.setStatus("COMPLETED");
                      capture.setGatewayReference(event.getGatewayReference());
                      captureRepository.save(capture);
                  });
          }
          return WebhookProcessingResult.success(event.getEventId(), "Processed");
      }

}
```

### Outbound Webhook (Transactional Outbox)

- OutboundWebhookService.java implements reliable delivery:

```java
@Service
public class OutboundWebhookService {

      // Queue webhook for delivery
      public void queueWebhook(String channel, String url, Object payload) {
          WebhookOutboxEntity entity = WebhookOutboxEntity.builder()
              .channel(channel)
              .endpointUrl(url)
              .payload(objectMapper.convertValue(payload, Map.class))
              .status("PENDING")
              .maxRetries(5)
              .build();
          outboxRepository.save(entity);
      }

      // Exponential backoff on failure
      private void handleFailure(WebhookOutboxEntity entity, String error) {
          entity.setRetryCount(entity.getRetryCount() + 1);
          if (entity.getRetryCount() >= entity.getMaxRetries()) {
              entity.setStatus("EXHAUSTED");
          } else {
              // Backoff: 60s, 120s, 240s, 480s, 960s (capped at 1 hour)
              long delay = Math.min(
                  initialDelay * (1L << (entity.getRetryCount() - 1)),
                  maxDelay
              );
              entity.setNextRetryAt(Instant.now().plusSeconds(delay));
              entity.setStatus("RETRY_SCHEDULED");
          }
      }

}
```

### Outbound Webhook Poller

- OutboundWebhookPoller.java polls and sends:

```java
@Scheduled(fixedDelayString = "${webhook.outbound.poll-interval-ms:500}")
public void pollAndSend() {
// Poll pending webhooks
List<WebhookOutboxEntity> pending = outboxRepository.findPendingWebhooks(batchSize);
pending.forEach(outboundService::sendWebhook);

      // Poll retryable webhooks (next_retry_at <= now)
      List<WebhookOutboxEntity> retryable = outboxRepository.findRetryableWebhooks(batchSize);
      retryable.forEach(outboundService::sendWebhook);

}
```

### Key Design Decisions

1. Synchronous processing: Webhooks processed in request thread (no Kafka for inbound)
2. Idempotency: Database-backed with 24-hour TTL, checked before processing
3. Strategy pattern: Extensible processor selection by event type
4. Multi-channel support: Transformers/verifiers per payment provider
5. Transactional outbox: Only for outbound webhooks (reliable delivery)
6. Error mapping: Per-channel error code normalization with caching
