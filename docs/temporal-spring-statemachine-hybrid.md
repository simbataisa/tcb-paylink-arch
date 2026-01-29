# Hybrid SAGA Architecture: Temporal + Spring State Machine
## Best-of-Both-Worlds Approach for Enterprise Payment Systems

---

## Table of Contents

1. [Architecture Overview](#architecture-overview)
2. [Why Hybrid Architecture?](#why-hybrid-architecture)
3. [Integration Patterns](#integration-patterns)
4. [Implementation Guide](#implementation-guide)
5. [State Synchronization](#state-synchronization)
6. [Use Cases & Decision Matrix](#use-cases--decision-matrix)
7. [Production Implementation](#production-implementation)
8. [Migration Strategy](#migration-strategy)

---

## Architecture Overview

### The Hybrid Model

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                          HYBRID SAGA ARCHITECTURE                                │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                   │
│  ┌─────────────────────────────────────────────────────────────────────────┐   │
│  │                    TEMPORAL WORKFLOW ORCHESTRATOR                        │   │
│  │                     (Workflow State & Durability)                        │   │
│  ├─────────────────────────────────────────────────────────────────────────┤   │
│  │                                                                           │   │
│  │  [START] → ValidateOrder → ReserveInventory → AuthorizePayment →        │   │
│  │            CapturePayment → UpdateOrder → [COMPLETED]                    │   │
│  │                                                                           │   │
│  │  Compensation Flow: ← CancelOrder ← ReleaseInventory ← VoidPayment      │   │
│  │                                                                           │   │
│  └─────────────────────────────────────────────────────────────────────────┘   │
│                                    ↕                                             │
│                          (Activity Execution)                                    │
│                                    ↕                                             │
│  ┌─────────────────────────────────────────────────────────────────────────┐   │
│  │               SPRING STATE MACHINE (Domain State Model)                  │   │
│  │                    (Business State & Transitions)                        │   │
│  ├─────────────────────────────────────────────────────────────────────────┤   │
│  │                                                                           │   │
│  │   PENDING → VALIDATING → RESERVED → AUTHORIZED → CAPTURED → COMPLETED   │   │
│  │                            ↓             ↓                                │   │
│  │                         RELEASING     VOIDING                             │   │
│  │                            ↓             ↓                                │   │
│  │                          CANCELLED ← CANCELLING                           │   │
│  │                                                                           │   │
│  └─────────────────────────────────────────────────────────────────────────┘   │
│                                    ↕                                             │
│                          (Event Publishing)                                      │
│                                    ↕                                             │
│  ┌─────────────────────────────────────────────────────────────────────────┐   │
│  │                      EVENT-DRIVEN INTEGRATION LAYER                      │   │
│  │                         (Kafka Event Streaming)                          │   │
│  ├─────────────────────────────────────────────────────────────────────────┤   │
│  │                                                                           │   │
│  │  Topics: order.validated | inventory.reserved | payment.authorized      │   │
│  │          payment.captured | order.completed | compensation.triggered     │   │
│  │                                                                           │   │
│  └─────────────────────────────────────────────────────────────────────────┘   │
│                                                                                   │
│  Key Benefits:                                                                   │
│  ✓ Temporal: Workflow durability, built-in retries, timeouts, visibility       │
│  ✓ State Machine: Domain modeling, business rules, fine-grained transitions    │
│  ✓ Events: Loose coupling, observability, integration with other systems        │
│                                                                                   │
└─────────────────────────────────────────────────────────────────────────────────┘
```

### Responsibility Separation

| Layer | Technology | Responsibility | Why? |
|-------|------------|----------------|------|
| **Orchestration** | Temporal Workflow | SAGA step ordering, compensation logic, durability, retries, timeouts | Built-in workflow engine, automatic persistence, excellent observability |
| **Domain State** | Spring State Machine | Business state modeling, fine-grained transitions, business rules, state guards | Type-safe state modeling, integration with Spring ecosystem |
| **Integration** | Kafka Events | Async communication, event sourcing, external system integration | Event-driven architecture, loose coupling |
| **Data** | PostgreSQL | State persistence, event store, audit trail | ACID guarantees, queryable history |
| **Cache** | Redis | Idempotency, distributed locks, session state | Fast lookups, distributed coordination |

---

## Why Hybrid Architecture?

### The Problem with Single-Layer Approaches

#### Temporal-Only Approach ⚠️

**Limitations:**
- Workflow state is coarse-grained (activity level)
- Business domain state modeling is implicit
- No rich state machine DSL for complex business rules
- Difficult to query current business state independently
- State guards and transitions are code-based, not declarative

```java
// Temporal-only: Implicit state through workflow execution
@WorkflowMethod
public PaymentResult processPayment(PaymentRequest request) {
    // State is implicit - we're "in payment authorization" when this line executes
    PaymentAuth auth = activities.authorizePayment(request);
    
    // Hard to answer: "What business state is this order in right now?"
    // Hard to query: "Show me all orders in AUTHORIZED state"
    // Hard to add: "Don't allow capture if authorization expired"
}
```

#### Spring State Machine-Only Approach ⚠️

**Limitations:**
- Must implement workflow durability manually
- Need custom retry/timeout logic
- No built-in workflow visualization
- Complex compensation orchestration
- Must build workflow engine features from scratch

```java
// State machine-only: Great for state modeling, but lacking orchestration
@Configuration
public class PaymentStateMachine {
    // Rich state modeling ✓
    // But how do we persist workflow execution? 
    // How do we handle retries across service restarts?
    // How do we visualize the running workflow?
    // All manual implementation required ✗
}
```

### The Hybrid Solution ✅

**Combines the best of both:**

```java
// Temporal handles: orchestration, durability, retries, compensation
@WorkflowImpl
public class PaymentSagaWorkflow {
    
    private final PaymentStateMachine stateMachine; // Injected
    
    @WorkflowMethod
    public PaymentResult processPayment(PaymentRequest request) {
        
        // Temporal provides: durability, retries, timeout
        try {
            // Step 1: Validate (Temporal activity)
            OrderValidation validation = activities.validateOrder(request);
            
            // Update business state (State Machine)
            stateMachine.transition(PaymentEvent.ORDER_VALIDATED);
            
            // Step 2: Reserve inventory
            InventoryReservation reservation = activities.reserveInventory(request);
            stateMachine.transition(PaymentEvent.INVENTORY_RESERVED);
            
            // Step 3: Authorize payment
            PaymentAuth auth = activities.authorizePayment(request);
            stateMachine.transition(PaymentEvent.PAYMENT_AUTHORIZED);
            
            // Step 4: Capture payment
            PaymentCapture capture = activities.capturePayment(auth.getAuthId());
            stateMachine.transition(PaymentEvent.PAYMENT_CAPTURED);
            
            // Now we can query: "What state is this payment in?" → CAPTURED
            // State machine provides: rich domain model, queryable state
            
            return PaymentResult.success(capture);
            
        } catch (Exception e) {
            // Temporal compensation
            compensate();
            throw e;
        }
    }
}
```

**Benefits:**
- ✅ Temporal handles workflow lifecycle (durability, retries, timeouts, compensation)
- ✅ State Machine provides rich business state model (queryable, rules, guards)
- ✅ Clear separation of concerns
- ✅ Best tooling from both ecosystems
- ✅ Gradual migration path

---

## Integration Patterns

### Pattern 1: Temporal Orchestrates, State Machine Models

**Use Case:** Default pattern for most SAGA workflows

```
Temporal Workflow (Orchestration)
    ↓ executes activity
    ↓
Activity Implementation
    ↓ performs business operation
    ↓ triggers state transition
    ↓
Spring State Machine (Domain State)
    ↓ validates transition
    ↓ updates persisted state
    ↓ publishes domain event
    ↓
Kafka (Event Stream)
```

**Implementation:**

```java
package com.payment.saga.workflow;

import io.temporal.workflow.*;
import io.temporal.activity.*;
import com.payment.saga.statemachine.PaymentStateMachineService;

/**
 * Temporal Workflow - Orchestration Layer
 * Handles: Durability, Retries, Timeouts, Compensation
 */
@WorkflowInterface
public interface PaymentSagaWorkflow {
    
    @WorkflowMethod
    PaymentResult processPayment(PaymentRequest request);
    
    @SignalMethod
    void cancel();
    
    @QueryMethod
    PaymentState getCurrentState();
}

@Component
public class PaymentSagaWorkflowImpl implements PaymentSagaWorkflow {
    
    private static final Logger logger = LoggerFactory.getLogger(PaymentSagaWorkflowImpl.class);
    
    // Temporal Activities (external operations)
    private final PaymentActivities activities = Workflow.newActivityStub(
        PaymentActivities.class,
        ActivityOptions.newBuilder()
            .setStartToCloseTimeout(Duration.ofMinutes(5))
            .setRetryOptions(RetryOptions.newBuilder()
                .setMaximumAttempts(3)
                .setBackoffCoefficient(2.0)
                .build())
            .build()
    );
    
    // Spring State Machine proxy (workflow-local)
    private final StateMachineProxy stateMachineProxy = new StateMachineProxy();
    
    private String sagaId;
    private PaymentState currentState = PaymentState.PENDING;
    private boolean cancelRequested = false;
    
    @Override
    public PaymentResult processPayment(PaymentRequest request) {
        
        this.sagaId = Workflow.getInfo().getWorkflowId();
        
        logger.info("Starting payment SAGA: {}", sagaId);
        
        try {
            // Initialize state machine
            stateMachineProxy.initialize(sagaId, request);
            
            // Step 1: Validate Order
            executeStep("ValidateOrder", () -> {
                OrderValidation validation = activities.validateOrder(request);
                stateMachineProxy.sendEvent(PaymentEvent.ORDER_VALIDATED, validation);
                return validation;
            });
            
            checkCancellation();
            
            // Step 2: Reserve Inventory
            executeStep("ReserveInventory", () -> {
                InventoryReservation reservation = activities.reserveInventory(
                    request.getOrderId(), 
                    request.getItems()
                );
                stateMachineProxy.sendEvent(PaymentEvent.INVENTORY_RESERVED, reservation);
                return reservation;
            });
            
            checkCancellation();
            
            // Step 3: Authorize Payment
            executeStep("AuthorizePayment", () -> {
                PaymentAuth auth = activities.authorizePayment(
                    request.getPaymentDetails()
                );
                stateMachineProxy.sendEvent(PaymentEvent.PAYMENT_AUTHORIZED, auth);
                return auth;
            });
            
            checkCancellation();
            
            // Step 4: Capture Payment
            executeStep("CapturePayment", () -> {
                PaymentCapture capture = activities.capturePayment(
                    stateMachineProxy.getAuthorizationId()
                );
                stateMachineProxy.sendEvent(PaymentEvent.PAYMENT_CAPTURED, capture);
                return capture;
            });
            
            // Step 5: Update Order
            executeStep("UpdateOrder", () -> {
                OrderUpdate update = activities.updateOrderStatus(
                    request.getOrderId(),
                    OrderStatus.COMPLETED
                );
                stateMachineProxy.sendEvent(PaymentEvent.ORDER_COMPLETED, update);
                return update;
            });
            
            logger.info("Payment SAGA completed successfully: {}", sagaId);
            
            return PaymentResult.builder()
                .sagaId(sagaId)
                .orderId(request.getOrderId())
                .status(PaymentResultStatus.SUCCESS)
                .finalState(stateMachineProxy.getCurrentState())
                .build();
                
        } catch (Exception e) {
            logger.error("Payment SAGA failed: {}", sagaId, e);
            
            // Trigger compensation through state machine
            stateMachineProxy.sendEvent(PaymentEvent.START_COMPENSATION, e);
            
            // Execute Temporal compensation
            compensate();
            
            return PaymentResult.builder()
                .sagaId(sagaId)
                .orderId(request.getOrderId())
                .status(PaymentResultStatus.FAILED)
                .finalState(stateMachineProxy.getCurrentState())
                .errorMessage(e.getMessage())
                .build();
        }
    }
    
    private <T> T executeStep(String stepName, Supplier<T> step) {
        logger.info("Executing step: {}", stepName);
        
        long startTime = Workflow.currentTimeMillis();
        
        try {
            T result = step.get();
            
            long duration = Workflow.currentTimeMillis() - startTime;
            logger.info("Step completed: {} in {}ms", stepName, duration);
            
            return result;
            
        } catch (Exception e) {
            logger.error("Step failed: {}", stepName, e);
            throw new WorkflowExecutionException("Step failed: " + stepName, e);
        }
    }
    
    private void compensate() {
        logger.info("Starting compensation for SAGA: {}", sagaId);
        
        PaymentState state = stateMachineProxy.getCurrentState();
        
        // Compensation order (reverse of execution)
        try {
            if (state.isAfter(PaymentState.PAYMENT_CAPTURED)) {
                activities.refundPayment(stateMachineProxy.getCaptureId());
                stateMachineProxy.sendEvent(PaymentEvent.PAYMENT_REFUNDED, null);
            } else if (state.isAfter(PaymentState.PAYMENT_AUTHORIZED)) {
                activities.voidAuthorization(stateMachineProxy.getAuthorizationId());
                stateMachineProxy.sendEvent(PaymentEvent.PAYMENT_VOIDED, null);
            }
            
            if (state.isAfter(PaymentState.INVENTORY_RESERVED)) {
                activities.releaseInventory(stateMachineProxy.getReservationId());
                stateMachineProxy.sendEvent(PaymentEvent.INVENTORY_RELEASED, null);
            }
            
            if (state.isAfter(PaymentState.ORDER_VALIDATED)) {
                activities.cancelOrder(stateMachineProxy.getOrderId());
                stateMachineProxy.sendEvent(PaymentEvent.ORDER_CANCELLED, null);
            }
            
            stateMachineProxy.sendEvent(PaymentEvent.COMPENSATION_COMPLETED, null);
            
            logger.info("Compensation completed for SAGA: {}", sagaId);
            
        } catch (Exception e) {
            logger.error("Compensation failed for SAGA: {}", sagaId, e);
            stateMachineProxy.sendEvent(PaymentEvent.COMPENSATION_FAILED, e);
            throw e;
        }
    }
    
    @Override
    public void cancel() {
        logger.info("Cancel requested for SAGA: {}", sagaId);
        this.cancelRequested = true;
        stateMachineProxy.sendEvent(PaymentEvent.CANCEL_REQUESTED, null);
    }
    
    private void checkCancellation() {
        if (cancelRequested) {
            throw new WorkflowCancelledException("Workflow cancelled by user");
        }
    }
    
    @Override
    public PaymentState getCurrentState() {
        return stateMachineProxy.getCurrentState();
    }
    
    /**
     * Workflow-local proxy to Spring State Machine
     * Serializable state for Temporal persistence
     */
    private static class StateMachineProxy {
        
        private String sagaId;
        private PaymentState currentState = PaymentState.PENDING;
        private String orderId;
        private String reservationId;
        private String authorizationId;
        private String captureId;
        
        public void initialize(String sagaId, PaymentRequest request) {
            this.sagaId = sagaId;
            this.orderId = request.getOrderId();
            
            // Notify Spring State Machine via activity
            PaymentActivities activities = Workflow.newLocalActivityStub(
                PaymentActivities.class,
                LocalActivityOptions.newBuilder()
                    .setStartToCloseTimeout(Duration.ofSeconds(5))
                    .build()
            );
            
            activities.initializeStateMachine(sagaId, request);
        }
        
        public void sendEvent(PaymentEvent event, Object payload) {
            
            // Call Spring State Machine through local activity
            PaymentActivities activities = Workflow.newLocalActivityStub(
                PaymentActivities.class,
                LocalActivityOptions.newBuilder()
                    .setStartToCloseTimeout(Duration.ofSeconds(5))
                    .build()
            );
            
            StateTransitionResult result = activities.transitionState(sagaId, event, payload);
            
            // Update local state
            this.currentState = result.getNewState();
            
            // Update reference IDs based on event
            switch (event) {
                case INVENTORY_RESERVED:
                    this.reservationId = ((InventoryReservation) payload).getReservationId();
                    break;
                case PAYMENT_AUTHORIZED:
                    this.authorizationId = ((PaymentAuth) payload).getAuthId();
                    break;
                case PAYMENT_CAPTURED:
                    this.captureId = ((PaymentCapture) payload).getCaptureId();
                    break;
            }
        }
        
        public PaymentState getCurrentState() {
            return currentState;
        }
        
        public String getOrderId() {
            return orderId;
        }
        
        public String getReservationId() {
            return reservationId;
        }
        
        public String getAuthorizationId() {
            return authorizationId;
        }
        
        public String getCaptureId() {
            return captureId;
        }
    }
}
```

**Spring State Machine Activity Implementation:**

```java
package com.payment.saga.activity;

import io.temporal.activity.ActivityInterface;
import io.temporal.activity.ActivityMethod;

/**
 * Bridge Activities - Connect Temporal to Spring State Machine
 */
@ActivityInterface
public interface PaymentActivities {
    
    // Business activities (external operations)
    @ActivityMethod
    OrderValidation validateOrder(PaymentRequest request);
    
    @ActivityMethod
    InventoryReservation reserveInventory(String orderId, List<OrderItem> items);
    
    @ActivityMethod
    PaymentAuth authorizePayment(PaymentDetails details);
    
    @ActivityMethod
    PaymentCapture capturePayment(String authId);
    
    @ActivityMethod
    OrderUpdate updateOrderStatus(String orderId, OrderStatus status);
    
    // Compensation activities
    @ActivityMethod
    void voidAuthorization(String authId);
    
    @ActivityMethod
    void refundPayment(String captureId);
    
    @ActivityMethod
    void releaseInventory(String reservationId);
    
    @ActivityMethod
    void cancelOrder(String orderId);
    
    // State machine integration activities
    @ActivityMethod
    void initializeStateMachine(String sagaId, PaymentRequest request);
    
    @ActivityMethod
    StateTransitionResult transitionState(String sagaId, PaymentEvent event, Object payload);
}

@Component
public class PaymentActivitiesImpl implements PaymentActivities {
    
    private final OrderService orderService;
    private final InventoryService inventoryService;
    private final PaymentGateway paymentGateway;
    private final PaymentStateMachineService stateMachineService;
    
    @Override
    public OrderValidation validateOrder(PaymentRequest request) {
        
        // Perform business validation
        OrderValidation validation = orderService.validate(request);
        
        // State machine handles state transition internally
        // No explicit call needed here - will be called from workflow
        
        return validation;
    }
    
    @Override
    public InventoryReservation reserveInventory(String orderId, List<OrderItem> items) {
        
        // Reserve inventory
        InventoryReservation reservation = inventoryService.reserve(orderId, items);
        
        return reservation;
    }
    
    @Override
    public PaymentAuth authorizePayment(PaymentDetails details) {
        
        // Call payment gateway
        PaymentAuth auth = paymentGateway.authorize(details);
        
        return auth;
    }
    
    @Override
    public PaymentCapture capturePayment(String authId) {
        
        // Capture authorized payment
        PaymentCapture capture = paymentGateway.capture(authId);
        
        return capture;
    }
    
    // State machine integration
    
    @Override
    public void initializeStateMachine(String sagaId, PaymentRequest request) {
        
        // Create and start state machine for this SAGA
        stateMachineService.createStateMachine(sagaId, request);
        stateMachineService.startStateMachine(sagaId);
    }
    
    @Override
    public StateTransitionResult transitionState(String sagaId, PaymentEvent event, Object payload) {
        
        // Transition state machine
        boolean success = stateMachineService.sendEvent(sagaId, event, payload);
        
        PaymentState newState = stateMachineService.getCurrentState(sagaId);
        
        return new StateTransitionResult(success, newState);
    }
}
```

**Spring State Machine Service:**

```java
package com.payment.saga.statemachine.service;

import org.springframework.statemachine.*;
import org.springframework.statemachine.config.StateMachineFactory;
import org.springframework.statemachine.persist.StateMachinePersister;
import org.springframework.stereotype.Service;

@Service
public class PaymentStateMachineService {
    
    private final StateMachineFactory<PaymentState, PaymentEvent> stateMachineFactory;
    private final StateMachinePersister<PaymentState, PaymentEvent, String> persister;
    private final DomainEventPublisher eventPublisher;
    private final SagaAuditService auditService;
    
    /**
     * Create new state machine for SAGA
     */
    @Transactional
    public void createStateMachine(String sagaId, PaymentRequest request) {
        
        StateMachine<PaymentState, PaymentEvent> stateMachine = 
            stateMachineFactory.getStateMachine(sagaId);
        
        // Set initial context
        stateMachine.getExtendedState().getVariables().put("sagaId", sagaId);
        stateMachine.getExtendedState().getVariables().put("orderId", request.getOrderId());
        stateMachine.getExtendedState().getVariables().put("request", request);
        
        // Persist initial state
        persister.persist(stateMachine, sagaId);
        
        logger.info("State machine created for SAGA: {}", sagaId);
    }
    
    /**
     * Start state machine
     */
    public void startStateMachine(String sagaId) {
        
        StateMachine<PaymentState, PaymentEvent> stateMachine = 
            stateMachineFactory.getStateMachine(sagaId);
        
        stateMachine.start();
        
        logger.info("State machine started for SAGA: {}", sagaId);
    }
    
    /**
     * Send event to state machine
     * Returns true if transition was accepted
     */
    @Transactional
    public boolean sendEvent(String sagaId, PaymentEvent event, Object payload) {
        
        StateMachine<PaymentState, PaymentEvent> stateMachine = 
            stateMachineFactory.getStateMachine(sagaId);
        
        // Restore state
        persister.restore(stateMachine, sagaId);
        
        PaymentState previousState = stateMachine.getState().getId();
        
        // Build message
        Message<PaymentEvent> message = MessageBuilder
            .withPayload(event)
            .setHeader("payload", payload)
            .setHeader("sagaId", sagaId)
            .build();
        
        // Send event
        boolean accepted = stateMachine.sendEvent(message);
        
        if (accepted) {
            PaymentState newState = stateMachine.getState().getId();
            
            // Persist new state
            persister.persist(stateMachine, sagaId);
            
            // Publish domain event
            publishStateChangedEvent(sagaId, previousState, newState, event, payload);
            
            // Record audit
            auditService.recordStateTransition(sagaId, previousState, newState, event, true, 0L);
            
            logger.info("State transition: SAGA={} {}→{} via {}", 
                       sagaId, previousState, newState, event);
        } else {
            logger.warn("Event rejected: SAGA={} event={} currentState={}", 
                       sagaId, event, stateMachine.getState().getId());
        }
        
        return accepted;
    }
    
    /**
     * Get current state
     */
    public PaymentState getCurrentState(String sagaId) {
        
        StateMachine<PaymentState, PaymentEvent> stateMachine = 
            stateMachineFactory.getStateMachine(sagaId);
        
        persister.restore(stateMachine, sagaId);
        
        return stateMachine.getState().getId();
    }
    
    /**
     * Publish domain event when state changes
     */
    private void publishStateChangedEvent(String sagaId, PaymentState from, 
                                         PaymentState to, PaymentEvent trigger, 
                                         Object payload) {
        
        PaymentStateChangedEvent event = PaymentStateChangedEvent.builder()
            .sagaId(sagaId)
            .fromState(from)
            .toState(to)
            .trigger(trigger)
            .payload(payload)
            .timestamp(Instant.now())
            .build();
        
        eventPublisher.publish(event);
    }
}
```

---

### Pattern 2: State Machine Drives Temporal Activities

**Use Case:** When business rules in state machine should trigger workflow steps

```
Spring State Machine (Business Rules)
    ↓ state transition occurs
    ↓ guard evaluates to true
    ↓
State Machine Action
    ↓ triggers workflow signal
    ↓
Temporal Workflow (receives signal)
    ↓ executes next activity
    ↓
Activity Implementation
```

**Implementation:**

```java
/**
 * State machine action that signals Temporal workflow
 */
@Component
public class PaymentStateActions {
    
    private final WorkflowClient temporalClient;
    
    /**
     * When state transitions to AUTHORIZED, signal workflow to proceed with capture
     */
    public Action<PaymentState, PaymentEvent> authorizedAction() {
        
        return context -> {
            String sagaId = context.getExtendedState()
                .get("sagaId", String.class);
            
            // Get Temporal workflow handle
            PaymentSagaWorkflow workflow = temporalClient.newWorkflowStub(
                PaymentSagaWorkflow.class, sagaId);
            
            // Signal workflow to proceed
            workflow.proceedToCapture();
            
            logger.info("Signaled workflow to capture payment: {}", sagaId);
        };
    }
    
    /**
     * When entering COMPENSATING state, signal workflow
     */
    public Action<PaymentState, PaymentEvent> compensatingEntryAction() {
        
        return context -> {
            String sagaId = context.getExtendedState()
                .get("sagaId", String.class);
            
            PaymentSagaWorkflow workflow = temporalClient.newWorkflowStub(
                PaymentSagaWorkflow.class, sagaId);
            
            // Signal workflow to start compensation
            workflow.startCompensation();
        };
    }
}

/**
 * Temporal workflow with signals from state machine
 */
@Component
public class PaymentSagaWorkflowImpl implements PaymentSagaWorkflow {
    
    private boolean proceedToCapture = false;
    private boolean startCompensation = false;
    
    @Override
    public PaymentResult processPayment(PaymentRequest request) {
        
        // Execute authorization
        PaymentAuth auth = activities.authorizePayment(request);
        
        // Notify state machine
        stateMachineProxy.sendEvent(PaymentEvent.PAYMENT_AUTHORIZED, auth);
        
        // Wait for state machine to allow capture
        // State machine may check: authorization not expired, fraud check passed, etc.
        Workflow.await(() -> proceedToCapture || startCompensation);
        
        if (startCompensation) {
            compensate();
            return PaymentResult.compensated();
        }
        
        // Proceed with capture
        PaymentCapture capture = activities.capturePayment(auth.getAuthId());
        stateMachineProxy.sendEvent(PaymentEvent.PAYMENT_CAPTURED, capture);
        
        return PaymentResult.success();
    }
    
    @SignalMethod
    @Override
    public void proceedToCapture() {
        this.proceedToCapture = true;
    }
    
    @SignalMethod
    @Override
    public void startCompensation() {
        this.startCompensation = true;
    }
}
```

---

### Pattern 3: Event-Driven Coordination

**Use Case:** Loose coupling between Temporal and State Machine through events

```
Temporal Workflow
    ↓ executes activity
    ↓ publishes event to Kafka
    ↓
Kafka Topic (payment.authorized)
    ↓ consumed by
    ↓
State Machine Event Consumer
    ↓ sends event to state machine
    ↓
Spring State Machine
    ↓ transitions state
    ↓ publishes domain event
    ↓
Kafka Topic (payment.state.changed)
    ↓ consumed by
    ↓
Temporal Workflow (via activity polling)
```

**Implementation:**

```java
/**
 * Temporal workflow publishes events
 */
@Component
public class PaymentSagaWorkflowImpl implements PaymentSagaWorkflow {
    
    @Override
    public PaymentResult processPayment(PaymentRequest request) {
        
        // Execute authorization
        PaymentAuth auth = activities.authorizePayment(request);
        
        // Publish event (through activity)
        activities.publishEvent(new PaymentAuthorizedEvent(
            sagaId, 
            auth.getAuthId(), 
            auth.getAmount()
        ));
        
        // Wait for state machine confirmation via event
        Workflow.await(Duration.ofMinutes(5), () -> 
            activities.checkStateTransition(sagaId, PaymentState.PAYMENT_AUTHORIZED));
        
        // Continue...
    }
}

/**
 * State machine consumes events from Kafka
 */
@Component
public class PaymentEventConsumer {
    
    @KafkaListener(topics = "payment.authorized")
    public void handlePaymentAuthorized(PaymentAuthorizedEvent event) {
        
        // Transition state machine
        stateMachineService.sendEvent(
            event.getSagaId(),
            PaymentEvent.PAYMENT_AUTHORIZED,
            event
        );
    }
}
```

---

## State Synchronization

### Synchronization Strategy

```java
package com.payment.saga.sync;

/**
 * Keeps Temporal workflow state and Spring State Machine in sync
 */
@Service
public class StateSynchronizationService {
    
    private final PaymentStateMachineService stateMachineService;
    private final WorkflowClient temporalClient;
    private final StateSyncRepository syncRepository;
    
    /**
     * Synchronize states after each transition
     */
    @Transactional
    public void syncStates(String sagaId, PaymentEvent event, Object payload) {
        
        try {
            // 1. Update state machine
            boolean smSuccess = stateMachineService.sendEvent(sagaId, event, payload);
            PaymentState smState = stateMachineService.getCurrentState(sagaId);
            
            // 2. Query Temporal workflow state
            PaymentSagaWorkflow workflow = temporalClient.newWorkflowStub(
                PaymentSagaWorkflow.class, sagaId);
            PaymentState workflowState = workflow.getCurrentState();
            
            // 3. Record sync state
            StateSyncRecord record = new StateSyncRecord();
            record.setSagaId(sagaId);
            record.setStateMachineState(smState);
            record.setWorkflowState(workflowState);
            record.setSynced(smState == workflowState);
            record.setTimestamp(Instant.now());
            
            syncRepository.save(record);
            
            // 4. Alert on mismatch
            if (smState != workflowState) {
                logger.error("State mismatch detected! SAGA={} SM={} Workflow={}", 
                           sagaId, smState, workflowState);
                alertOps(sagaId, smState, workflowState);
            }
            
        } catch (Exception e) {
            logger.error("State sync failed for SAGA: {}", sagaId, e);
            throw new StateSyncException("Failed to sync states", e);
        }
    }
    
    /**
     * Scheduled reconciliation of states
     */
    @Scheduled(fixedDelay = 60000) // Every minute
    public void reconcileStates() {
        
        List<StateSyncRecord> mismatched = 
            syncRepository.findBySyncedFalseAndTimestampAfter(
                Instant.now().minusMinutes(10));
        
        for (StateSyncRecord record : mismatched) {
            try {
                reconcileSaga(record.getSagaId());
            } catch (Exception e) {
                logger.error("Reconciliation failed for SAGA: {}", 
                           record.getSagaId(), e);
            }
        }
    }
    
    /**
     * Reconcile individual SAGA
     */
    private void reconcileSaga(String sagaId) {
        
        PaymentState smState = stateMachineService.getCurrentState(sagaId);
        
        PaymentSagaWorkflow workflow = temporalClient.newWorkflowStub(
            PaymentSagaWorkflow.class, sagaId);
        PaymentState workflowState = workflow.getCurrentState();
        
        // Temporal is source of truth for workflow state
        if (smState != workflowState) {
            logger.info("Reconciling state: SAGA={} SM={} -> Workflow={}", 
                       sagaId, smState, workflowState);
            
            // Replay events to bring state machine to correct state
            replayToState(sagaId, workflowState);
        }
    }
    
    private void replayToState(String sagaId, PaymentState targetState) {
        // Implementation: replay events from event store
        // to bring state machine to target state
    }
}
```

---

## Use Cases & Decision Matrix

### When to Use Each Component

| Requirement | Temporal | State Machine | Both | Why? |
|-------------|----------|---------------|------|------|
| **Step ordering & coordination** | ✅ | ❌ | - | Temporal's workflow engine excels at this |
| **Automatic retries** | ✅ | ❌ | - | Built into Temporal |
| **Timeout handling** | ✅ | ❌ | - | Temporal's timer system |
| **Workflow visibility** | ✅ | ❌ | - | Temporal UI shows execution |
| **Compensation logic** | ✅ | ❌ | - | Temporal's saga pattern support |
| **Business state modeling** | ❌ | ✅ | - | State machine's DSL is better |
| **Fine-grained transitions** | ❌ | ✅ | - | State machine's transition system |
| **State guards (business rules)** | ❌ | ✅ | - | State machine's guard conditions |
| **Queryable business state** | ❌ | ✅ | - | State machine persists domain state |
| **Domain events** | ❌ | ✅ | - | State machine publishes on transitions |
| **Complex state hierarchies** | ❌ | ✅ | - | State machine supports nested states |
| **State-based notifications** | ❌ | ✅ | - | State machine actions |
| **Audit trail** | ✅ | ✅ | ✅ | Both provide, combine for complete picture |
| **Long-running workflows** | ✅ | ❌ | - | Temporal handles days/months duration |
| **Distributed execution** | ✅ | ⚠️ | - | Temporal multi-DC, SM needs Redis |
| **Event sourcing** | ⚠️ | ✅ | ✅ | SM better for domain events, Temporal for workflow history |

### Decision Tree

```
Start: Do I need a SAGA?
  │
  ├─ YES → Do I have complex business state transitions?
  │         │
  │         ├─ YES → Do I need workflow durability & retries?
  │         │         │
  │         │         ├─ YES → Use TEMPORAL + STATE MACHINE ✅
  │         │         │
  │         │         └─ NO → Use STATE MACHINE only
  │         │
  │         └─ NO → Do I need workflow durability & retries?
  │                   │
  │                   ├─ YES → Use TEMPORAL only
  │                   │
  │                   └─ NO → Use simple service orchestration
  │
  └─ NO → Use simple transactional processing
```

---

## Production Implementation

### Complete Example: Payment SAGA

```java
package com.payment.saga;

/**
 * COMPLETE IMPLEMENTATION: Hybrid Temporal + Spring State Machine
 */

// ============================================================================
// TEMPORAL WORKFLOW
// ============================================================================

@WorkflowInterface
public interface PaymentSagaWorkflow {
    @WorkflowMethod
    PaymentResult processPayment(PaymentRequest request);
    
    @SignalMethod
    void cancel();
    
    @QueryMethod
    PaymentWorkflowState getWorkflowState();
}

@Component
public class PaymentSagaWorkflowImpl implements PaymentSagaWorkflow {
    
    private final PaymentActivities activities;
    private final StateMachineProxy stateMachine;
    
    private String sagaId;
    private PaymentWorkflowState workflowState = PaymentWorkflowState.INITIALIZED;
    private boolean cancelled = false;
    
    public PaymentSagaWorkflowImpl() {
        this.activities = Workflow.newActivityStub(
            PaymentActivities.class,
            ActivityOptions.newBuilder()
                .setStartToCloseTimeout(Duration.ofMinutes(5))
                .setRetryOptions(RetryOptions.newBuilder()
                    .setMaximumAttempts(3)
                    .setInitialInterval(Duration.ofSeconds(1))
                    .setMaximumInterval(Duration.ofSeconds(30))
                    .setBackoffCoefficient(2.0)
                    .build())
                .build()
        );
        this.stateMachine = new StateMachineProxy();
    }
    
    @Override
    public PaymentResult processPayment(PaymentRequest request) {
        
        this.sagaId = Workflow.getInfo().getWorkflowId();
        
        logger.info("[WORKFLOW] Starting payment SAGA: {}", sagaId);
        
        try {
            // Initialize
            workflowState = PaymentWorkflowState.RUNNING;
            stateMachine.initialize(sagaId, request);
            
            // Step 1: Validate Order
            workflowState = PaymentWorkflowState.VALIDATING_ORDER;
            OrderValidation validation = executeWithStateTransition(
                "ValidateOrder",
                () -> activities.validateOrder(request),
                PaymentEvent.ORDER_VALIDATED
            );
            
            checkCancellation();
            
            // Step 2: Reserve Inventory
            workflowState = PaymentWorkflowState.RESERVING_INVENTORY;
            InventoryReservation reservation = executeWithStateTransition(
                "ReserveInventory",
                () -> activities.reserveInventory(request.getOrderId(), request.getItems()),
                PaymentEvent.INVENTORY_RESERVED
            );
            
            checkCancellation();
            
            // Step 3: Authorize Payment
            workflowState = PaymentWorkflowState.AUTHORIZING_PAYMENT;
            PaymentAuth auth = executeWithStateTransition(
                "AuthorizePayment",
                () -> activities.authorizePayment(request.getPaymentDetails()),
                PaymentEvent.PAYMENT_AUTHORIZED
            );
            
            checkCancellation();
            
            // Step 4: Capture Payment
            workflowState = PaymentWorkflowState.CAPTURING_PAYMENT;
            PaymentCapture capture = executeWithStateTransition(
                "CapturePayment",
                () -> activities.capturePayment(auth.getAuthId()),
                PaymentEvent.PAYMENT_CAPTURED
            );
            
            // Step 5: Complete Order
            workflowState = PaymentWorkflowState.COMPLETING_ORDER;
            OrderUpdate update = executeWithStateTransition(
                "CompleteOrder",
                () -> activities.updateOrderStatus(request.getOrderId(), OrderStatus.COMPLETED),
                PaymentEvent.ORDER_COMPLETED
            );
            
            // Success!
            workflowState = PaymentWorkflowState.COMPLETED;
            
            logger.info("[WORKFLOW] Payment SAGA completed: {}", sagaId);
            
            return PaymentResult.builder()
                .sagaId(sagaId)
                .orderId(request.getOrderId())
                .status(PaymentResultStatus.SUCCESS)
                .workflowState(workflowState)
                .domainState(stateMachine.getCurrentState())
                .captureId(capture.getCaptureId())
                .build();
                
        } catch (Exception e) {
            logger.error("[WORKFLOW] Payment SAGA failed: {}", sagaId, e);
            
            workflowState = PaymentWorkflowState.COMPENSATING;
            stateMachine.sendEvent(PaymentEvent.START_COMPENSATION, e);
            
            compensate();
            
            workflowState = cancelled ? 
                PaymentWorkflowState.CANCELLED : PaymentWorkflowState.FAILED;
            
            return PaymentResult.builder()
                .sagaId(sagaId)
                .orderId(request.getOrderId())
                .status(cancelled ? PaymentResultStatus.CANCELLED : PaymentResultStatus.FAILED)
                .workflowState(workflowState)
                .domainState(stateMachine.getCurrentState())
                .errorMessage(e.getMessage())
                .build();
        }
    }
    
    private <T> T executeWithStateTransition(String stepName, 
                                             Supplier<T> operation,
                                             PaymentEvent successEvent) {
        
        logger.info("[WORKFLOW] Executing step: {}", stepName);
        
        try {
            // Execute business operation
            T result = operation.get();
            
            // Transition state machine on success
            stateMachine.sendEvent(successEvent, result);
            
            logger.info("[WORKFLOW] Step completed: {}", stepName);
            
            return result;
            
        } catch (Exception e) {
            logger.error("[WORKFLOW] Step failed: {}", stepName, e);
            
            // Determine appropriate failure event
            PaymentEvent failureEvent = mapToFailureEvent(successEvent);
            stateMachine.sendEvent(failureEvent, e);
            
            throw e;
        }
    }
    
    private void compensate() {
        logger.info("[WORKFLOW] Starting compensation for SAGA: {}", sagaId);
        
        PaymentState currentState = stateMachine.getCurrentState();
        
        try {
            // Compensate in reverse order
            if (currentState.isAfterOrEqual(PaymentState.PAYMENT_CAPTURED)) {
                activities.refundPayment(stateMachine.getCaptureId());
                stateMachine.sendEvent(PaymentEvent.PAYMENT_REFUNDED, null);
            } else if (currentState.isAfterOrEqual(PaymentState.PAYMENT_AUTHORIZED)) {
                activities.voidAuthorization(stateMachine.getAuthorizationId());
                stateMachine.sendEvent(PaymentEvent.PAYMENT_VOIDED, null);
            }
            
            if (currentState.isAfterOrEqual(PaymentState.INVENTORY_RESERVED)) {
                activities.releaseInventory(stateMachine.getReservationId());
                stateMachine.sendEvent(PaymentEvent.INVENTORY_RELEASED, null);
            }
            
            if (currentState.isAfterOrEqual(PaymentState.ORDER_VALIDATED)) {
                activities.cancelOrder(stateMachine.getOrderId());
                stateMachine.sendEvent(PaymentEvent.ORDER_CANCELLED, null);
            }
            
            stateMachine.sendEvent(PaymentEvent.COMPENSATION_COMPLETED, null);
            
            logger.info("[WORKFLOW] Compensation completed for SAGA: {}", sagaId);
            
        } catch (Exception e) {
            logger.error("[WORKFLOW] Compensation failed for SAGA: {}", sagaId, e);
            stateMachine.sendEvent(PaymentEvent.COMPENSATION_FAILED, e);
            throw e;
        }
    }
    
    @Override
    public void cancel() {
        logger.info("[WORKFLOW] Cancellation requested for SAGA: {}", sagaId);
        this.cancelled = true;
        stateMachine.sendEvent(PaymentEvent.CANCEL_REQUESTED, null);
    }
    
    private void checkCancellation() {
        if (cancelled) {
            throw new WorkflowCancelledException("Payment SAGA cancelled");
        }
    }
    
    @Override
    public PaymentWorkflowState getWorkflowState() {
        return workflowState;
    }
    
    // Workflow state enum
    public enum PaymentWorkflowState {
        INITIALIZED,
        RUNNING,
        VALIDATING_ORDER,
        RESERVING_INVENTORY,
        AUTHORIZING_PAYMENT,
        CAPTURING_PAYMENT,
        COMPLETING_ORDER,
        COMPLETED,
        COMPENSATING,
        FAILED,
        CANCELLED
    }
}

// ============================================================================
// SPRING STATE MACHINE CONFIGURATION
// ============================================================================

@Configuration
@EnableStateMachineFactory
public class PaymentStateMachineConfig 
    extends EnumStateMachineConfigurerAdapter<PaymentState, PaymentEvent> {
    
    @Autowired
    private PaymentStateActions actions;
    
    @Autowired
    private PaymentStateGuards guards;
    
    @Override
    public void configure(StateMachineConfigurationConfigurer<PaymentState, PaymentEvent> config)
            throws Exception {
        config
            .withConfiguration()
                .autoStartup(false)
            .and()
            .withPersistence()
                .runtimePersister(stateMachineRuntimePersister());
    }
    
    @Override
    public void configure(StateMachineStateConfigurer<PaymentState, PaymentEvent> states)
            throws Exception {
        states
            .withStates()
                .initial(PaymentState.PENDING)
                .end(PaymentState.COMPLETED)
                .end(PaymentState.COMPENSATED)
                .end(PaymentState.FAILED)
                .states(EnumSet.allOf(PaymentState.class));
    }
    
    @Override
    public void configure(StateMachineTransitionConfigurer<PaymentState, PaymentEvent> transitions)
            throws Exception {
        transitions
            // Forward transitions
            .withExternal()
                .source(PaymentState.PENDING).target(PaymentState.VALIDATING)
                .event(PaymentEvent.START_PAYMENT)
            .and()
            .withExternal()
                .source(PaymentState.VALIDATING).target(PaymentState.VALIDATED)
                .event(PaymentEvent.ORDER_VALIDATED)
                .action(actions.orderValidatedAction())
            .and()
            .withExternal()
                .source(PaymentState.VALIDATED).target(PaymentState.RESERVING)
                .event(PaymentEvent.ORDER_VALIDATED)
            .and()
            .withExternal()
                .source(PaymentState.RESERVING).target(PaymentState.RESERVED)
                .event(PaymentEvent.INVENTORY_RESERVED)
                .action(actions.inventoryReservedAction())
            .and()
            .withExternal()
                .source(PaymentState.RESERVED).target(PaymentState.AUTHORIZING)
                .event(PaymentEvent.INVENTORY_RESERVED)
            .and()
            .withExternal()
                .source(PaymentState.AUTHORIZING).target(PaymentState.AUTHORIZED)
                .event(PaymentEvent.PAYMENT_AUTHORIZED)
                .action(actions.paymentAuthorizedAction())
                .guard(guards.authorizationValid())
            .and()
            .withExternal()
                .source(PaymentState.AUTHORIZED).target(PaymentState.CAPTURING)
                .event(PaymentEvent.PAYMENT_AUTHORIZED)
            .and()
            .withExternal()
                .source(PaymentState.CAPTURING).target(PaymentState.CAPTURED)
                .event(PaymentEvent.PAYMENT_CAPTURED)
                .action(actions.paymentCapturedAction())
            .and()
            .withExternal()
                .source(PaymentState.CAPTURED).target(PaymentState.COMPLETING)
                .event(PaymentEvent.PAYMENT_CAPTURED)
            .and()
            .withExternal()
                .source(PaymentState.COMPLETING).target(PaymentState.COMPLETED)
                .event(PaymentEvent.ORDER_COMPLETED)
                .action(actions.orderCompletedAction())
            
            // Compensation transitions
            .and()
            .withExternal()
                .source(PaymentState.VALIDATING).target(PaymentState.COMPENSATING)
                .event(PaymentEvent.ORDER_VALIDATION_FAILED)
            .and()
            .withExternal()
                .source(PaymentState.RESERVING).target(PaymentState.COMPENSATING)
                .event(PaymentEvent.INVENTORY_RESERVATION_FAILED)
            .and()
            .withExternal()
                .source(PaymentState.AUTHORIZING).target(PaymentState.COMPENSATING)
                .event(PaymentEvent.PAYMENT_AUTHORIZATION_FAILED)
            .and()
            .withExternal()
                .source(PaymentState.CAPTURING).target(PaymentState.COMPENSATING)
                .event(PaymentEvent.PAYMENT_CAPTURE_FAILED)
            
            // Compensation actions
            .and()
            .withExternal()
                .source(PaymentState.COMPENSATING).target(PaymentState.COMPENSATED)
                .event(PaymentEvent.COMPENSATION_COMPLETED)
                .action(actions.compensationCompletedAction());
    }
    
    @Bean
    public StateMachineRuntimePersister<PaymentState, PaymentEvent, String> 
            stateMachineRuntimePersister() {
        return new JpaStateMachineRuntimePersister<>();
    }
}

// ============================================================================
// STATE MACHINE ACTIONS (Business Logic on Transitions)
// ============================================================================

@Component
public class PaymentStateActions {
    
    @Autowired
    private DomainEventPublisher eventPublisher;
    
    @Autowired
    private SagaAuditService auditService;
    
    public Action<PaymentState, PaymentEvent> orderValidatedAction() {
        return context -> {
            String sagaId = getSagaId(context);
            OrderValidation validation = getPayload(context);
            
            // Publish domain event
            eventPublisher.publish(new OrderValidatedEvent(
                sagaId, 
                validation.getOrderId(),
                validation.getValidationId()
            ));
            
            // Record audit
            auditService.recordTransition(sagaId, PaymentState.VALIDATING, 
                                         PaymentState.VALIDATED);
            
            logger.info("[STATE MACHINE] Order validated: {}", sagaId);
        };
    }
    
    public Action<PaymentState, PaymentEvent> inventoryReservedAction() {
        return context -> {
            String sagaId = getSagaId(context);
            InventoryReservation reservation = getPayload(context);
            
            // Store reservation ID in state machine context
            context.getExtendedState().getVariables()
                .put("reservationId", reservation.getReservationId());
            
            // Publish event
            eventPublisher.publish(new InventoryReservedEvent(
                sagaId,
                reservation.getReservationId(),
                reservation.getItems()
            ));
            
            logger.info("[STATE MACHINE] Inventory reserved: {}", sagaId);
        };
    }
    
    public Action<PaymentState, PaymentEvent> paymentAuthorizedAction() {
        return context -> {
            String sagaId = getSagaId(context);
            PaymentAuth auth = getPayload(context);
            
            // Store auth ID
            context.getExtendedState().getVariables()
                .put("authorizationId", auth.getAuthId());
            context.getExtendedState().getVariables()
                .put("authorizationExpiry", auth.getExpiresAt());
            
            // Publish event
            eventPublisher.publish(new PaymentAuthorizedEvent(
                sagaId,
                auth.getAuthId(),
                auth.getAmount(),
                auth.getExpiresAt()
            ));
            
            logger.info("[STATE MACHINE] Payment authorized: {} auth: {}", 
                       sagaId, auth.getAuthId());
        };
    }
    
    public Action<PaymentState, PaymentEvent> paymentCapturedAction() {
        return context -> {
            String sagaId = getSagaId(context);
            PaymentCapture capture = getPayload(context);
            
            // Store capture ID
            context.getExtendedState().getVariables()
                .put("captureId", capture.getCaptureId());
            
            // Publish event
            eventPublisher.publish(new PaymentCapturedEvent(
                sagaId,
                capture.getCaptureId(),
                capture.getAmount()
            ));
            
            logger.info("[STATE MACHINE] Payment captured: {} capture: {}", 
                       sagaId, capture.getCaptureId());
        };
    }
    
    public Action<PaymentState, PaymentEvent> orderCompletedAction() {
        return context -> {
            String sagaId = getSagaId(context);
            
            // Publish completion event
            eventPublisher.publish(new OrderCompletedEvent(
                sagaId,
                getOrderId(context)
            ));
            
            logger.info("[STATE MACHINE] Order completed: {}", sagaId);
        };
    }
    
    public Action<PaymentState, PaymentEvent> compensationCompletedAction() {
        return context -> {
            String sagaId = getSagaId(context);
            
            // Publish compensation completed event
            eventPublisher.publish(new CompensationCompletedEvent(sagaId));
            
            logger.info("[STATE MACHINE] Compensation completed: {}", sagaId);
        };
    }
    
    // Helper methods
    private String getSagaId(StateContext<PaymentState, PaymentEvent> context) {
        return context.getExtendedState().get("sagaId", String.class);
    }
    
    private String getOrderId(StateContext<PaymentState, PaymentEvent> context) {
        return context.getExtendedState().get("orderId", String.class);
    }
    
    private <T> T getPayload(StateContext<PaymentState, PaymentEvent> context) {
        return (T) context.getMessage().getHeaders().get("payload");
    }
}

// ============================================================================
// STATE MACHINE GUARDS (Business Rules)
// ============================================================================

@Component
public class PaymentStateGuards {
    
    /**
     * Guard: Only allow capture if authorization is still valid
     */
    public Guard<PaymentState, PaymentEvent> authorizationValid() {
        return context -> {
            Instant expiresAt = context.getExtendedState()
                .get("authorizationExpiry", Instant.class);
            
            if (expiresAt == null) {
                return false;
            }
            
            boolean valid = Instant.now().isBefore(expiresAt);
            
            if (!valid) {
                logger.warn("Authorization expired for SAGA: {}", 
                           context.getExtendedState().get("sagaId", String.class));
            }
            
            return valid;
        };
    }
    
    /**
     * Guard: Only allow compensation if not already compensated
     */
    public Guard<PaymentState, PaymentEvent> canCompensate() {
        return context -> {
            PaymentState currentState = context.getSource().getId();
            return currentState != PaymentState.COMPENSATED 
                && currentState != PaymentState.FAILED;
        };
    }
}
```

### Configuration

```yaml
# application.yml - Hybrid configuration

# Temporal configuration
temporal:
  host: ${TEMPORAL_HOST:localhost}
  port: ${TEMPORAL_PORT:7233}
  namespace: ${TEMPORAL_NAMESPACE:payment-saga}
  task-queue: payment-saga-queue
  worker:
    max-concurrent-activities: 20
    max-concurrent-workflows: 100

# Spring State Machine
spring:
  statemachine:
    distributed:
      enabled: true
    persistence:
      enabled: true
    
# State sync
state-sync:
  enabled: true
  check-interval: 60s
  reconciliation:
    enabled: true
    interval: 5m
```

### Deployment Architecture

```yaml
# Kubernetes deployment

# Temporal Server (if self-hosted)
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: temporal-server
spec:
  replicas: 3
  template:
    spec:
      containers:
      - name: temporal
        image: temporalio/auto-setup:1.24.0
        
# Payment Saga Service (Workers + State Machine)
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: payment-saga-service
spec:
  replicas: 5
  template:
    spec:
      containers:
      - name: saga-service
        image: payment-saga:latest
        env:
        - name: TEMPORAL_HOST
          value: temporal-server
        - name: DB_HOST
          value: postgres
        - name: REDIS_HOST
          value: redis
```

---

## Migration Strategy

### Phase 1: Parallel Run (1-2 months)

```java
/**
 * Run both systems in parallel, compare results
 */
@Service
public class MigrationService {
    
    @Autowired
    private TemporalPaymentService temporalService;
    
    @Autowired
    private HybridPaymentService hybridService;
    
    public PaymentResult processPayment(PaymentRequest request) {
        
        // Process with both systems
        CompletableFuture<PaymentResult> temporal = 
            CompletableFuture.supplyAsync(() -> 
                temporalService.process(request));
        
        CompletableFuture<PaymentResult> hybrid = 
            CompletableFuture.supplyAsync(() -> 
                hybridService.process(request));
        
        // Wait for both
        PaymentResult temporalResult = temporal.join();
        PaymentResult hybridResult = hybrid.join();
        
        // Compare and alert on differences
        compareResults(temporalResult, hybridResult);
        
        // Return Temporal result (existing system)
        return temporalResult;
    }
}
```

### Phase 2: Gradual Cutover (1-2 months)

```java
/**
 * Route percentage of traffic to hybrid system
 */
@Service
public class CanaryMigrationService {
    
    @Value("${migration.hybrid.percentage:0}")
    private int hybridPercentage;
    
    public PaymentResult processPayment(PaymentRequest request) {
        
        // Random routing based on percentage
        if (ThreadLocalRandom.current().nextInt(100) < hybridPercentage) {
            return hybridService.process(request);
        } else {
            return temporalService.process(request);
        }
    }
}
```

### Phase 3: Full Migration (1 month)

- Switch all traffic to hybrid system
- Keep Temporal-only as fallback
- Monitor for issues
- Decommission Temporal-only after stabilization

---

## Summary

The hybrid Temporal + Spring State Machine architecture provides:

✅ **Best of Both Worlds**
- Temporal: Workflow orchestration, durability, retries, visibility
- State Machine: Rich domain modeling, business rules, queryable state

✅ **Clear Separation of Concerns**
- Temporal handles "how" (orchestration)
- State Machine handles "what" (business state)

✅ **Production Ready**
- Battle-tested components
- Excellent tooling and observability
- Gradual migration path

✅ **Flexible Integration**
- Multiple patterns: direct, event-driven, signal-based
- Choose pattern based on coupling requirements

✅ **Enterprise Grade**
- Full auditability
- State synchronization
- Disaster recovery
- Monitoring and alerting

**Recommendation**: Use this hybrid approach when you need both sophisticated workflow orchestration AND rich business domain modeling. The added complexity is justified for complex payment workflows with intricate business rules.

---

**Author**: Payment Systems Architecture Team
**Version**: 2.0 - Hybrid Architecture
**Last Updated**: January 2026
