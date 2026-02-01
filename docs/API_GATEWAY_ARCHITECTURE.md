# API Gateway Architecture

This document describes the request handling architecture for the Payment SAGA platform, covering external API gateway routing via Kong and internal service-to-service communication via Istio service mesh.

## Table of Contents

- [Architecture Overview](#architecture-overview)
- [Request Flow](#request-flow)
- [Component Responsibilities](#component-responsibilities)
- [Security Model](#security-model)
- [Service Access Matrix](#service-access-matrix)
- [Kong Gateway Configuration](#kong-gateway-configuration)
- [Istio Service Mesh](#istio-service-mesh)
- [Distributed Tracing](#distributed-tracing)

---

## Architecture Overview

The Payment SAGA platform uses a **layered gateway architecture**:

1. **Kong Gateway** - Handles external client traffic (authentication, rate limiting, security headers)
2. **Istio Service Mesh** - Handles internal service-to-service traffic (mTLS, authorization, observability)

```mermaid
graph TB
    subgraph External["External Zone"]
        Client[External Client]
    end

    subgraph DMZ["DMZ / Edge"]
        Kong[Kong API Gateway]
    end

    subgraph Mesh["Istio Service Mesh (mTLS)"]
        subgraph Orchestration["Orchestration Layer"]
            Orch[Payment Orchestrator<br/>+ Envoy Sidecar]
        end

        subgraph Services["Domain Services"]
            Order[Order Service<br/>+ Envoy Sidecar]
            Inv[Inventory Service<br/>+ Envoy Sidecar]
            Pay[Payment Gateway<br/>+ Envoy Sidecar]
            OB[Open Banking API<br/>+ Envoy Sidecar]
        end

        subgraph Infrastructure["Infrastructure"]
            Temporal[Temporal Server]
            Kafka[Kafka]
            DB[(PostgreSQL DBs)]
        end
    end

    Client -->|HTTPS| Kong
    Kong -->|HTTP| Orch
    Kong -->|HTTP| Order
    Kong -->|HTTP| Inv
    Kong -->|HTTP| Pay
    Kong -->|HTTP| OB

    Orch -.->|mTLS| Order
    Orch -.->|mTLS| Inv
    Orch -.->|mTLS| Pay
    Orch -.->|gRPC| Temporal
    OB -.->|mTLS| Order
    OB -.->|mTLS| Pay

    Order --> DB
    Inv --> DB
    Pay --> DB
    OB --> DB
    Pay --> Kafka

    style Kong fill:#f9f,stroke:#333
    style Orch fill:#bbf,stroke:#333
    style Order fill:#bfb,stroke:#333
    style Inv fill:#bfb,stroke:#333
    style Pay fill:#bfb,stroke:#333
    style OB fill:#fbf,stroke:#333
```

---

## Request Flow

### External Payment Request Flow

```mermaid
sequenceDiagram
    autonumber
    participant Client as External Client
    participant Kong as Kong Gateway
    participant Envoy1 as Orchestrator Envoy
    participant Orch as Orchestrator
    participant Envoy2 as Order Envoy
    participant Order as Order Service
    participant Envoy3 as Inventory Envoy
    participant Inv as Inventory Service
    participant Envoy4 as Payment Envoy
    participant Pay as Payment Gateway
    participant Temporal as Temporal Server

    Client->>Kong: POST /api/v1/payments

    Note over Kong: Kong Gateway Processing
    Kong->>Kong: 1. Inject X-Correlation-ID
    Kong->>Kong: 2. Validate JWT/API Key
    Kong->>Kong: 3. Check ACL Groups
    Kong->>Kong: 4. Apply Rate Limit (100/min)
    Kong->>Kong: 5. Add Security Headers

    Kong->>Envoy1: Forward Request
    Envoy1->>Orch: Local Request (mTLS terminated)

    Orch->>Temporal: Start PaymentSagaWorkflow
    Temporal-->>Orch: Workflow Started

    Note over Orch,Order: Istio mTLS Communication

    rect rgb(230, 245, 255)
        Note over Envoy1,Envoy2: Step 1: Order Validation
        Orch->>Envoy1: Call Order Service
        Envoy1->>Envoy2: mTLS Connection
        Envoy2->>Order: POST /api/orders/validate
        Order-->>Envoy2: ValidationResult
        Envoy2-->>Envoy1: mTLS Response
        Envoy1-->>Orch: ValidationResult
    end

    rect rgb(230, 255, 230)
        Note over Envoy1,Envoy3: Step 2: Inventory Reservation
        Orch->>Envoy1: Call Inventory Service
        Envoy1->>Envoy3: mTLS Connection
        Envoy3->>Inv: POST /api/inventory/reserve
        Inv-->>Envoy3: ReservationResult
        Envoy3-->>Envoy1: mTLS Response
        Envoy1-->>Orch: ReservationResult
    end

    rect rgb(255, 245, 230)
        Note over Envoy1,Envoy4: Step 3: Payment Authorization
        Orch->>Envoy1: Call Payment Gateway
        Envoy1->>Envoy4: mTLS Connection
        Envoy4->>Pay: POST /api/payments/authorize
        Pay-->>Envoy4: AuthorizationResult
        Envoy4-->>Envoy1: mTLS Response
        Envoy1-->>Orch: AuthorizationResult
    end

    Orch-->>Envoy1: Payment Response
    Envoy1-->>Kong: Response
    Kong-->>Client: 200 OK + Security Headers
```

### Webhook Event Flow

```mermaid
sequenceDiagram
    autonumber
    participant PSP as Payment Provider<br/>(Stripe/PayPal)
    participant Kong as Kong Gateway
    participant Envoy as Payment GW Envoy
    participant Pay as Payment Gateway
    participant Kafka as Kafka
    participant Consumer as Webhook Consumer
    participant Temporal as Temporal Server

    PSP->>Kong: POST /api/webhooks/stripe
    Note over Kong: No JWT Auth (Webhook route)
    Kong->>Kong: Inject Correlation ID
    Kong->>Kong: Apply Rate Limit

    Kong->>Envoy: Forward Webhook
    Envoy->>Pay: Webhook Request

    Pay->>Pay: Verify Signature
    Pay->>Pay: Process Webhook
    Pay->>Kafka: Publish to webhook.payment.events
    Pay-->>Envoy: 200 OK
    Envoy-->>Kong: Response
    Kong-->>PSP: 200 OK

    Kafka->>Consumer: Consume Event
    Consumer->>Consumer: Idempotency Check
    Consumer->>Temporal: Signal Workflow
    Temporal-->>Consumer: Signal Accepted
```

---

## Component Responsibilities

### Kong Gateway (Edge)

| Responsibility | Description |
|----------------|-------------|
| **Authentication** | JWT validation via JWKS, API Key validation |
| **Authorization** | ACL group enforcement (payment-processors, admin) |
| **Rate Limiting** | Global (1000/min) and per-API (100/min for payments) |
| **Security Headers** | HSTS, X-Frame-Options, X-Content-Type-Options, X-XSS-Protection |
| **Correlation ID** | UUID generation and propagation |
| **Request Validation** | Size limiting (10MB for payments, 1MB for Open Banking) |
| **Bot Protection** | Block automated tools (curl, wget in production) |
| **Routing** | Path-based routing to backend services |

### Istio Service Mesh (Internal)

| Responsibility | Description |
|----------------|-------------|
| **mTLS Encryption** | All service-to-service traffic encrypted |
| **Authorization Policies** | Fine-grained access control (who can call whom) |
| **Traffic Management** | Retries, timeouts, circuit breaking |
| **Observability** | Distributed tracing, metrics collection |
| **Load Balancing** | Round-robin with health checks |

### Payment Orchestrator

| Responsibility | Description |
|----------------|-------------|
| **Workflow Orchestration** | Temporal workflow execution |
| **Service Coordination** | Calls Order, Inventory, Payment services |
| **State Management** | Spring State Machine for business states |
| **Compensation** | SAGA rollback on failures |
| **Event Publishing** | Domain events via Kafka |

### Domain Services

| Service | Responsibility |
|---------|----------------|
| **Order Service** | Order validation, status management, cancellation |
| **Inventory Service** | Stock reservation, release |
| **Payment Gateway** | Payment authorization, capture, refund, webhooks |
| **Open Banking API** | PSD2/SBV64 compliant APIs, TPP management, consents |

---

## Security Model

### External Traffic Security (Kong)

```mermaid
flowchart LR
    subgraph "External Request"
        A[Client Request]
    end

    subgraph "Kong Gateway"
        B[TLS Termination]
        C[JWT Validation]
        D[ACL Check]
        E[Rate Limiting]
        F[Security Headers]
    end

    subgraph "Backend"
        G[Service]
    end

    A --> B --> C --> D --> E --> F --> G
```

**Authentication Methods:**

| Route Type | Auth Method | Example Routes |
|------------|-------------|----------------|
| Authenticated | JWT + ACL | `/api/v1/payments`, `/api/orders` |
| Webhook | Signature Verification (App-level) | `/api/webhooks/*` |
| Public | None | `/swagger-ui`, `/api-docs` |
| Open Banking | TPP JWT + Consent | `/open-banking/v1/*` |

### Internal Traffic Security (Istio)

```mermaid
flowchart LR
    subgraph "Service A Pod"
        A1[Application]
        A2[Envoy Sidecar]
    end

    subgraph "Istio Control Plane"
        I1[istiod]
        I2[Cert Authority]
    end

    subgraph "Service B Pod"
        B1[Envoy Sidecar]
        B2[Application]
    end

    A1 -->|plaintext| A2
    A2 <-->|mTLS| B1
    B1 -->|plaintext| B2

    I1 -->|Config| A2
    I1 -->|Config| B1
    I2 -->|Certs| A2
    I2 -->|Certs| B1
```

**mTLS Configuration:**

| Setting | Value | Description |
|---------|-------|-------------|
| Mode | `STRICT` | All traffic must be mTLS |
| Certificate Rotation | Automatic | 24-hour rotation by Istio |
| Cipher Suites | TLS 1.3 preferred | Strong encryption |

---

## Service Access Matrix

This matrix defines which services can communicate with each other. Enforced by Istio AuthorizationPolicy.

```
┌─────────────────────┬────────┬────────┬────────┬────────┬──────────┬───────────┐
│ FROM \ TO           │ Order  │ Inv    │ Pay    │ OB-API │ Temporal │ Kafka     │
├─────────────────────┼────────┼────────┼────────┼────────┼──────────┼───────────┤
│ Kong Gateway        │   ✅   │   ✅   │   ✅   │   ✅   │    ❌    │    ❌     │
│ Orchestrator        │   ✅   │   ✅   │   ✅   │   ❌   │    ✅    │    ✅     │
│ Order Service       │   -    │   ❌   │   ❌   │   ❌   │    ❌    │    ✅     │
│ Inventory Service   │   ❌   │   -    │   ❌   │   ❌   │    ❌    │    ✅     │
│ Payment Gateway     │   ❌   │   ❌   │   -    │   ❌   │    ❌    │    ✅     │
│ Open Banking API    │   ✅   │   ❌   │   ✅   │   -    │    ❌    │    ✅     │
└─────────────────────┴────────┴────────┴────────┴────────┴──────────┴───────────┘

Legend: ✅ = Allowed, ❌ = Denied, - = Self
```

**Rationale:**
- **Orchestrator → All Services**: Needs to coordinate the payment saga
- **Open Banking → Order, Payment**: Needs to initiate payments and query orders
- **Domain Services → Kafka**: Event publishing for domain events
- **No cross-domain calls**: Order cannot call Inventory directly (prevents tight coupling)

---

## Kong Gateway Configuration

### Route Categories

| Category | Host | Routes | Authentication |
|----------|------|--------|----------------|
| **Authenticated** | `payment-saga.example.com` | `/api/v1/payments`, `/api/orders`, `/api/inventory`, `/api/payments` | JWT + ACL |
| **Webhooks** | `payment-saga.example.com` | `/api/webhooks/*` | None (signature at app) |
| **Public** | `payment-saga.example.com` | `/swagger-ui`, `/api-docs` | None |
| **Open Banking Tier 1** | `openbanking.paylink.com` | `/open-banking/v1/accounts` | TPP JWT |
| **Open Banking Tier 2** | `openbanking.paylink.com` | `/open-banking/v1/accounts/*/balance`, `*/transactions` | TPP JWT + Consent |
| **Open Banking Tier 3** | `openbanking.paylink.com` | `/open-banking/v1/payments` | TPP JWT + Consent + SCA |

### Plugin Stack

```mermaid
flowchart TB
    subgraph "Global Plugins"
        G1[Global Rate Limiting<br/>1000/min, 10000/hr]
        G2[Prometheus Metrics]
    end

    subgraph "Route Plugins"
        R1[Correlation ID]
        R2[Request Size Limit]
        R3[Security Headers]
        R4[Route Rate Limit<br/>100/min]
        R5[JWT Authentication]
        R6[ACL Authorization]
    end

    G1 --> R1 --> R2 --> R3 --> R4 --> R5 --> R6
```

---

## Istio Service Mesh

### Traffic Flow

```mermaid
flowchart TB
    subgraph "Istio Control Plane"
        Istiod[istiod]
    end

    subgraph "Data Plane"
        subgraph "Pod: Orchestrator"
            E1[Envoy Proxy<br/>:15001 inbound<br/>:15006 outbound]
            App1[Orchestrator App<br/>:9090]
        end

        subgraph "Pod: Order Service"
            E2[Envoy Proxy]
            App2[Order App<br/>:8081]
        end
    end

    Istiod -->|xDS Config| E1
    Istiod -->|xDS Config| E2

    E1 <-->|mTLS| E2
    App1 --> E1
    E2 --> App2
```

### Key Resources

| Resource | Purpose | File |
|----------|---------|------|
| **PeerAuthentication** | Enable mTLS STRICT mode | `peer-authentication.yaml` |
| **AuthorizationPolicy** | Service-to-service ACLs | `authorization-policies.yaml` |
| **DestinationRule** | mTLS client settings | `destination-rules.yaml` |
| **VirtualService** | Routing, retries, timeouts | `virtual-services.yaml` |
| **ServiceEntry** | External service definitions | `service-entries.yaml` |

---

## Distributed Tracing

### Trace Propagation

```mermaid
sequenceDiagram
    participant Client
    participant Kong
    participant Envoy1 as Envoy (Orch)
    participant App1 as Orchestrator
    participant Envoy2 as Envoy (Order)
    participant App2 as Order Service
    participant Zipkin

    Client->>Kong: Request
    Kong->>Kong: Generate X-Correlation-ID
    Kong->>Envoy1: X-Correlation-ID: abc-123

    Envoy1->>Envoy1: Generate B3 Headers
    Note over Envoy1: X-B3-TraceId: abc123<br/>X-B3-SpanId: span1

    Envoy1->>App1: Request + B3 Headers
    App1->>App1: Log with Correlation ID

    App1->>Envoy1: Call Order Service
    Envoy1->>Envoy2: Propagate B3 Headers
    Note over Envoy1,Envoy2: X-B3-TraceId: abc123<br/>X-B3-SpanId: span2<br/>X-B3-ParentSpanId: span1

    Envoy2->>App2: Request + B3 Headers
    App2->>App2: Log with Correlation ID

    Envoy1->>Zipkin: Report Span
    Envoy2->>Zipkin: Report Span
```

### Header Propagation

| Header | Source | Purpose |
|--------|--------|---------|
| `X-Correlation-ID` | Kong | Business correlation ID |
| `X-B3-TraceId` | Istio/App | Distributed trace ID |
| `X-B3-SpanId` | Istio/App | Current span ID |
| `X-B3-ParentSpanId` | Istio/App | Parent span ID |
| `X-B3-Sampled` | Istio/App | Sampling decision |

### Observability Stack

| Component | Purpose | Endpoint |
|-----------|---------|----------|
| **Zipkin** | Distributed tracing | http://localhost:9411 |
| **Prometheus** | Metrics collection | http://localhost:9090 |
| **Grafana** | Dashboards | http://localhost:3000 |
| **Kiali** | Service mesh visualization | http://localhost:20001 |

---

## Configuration Files

| File | Location | Description |
|------|----------|-------------|
| Kong Ingress | `k8s/base/kong/kong-ingress.yaml` | Route definitions |
| Kong Plugins | `k8s/base/kong/kong-plugins.yaml` | Plugin configurations |
| Istio PeerAuth | `k8s/base/istio/peer-authentication.yaml` | mTLS settings |
| Istio AuthZ | `k8s/base/istio/authorization-policies.yaml` | Access control |
| Istio DestRule | `k8s/base/istio/destination-rules.yaml` | Traffic policies |
| Istio VirtualSvc | `k8s/base/istio/virtual-services.yaml` | Routing rules |
| Istio ServiceEntry | `k8s/base/istio/service-entries.yaml` | External services |

---

## Related Documentation

- [Security Architecture](./Security.md) - Overall security design
- [Kong Migration Plan](./KONG_MIGRATION_PLAN.md) - Migration from Eureka
- [Temporal Scaling Architecture](./TEMPORAL_SCALING_ARCHITECTURE.md) - Workflow scaling

### Istio Configuration Files

| File | Purpose |
|------|---------|
| `k8s/base/istio/peer-authentication.yaml` | mTLS STRICT mode |
| `k8s/base/istio/authorization-policies.yaml` | Service access control |
| `k8s/base/istio/destination-rules.yaml` | Traffic policies |
| `k8s/base/istio/virtual-services.yaml` | Routing rules |
| `k8s/base/istio/service-entries.yaml` | External services |
