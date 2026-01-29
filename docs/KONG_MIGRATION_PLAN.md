# Migration: Eureka to Kubernetes-Native Service Discovery with Kong Gateway

## Table of Contents

- [Overview](#overview)
- [Current State](#current-state)
- [Target State](#target-state)
- [Phase 1: Remove Eureka Dependencies](#phase-1-remove-eureka-dependencies)
- [Phase 2: Application Configuration Updates](#phase-2-application-configuration-updates)
- [Phase 3: Kubernetes Manifest Updates](#phase-3-kubernetes-manifest-updates)
- [Phase 4: Kong Gateway Configuration](#phase-4-kong-gateway-configuration)
- [Phase 5: EKS Overlay](#phase-5-eks-overlay)
- [Phase 6: Cleanup](#phase-6-cleanup)
- [Files Summary](#files-summary)
- [Verification](#verification)
- [Rollback Plan](#rollback-plan)

---

## Overview

Migrate from Eureka-based service discovery to Kubernetes-native DNS with Kong Gateway as Ingress Controller for AWS EKS deployment.

## Current State

- **Service Discovery**: Eureka Server (port 8761) with all services registering via `spring-cloud-starter-netflix-eureka-client`
- **Feign Clients**: Name-based discovery (`@FeignClient(name = "order-service")`)
- **Ingress**: NGINX Ingress Controller
- **K8s Manifests**: `k8s/base/` with Kustomize

## Target State

- **Service Discovery**: Kubernetes DNS + Spring Cloud Kubernetes LoadBalancer
- **Feign Clients**: Profile-based URL resolution (direct URLs for local, K8s DNS for cluster)
- **Ingress**: Kong Gateway Ingress Controller with plugins
- **Deployment**: AWS EKS with Kong + ALB

---

## Phase 1: Remove Eureka Dependencies

### 1.1 Parent POM (`pom.xml`)

Remove eureka-server from modules:
```xml
<!-- Remove: <module>eureka-server</module> -->
```

### 1.2 Service POMs - Remove Eureka Client

**Files to modify:**
- `order-service/pom.xml`
- `inventory-service/pom.xml`
- `payment-gateway-service/pom.xml`
- `payment-saga-orchestrator/pom.xml`

Remove from each:
```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
</dependency>
```

### 1.3 Orchestrator POM - Add Spring Cloud Kubernetes

**File:** `payment-saga-orchestrator/pom.xml`

Add:
```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-kubernetes-client</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-kubernetes-client-loadbalancer</artifactId>
</dependency>
```

---

## Phase 2: Application Configuration Updates

### 2.1 Orchestrator application.yml

**File:** `payment-saga-orchestrator/src/main/resources/application.yml`

**Remove** entire `eureka:` block (lines 27-36)

**Add** profile-based configuration:

```yaml
---
# Local Development Profile
spring:
  config:
    activate:
      on-profile: local
  cloud:
    kubernetes:
      enabled: false
    loadbalancer:
      enabled: false
    openfeign:
      client:
        config:
          order-service:
            url: http://localhost:8081
          inventory-service:
            url: http://localhost:8082
          payment-gateway-service:
            url: http://localhost:8083

---
# Docker Compose Profile
spring:
  config:
    activate:
      on-profile: docker
  cloud:
    kubernetes:
      enabled: false
    openfeign:
      client:
        config:
          order-service:
            url: http://order-service:8081
          inventory-service:
            url: http://inventory-service:8082
          payment-gateway-service:
            url: http://payment-gateway-service:8083

---
# Kubernetes Profile (K8s DNS discovery)
spring:
  config:
    activate:
      on-profile: k8s
  cloud:
    kubernetes:
      enabled: true
      discovery:
        enabled: true
        all-namespaces: false
      loadbalancer:
        enabled: true
        mode: service

---
# EKS Production Profile
spring:
  config:
    activate:
      on-profile: eks
  cloud:
    kubernetes:
      enabled: true
      discovery:
        enabled: true
      loadbalancer:
        enabled: true
        mode: service
```

### 2.2 Update Feign Client Annotations

**Files:**
- `payment-saga-orchestrator/src/main/java/com/payment/saga/client/OrderClient.java`
- `payment-saga-orchestrator/src/main/java/com/payment/saga/client/InventoryClient.java`
- `payment-saga-orchestrator/src/main/java/com/payment/saga/client/PaymentGatewayClient.java`

Update annotation to support conditional URL:
```java
@FeignClient(
    name = "order-service",
    url = "${spring.cloud.openfeign.client.config.order-service.url:}"
)
```

When `url` is empty, Feign uses service discovery; when set, uses direct URL.

### 2.3 Backend Services - Remove Eureka Config

**Files:**
- `order-service/src/main/resources/application.yml`
- `inventory-service/src/main/resources/application.yml`
- `payment-gateway-service/src/main/resources/application.yml`

Remove `eureka:` block from each.

---

## Phase 3: Kubernetes Manifest Updates

### 3.1 Delete Eureka Server Manifest

**Delete:** `k8s/base/eureka-server.yaml`

### 3.2 Update kustomization.yaml

**File:** `k8s/base/kustomization.yaml`

```yaml
resources:
  - namespace.yaml
  - configmap.yaml
  - secret.yaml
  - rbac.yaml              # NEW
  - orchestrator.yaml
  - order-service.yaml
  - inventory-service.yaml
  - payment-gateway-service.yaml
  - kong/                  # NEW (replaces ingress.yaml)
```

### 3.3 Update ConfigMap

**File:** `k8s/base/configmap.yaml`

- Remove: `EUREKA_URI`
- Add: `SPRING_PROFILES_ACTIVE: "k8s"`

### 3.4 Create RBAC for Spring Cloud Kubernetes

**Create:** `k8s/base/rbac.yaml`

ServiceAccount + Role + RoleBinding for service/endpoint discovery.

### 3.5 Update Deployments

Add to each deployment:
- `serviceAccountName: payment-saga-orchestrator` (for orchestrator)
- Environment variable: `SPRING_PROFILES_ACTIVE` from ConfigMap

---

## Phase 4: Kong Gateway Configuration

### 4.1 Create Kong Directory

**Create:** `k8s/base/kong/kustomization.yaml`

### 4.2 Kong Ingress

**Create:** `k8s/base/kong/kong-ingress.yaml`

Routes:
| Path | Service | Port |
|------|---------|------|
| `/api/v1/payments` | payment-saga-orchestrator | 9090 |
| `/api/orders` | order-service | 8081 |
| `/api/inventory` | inventory-service | 8082 |
| `/api/webhooks` | payment-gateway-service | 8083 |
| `/api/payments` | payment-gateway-service | 8083 |

### 4.3 Kong Plugins

**Create:** `k8s/base/kong/kong-plugins.yaml`

Plugins:
- **rate-limiting**: 100/min for payment API, 1000/min global
- **correlation-id**: X-Correlation-ID header injection
- **request-size-limiting**: 10MB max
- **prometheus**: Metrics collection
- **response-transformer**: Security headers

### 4.4 Service Annotations

Add to each K8s Service:
```yaml
annotations:
  konghq.com/plugins: correlation-id,request-size-limiting
```

---

## Phase 5: EKS Overlay

### 5.1 Create EKS Overlay

**Create:** `k8s/overlays/eks/kustomization.yaml`

- Patches for EKS-specific configs (RDS, MSK, ElastiCache endpoints)
- ECR image references
- ALB Ingress annotations
- TLS configuration

### 5.2 AWS Load Balancer Integration

**Create:** `k8s/overlays/eks/aws-load-balancer.yaml`

- ALB annotations for SSL termination
- WAF integration (optional)
- Health check configuration

---

## Phase 6: Cleanup

### 6.1 Delete Eureka Server Module

**Delete entire directory:** `eureka-server/`

### 6.2 Delete Old Ingress

**Delete:** `k8s/base/ingress.yaml` (replaced by Kong)

---

## Files Summary

### Delete
- `eureka-server/` (entire module)
- `k8s/base/eureka-server.yaml`
- `k8s/base/ingress.yaml`

### Modify
| File | Changes |
|------|---------|
| `pom.xml` | Remove eureka-server module |
| `payment-saga-orchestrator/pom.xml` | Remove Eureka, add Spring Cloud K8s |
| `order-service/pom.xml` | Remove Eureka client |
| `inventory-service/pom.xml` | Remove Eureka client |
| `payment-gateway-service/pom.xml` | Remove Eureka client |
| `payment-saga-orchestrator/src/main/resources/application.yml` | Profile-based config |
| `order-service/src/main/resources/application.yml` | Remove Eureka block |
| `inventory-service/src/main/resources/application.yml` | Remove Eureka block |
| `payment-gateway-service/src/main/resources/application.yml` | Remove Eureka block |
| `payment-saga-orchestrator/.../client/OrderClient.java` | Add URL parameter |
| `payment-saga-orchestrator/.../client/InventoryClient.java` | Add URL parameter |
| `payment-saga-orchestrator/.../client/PaymentGatewayClient.java` | Add URL parameter |
| `k8s/base/kustomization.yaml` | Remove eureka, add kong/ |
| `k8s/base/configmap.yaml` | Remove EUREKA_URI, add profiles |
| `k8s/base/orchestrator.yaml` | Add serviceAccount, env |
| `k8s/base/order-service.yaml` | Add Kong annotations, env |
| `k8s/base/inventory-service.yaml` | Add Kong annotations, env |
| `k8s/base/payment-gateway-service.yaml` | Add Kong annotations, env |

### Create
| File | Purpose |
|------|---------|
| `k8s/base/rbac.yaml` | ServiceAccount + RBAC for K8s discovery |
| `k8s/base/kong/kustomization.yaml` | Kong resources |
| `k8s/base/kong/kong-ingress.yaml` | Kong Ingress routing |
| `k8s/base/kong/kong-plugins.yaml` | Rate limiting, correlation ID, etc. |
| `k8s/overlays/eks/kustomization.yaml` | EKS-specific overlay |
| `k8s/overlays/eks/aws-load-balancer.yaml` | ALB configuration |
| `k8s/overlays/eks/configmap-patch.yaml` | AWS service endpoints |

---

## Verification

### 1. Build Verification
```bash
mvn clean compile -pl !eureka-server
mvn test -pl payment-saga-orchestrator
```

### 2. Local Development Test
```bash
# Start services with local profile
java -jar -Dspring.profiles.active=local order-service.jar &
java -jar -Dspring.profiles.active=local inventory-service.jar &
java -jar -Dspring.profiles.active=local payment-gateway-service.jar &
java -jar -Dspring.profiles.active=local payment-saga-orchestrator.jar &

# Test payment flow
curl -X POST http://localhost:9090/api/v1/payments \
  -H "Content-Type: application/json" \
  -d '{"orderId":"ORD-001","customerId":"CUST-001","amount":100}'
```

### 3. Kubernetes Test (Minikube/Kind)
```bash
# Install Kong Ingress Controller
kubectl apply -f https://raw.githubusercontent.com/Kong/kubernetes-ingress-controller/main/deploy/single/all-in-one-dbless.yaml

# Deploy application
kubectl apply -k k8s/base/

# Verify pods
kubectl get pods -n payment-saga

# Check service discovery logs
kubectl logs deployment/payment-saga-orchestrator -n payment-saga | grep -i "discovery"

# Test through Kong
KONG_IP=$(kubectl get svc kong-proxy -n kong -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
curl -H "Host: payment-saga.example.com" http://$KONG_IP/api/v1/payments
```

### 4. EKS Deployment
```bash
# Deploy to EKS
kubectl apply -k k8s/overlays/eks/

# Get ALB DNS
kubectl get ingress -n payment-saga

# Test external access
curl https://payment-saga.example.com/api/v1/payments
```

### 5. Verify Kong Plugins
```bash
# Check rate limiting headers
curl -i https://payment-saga.example.com/api/v1/payments
# Look for: X-RateLimit-Remaining, X-Correlation-ID

# Check Prometheus metrics
curl http://<KONG_ADMIN>:8001/metrics | grep payment
```

---

## Rollback Plan

If issues occur:

1. Restore eureka-server module in parent pom.xml
2. Add back Eureka client dependencies to service POMs
3. Restore eureka sections in application.yml files
4. Restore k8s/base/eureka-server.yaml and ingress.yaml
5. Redeploy: `kubectl apply -k k8s/base/`
