# Payment Product Funnel System - Implementation Plan

## Table of Contents

- [Overview](#overview)
- [Supported Payment Products](#supported-payment-products)
- [Phase 1: Domain Model & Database (Week 1-2)](#phase-1-domain-model--database-week-1-2)
- [Phase 2: Configuration & Fee Engine (Week 2-3)](#phase-2-configuration--fee-engine-week-2-3)
- [Phase 3: Product Router Enhancement (Week 3-4)](#phase-3-product-router-enhancement-week-3-4)
- [Phase 4: Workflow Factory & Step Registry (Week 4-5)](#phase-4-workflow-factory--step-registry-week-4-5)
- [Phase 5: State Machine Extensions (Week 5-6)](#phase-5-state-machine-extensions-week-5-6)
- [Phase 6: Product-Specific Activities (Week 6-8)](#phase-6-product-specific-activities-week-6-8)
- [Phase 7: Multi-Source Payment Orchestration (Week 8-9)](#phase-7-multi-source-payment-orchestration-week-8-9)
- [Phase 8: Async Confirmation Handling (Week 9-10)](#phase-8-async-confirmation-handling-week-9-10)
- [Configuration Schema (application.yml)](#configuration-schema-applicationyml)
- [Critical Files to Modify](#critical-files-to-modify)
- [Implementation Order](#implementation-order)
- [Verification](#verification)
- [Performance Considerations](#performance-considerations)

---

## Overview

Implement a highly configurable Payment Product Funnel system supporting 10 payment types with configuration-driven workflows, fee calculation engine, and rule-based routing.

## Supported Payment Products

| Product | Category | Key Characteristics |
|---------|----------|---------------------|
| Bank Transfer | FUNDS_TRANSFER | ACH/Wire/SEPA, async confirmation (1-3 days) |
| Card Payment | CARD | Existing flow enhanced with fees |
| Points & Loyalty | ALTERNATIVE | Instant redemption, no inventory |
| Loan Disbursement | LENDING | Heavy compliance (KYC/AML) |
| Loan Settlement | LENDING | Links to loan system |
| Investment Trading | TRADING | T+2 settlement |
| Bill Payment | RECURRING | Scheduling support |
| Multi-Source | COMPOSITE | Split across multiple sources (child workflows) |
| Crypto | CRYPTO | Blockchain confirmations |
| Merchant/Affiliate | B2B | Custom fees, payout reconciliation |

---

## Phase 1: Domain Model & Database (Week 1-2)

### New Files to Create

**Enums in `payment-saga-common`:**

| File | Purpose |
|------|---------|
| `domain/PaymentProductType.java` | 10 product types with category |
| `domain/PaymentCategory.java` | FUNDS_TRANSFER, CARD, ALTERNATIVE, LENDING, TRADING, RECURRING, COMPOSITE, CRYPTO, B2B |

**Extend Existing:**
- `domain/PaymentDetails.java` - Add new PaymentMethod values: ACH_TRANSFER, WIRE_TRANSFER, LOYALTY_POINTS, GIFT_CARD, LOAN_ACCOUNT, BROKERAGE_ACCOUNT, BITCOIN, ETHEREUM, etc.

**Database Migration `V6__payment_product_funnel.sql`:**

```sql
-- Product configurations (steps, validations, channels per product)
CREATE TABLE payment_product_configs (
    product_type VARCHAR(50) PRIMARY KEY,
    enabled BOOLEAN DEFAULT TRUE,
    workflow_steps JSONB NOT NULL,
    required_validations JSONB,
    supported_channels JSONB NOT NULL,
    default_priority VARCHAR(20) DEFAULT 'NORMAL',
    default_timeout_ms BIGINT DEFAULT 300000,
    fee_rule_id VARCHAR(50),
    routing_rule_id VARCHAR(50)
);

-- Fee rules (FLAT, PERCENTAGE, TIERED, HYBRID)
CREATE TABLE fee_rules (
    rule_id VARCHAR(50) PRIMARY KEY,
    product_type VARCHAR(50) NOT NULL,
    merchant_id VARCHAR(50),  -- NULL for global
    fee_type VARCHAR(20) NOT NULL,
    flat_amount DECIMAL(15,4),
    percentage_rate DECIMAL(10,6),
    tier_config JSONB,
    min_fee DECIMAL(15,4),
    max_fee DECIMAL(15,4),
    merchant_absorbs BOOLEAN DEFAULT TRUE
);

-- Routing rules (SpEL expressions)
CREATE TABLE routing_rules (
    rule_id VARCHAR(50) PRIMARY KEY,
    product_type VARCHAR(50) NOT NULL,
    condition_expression TEXT NOT NULL,
    target_channel VARCHAR(50) NOT NULL,
    fallback_channels JSONB DEFAULT '[]',
    weight INT DEFAULT 100,
    enabled BOOLEAN DEFAULT TRUE
);

-- Multi-source payments
CREATE TABLE multi_source_payments (
    payment_id VARCHAR(36) PRIMARY KEY,
    order_id VARCHAR(36) NOT NULL,
    parent_saga_id VARCHAR(36) NOT NULL,
    total_amount DECIMAL(15,2) NOT NULL,
    allocation_strategy VARCHAR(30) NOT NULL
);

CREATE TABLE payment_sources (
    source_id VARCHAR(36) PRIMARY KEY,
    multi_source_payment_id VARCHAR(36) REFERENCES multi_source_payments,
    payment_method VARCHAR(50) NOT NULL,
    allocated_amount DECIMAL(15,2) NOT NULL,
    priority INT NOT NULL,
    child_workflow_id VARCHAR(100)
);

-- Fee audit log
CREATE TABLE calculated_fees_log (
    id BIGSERIAL PRIMARY KEY,
    order_id VARCHAR(36) NOT NULL,
    product_type VARCHAR(50) NOT NULL,
    total_fee DECIMAL(15,4) NOT NULL,
    calculation_breakdown JSONB NOT NULL
);
```

---

## Phase 2: Configuration & Fee Engine (Week 2-3)

### New Files in `payment-saga-orchestrator`

**Configuration Service:**

| File | Purpose |
|------|---------|
| `product/config/ProductConfigService.java` | Load/cache product configurations |
| `product/config/ProductConfigRepository.java` | JPA repository |
| `product/config/ProductConfigEntity.java` | JPA entity |

**Fee Calculation Engine:**

| File | Purpose |
|------|---------|
| `fee/FeeCalculationService.java` | Main interface |
| `fee/FeeCalculationServiceImpl.java` | FLAT/PERCENTAGE/TIERED/HYBRID logic |
| `fee/FeeRule.java` | Domain object |
| `fee/FeeRuleRepository.java` | JPA repository |
| `fee/FeeRuleCache.java` | Redis caching (15min TTL) |
| `fee/FeeCalculationResult.java` | Result DTO |

**Fee Calculation Logic:**
```java
public FeeCalculationResult calculateFees(FeeCalculationRequest request) {
    List<FeeRule> rules = resolveRules(request.getProductType(), request.getMerchantId());
    BigDecimal baseFee = switch (rules.get(0).getFeeType()) {
        case FLAT -> rule.getFlatAmount();
        case PERCENTAGE -> amount.multiply(rule.getPercentageRate());
        case HYBRID -> flatAmount.add(amount.multiply(percentageRate));
        case TIERED -> calculateTieredFee(amount, rule.getTierConfig());
    };
    return applyBounds(baseFee, rules);
}
```

---

## Phase 3: Product Router Enhancement (Week 3-4)

### Enhance `PaymentRouter.java`

**New Files:**

| File | Purpose |
|------|---------|
| `routing/product/ProductRouter.java` | Product-aware routing interface |
| `routing/product/RuleBasedProductRouter.java` | SpEL-based rule evaluation |
| `routing/product/RoutingRule.java` | Rule domain object |
| `routing/product/RoutingRuleRepository.java` | JPA repository |

**Key Enhancement:**
```java
public ProductRoutingResult route(ProductPaymentRequest request) {
    PaymentProductType productType = request.getProductType();
    List<RoutingRule> rules = getRoutingRules(productType);
    RoutingRule matched = evaluateRules(rules, request);  // SpEL evaluation
    PaymentChannel channel = selectHealthyChannel(matched.getTargetChannel(), matched.getFallbackChannels());
    return ProductRoutingResult.builder()
        .productType(productType)
        .channel(channel)
        .priority(calculatePriority(request, productType))
        .taskQueue(priority.getTaskQueue())
        .build();
}
```

---

## Phase 4: Workflow Factory & Step Registry (Week 4-5)

### New Workflow Architecture

**Files:**

| File | Purpose |
|------|---------|
| `product/workflow/ProductWorkflowFactory.java` | Creates product-specific workflows |
| `product/workflow/WorkflowStepRegistry.java` | Loads steps from config |
| `product/workflow/WorkflowStep.java` | Step definition interface |

**Workflow Step Configuration (application.yml):**
```yaml
payment-saga:
  products:
    card:
      steps: [VALIDATE, RESERVE_INVENTORY, AUTHORIZE, CAPTURE, COMPLETE]
      timeout-ms: 300000
    bank-transfer:
      steps: [VALIDATE, VERIFY_SOURCE, VERIFY_DEST, INITIATE, AWAIT_CONFIRMATION]
      timeout-ms: 259200000  # 3 days
    multi-source:
      steps: [VALIDATE, ALLOCATE, PROCESS_SOURCES, AGGREGATE]
      parallel-step: PROCESS_SOURCES
```

---

## Phase 5: State Machine Extensions (Week 5-6)

### Extend `PaymentState.java`

```java
// New product-specific states
VERIFYING_SOURCE(15),      // Bank transfer
VERIFYING_DESTINATION(16),
AWAITING_EXTERNAL(17),     // Async confirmation
CHECKING_BALANCE(20),      // Points/Crypto
REDEEMING(21),             // Points
DISBURSING(25),            // Loan
SETTLING(26),
EXECUTING_TRADE(30),       // Investment
TRADE_SETTLING(31),
SCHEDULING(35),            // Bill payment
ALLOCATING_SOURCES(40),    // Multi-source
PROCESSING_SOURCES(41),
AGGREGATING(42),
AWAITING_BLOCKCHAIN(45);   // Crypto
```

### Extend `PaymentEvent.java`

```java
// Bank Transfer
SOURCE_VERIFIED, DESTINATION_VERIFIED, TRANSFER_INITIATED, TRANSFER_CONFIRMED,
// Points
BALANCE_CHECKED, POINTS_REDEEMED,
// Loan
KYC_PASSED, CREDIT_APPROVED, DISBURSEMENT_COMPLETED,
// Multi-source
SOURCES_ALLOCATED, ALL_SOURCES_COMPLETED, AGGREGATION_COMPLETED,
// Crypto
BLOCKCHAIN_CONFIRMED;
```

---

## Phase 6: Product-Specific Activities (Week 6-8)

### New Activity Interfaces

| File | Product | Key Methods |
|------|---------|-------------|
| `activity/BankTransferActivities.java` | Bank Transfer | verifySourceAccount, verifyDestAccount, initiateTransfer, awaitConfirmation |
| `activity/PointsLoyaltyActivities.java` | Points | checkBalance, redeemPoints |
| `activity/LoanActivities.java` | Loan | performKycCheck, creditCheck, disburse, settlePayment |
| `activity/InvestmentActivities.java` | Investment | executeTradeOrder, awaitSettlement |
| `activity/BillPaymentActivities.java` | Bill Payment | verifyBiller, schedulePayment |
| `activity/MultiSourceActivities.java` | Multi-Source | allocateSources, createChildWorkflows, aggregateResults |
| `activity/CryptoActivities.java` | Crypto | checkWalletBalance, initiateTransfer, awaitBlockchainConfirmation |
| `activity/MerchantPayoutActivities.java` | Merchant | calculatePayout, executePayout, reconcile |

---

## Phase 7: Multi-Source Payment Orchestration (Week 8-9)

### Child Workflow Pattern

**Files:**

| File | Purpose |
|------|---------|
| `multisource/MultiSourcePaymentWorkflow.java` | Parent workflow interface |
| `multisource/MultiSourcePaymentWorkflowImpl.java` | Orchestrates child workflows |
| `multisource/SourceAllocationStrategy.java` | Strategy interface |
| `multisource/PriorityBasedAllocation.java` | Use sources in priority order |
| `multisource/ProportionalAllocation.java` | Split proportionally |
| `multisource/MinimizeFeeAllocation.java` | Optimize for lowest fees |

**Flow:**
```
VALIDATE → ALLOCATE_SOURCES → [PARALLEL: child workflows] → AGGREGATE → COMPLETE
```

---

## Phase 8: Async Confirmation Handling (Week 9-10)

For Bank Transfer, Crypto, Investment:

**Webhook Signal Integration:**
```java
@SignalMethod
void externalTransferConfirmed(TransferConfirmation confirmation);

@SignalMethod
void blockchainConfirmed(BlockchainConfirmation confirmation);
```

**Polling Activity (fallback):**
```java
@ActivityMethod
TransferStatus pollTransferStatus(String transferId, Duration timeout);
```

---

## Configuration Schema (application.yml)

```yaml
payment-saga:
  products:
    card:
      enabled: true
      default-priority: NORMAL
      timeout-ms: 300000
      channels: [STRIPE, ADYEN, PAYPAL]
      fees:
        rule-id: CARD_DEFAULT
    bank-transfer:
      enabled: true
      default-priority: NORMAL
      timeout-ms: 259200000
      channels: [PLAID, STRIPE_ACH]
      fees:
        rule-id: BANK_TRANSFER_DEFAULT
    crypto:
      enabled: true
      default-priority: HIGH
      timeout-ms: 3600000
      confirmations-required: 6
      supported-currencies: [BTC, ETH, USDC]

  fee-rules:
    CARD_DEFAULT:
      type: HYBRID
      percentage: 0.029
      flat-amount: 0.30
    BANK_TRANSFER_DEFAULT:
      type: TIERED
      tiers:
        - max-amount: 1000
          fee: 0.50
        - max-amount: 10000
          fee: 1.00

  routing:
    rules:
      - product: bank-transfer
        condition: "amount > 50000"
        channel: WIRE_TRANSFER
      - product: bank-transfer
        condition: "currency == 'EUR'"
        channel: SEPA
```

---

## Critical Files to Modify

| File | Changes |
|------|---------|
| `payment-saga-common/src/main/.../PaymentState.java` | Add 15+ new states |
| `payment-saga-common/src/main/.../PaymentEvent.java` | Add 20+ new events |
| `payment-saga-common/src/main/.../PaymentDetails.java` | Extend PaymentMethod enum |
| `payment-saga-orchestrator/src/main/.../PaymentRouter.java` | Add product-type routing |
| `payment-saga-orchestrator/src/main/.../PaymentController.java` | Use ProductRouter, dynamic task queue |
| `payment-saga-orchestrator/src/main/.../PaymentStateMachineConfig.java` | Add product-specific transitions |
| `payment-saga-orchestrator/src/main/resources/application.yml` | Product configurations |

---

## Implementation Order

| Week | Phase | Deliverables |
|------|-------|--------------|
| 1-2 | Domain & DB | Enums, entities, V6 migration |
| 2-3 | Fee Engine | FeeCalculationService, caching |
| 3-4 | Routing | RuleBasedProductRouter, SpEL evaluation |
| 4-5 | Workflow Factory | ProductWorkflowFactory, StepRegistry |
| 5-6 | State Machine | New states/events, transitions |
| 6-8 | Activities | Product-specific activity implementations |
| 8-9 | Multi-Source | Child workflow orchestration |
| 9-10 | Async | Webhook signals, polling fallback |

---

## Verification

### 1. Fee Calculation
```bash
# Unit tests
mvn test -Dtest="FeeCalculation*Test"

# Integration test
curl -X POST http://localhost:9090/api/v1/fees/calculate \
  -d '{"productType":"CARD_PAYMENT","amount":100,"merchantId":"M001"}'
```

### 2. Product Routing
```bash
mvn test -Dtest="ProductRouter*Test,RoutingRule*Test"
```

### 3. Multi-Source Payment
```java
// Test child workflow orchestration
@Test
void shouldProcessMultiSourcePayment() {
    ProductPaymentRequest request = createMultiSourceRequest(
        List.of(
            allocation("CREDIT_CARD", 50.00),
            allocation("LOYALTY_POINTS", 30.00),
            allocation("GIFT_CARD", 20.00)
        )
    );
    PaymentResult result = workflow.processPayment(request.getOrderId());
    assertThat(result.isSuccess()).isTrue();
}
```

### 4. Full Test Suite
```bash
mvn test -Dtest="*ProductType*Test,*FeeCalculation*Test,*ProductRouter*Test,*MultiSource*Test"
```

### 5. End-to-End
```bash
# Start infrastructure
docker compose up -d

# Test bank transfer (async)
curl -X POST http://localhost:9090/api/v1/payments \
  -H "Content-Type: application/json" \
  -d '{
    "productType": "BANK_TRANSFER",
    "orderId": "ORD-001",
    "amount": 1000.00,
    "productSpecificData": {
      "sourceAccountId": "ACC-SRC-001",
      "destinationAccountId": "ACC-DST-001"
    }
  }'
```

---

## Performance Considerations

1. **Caching**: Product configs (5min), fee rules (15min), routing rules (5min) in Redis
2. **Parallel Execution**: Fee calculation + routing + validation run concurrently
3. **Workflow History**: Store product data in `product_payment_data` table, pass only orderId
4. **Database Indexes**: Composite indexes on (product_type, merchant_id), partial indexes for enabled rules
