# T24 Core Banking Integration Architecture

This document describes the architecture design and principles for integrating the Payment SAGA Platform with Temenos T24 Core Banking System, addressing blocking process issues through the SAGA pattern.

## Table of Contents

- [Executive Summary](#executive-summary)
- [Why SAGA Pattern for Core Banking](#why-saga-pattern-for-core-banking)
  - [Why NOT Traditional 2PC](#why-not-traditional-2pc-two-phase-commit)
  - [Why NOT TCC Pattern](#why-not-tcc-try-confirm-cancel-pattern)
  - [SAGA Pattern Advantages](#saga-pattern-advantages-for-t24-integration)
- [T24 Integration Challenges](#t24-integration-challenges)
- [Architecture Design](#architecture-design)
- [Integration Principles](#integration-principles)
- [T24 Adapter Design](#t24-adapter-design)
- [Compensation Strategies](#compensation-strategies)
- [Implementation Patterns](#implementation-patterns)
- [Monitoring and Reconciliation](#monitoring-and-reconciliation)
- [Summary](#summary)

---

## Executive Summary

### The Problem

Temenos T24 core banking systems present unique integration challenges:

```mermaid
flowchart LR
    subgraph Traditional["TRADITIONAL APPROACH"]
        A[Payment Request] --> B[T24 - blocking]
        B --> C[Response]
        B --> D[PROBLEMS]
    end

    subgraph Problems[" "]
        D --> P1["30-60s response times"]
        D --> P2["Batch processing windows"]
        D --> P3["Limited concurrent connections"]
        D --> P4["No distributed transaction support"]
        D --> P5["Account locks during processing"]
        D --> P6["EOD/SOD processing blackouts"]
    end
```

### The Solution

Apply SAGA pattern with asynchronous T24 integration:

```mermaid
flowchart TB
    subgraph SAGA["SAGA-BASED APPROACH"]
        A[Payment Request] --> B

        subgraph Temporal["Payment SAGA (Temporal)"]
            B["• Non-blocking<br/>• Compensation<br/>• State Tracking"]
        end

        subgraph Adapter["T24 Adapter (Async/Queue)"]
            C["• Request Queue<br/>• Response Queue<br/>• Idempotent"]
        end

        B <--> C
        C --> D[(T24<br/>Core Banking)]
    end
```

---

## Why SAGA Pattern for Core Banking

### Core Banking Characteristics vs SAGA Benefits

| T24 Characteristic | Challenge | SAGA Solution |
|-------------------|-----------|---------------|
| **Slow Response (30-60s)** | HTTP timeouts, blocked threads | Async activities with durable state |
| **Batch Processing Windows** | Operations fail during EOD | Queue-based retry with backpressure |
| **Limited Concurrency** | Connection pool exhaustion | Bulkhead + rate limiting per account |
| **No Distributed Transactions** | Partial failures leave inconsistent state | Compensation-based rollback |
| **Account Locks** | Concurrent operations deadlock | Workflow-level serialization |
| **Non-Idempotent Operations** | Duplicate debits on retry | Idempotency keys + transaction reference |

### Why NOT Traditional 2PC (Two-Phase Commit)

```mermaid
flowchart TB
    subgraph TwoPC["TWO-PHASE COMMIT (2PC)"]
        direction TB
        C1[Coordinator]
        C1 -->|PREPARE| PS1[Payment Service ✓]
        C1 -->|PREPARE| T24_1["T24 Core Banking ✗<br/>(No 2PC support)"]

        F1["❌ FAILURE: T24 doesn't support distributed transactions"]
        F2["❌ PROBLEM: Locks held across network boundaries"]
        F3["❌ PROBLEM: Single coordinator = single point of failure"]
    end
```

```mermaid
flowchart TB
    subgraph SAGA["SAGA PATTERN"]
        direction TB
        O[Orchestrator - Temporal]

        O -->|Step 1| S1[Validate Account ✓]
        O -->|Step 2| S2[Reserve Amount ✓]
        O -->|Step 3| S3["Debit T24 Account ✗<br/>(Failed)"]

        O -.->|Compensate| C1[Release Reserve ✓]
        O -.->|Compensate| C2[Cancel Validation ✓]

        B1["✓ Each step is independent transaction"]
        B2["✓ Compensation reverses completed steps"]
        B3["✓ State persisted, survives failures"]
        B4["✓ Works with T24's existing transaction model"]
    end
```

### Why NOT TCC (Try-Confirm-Cancel) Pattern

TCC is another distributed transaction pattern that appears suitable at first glance but presents fundamental incompatibilities with T24's operational model.

#### How TCC Works

```mermaid
flowchart TB
    subgraph TCC["TRY-CONFIRM-CANCEL (TCC) PATTERN"]
        direction TB

        subgraph Try["Phase 1: TRY (Reserve)"]
            T1["Reserve resources"]
            T2["Create tentative state"]
            T3["Lock resources for timeout period"]
        end

        subgraph Confirm["Phase 2: CONFIRM (Commit)"]
            C1["Make reservation permanent"]
            C2["Release locks"]
            C3["Must be idempotent"]
        end

        subgraph Cancel["Phase 2: CANCEL (Rollback)"]
            X1["Release reservations"]
            X2["Restore original state"]
            X3["Must be idempotent"]
        end

        Try -->|All Try succeed| Confirm
        Try -->|Any Try fails| Cancel
    end
```

#### TCC vs T24: Fundamental Incompatibilities

```mermaid
flowchart LR
    subgraph TCC_Req["TCC REQUIREMENTS"]
        R1["Tentative reservations<br/>(soft locks)"]
        R2["Quick confirmation<br/>(< 5 seconds)"]
        R3["Native Cancel support<br/>(undo tentative)"]
        R4["Timeout-based<br/>auto-cancellation"]
        R5["All participants must<br/>implement TCC protocol"]
    end

    subgraph T24_Reality["T24 REALITY"]
        T1["Postings are final<br/>(no tentative state)"]
        T2["30-60s response times<br/>(exceeds TCC timeout)"]
        T3["Reversal = new posting<br/>(not cancel)"]
        T4["No auto-cancel<br/>(requires explicit reversal)"]
        T5["OFS protocol only<br/>(no TCC support)"]
    end

    R1 -.->|"❌ Incompatible"| T1
    R2 -.->|"❌ Incompatible"| T2
    R3 -.->|"❌ Incompatible"| T3
    R4 -.->|"❌ Incompatible"| T4
    R5 -.->|"❌ Incompatible"| T5
```

#### Detailed Analysis: Why TCC Fails with T24

| TCC Requirement | T24 Behavior | Problem |
|-----------------|--------------|---------|
| **Tentative Reservations** | T24 postings are immediately final. There is no "tentative" state - once a debit posts, funds are moved. | Cannot implement true TRY phase without custom T24 modifications |
| **Quick Confirmation** | T24 operations take 30-60 seconds. TCC assumes sub-second TRY phase with quick CONFIRM. | TCC timeout would expire before T24 responds |
| **Native Cancel** | T24 has no "cancel" - only reversal postings. A cancel in T24 creates a new offsetting transaction. | "Cancel" leaves audit trail, doesn't truly undo |
| **Resource Locking** | T24 locks accounts during posting, but releases immediately after. TCC requires holding locks until CONFIRM/CANCEL. | Cannot extend T24's lock duration |
| **Coordinator Timeout** | TCC coordinators auto-cancel after timeout. T24 during EOD may not respond for hours. | Mass cancellations during batch windows |
| **Protocol Support** | TCC requires all participants to implement Try/Confirm/Cancel interfaces. T24 uses OFS messaging. | Would require building TCC facade over OFS |

#### TCC Implementation Attempt with T24

```mermaid
sequenceDiagram
    participant Coord as TCC Coordinator
    participant PS as Payment Service
    participant T24

    Note over Coord: TCC Timeout: 30 seconds

    Coord->>PS: TRY: Reserve $100
    PS-->>Coord: Reserved ✓

    Coord->>T24: TRY: Reserve $100 from Account
    Note over T24: Processing...<br/>30-60 seconds

    Note over Coord: ⏰ TIMEOUT (30s)!<br/>Must CANCEL all

    Coord->>PS: CANCEL: Release $100
    PS-->>Coord: Released ✓

    Note over T24: Still processing...

    T24-->>Coord: Reserved ✓ (too late!)

    Note over Coord,T24: ❌ INCONSISTENT STATE<br/>T24 has reservation<br/>Payment Service cancelled
```

#### The "Pseudo-TCC" Anti-Pattern

Some teams attempt to build TCC over T24 using this approach:

```mermaid
flowchart TB
    subgraph Pseudo["PSEUDO-TCC ANTI-PATTERN"]
        direction TB

        subgraph PseudoTry["Pseudo-TRY"]
            PT1["Create HOLD transaction in T24"]
            PT2["Block funds without posting"]
        end

        subgraph PseudoConfirm["Pseudo-CONFIRM"]
            PC1["Convert HOLD to actual debit"]
            PC2["Two T24 operations required"]
        end

        subgraph PseudoCancel["Pseudo-CANCEL"]
            PX1["Release HOLD"]
            PX2["Another T24 operation"]
        end

        PseudoTry --> PseudoConfirm
        PseudoTry --> PseudoCancel
    end

    subgraph Problems["WHY THIS FAILS"]
        P1["❌ HOLD + CONFIRM = 2x latency (60-120s)"]
        P2["❌ HOLD may fail during EOD"]
        P3["❌ CONFIRM may fail after HOLD succeeds"]
        P4["❌ Orphaned HOLDs if coordinator crashes"]
        P5["❌ T24 HOLD limits per account"]
    end

    Pseudo --> Problems
```

**Problems with Pseudo-TCC:**
1. **Double Latency**: TRY (hold) + CONFIRM (post) = 60-120 seconds total
2. **Partial Failures**: HOLD succeeds but CONFIRM fails during EOD
3. **Orphaned Holds**: Coordinator crash leaves funds locked
4. **Hold Limits**: T24 has limits on concurrent holds per account
5. **Complexity**: More failure modes than direct posting

#### Why SAGA is Superior to TCC for T24

```mermaid
flowchart TB
    subgraph Comparison["SAGA vs TCC for T24"]
        direction LR

        subgraph SAGA_Adv["SAGA ADVANTAGES"]
            S1["✓ Works with T24's posting model"]
            S2["✓ No timeout constraints"]
            S3["✓ Handles 30-60s latency"]
            S4["✓ Survives EOD windows"]
            S5["✓ Compensation = reversal posting"]
            S6["✓ Durable state in Temporal"]
        end

        subgraph TCC_Dis["TCC DISADVANTAGES"]
            T1["❌ Requires tentative state"]
            T2["❌ Timeout-based coordination"]
            T3["❌ Cannot handle slow responses"]
            T4["❌ Fails during batch windows"]
            T5["❌ Cancel ≠ T24 reversal"]
            T6["❌ Coordinator is SPOF"]
        end
    end
```

| Aspect | TCC | SAGA | Winner for T24 |
|--------|-----|------|----------------|
| **Latency Tolerance** | Low (seconds) | High (hours/days) | SAGA |
| **T24 Protocol** | Requires custom facade | Works with OFS directly | SAGA |
| **EOD/SOD Handling** | Times out, mass cancels | Queues, retries automatically | SAGA |
| **Failure Recovery** | Coordinator must be available | Temporal persists state | SAGA |
| **Audit Trail** | TRY + CONFIRM/CANCEL | Single posting + reversal if needed | SAGA |
| **Implementation Effort** | High (build TCC over T24) | Low (use T24 as-is) | SAGA |
| **Operational Complexity** | High (orphaned reservations) | Low (compensation is explicit) | SAGA |

#### Summary: TCC is Not Suitable for T24

```mermaid
flowchart TB
    subgraph Decision["PATTERN SELECTION FOR T24"]
        Q1{{"Does T24 support<br/>tentative reservations?"}}
        Q2{{"Can T24 respond<br/>within TCC timeout?"}}
        Q3{{"Does T24 have native<br/>Cancel operation?"}}

        Q1 -->|"No"| TCC_Out["❌ TCC Not Viable"]
        Q2 -->|"No (30-60s)"| TCC_Out
        Q3 -->|"No (reversal only)"| TCC_Out

        TCC_Out --> SAGA_Win["✓ Use SAGA Pattern"]

        SAGA_Win --> Benefits["Benefits:<br/>• Works with T24's posting model<br/>• Handles long latencies<br/>• Survives batch windows<br/>• Clear compensation via reversal"]
    end
```

**Key Takeaway**: TCC assumes a world of fast, tentative operations with native cancel support. T24 operates in a world of slow, final postings with reversal-based compensation. SAGA's "execute and compensate" model aligns perfectly with T24's "post and reverse" model.

### SAGA Pattern Advantages for T24 Integration

1. **Loose Coupling**
   - T24 operations remain self-contained transactions
   - No need to modify T24's internal transaction handling
   - Each service manages its own data consistency

2. **Failure Isolation**
   - T24 failures don't lock up Payment SAGA resources
   - Circuit breaker prevents cascade to healthy services
   - Workflow state preserved during T24 outages

3. **Long-Running Operations**
   - Temporal durably stores workflow state
   - Activities can wait for T24 batch completion
   - Human intervention points for manual approval

4. **Visibility & Auditability**
   - Every step recorded in workflow history
   - Compensation actions logged for compliance
   - Full trace from payment request to T24 posting

5. **Graceful Degradation**
   - Payments queued when T24 unavailable
   - Automatic retry when T24 returns online
   - Priority-based processing for critical payments

---

## T24 Integration Challenges

### Challenge 1: Blocking API Calls

**Problem:** T24 operations take 30-60 seconds, causing thread exhaustion.

**Solution:** Async Activity with Heartbeat

```java
@ActivityMethod
public T24Response debitAccount(T24DebitRequest request) {
    // Submit to T24 request queue
    String transactionRef = t24Gateway.submitDebit(request);

    // Poll for completion with heartbeat (keeps workflow alive)
    while (!t24Gateway.isComplete(transactionRef)) {
        Activity.getExecutionContext().heartbeat(
            "Waiting for T24: ref=" + transactionRef);
        Thread.sleep(Duration.ofSeconds(5));
    }

    return t24Gateway.getResult(transactionRef);
}
```

### Challenge 2: Batch Processing Windows (EOD/SOD)

**Problem:** T24 rejects operations during End-of-Day processing.

**Solution:** Backpressure Queue with Scheduled Retry

```mermaid
flowchart TB
    subgraph Handling["T24 AVAILABILITY HANDLING"]
        A[Payment Request] --> B{T24 Availability<br/>Check}

        B -->|AVAILABLE| C[Process<br/>Immediately]

        B -->|UNAVAILABLE| D[Pending<br/>Queue]
        D --> E{T24 Online<br/>Trigger}
        F[Scheduled Check<br/>every 5min] --> E
        E --> G[Process Queued Payments<br/>Priority Order]
    end
```

### Challenge 3: Non-Idempotent Operations

**Problem:** Retrying a debit creates duplicate transactions.

**Solution:** T24 Transaction Reference + External Idempotency Key

```java
public class T24DebitRequest {
    private String idempotencyKey;      // Our unique key
    private String t24TransactionRef;   // T24's reference (if known)
    private String accountNumber;
    private BigDecimal amount;
    private String currency;
    private String narrative;

    // T24 checks: Has this idempotencyKey been processed?
    // If yes: Return existing transaction reference
    // If no: Process and return new reference
}
```

### Challenge 4: Account-Level Locking

**Problem:** Concurrent operations on same account cause deadlocks.

**Solution:** Customer/Account-Based Sharding

```mermaid
flowchart LR
    subgraph Serialization["ACCOUNT-LEVEL SERIALIZATION"]
        P1["Payment 1<br/>(Account: 123456)"] --> S3["Shard-3<br/>(serialized)"]
        P2["Payment 2<br/>(Account: 123456)"] --> S3
        P3["Payment 3<br/>(Account: 123456)"] --> S3

        P4["Payment 4<br/>(Account: 789012)"] --> S7["Shard-7<br/>(serialized)"]
        P5["Payment 5<br/>(Account: 789012)"] --> S7
    end

    Formula["Shard = hash(accountNumber) % shardCount<br/>Same account → Same shard → Serial execution"]
```

### Challenge 5: Partial Failures

**Problem:** Debit succeeds but credit fails (or vice versa).

**Solution:** Compensating Transactions

```
SCENARIO: Transfer $100 from Account A to Account B

Forward Flow:
1. Reserve $100 from Account A (soft hold)
2. Debit $100 from Account A → T24 posting
3. Credit $100 to Account B → T24 posting
4. Release reservation on Account A

If Step 3 (Credit) fails AFTER Step 2 (Debit) succeeded:

Compensation Flow:
1. Reverse Debit on Account A (T24 REVERSAL posting)
2. Release reservation on Account A
3. Mark workflow as COMPENSATED

T24 Reversal = New posting that negates original posting
```

---

## Architecture Design

### High-Level Architecture

```mermaid
flowchart TB
    subgraph Platform["PAYMENT SAGA PLATFORM"]
        subgraph Temporal["TEMPORAL WORKFLOW ENGINE"]
            subgraph Workflow["PaymentSagaWorkflow"]
                W1["1. validatePayment()"]
                W2["2. checkAccountBalance()"]
                W3["3. reserveFunds()"]
                W4["4. debitSourceAccount()"]
                W5["5. creditDestAccount()"]
                W6["6. completePayment()"]

                W1 --> W2 --> W3 --> W4 --> W5 --> W6

                subgraph Compensation["Compensation Stack"]
                    C1["reverseCredit()"]
                    C2["reverseDebit()"]
                    C3["releaseReservation()"]
                end
            end

            T24Act["T24Activities<br/>(Async)"]
            W2 -.-> T24Act
            W3 -.-> T24Act
            W4 -.-> T24Act
            W5 -.-> T24Act
        end

        subgraph Adapter["T24 ADAPTER LAYER"]
            G["T24 Gateway<br/>(Circuit Breaker)"]
            Q["Request Queue<br/>(Priority Based)"]
            I["Idempotency<br/>Registry<br/>(Redis/DB)"]
            M["T24 Status<br/>Monitor<br/>(EOD/SOD)"]
            R["Response<br/>Handler<br/>(Callback)"]
            Rec["Reconcile<br/>Service<br/>(Daily)"]
        end

        Temporal --> Adapter
    end

    subgraph T24["TEMENOS T24 CORE BANKING"]
        OFS["OFS<br/>(Open Financial<br/>Services)<br/>Message Bus"]
        TAFJ["TAFJ<br/>(Temenos<br/>Application<br/>Framework)"]
        DB[(T24 Database<br/>Account Ledger)]

        OFS --> TAFJ --> DB

        TxTypes["Transaction Types:<br/>• AC (Account Transfer)<br/>• FT (Funds Transfer)<br/>• TT (Teller Transaction)<br/>• LD (Lending)"]
    end

    Adapter --> T24
```

### Sequence Diagram: Payment with T24 Integration

```mermaid
sequenceDiagram
    participant Client
    participant API as API Gateway
    participant WF as Workflow (Temporal)
    participant Adapter as T24 Adapter
    participant Queue as Request Queue
    participant T24

    Client->>API: POST /payments
    API->>WF: Start Workflow
    API-->>Client: 202 Accepted

    WF->>Adapter: 1. Validate
    Adapter-->>WF: OK

    WF->>Queue: 2. Check Balance
    Queue->>T24: Submit OFS
    Note over WF: Heartbeat while waiting
    T24-->>Queue: Response
    Queue-->>WF: Balance OK

    WF->>Queue: 3. Debit Account
    Queue->>T24: Submit FT
    Note over WF: Heartbeat while waiting
    T24-->>Queue: Posted
    Queue-->>WF: TxnRef

    Note over WF: Push to Compensation Stack

    WF->>Queue: 4. Credit Account
    Queue->>T24: Submit FT
    T24-->>Queue: Posted
    Queue-->>WF: TxnRef

    WF->>Adapter: 5. Complete

    WF-->>Client: Webhook: Payment Complete
```

---

## Integration Principles

### Principle 1: Asynchronous by Default

**Never make synchronous blocking calls to T24.**

```java
// ❌ BAD: Synchronous blocking call
public T24Response debitAccount(T24Request request) {
    return t24Client.debit(request);  // Blocks for 30-60s
}

// ✓ GOOD: Async with polling
public T24Response debitAccount(T24Request request) {
    String txnRef = t24Gateway.submitAsync(request);

    while (!isComplete(txnRef)) {
        Activity.getExecutionContext().heartbeat(txnRef);
        sleep(5000);
    }

    return t24Gateway.getResult(txnRef);
}
```

### Principle 2: Idempotency at Every Layer

```mermaid
flowchart TB
    subgraph Layers["IDEMPOTENCY LAYERS"]
        direction TB

        subgraph L1["Layer 1: Workflow Level"]
            L1A["Workflow ID = payment-{orderId}-{uuid}"]
            L1B["Temporal prevents duplicate workflow starts"]
        end

        subgraph L2["Layer 2: Activity Level"]
            L2A["Idempotency Key = {orderId}-{step}-{attempt}"]
            L2B["Redis/DB check before execution"]
        end

        subgraph L3["Layer 3: T24 Adapter Level"]
            L3A["T24 Transaction Reference = {idempotencyKey}"]
            L3B["T24's OFS deduplication by reference"]
        end

        subgraph L4["Layer 4: T24 Core Level"]
            L4A["Account posting reference"]
            L4B["T24's internal duplicate detection"]
        end

        L1 --> L2 --> L3 --> L4
    end
```

### Principle 3: Compensation Over Rollback

**Design compensation, not rollback.**

| Operation | Compensation | Notes |
|-----------|--------------|-------|
| Reserve Funds | Release Reserve | Soft hold → Released |
| Debit Account | Reversal Posting | New credit entry |
| Credit Account | Reversal Posting | New debit entry |
| Update Status | Revert Status | Status = CANCELLED |

```java
// Compensation is a NEW transaction, not an UNDO
public void compensateDebit(String originalTxnRef) {
    T24ReversalRequest reversal = T24ReversalRequest.builder()
        .originalReference(originalTxnRef)
        .reversalType(ReversalType.FULL)
        .narrative("SAGA compensation: " + workflowId)
        .build();

    t24Gateway.postReversal(reversal);  // Creates new posting
}
```

### Principle 4: Fail-Open for Compensations

**Compensation failures should not block other compensations.**

```java
private void compensate() {
    int successCount = 0;
    int failureCount = 0;

    for (CompensationAction action : compensationStack) {
        try {
            action.execute();
            successCount++;
        } catch (Exception e) {
            failureCount++;
            // LOG but CONTINUE - don't throw
            log.error("[COMPENSATION-FAILED] {} - manual reconciliation required",
                action.getName(), e);
            alertService.raiseCompensationFailure(workflowId, action);
        }
    }

    // Even with failures, workflow completes (for visibility)
    // Failed compensations trigger manual reconciliation
}
```

### Principle 5: Account-Level Serialization

**Serialize operations on the same account to prevent deadlocks.**

```java
// Shard by account number ensures same-account operations are serialized
public String calculateTaskQueue(String accountNumber, PaymentPriority priority) {
    int shardId = Math.abs(accountNumber.hashCode()) % shardCount;
    return String.format("t24-saga-queue-%s-shard-%d",
        priority.name().toLowerCase(), shardId);
}
```

### Principle 6: Eventual Consistency with Reconciliation

**Accept eventual consistency, implement daily reconciliation.**

```mermaid
flowchart TB
    subgraph Strategy["RECONCILIATION STRATEGY"]
        direction TB

        subgraph RT["Real-Time (Best Effort)"]
            RT1["T24 posts → Webhook/Queue → Update Payment Status"]
            RT2["~95% of payments reconciled within 5 minutes"]
        end

        subgraph NRT["Near Real-Time (Catch-up)"]
            NRT1["Every 15 minutes: Query T24 for pending transactions"]
            NRT2["Match against Payment SAGA database"]
        end

        subgraph Daily["Daily (Full Reconciliation)"]
            D1["SOD+1: Full extract from T24"]
            D2["Compare with Payment SAGA ledger"]
            D3["Generate discrepancy report"]
            D4["Auto-correct where safe, alert for manual review"]
            D1 --> D2 --> D3 --> D4
        end

        RT --> NRT --> Daily
    end
```

### Principle 7: Circuit Breaker for T24 Protection

```yaml
resilience4j:
  circuitbreaker:
    instances:
      t24-gateway:
        slidingWindowType: COUNT_BASED
        slidingWindowSize: 20
        minimumNumberOfCalls: 10
        failureRateThreshold: 50
        waitDurationInOpenState: 120s  # T24 recovery time
        permittedNumberOfCallsInHalfOpenState: 5
        slowCallDurationThreshold: 30s  # T24 is slow
        slowCallRateThreshold: 80
```

---

## T24 Adapter Design

### T24 Gateway Interface

```java
public interface T24Gateway {

    // Account Operations
    T24BalanceResponse getAccountBalance(String accountNumber);
    T24AccountResponse getAccountDetails(String accountNumber);

    // Transaction Operations
    String submitDebit(T24DebitRequest request);    // Returns txnRef
    String submitCredit(T24CreditRequest request);  // Returns txnRef
    String submitTransfer(T24TransferRequest request);

    // Transaction Status
    T24TransactionStatus getTransactionStatus(String txnRef);
    boolean isTransactionComplete(String txnRef);
    T24TransactionResponse getTransactionResult(String txnRef);

    // Reversal Operations
    String submitReversal(T24ReversalRequest request);

    // System Status
    T24SystemStatus getSystemStatus();  // EOD, SOD, ONLINE
    boolean isSystemAvailable();
}
```

### T24 Activities Implementation

```java
@Component
public class T24ActivitiesImpl implements T24Activities {

    private final T24Gateway t24Gateway;
    private final IdempotencyService idempotencyService;
    private final T24StatusMonitor statusMonitor;

    @Override
    public T24DebitResponse debitAccount(T24DebitRequest request) {
        String idempotencyKey = buildIdempotencyKey(
            "debit", request.getAccountNumber(), request.getAmount());

        // Check idempotency
        if (idempotencyService.isProcessed(idempotencyKey)) {
            return idempotencyService.getCachedResult(
                idempotencyKey, T24DebitResponse.class);
        }

        // Check T24 availability
        if (!statusMonitor.isAvailable()) {
            throw new T24UnavailableException(
                "T24 is in " + statusMonitor.getStatus() + " mode");
        }

        // Submit to T24
        String txnRef = t24Gateway.submitDebit(request);

        // Poll for completion with heartbeat
        while (!t24Gateway.isTransactionComplete(txnRef)) {
            Activity.getExecutionContext().heartbeat(
                Map.of("txnRef", txnRef, "status", "PENDING"));
            Thread.sleep(Duration.ofSeconds(5));
        }

        // Get result
        T24TransactionResponse t24Response = t24Gateway.getTransactionResult(txnRef);

        if (!t24Response.isSuccess()) {
            throw new T24TransactionException(t24Response.getErrorCode(),
                t24Response.getErrorMessage());
        }

        T24DebitResponse response = T24DebitResponse.builder()
            .transactionReference(txnRef)
            .postingDate(t24Response.getPostingDate())
            .valueDate(t24Response.getValueDate())
            .build();

        // Cache for idempotency
        idempotencyService.markProcessed(idempotencyKey, response);

        return response;
    }

    @Override
    public void reverseDebit(String originalTxnRef) {
        T24ReversalRequest reversal = T24ReversalRequest.builder()
            .originalReference(originalTxnRef)
            .reversalType(ReversalType.FULL)
            .build();

        // Reversal must be idempotent too
        String reversalKey = "reversal-" + originalTxnRef;
        if (idempotencyService.isProcessed(reversalKey)) {
            return;
        }

        String reversalRef = t24Gateway.submitReversal(reversal);

        // Wait for reversal completion
        while (!t24Gateway.isTransactionComplete(reversalRef)) {
            Activity.getExecutionContext().heartbeat(
                Map.of("reversalRef", reversalRef));
            Thread.sleep(Duration.ofSeconds(5));
        }

        idempotencyService.markProcessed(reversalKey, reversalRef);
    }
}
```

### T24 System Status Monitor

```java
@Component
public class T24StatusMonitor {

    private volatile T24SystemStatus currentStatus = T24SystemStatus.UNKNOWN;
    private volatile Instant lastCheck = Instant.MIN;

    @Scheduled(fixedRate = 30000)  // Check every 30 seconds
    public void checkStatus() {
        try {
            currentStatus = t24Gateway.getSystemStatus();
            lastCheck = Instant.now();

            if (currentStatus == T24SystemStatus.EOD_PROCESSING) {
                log.warn("[T24-STATUS] T24 is in EOD processing mode");
                metricsService.recordEodStart();
            } else if (currentStatus == T24SystemStatus.ONLINE) {
                log.info("[T24-STATUS] T24 is online");
            }
        } catch (Exception e) {
            currentStatus = T24SystemStatus.UNAVAILABLE;
            log.error("[T24-STATUS] Failed to check T24 status", e);
        }
    }

    public boolean isAvailable() {
        return currentStatus == T24SystemStatus.ONLINE;
    }

    public T24SystemStatus getStatus() {
        return currentStatus;
    }
}

public enum T24SystemStatus {
    ONLINE,           // Normal operations
    EOD_PROCESSING,   // End of Day batch
    SOD_PROCESSING,   // Start of Day batch
    MAINTENANCE,      // Scheduled maintenance
    UNAVAILABLE,      // Connection issues
    UNKNOWN           // Status check failed
}
```

---

## Compensation Strategies

### Strategy Matrix

| Scenario | Detection | Compensation | Recovery |
|----------|-----------|--------------|----------|
| Debit succeeded, Credit failed | Workflow state | Reverse debit | Automatic |
| Credit succeeded, Debit failed | Workflow state | Reverse credit | Automatic |
| T24 timeout during debit | No confirmation | Query T24 status | Manual if ambiguous |
| T24 EOD during operation | EOD status | Queue for SOD+1 | Automatic retry |
| Double posting detected | Reconciliation | Reverse duplicate | Manual approval |
| Account locked | T24 error code | Retry with backoff | Automatic |

### Compensation Flow

```mermaid
flowchart TB
    subgraph Decision["COMPENSATION DECISION TREE"]
        A[Payment Failed]

        A --> B[T24 Error]
        A --> C[Timeout Error]
        A --> D[Business Error]

        B --> E[Query T24<br/>Error Code]
        E --> F[Retryable]
        E --> G[Fatal]
        F --> H[Retry]
        G --> I[Compensate]

        C --> J[Query T24<br/>for status]
        J --> K[Posted]
        J --> L[Not Posted]
        K --> M["Compensate<br/>(reversal)"]
        L --> N["Continue<br/>(no action)"]

        D --> O["No retry<br/>Mark failed<br/>Compensate"]
    end
```

---

## Implementation Patterns

### Pattern 1: Reserve-Execute-Confirm (REC)

```java
public PaymentResult processT24Payment(String orderId) {

    // STEP 1: Reserve (soft hold)
    T24ReservationResponse reservation =
        t24Activities.reserveFunds(accountNumber, amount);
    compensationStack.push(() ->
        t24Activities.releaseReservation(reservation.getReservationId()));

    // STEP 2: Execute (actual posting)
    T24DebitResponse debit =
        t24Activities.debitAccount(request);
    compensationStack.push(() ->
        t24Activities.reverseDebit(debit.getTransactionReference()));

    T24CreditResponse credit =
        t24Activities.creditAccount(request);
    compensationStack.push(() ->
        t24Activities.reverseCredit(credit.getTransactionReference()));

    // STEP 3: Confirm (release reservation, already posted)
    t24Activities.releaseReservation(reservation.getReservationId());

    return PaymentResult.success();
}
```

### Pattern 2: Try-Confirm-Cancel (TCC)

```java
// Try Phase: Tentative reservation
T24TryResponse tryResult = t24Activities.tryReservation(request);

// Confirm Phase: Make reservation permanent
if (allStepsSucceeded) {
    t24Activities.confirmReservation(tryResult.getReservationId());
}

// Cancel Phase: Release reservation
else {
    t24Activities.cancelReservation(tryResult.getReservationId());
}
```

### Pattern 3: Outbox for T24 Commands

```java
// Write T24 command to outbox (transactional with business data)
@Transactional
public void submitT24Command(T24Command command) {
    // Save business state
    paymentRepository.save(payment);

    // Save T24 command to outbox
    t24OutboxRepository.save(T24OutboxEntry.builder()
        .commandType(command.getType())
        .payload(objectMapper.writeValueAsString(command))
        .status(OutboxStatus.PENDING)
        .createdAt(Instant.now())
        .build());
}

// Poller sends to T24
@Scheduled(fixedRate = 100)
public void pollT24Outbox() {
    List<T24OutboxEntry> pending = t24OutboxRepository
        .findPendingForUpdate(batchSize);

    for (T24OutboxEntry entry : pending) {
        try {
            t24Gateway.submit(entry.getPayload());
            entry.setStatus(OutboxStatus.SENT);
        } catch (Exception e) {
            entry.incrementRetryCount();
            if (entry.getRetryCount() > maxRetries) {
                entry.setStatus(OutboxStatus.FAILED);
            }
        }
        t24OutboxRepository.save(entry);
    }
}
```

---

## Monitoring and Reconciliation

### Key Metrics

```yaml
metrics:
  t24_transaction_duration_seconds:
    type: histogram
    labels: [operation, status]
    buckets: [1, 5, 10, 30, 60, 120]

  t24_circuit_breaker_state:
    type: gauge
    labels: [state]  # CLOSED, OPEN, HALF_OPEN

  t24_pending_transactions:
    type: gauge
    labels: [type]  # DEBIT, CREDIT, REVERSAL

  t24_compensation_total:
    type: counter
    labels: [operation, result]  # success, failure

  t24_reconciliation_discrepancies:
    type: counter
    labels: [type]  # MISSING_IN_T24, MISSING_IN_SAGA, AMOUNT_MISMATCH
```

### Reconciliation Report

```sql
-- Daily Reconciliation Query
SELECT
    p.payment_id,
    p.amount,
    p.status as saga_status,
    t.transaction_ref,
    t.posting_status as t24_status,
    CASE
        WHEN t.transaction_ref IS NULL THEN 'MISSING_IN_T24'
        WHEN p.payment_id IS NULL THEN 'MISSING_IN_SAGA'
        WHEN p.amount != t.amount THEN 'AMOUNT_MISMATCH'
        WHEN p.status != t24_to_saga_status(t.posting_status) THEN 'STATUS_MISMATCH'
        ELSE 'MATCHED'
    END as reconciliation_status
FROM payments p
FULL OUTER JOIN t24_transactions t
    ON p.t24_transaction_ref = t.transaction_ref
WHERE p.created_at >= CURRENT_DATE - 1
    AND (p.status != 'COMPLETED' OR t.posting_status != 'POSTED'
         OR p.amount != t.amount);
```

---

## Summary

### Pattern Comparison: 2PC vs TCC vs SAGA for T24

```mermaid
flowchart TB
    subgraph Patterns["DISTRIBUTED TRANSACTION PATTERNS"]
        direction LR

        subgraph TwoPC["2PC"]
            PC1["Prepare all"]
            PC2["Commit all"]
            PC3["❌ T24 has no 2PC"]
        end

        subgraph TCC["TCC"]
            TC1["Try (reserve)"]
            TC2["Confirm/Cancel"]
            TC3["❌ T24 postings are final"]
        end

        subgraph SAGA["SAGA"]
            SG1["Execute step"]
            SG2["Compensate if fail"]
            SG3["✓ Works with T24"]
        end
    end
```

| Criteria | 2PC | TCC | SAGA |
|----------|-----|-----|------|
| **T24 Compatibility** | ❌ No XA support | ❌ No tentative state | ✓ Works with OFS |
| **Latency Tolerance** | ❌ Seconds | ❌ Seconds | ✓ Hours/Days |
| **EOD/SOD Handling** | ❌ Blocks | ❌ Times out | ✓ Queues & retries |
| **Failure Recovery** | ❌ Coordinator SPOF | ❌ Coordinator SPOF | ✓ Durable workflow |
| **Resource Locking** | ❌ Holds locks | ❌ Requires tentative locks | ✓ No locks held |
| **Implementation** | ❌ Impossible | ❌ Complex facade needed | ✓ Direct integration |
| **Cancel/Rollback** | Rollback | Cancel tentative | Compensate (reversal) |
| **Verdict for T24** | **Not viable** | **Not recommended** | **Recommended** |

### Why SAGA Pattern is the Right Fit for T24

| Challenge | SAGA Solution | Benefit |
|-----------|---------------|---------|
| Long-running T24 operations | Durable workflow state | No timeout issues |
| T24 batch windows (EOD/SOD) | Queue-based retry | Automatic recovery |
| No distributed transactions | Compensation-based rollback | Consistent outcomes |
| Account-level locking | Customer-hash sharding | No deadlocks |
| Non-idempotent operations | Multi-layer idempotency | Safe retries |
| Partial failures | LIFO compensation stack | Clean rollback |
| Visibility requirements | Workflow history | Full audit trail |

### Architecture Principles Summary

1. **Asynchronous by Default** - Never block on T24 calls
2. **Idempotency at Every Layer** - Safe retries guaranteed
3. **Compensation Over Rollback** - Forward-only transaction design
4. **Fail-Open for Compensations** - Don't compound failures
5. **Account-Level Serialization** - Prevent deadlocks
6. **Eventual Consistency** - Accept with reconciliation
7. **Circuit Breaker Protection** - Protect both systems

---

## Related Documentation

- [TEMPORAL_SCALING_ARCHITECTURE.md](TEMPORAL_SCALING_ARCHITECTURE.md) - Scaling for high throughput
- [Observability.md](Observability.md) - Monitoring and tracing
- [CLAUDE.md](../CLAUDE.md) - Development guide
