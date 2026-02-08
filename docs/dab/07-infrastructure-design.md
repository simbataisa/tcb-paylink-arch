# II.6 Infrastructure Design

[< Back to Index](../DAB_Payment_SAGA_Platform.md) | [← Previous: II.5 Integration Design](06-integration-design.md)

---

## EKS Cluster Topology

```mermaid
flowchart TB
    subgraph AWS["AWS CLOUD"]
        subgraph Edge["EDGE"]
            WAF["AWS WAF<br/>+ Shield Standard"]
            ALB["Application Load Balancer<br/>(SSL Termination)"]
        end

        subgraph EKS["EKS CLUSTER (1.28)"]
            subgraph KongNS["kong-system namespace"]
                KongIC["Kong Ingress Controller<br/>(2 replicas)"]
            end

            subgraph IstioNS["istio-system namespace"]
                Istiod["istiod<br/>(3 replicas)"]
            end

            subgraph AppNS["payment-saga namespace"]
                Orch["Orchestrator<br/>(8-50 pods, HPA)"]
                OrderPods["Order Service<br/>(2 replicas)"]
                InvPods["Inventory Service<br/>(2 replicas)"]
                PayPods["Payment Gateway<br/>(2 replicas)"]
                OBPods["Open Banking API<br/>(2-10 pods, HPA)"]
                Connect["Kafka Connect<br/>(Debezium, 1 pod)"]
            end

            subgraph TemporalNS["temporal namespace"]
                TFront["Frontend (3)"]
                THistory["History (4)"]
                TMatching["Matching (3)"]
                TWorker["Worker (2)"]
            end
        end

        subgraph Data["MANAGED DATA SERVICES"]
            Aurora["Aurora PostgreSQL<br/>Serverless v2<br/>(2-64 ACU, Multi-AZ)"]
            MSK["Amazon MSK<br/>(3 brokers, Multi-AZ)"]
            ElastiCache["ElastiCache Redis 7<br/>(3-node cluster, Multi-AZ)"]
            SecretsManager["AWS Secrets Manager<br/>+ External Secrets Operator"]
        end

        subgraph Observability["OBSERVABILITY"]
            Prometheus["Prometheus<br/>:9090"]
            Grafana["Grafana<br/>:3000"]
            Zipkin["Zipkin<br/>:9411"]
        end
    end

    WAF --> ALB --> KongIC
    KongIC --> Orch
    KongIC --> OrderPods
    KongIC --> InvPods
    KongIC --> PayPods
    KongIC --> OBPods

    Orch --> TFront
    Orch --> Aurora
    OrderPods --> Aurora
    InvPods --> Aurora
    PayPods --> Aurora
    OBPods --> Aurora

    PayPods --> Connect --> MSK
    MSK --> Orch

    Orch --> ElastiCache
    SecretsManager -.-> AppNS
```

## Infrastructure Components

| Component | Specification | Replicas / Sizing |
|---|---|---|
| **EKS Cluster** | Kubernetes 1.28, managed control plane | 3 AZ, m5.xlarge nodes |
| **SAGA Orchestrator** | Java 21, Spring Boot 3.2.1, 500m-2000m CPU, 1-2Gi RAM | 8-50 pods (HPA) |
| **Order/Inventory/Payment Services** | Java 21, 100m-500m CPU, 256-512Mi RAM | 2 replicas each |
| **Open Banking API** | Java 21, 200m-1000m CPU, 512Mi-1Gi RAM | 2-10 pods (HPA) |
| **Aurora PostgreSQL** | Serverless v2, Multi-AZ | 2-64 ACU, 4 databases |
| **Amazon MSK** | Kafka 3.6, Multi-AZ | 3 brokers, m5.large |
| **ElastiCache Redis** | Redis 7, Multi-AZ | 3-node cluster, r6g.large |
| **Temporal Cluster** | Frontend 3, History 4, Matching 3, Worker 2 | 12 pods total |
| **Kafka Connect** | Debezium 2.5, 500m-1000m CPU, 512Mi-1Gi RAM | 1 pod |
| **Kong Ingress** | Kong 3.4 | 2 replicas |
| **Istio** | istiod 1.20 + Envoy sidecars | 3 replicas + per-pod |
| **Prometheus** | Time-series metrics | 1 pod |
| **Grafana** | Dashboards, alerting | 1 pod |
| **Zipkin** | Distributed tracing | 1 pod |

## HPA Configuration

| Deployment | Min Replicas | Max Replicas | CPU Target | Memory Target |
|---|---|---|---|---|
| `payment-saga-orchestrator` | 8 | 50 | 70% | 80% |
| `open-banking-api` | 2 | 10 | 70% | 80% |

Scale-up policy: 50% or 4 pods per 60s (whichever is greater). Scale-down policy: 25% per 120s with 300s stabilization.

---

**Previous:** [← II.5 Integration Design](06-integration-design.md) | **Next:** [II.7 Security Design →](08-security-design.md)
