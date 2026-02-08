# Payment SAGA Platform — REST API Reference

> Auto-generated from [`openapi.yaml`](openapi.yaml) (OpenAPI 3.0.3)

[< Back to Integration Design](06-integration-design.md)

---

## Servers

| Environment | URL                     | Service                   |
| ----------- | ----------------------- | ------------------------- |
| Local       | `http://localhost:9090` | Payment SAGA Orchestrator |
| Local       | `http://localhost:8081` | Order Service             |
| Local       | `http://localhost:8082` | Inventory Service         |
| Local       | `http://localhost:8083` | Payment Gateway Service   |
| Local       | `http://localhost:8084` | Open Banking API          |

## Authentication

| Scheme       | Type                         | Description                                                                                                                    |
| ------------ | ---------------------------- | ------------------------------------------------------------------------------------------------------------------------------ |
| `bearerAuth` | HTTP Bearer (JWT)            | JWT token with claims: `sub`, `tpp_id`, `customer_id`, `consent_id`                                                            |
| `oauth2`     | OAuth 2.0 Authorization Code | Scopes: `payment:create/read/cancel`, `consent:create/read/authorize/revoke`, `tpp:register/read/admin`, `openbanking:ais/pis` |

---

## Table of Contents

1. [Payment SAGA](#1-payment-saga)
2. [Audit Trail](#2-audit-trail)
3. [Reconciliation](#3-reconciliation)
4. [Orders](#4-orders)
5. [Inventory](#5-inventory)
6. [Payment Gateway](#6-payment-gateway)
7. [Webhooks](#7-webhooks)
8. [Consent Management](#8-consent-management)
9. [TPP Registration](#9-tpp-registration)
10. [Account Information](#10-account-information)
11. [Payment Initiation](#11-payment-initiation)
12. [NAPAS Integration](#12-napas-integration)
13. [Kong Audit](#13-kong-audit)
14. [Schemas](#14-schemas)

---

## 1. Payment SAGA

Payment workflow orchestration operations (Orchestrator `:9090`)

| Method | Endpoint                               | Summary                           | Auth             |
| ------ | -------------------------------------- | --------------------------------- | ---------------- |
| `POST` | `/api/v1/payments`                     | Initiate a payment SAGA workflow  | `payment:create` |
| `GET`  | `/api/v1/payments/{workflowId}`        | Get payment workflow status       | `payment:read`   |
| `GET`  | `/api/v1/payments/{workflowId}/result` | Get payment result (blocking)     | `payment:read`   |
| `POST` | `/api/v1/payments/{workflowId}/cancel` | Cancel a running payment workflow | `payment:cancel` |
| `GET`  | `/api/v1/payments/health`              | Payment service health check      | Public           |

### POST `/api/v1/payments`

**Initiate a payment SAGA workflow**

- **Operation ID:** `processPayment`
- **Security:** Bearer JWT — scope: `payment:create`

**Responses:**

| Code  | Description                          |
| ----- | ------------------------------------ |
| `202` | Payment workflow initiated           |
| `400` | Invalid request                      |
| `401` | Unauthorized                         |
| `403` | Forbidden - insufficient permissions |

### GET `/api/v1/payments/{workflowId}`

**Get payment workflow status**

- **Operation ID:** `getPaymentStatus`
- **Security:** Bearer JWT — scope: `payment:read`

**Responses:**

| Code  | Description              |
| ----- | ------------------------ |
| `200` | Payment status retrieved |
| `404` | Workflow not found       |

### GET `/api/v1/payments/{workflowId}/result`

**Get payment result (blocking)**

- **Operation ID:** `getPaymentResult`
- **Security:** Bearer JWT — scope: `payment:read`

**Responses:**

| Code  | Description        |
| ----- | ------------------ |
| `200` | Payment result     |
| `404` | Workflow not found |

### POST `/api/v1/payments/{workflowId}/cancel`

**Cancel a running payment workflow**

- **Operation ID:** `cancelPayment`
- **Security:** Bearer JWT — scope: `payment:cancel`

**Responses:**

| Code  | Description            |
| ----- | ---------------------- |
| `200` | Cancellation requested |
| `404` | Workflow not found     |

### GET `/api/v1/payments/health`

**Payment service health check**

- **Operation ID:** `paymentHealth`
- **Security:** None (public)

**Responses:**

| Code  | Description        |
| ----- | ------------------ |
| `200` | Service is healthy |

---

## 2. Audit Trail

Audit trail and compliance reporting (Orchestrator `:9090`)

| Method | Endpoint                         | Summary                                      | Auth       |
| ------ | -------------------------------- | -------------------------------------------- | ---------- |
| `GET`  | `/api/v1/audit/{sagaId}`         | Get audit trail by SAGA ID                   | Bearer JWT |
| `GET`  | `/api/v1/audit/search`           | Search audit records                         | Bearer JWT |
| `GET`  | `/api/v1/audit/export`           | Export audit records                         | Bearer JWT |
| `GET`  | `/api/v1/audit/archival/status`  | Get archival status                          | Bearer JWT |
| `POST` | `/api/v1/audit/archival/trigger` | Trigger manual archival of old audit records | Bearer JWT |

### GET `/api/v1/audit/{sagaId}`

**Get audit trail by SAGA ID**

- **Operation ID:** `getAuditTrail`
- **Security:** Bearer JWT

**Responses:**

| Code  | Description           |
| ----- | --------------------- |
| `200` | Audit trail retrieved |
| `404` | SAGA not found        |

### GET `/api/v1/audit/search`

**Search audit records**

- **Operation ID:** `searchAudit`
- **Security:** Bearer JWT

**Responses:**

| Code  | Description    |
| ----- | -------------- |
| `200` | Search results |

### GET `/api/v1/audit/export`

**Export audit records**

- **Operation ID:** `exportAudit`
- **Security:** Bearer JWT

**Responses:**

| Code  | Description   |
| ----- | ------------- |
| `200` | Exported file |

### GET `/api/v1/audit/archival/status`

**Get archival status**

- **Operation ID:** `getArchivalStatus`
- **Security:** Bearer JWT

**Responses:**

| Code  | Description     |
| ----- | --------------- |
| `200` | Archival status |

### POST `/api/v1/audit/archival/trigger`

**Trigger manual archival of old audit records**

- **Operation ID:** `triggerArchival`
- **Security:** Bearer JWT

**Responses:**

| Code  | Description        |
| ----- | ------------------ |
| `200` | Archival triggered |

---

## 3. Reconciliation

Event reconciliation and reporting (Orchestrator `:9090`)

| Method | Endpoint                                                 | Summary                             | Auth       |
| ------ | -------------------------------------------------------- | ----------------------------------- | ---------- |
| `GET`  | `/api/v1/reconciliation/summary`                         | Get reconciliation summary          | Bearer JWT |
| `GET`  | `/api/v1/reconciliation/latency-report`                  | Get latency report with percentiles | Bearer JWT |
| `GET`  | `/api/v1/reconciliation/events/{eventId}`                | Get event timeline                  | Bearer JWT |
| `GET`  | `/api/v1/reconciliation/discrepancies`                   | List discrepancies                  | Bearer JWT |
| `POST` | `/api/v1/reconciliation/discrepancies/{eventId}/resolve` | Resolve a discrepancy               | Bearer JWT |
| `GET`  | `/api/v1/reconciliation/batches`                         | List reconciliation batches         | Bearer JWT |
| `GET`  | `/api/v1/reconciliation/batches/{batchId}`               | Get batch details                   | Bearer JWT |
| `POST` | `/api/v1/reconciliation/trigger`                         | Trigger manual reconciliation       | Bearer JWT |
| `GET`  | `/api/v1/reconciliation/config`                          | Get reconciliation configuration    | Bearer JWT |
| `GET`  | `/api/v1/reconciliation/health`                          | Reconciliation health check         | Bearer JWT |

### GET `/api/v1/reconciliation/summary`

**Get reconciliation summary**

- **Operation ID:** `getReconciliationSummary`
- **Security:** Bearer JWT

**Responses:**

| Code  | Description       |
| ----- | ----------------- |
| `200` | Summary retrieved |

### GET `/api/v1/reconciliation/latency-report`

**Get latency report with percentiles**

- **Operation ID:** `getLatencyReport`
- **Security:** Bearer JWT

**Responses:**

| Code  | Description    |
| ----- | -------------- |
| `200` | Latency report |

### GET `/api/v1/reconciliation/events/{eventId}`

**Get event timeline**

- **Operation ID:** `getEventTimeline`
- **Security:** Bearer JWT

**Responses:**

| Code  | Description     |
| ----- | --------------- |
| `200` | Event timeline  |
| `404` | Event not found |

### GET `/api/v1/reconciliation/discrepancies`

**List discrepancies**

- **Operation ID:** `getDiscrepancies`
- **Security:** Bearer JWT

**Responses:**

| Code  | Description        |
| ----- | ------------------ |
| `200` | Discrepancies page |

### POST `/api/v1/reconciliation/discrepancies/{eventId}/resolve`

**Resolve a discrepancy**

- **Operation ID:** `resolveDiscrepancy`
- **Security:** Bearer JWT

**Responses:**

| Code  | Description          |
| ----- | -------------------- |
| `200` | Discrepancy resolved |
| `404` | Event not found      |

### GET `/api/v1/reconciliation/batches`

**List reconciliation batches**

- **Operation ID:** `getBatches`
- **Security:** Bearer JWT

**Responses:**

| Code  | Description  |
| ----- | ------------ |
| `200` | Batches page |

### GET `/api/v1/reconciliation/batches/{batchId}`

**Get batch details**

- **Operation ID:** `getBatch`
- **Security:** Bearer JWT

**Responses:**

| Code  | Description     |
| ----- | --------------- |
| `200` | Batch details   |
| `404` | Batch not found |

### POST `/api/v1/reconciliation/trigger`

**Trigger manual reconciliation**

- **Operation ID:** `triggerReconciliation`
- **Security:** Bearer JWT

**Responses:**

| Code  | Description              |
| ----- | ------------------------ |
| `202` | Reconciliation triggered |

### GET `/api/v1/reconciliation/config`

**Get reconciliation configuration**

- **Operation ID:** `getReconciliationConfig`
- **Security:** Bearer JWT

**Responses:**

| Code  | Description   |
| ----- | ------------- |
| `200` | Configuration |

### GET `/api/v1/reconciliation/health`

**Reconciliation health check**

- **Operation ID:** `reconciliationHealth`
- **Security:** Bearer JWT

**Responses:**

| Code  | Description   |
| ----- | ------------- |
| `200` | Health status |

---

## 4. Orders

Order management (Order Service `:8081`)

| Method | Endpoint                       | Summary             | Auth       |
| ------ | ------------------------------ | ------------------- | ---------- |
| `POST` | `/api/orders/validate`         | Validate an order   | Bearer JWT |
| `PUT`  | `/api/orders/{orderId}/status` | Update order status | Bearer JWT |
| `POST` | `/api/orders/{orderId}/cancel` | Cancel an order     | Bearer JWT |

### POST `/api/orders/validate`

**Validate an order**

- **Operation ID:** `validateOrder`
- **Security:** Bearer JWT

**Responses:**

| Code  | Description       |
| ----- | ----------------- |
| `200` | Validation result |
| `400` | Invalid request   |

### PUT `/api/orders/{orderId}/status`

**Update order status**

- **Operation ID:** `updateOrderStatus`
- **Security:** Bearer JWT

**Responses:**

| Code  | Description     |
| ----- | --------------- |
| `200` | Status updated  |
| `404` | Order not found |

### POST `/api/orders/{orderId}/cancel`

**Cancel an order**

- **Operation ID:** `cancelOrder`
- **Security:** Bearer JWT

**Responses:**

| Code  | Description     |
| ----- | --------------- |
| `200` | Order cancelled |
| `404` | Order not found |

---

## 5. Inventory

Inventory reservation management (Inventory Service `:8082`)

| Method | Endpoint                                 | Summary                          | Auth       |
| ------ | ---------------------------------------- | -------------------------------- | ---------- |
| `POST` | `/api/inventory/reserve`                 | Reserve inventory for an order   | Bearer JWT |
| `POST` | `/api/inventory/release/{reservationId}` | Release an inventory reservation | Bearer JWT |

### POST `/api/inventory/reserve`

**Reserve inventory for an order**

- **Operation ID:** `reserveInventory`
- **Security:** Bearer JWT

**Responses:**

| Code  | Description        |
| ----- | ------------------ |
| `200` | Reservation result |
| `400` | Invalid request    |

### POST `/api/inventory/release/{reservationId}`

**Release an inventory reservation**

- **Operation ID:** `releaseInventory`
- **Security:** Bearer JWT

**Responses:**

| Code  | Description           |
| ----- | --------------------- |
| `200` | Reservation released  |
| `404` | Reservation not found |

---

## 6. Payment Gateway

Payment authorization and capture (Payment Gateway `:8083`)

| Method | Endpoint                           | Summary                       | Auth       |
| ------ | ---------------------------------- | ----------------------------- | ---------- |
| `POST` | `/api/payments/authorize`          | Authorize a payment           | Bearer JWT |
| `POST` | `/api/payments/capture/{authId}`   | Capture an authorized payment | Bearer JWT |
| `POST` | `/api/payments/void/{authId}`      | Void an authorization         | Bearer JWT |
| `POST` | `/api/payments/refund/{captureId}` | Refund a captured payment     | Bearer JWT |

### POST `/api/payments/authorize`

**Authorize a payment**

- **Operation ID:** `authorizePayment`
- **Security:** Bearer JWT

**Responses:**

| Code  | Description             |
| ----- | ----------------------- |
| `200` | Authorization result    |
| `400` | Invalid payment details |
| `402` | Payment declined        |

### POST `/api/payments/capture/{authId}`

**Capture an authorized payment**

- **Operation ID:** `capturePayment`
- **Security:** Bearer JWT

**Responses:**

| Code  | Description                              |
| ----- | ---------------------------------------- |
| `200` | Capture result                           |
| `404` | Authorization not found                  |
| `409` | Authorization already captured or voided |

### POST `/api/payments/void/{authId}`

**Void an authorization**

- **Operation ID:** `voidAuthorization`
- **Security:** Bearer JWT

**Responses:**

| Code  | Description                              |
| ----- | ---------------------------------------- |
| `200` | Authorization voided                     |
| `404` | Authorization not found                  |
| `409` | Authorization already captured or voided |

### POST `/api/payments/refund/{captureId}`

**Refund a captured payment**

- **Operation ID:** `refundPayment`
- **Security:** Bearer JWT

**Responses:**

| Code  | Description              |
| ----- | ------------------------ |
| `200` | Payment refunded         |
| `404` | Capture not found        |
| `409` | Payment already refunded |

---

## 7. Webhooks

External PSP webhook ingestion (Payment Gateway `:8083`)

| Method | Endpoint                  | Summary                              | Auth       |
| ------ | ------------------------- | ------------------------------------ | ---------- |
| `POST` | `/api/webhooks/{channel}` | Receive webhook from payment channel | Bearer JWT |

### POST `/api/webhooks/{channel}`

**Receive webhook from payment channel**

- **Operation ID:** `receiveWebhook`
- **Security:** Bearer JWT

**Responses:**

| Code  | Description                           |
| ----- | ------------------------------------- |
| `200` | Webhook processed                     |
| `400` | Invalid webhook payload               |
| `401` | Invalid webhook signature             |
| `409` | Duplicate webhook (already processed) |

---

## 8. Consent Management

Customer consent for Open Banking (Open Banking `:8084`)

| Method   | Endpoint                                          | Summary                       | Auth                |
| -------- | ------------------------------------------------- | ----------------------------- | ------------------- |
| `POST`   | `/open-banking/v1/consents`                       | Request customer consent      | `consent:create`    |
| `GET`    | `/open-banking/v1/consents`                       | List customer consents        | `consent:read`      |
| `GET`    | `/open-banking/v1/consents/{consentId}`           | Get consent details           | `consent:read`      |
| `DELETE` | `/open-banking/v1/consents/{consentId}`           | Revoke consent                | `consent:revoke`    |
| `POST`   | `/open-banking/v1/consents/{consentId}/authorize` | Authorize consent (after SCA) | `consent:authorize` |

### POST `/open-banking/v1/consents`

**Request customer consent**

- **Operation ID:** `requestConsent`
- **Security:** Bearer JWT — scope: `consent:create`

**Responses:**

| Code  | Description                                       |
| ----- | ------------------------------------------------- |
| `201` | Consent created (awaiting customer authorization) |
| `400` | Invalid request                                   |
| `401` | Unauthorized                                      |
| `403` | TPP not active                                    |

### GET `/open-banking/v1/consents`

**List customer consents**

- **Operation ID:** `getCustomerConsents`
- **Security:** Bearer JWT — scope: `consent:read`

**Responses:**

| Code  | Description       |
| ----- | ----------------- |
| `200` | Customer consents |

### GET `/open-banking/v1/consents/{consentId}`

**Get consent details**

- **Operation ID:** `getConsent`
- **Security:** Bearer JWT — scope: `consent:read`

**Responses:**

| Code  | Description       |
| ----- | ----------------- |
| `200` | Consent details   |
| `404` | Consent not found |

### DELETE `/open-banking/v1/consents/{consentId}`

**Revoke consent**

- **Operation ID:** `revokeConsent`
- **Security:** Bearer JWT — scope: `consent:revoke`

**Responses:**

| Code  | Description                           |
| ----- | ------------------------------------- |
| `204` | Consent revoked                       |
| `403` | Not authorized to revoke this consent |
| `404` | Consent not found                     |

### POST `/open-banking/v1/consents/{consentId}/authorize`

**Authorize consent (after SCA)**

- **Operation ID:** `authorizeConsent`
- **Security:** Bearer JWT — scope: `consent:authorize`

**Responses:**

| Code  | Description                |
| ----- | -------------------------- |
| `200` | Consent authorized         |
| `400` | Invalid consent state      |
| `403` | Not the consent's customer |
| `404` | Consent not found          |

---

## 9. TPP Registration

Third Party Provider management (Open Banking `:8084`)

| Method | Endpoint                                    | Summary                         | Auth              |
| ------ | ------------------------------------------- | ------------------------------- | ----------------- |
| `POST` | `/open-banking/v1/tpp/register`             | Register a Third Party Provider | `tpp:register`    |
| `GET`  | `/open-banking/v1/tpp/{tppId}`              | Get TPP details                 | `tpp:read`        |
| `PUT`  | `/open-banking/v1/tpp/{tppId}/activate`     | Activate a TPP                  | `tpp:admin`       |
| `PUT`  | `/open-banking/v1/tpp/{tppId}/suspend`      | Suspend a TPP                   | `tpp:admin`       |
| `POST` | `/open-banking/v1/tpp/{tppId}/credentials`  | Rotate API credentials          | `tpp:credentials` |
| `POST` | `/open-banking/v1/tpp/{tppId}/tiers/{tier}` | Grant API tier to TPP           | `tpp:admin`       |
| `GET`  | `/open-banking/v1/tpp`                      | List active TPPs                | `tpp:admin`       |

### POST `/open-banking/v1/tpp/register`

**Register a Third Party Provider**

- **Operation ID:** `registerTpp`
- **Security:** Bearer JWT — scope: `tpp:register`

**Responses:**

| Code  | Description                              |
| ----- | ---------------------------------------- |
| `201` | TPP registered                           |
| `400` | Invalid request or duplicate SBV license |
| `401` | Unauthorized                             |

### GET `/open-banking/v1/tpp/{tppId}`

**Get TPP details**

- **Operation ID:** `getTpp`
- **Security:** Bearer JWT — scope: `tpp:read`

**Responses:**

| Code  | Description   |
| ----- | ------------- |
| `200` | TPP details   |
| `404` | TPP not found |

### PUT `/open-banking/v1/tpp/{tppId}/activate`

**Activate a TPP**

- **Operation ID:** `activateTpp`
- **Security:** Bearer JWT — scope: `tpp:admin`

**Responses:**

| Code  | Description     |
| ----- | --------------- |
| `200` | TPP activated   |
| `400` | License expired |
| `404` | TPP not found   |

### PUT `/open-banking/v1/tpp/{tppId}/suspend`

**Suspend a TPP**

- **Operation ID:** `suspendTpp`
- **Security:** Bearer JWT — scope: `tpp:admin`

**Responses:**

| Code  | Description   |
| ----- | ------------- |
| `200` | TPP suspended |
| `404` | TPP not found |

### POST `/open-banking/v1/tpp/{tppId}/credentials`

**Rotate API credentials**

- **Operation ID:** `rotateCredentials`
- **Security:** Bearer JWT — scope: `tpp:credentials`

**Responses:**

| Code  | Description     |
| ----- | --------------- |
| `200` | New credentials |
| `404` | TPP not found   |

### POST `/open-banking/v1/tpp/{tppId}/tiers/{tier}`

**Grant API tier to TPP**

- **Operation ID:** `grantTier`
- **Security:** Bearer JWT — scope: `tpp:admin`

**Responses:**

| Code  | Description   |
| ----- | ------------- |
| `200` | Tier granted  |
| `404` | TPP not found |

### GET `/open-banking/v1/tpp`

**List active TPPs**

- **Operation ID:** `getActiveTpps`
- **Security:** Bearer JWT — scope: `tpp:admin`

**Responses:**

| Code  | Description |
| ----- | ----------- |
| `200` | Active TPPs |

---

## 10. Account Information

Tier 1 & 2 Account Information APIs (Open Banking `:8084`)

| Method | Endpoint                                             | Summary                           | Auth              |
| ------ | ---------------------------------------------------- | --------------------------------- | ----------------- |
| `GET`  | `/open-banking/v1/accounts`                          | List customer accounts (Tier 1)   | `openbanking:ais` |
| `GET`  | `/open-banking/v1/accounts/{accountId}`              | Get account details (Tier 1)      | `openbanking:ais` |
| `GET`  | `/open-banking/v1/accounts/{accountId}/balance`      | Get account balance (Tier 2)      | `openbanking:ais` |
| `GET`  | `/open-banking/v1/accounts/{accountId}/transactions` | Get account transactions (Tier 2) | `openbanking:ais` |

### GET `/open-banking/v1/accounts`

**List customer accounts (Tier 1)**

- **Operation ID:** `listAccounts`
- **Security:** Bearer JWT — scope: `openbanking:ais`

**Responses:**

| Code  | Description                  |
| ----- | ---------------------------- |
| `200` | Customer accounts            |
| `401` | Unauthorized                 |
| `403` | Not authorized for this tier |

### GET `/open-banking/v1/accounts/{accountId}`

**Get account details (Tier 1)**

- **Operation ID:** `getAccount`
- **Security:** Bearer JWT — scope: `openbanking:ais`

**Responses:**

| Code  | Description       |
| ----- | ----------------- |
| `200` | Account details   |
| `404` | Account not found |

### GET `/open-banking/v1/accounts/{accountId}/balance`

**Get account balance (Tier 2)**

- **Operation ID:** `getBalance`
- **Security:** Bearer JWT — scope: `openbanking:ais`

**Responses:**

| Code  | Description                         |
| ----- | ----------------------------------- |
| `200` | Account balance                     |
| `403` | Consent required for balance access |
| `404` | Account not found                   |

### GET `/open-banking/v1/accounts/{accountId}/transactions`

**Get account transactions (Tier 2)**

- **Operation ID:** `getTransactions`
- **Security:** Bearer JWT — scope: `openbanking:ais`

**Responses:**

| Code  | Description                             |
| ----- | --------------------------------------- |
| `200` | Transactions                            |
| `403` | Consent required for transaction access |
| `404` | Account not found                       |

---

## 11. Payment Initiation

Tier 3 Payment Initiation Services (Open Banking `:8084`)

| Method | Endpoint                                        | Summary                   | Auth              |
| ------ | ----------------------------------------------- | ------------------------- | ----------------- |
| `POST` | `/open-banking/v1/payments`                     | Initiate payment (Tier 3) | `openbanking:pis` |
| `GET`  | `/open-banking/v1/payments/{paymentId}`         | Get payment status        | `openbanking:pis` |
| `POST` | `/open-banking/v1/payments/{paymentId}/confirm` | Confirm payment after SCA | `openbanking:pis` |

### POST `/open-banking/v1/payments`

**Initiate payment (Tier 3)**

- **Operation ID:** `initiatePayment`
- **Security:** Bearer JWT — scope: `openbanking:pis`

**Responses:**

| Code  | Description                             |
| ----- | --------------------------------------- |
| `201` | Payment initiated (awaiting SCA)        |
| `400` | Invalid payment request                 |
| `403` | Consent required for payment initiation |

### GET `/open-banking/v1/payments/{paymentId}`

**Get payment status**

- **Operation ID:** `getOBPaymentStatus`
- **Security:** Bearer JWT — scope: `openbanking:pis`

**Responses:**

| Code  | Description       |
| ----- | ----------------- |
| `200` | Payment status    |
| `404` | Payment not found |

### POST `/open-banking/v1/payments/{paymentId}/confirm`

**Confirm payment after SCA**

- **Operation ID:** `confirmPayment`
- **Security:** Bearer JWT — scope: `openbanking:pis`

**Responses:**

| Code  | Description                              |
| ----- | ---------------------------------------- |
| `200` | Payment confirmed                        |
| `400` | Invalid SCA or payment already processed |
| `404` | Payment not found                        |

---

## 12. NAPAS Integration

Internal ISO 20022 NAPAS integration (Open Banking `:8084`)

| Method | Endpoint                           | Summary                                                | Auth       |
| ------ | ---------------------------------- | ------------------------------------------------------ | ---------- |
| `POST` | `/internal/napas/pain001/generate` | Generate ISO 20022 pain.001 Credit Transfer Initiation | Bearer JWT |
| `POST` | `/internal/napas/pain002/receive`  | Receive ISO 20022 pain.002 Payment Status Report       | Bearer JWT |
| `POST` | `/internal/napas/pain002/generate` | Generate ISO 20022 pain.002 response                   | Bearer JWT |
| `POST` | `/internal/napas/pain001/validate` | Validate pain.001 message                              | Bearer JWT |

### POST `/internal/napas/pain001/generate`

**Generate ISO 20022 pain.001 Credit Transfer Initiation**

- **Operation ID:** `generatePain001`
- **Security:** Bearer JWT

**Responses:**

| Code  | Description            |
| ----- | ---------------------- |
| `200` | pain.001 XML generated |

### POST `/internal/napas/pain002/receive`

**Receive ISO 20022 pain.002 Payment Status Report**

- **Operation ID:** `receivePain002`
- **Security:** Bearer JWT

**Responses:**

| Code  | Description             |
| ----- | ----------------------- |
| `200` | Status report processed |
| `400` | Invalid XML             |

### POST `/internal/napas/pain002/generate`

**Generate ISO 20022 pain.002 response**

- **Operation ID:** `generatePain002`
- **Security:** Bearer JWT

**Responses:**

| Code  | Description            |
| ----- | ---------------------- |
| `200` | pain.002 XML generated |

### POST `/internal/napas/pain001/validate`

**Validate pain.001 message**

- **Operation ID:** `validatePain001`
- **Security:** Bearer JWT

**Responses:**

| Code  | Description       |
| ----- | ----------------- |
| `200` | Validation result |

---

## 13. Kong Audit

Internal Kong Gateway audit logs (Open Banking `:8084`)

| Method | Endpoint               | Summary                        | Auth       |
| ------ | ---------------------- | ------------------------------ | ---------- |
| `POST` | `/internal/audit/kong` | Receive Kong Gateway audit log | Bearer JWT |

### POST `/internal/audit/kong`

**Receive Kong Gateway audit log**

- **Operation ID:** `receiveKongAuditLog`
- **Security:** Bearer JWT

**Responses:**

| Code  | Description  |
| ----- | ------------ |
| `200` | Log received |

---

## 14. Schemas

Complete schema definitions are in [`openapi.yaml`](openapi.yaml). Summary below.

### Request Schemas

| Schema                        | Required Fields                                                          |
| ----------------------------- | ------------------------------------------------------------------------ |
| **OrderRequest**              | `orderId`, `customerId`, `amount`, `currency`, `items`, `paymentDetails` |
| **OrderItem**                 | `sku`, `price`                                                           |
| **PaymentDetails**            | `paymentMethod`, `amount`, `currency`                                    |
| **CustomerInfo**              | —                                                                        |
| **ShippingAddress**           | —                                                                        |
| **BillingAddress**            | —                                                                        |
| **ThreeDSecureData**          | —                                                                        |
| **ConsentRequest**            | `customerId`, `consentType`, `permissions`                               |
| **TppRegistrationRequest**    | `organizationName`, `sbvLicenseNumber`, `licenseExpiry`                  |
| **PaymentInitiationRequest**  | `debtorAccountId`, `creditorAccountId`, `amount`, `currency`             |
| **ScaConfirmationRequest**    | `scaMethod`                                                              |
| **ResolveDiscrepancyRequest** | `resolutionStatus`                                                       |
| **Pain001Request**            | —                                                                        |
| **Pain002Request**            | —                                                                        |
| **KongLogEntry**              | —                                                                        |

### Response Schemas

| Schema                          | Key Fields                                                                    |
| ------------------------------- | ----------------------------------------------------------------------------- |
| **PaymentResult**               | `workflowId`, `sagaId`, `orderId`, `status`, `workflowState`                  |
| **OrderValidation**             | `validationId`, `orderId`, `valid`, `failureReason`, `checks`                 |
| **ValidationCheck**             | `checkName`, `passed`, `message`                                              |
| **OrderUpdate**                 | `orderId`, `previousStatus`, `newStatus`, `success`, `message`                |
| **InventoryReservation**        | `reservationId`, `orderId`, `success`, `failureReason`, `items`               |
| **ReservedItem**                | `sku`, `quantityReserved`, `warehouseId`                                      |
| **PaymentAuth**                 | `authId`, `orderId`, `approved`, `declineReason`, `declineCode`               |
| **PaymentCapture**              | `captureId`, `authId`, `orderId`, `success`, `failureReason`                  |
| **WebhookResponse**             | `status`, `eventId`, `message`                                                |
| **ConsentResponse**             | `consentId`, `customerId`, `tppId`, `consentType`, `status`                   |
| **TppResponse**                 | `tppId`, `organizationName`, `sbvLicenseNumber`, `licenseExpiry`, `status`    |
| **TppCredentials**              | `tppId`, `apiKey`, `apiKeyPrefix`, `warning`                                  |
| **TppRegistrationResult**       | `tpp`, `credentials`                                                          |
| **PaymentInitiationResponse**   | `paymentId`, `consentId`, `status`, `createdAt`, `scaUrl`                     |
| **PaymentStatusResponse**       | `paymentId`, `status`, `createdAt`, `statusUpdateAt`, `reasonCode`            |
| **PaymentConfirmationResponse** | `paymentId`, `status`, `confirmedAt`, `expectedSettlement`                    |
| **AuditTrailResponse**          | `sagaId`, `recordCount`, `records`                                            |
| **AuditSearchResponse**         | `totalRecords`, `page`, `size`, `records`                                     |
| **AuditRecordDto**              | `id`, `sagaId`, `workflowId`, `fromState`, `toState`                          |
| **ArchivalStatusResponse**      | `archivalEnabled`, `thresholdDays`, `pendingArchivalCount`                    |
| **ArchivalResultResponse**      | `recordsArchived`, `message`                                                  |
| **ReconciliationSummaryDto**    | `periodStart`, `periodEnd`, `totalEvents`, `matchedEvents`, `unmatchedEvents` |
| **LatencyReportDto**            | `periodStart`, `periodEnd`, `sampleSize`, `p50LatencyMs`, `p75LatencyMs`      |
| **EventTimelineDto**            | `eventId`, `orderId`, `sagaId`, `eventType`, `sourceService`                  |
| **DiscrepancyDto**              | `id`, `eventId`, `orderId`, `sagaId`, `eventType`                             |
| **PageDiscrepancyDto**          | `content`, `totalElements`, `totalPages`, `size`, `number`                    |
| **ReconciliationBatch**         | `id`, `batchId`, `batchType`, `periodStart`, `periodEnd`                      |
| **PageReconciliationBatch**     | `content`, `totalElements`, `totalPages`, `size`, `number`                    |
| **AccountSummary**              | `accountId`, `accountType`, `currency`, `nickname`, `status`                  |
| **AccountDetail**               | `accountId`, `accountType`, `currency`, `nickname`, `status`                  |
| **AccountBalance**              | `accountId`, `balances`                                                       |
| **TransactionList**             | `accountId`, `transactions`                                                   |
| **ErrorResponse**               | `timestamp`, `status`, `error`, `message`, `path`                             |

### Enumerations

#### `PaymentMethod`

| Value               |
| ------------------- |
| `CREDIT_CARD`       |
| `DEBIT_CARD`        |
| `BANK_TRANSFER`     |
| `ACH_TRANSFER`      |
| `WIRE_TRANSFER`     |
| `SEPA_TRANSFER`     |
| `DIGITAL_WALLET`    |
| `BUY_NOW_PAY_LATER` |
| `LOYALTY_POINTS`    |
| `GIFT_CARD`         |
| `STORE_CREDIT`      |
| `LOAN_ACCOUNT`      |
| `LINE_OF_CREDIT`    |
| `BROKERAGE_ACCOUNT` |
| `MUTUAL_FUND`       |
| `BITCOIN`           |
| `ETHEREUM`          |
| `USDC`              |
| `USDT`              |
| `MERCHANT_CREDIT`   |
| `AFFILIATE_PAYOUT`  |

#### `OrderStatus`

| Value                |
| -------------------- |
| `PENDING`            |
| `VALIDATED`          |
| `PROCESSING`         |
| `PAYMENT_PENDING`    |
| `PAYMENT_AUTHORIZED` |
| `PAYMENT_CAPTURED`   |
| `COMPLETED`          |
| `CANCELLED`          |
| `REFUNDED`           |
| `FAILED`             |

#### `CustomerStatus`

`ACTIVE` | `SUSPENDED` | `PENDING_VERIFICATION`

#### `PaymentResultStatus`

`SUCCESS` | `FAILED` | `CANCELLED` | `COMPENSATED` | `PENDING` | `REQUIRES_INTERVENTION`

#### `WorkflowState` — Temporal workflow lifecycle state

| Value                   |
| ----------------------- |
| `INITIALIZED`           |
| `INITIALIZING`          |
| `RUNNING`               |
| `VALIDATING_ORDER`      |
| `RESERVING_INVENTORY`   |
| `AUTHORIZING_PAYMENT`   |
| `CAPTURING_PAYMENT`     |
| `COMPLETING_ORDER`      |
| `COMPLETED`             |
| `COMPENSATING`          |
| `FAILED`                |
| `CANCELLED`             |
| `REQUIRES_INTERVENTION` |

#### `PaymentState` — Spring State Machine business state

| Value                 |
| --------------------- |
| `PENDING`             |
| `VALIDATING`          |
| `VALIDATED`           |
| `RESERVING`           |
| `RESERVED`            |
| `AUTHORIZING`         |
| `AUTHORIZED`          |
| `CAPTURING`           |
| `CAPTURED`            |
| `COMPLETING`          |
| `COMPLETED`           |
| `COMPENSATING`        |
| `COMPENSATED`         |
| `FAILED`              |
| `COMPENSATION_FAILED` |

#### `ConsentType`

`AIS` | `PIS` | `CBPII`

#### `ConsentStatus`

`AWAITING_AUTHORIZATION` | `AUTHORIZED` | `REVOKED` | `EXPIRED` | `REJECTED`

#### `PermissionType` — Tier 1: ACCOUNTS, PRODUCTS. Tier 2: BALANCES, TRANSACTIONS, etc. Tier 3: PAYMENTS, PAYMENT_STATUS.

| Value                 |
| --------------------- |
| `ACCOUNTS`            |
| `PRODUCTS`            |
| `BALANCES`            |
| `TRANSACTIONS`        |
| `TRANSACTIONS_DETAIL` |
| `STANDING_ORDERS`     |
| `DIRECT_DEBITS`       |
| `BENEFICIARIES`       |
| `PAYMENTS`            |
| `PAYMENT_STATUS`      |

#### `TppStatus`

`PENDING` | `ACTIVE` | `SUSPENDED` | `REVOKED`

#### `ApiTier` — Tier 1: Information Query. Tier 2: Account Info (requires consent). Tier 3: Payment Initiation (requires consent + SCA).

`TIER_1` | `TIER_2` | `TIER_3`

#### `OBPaymentStatus` — Open Banking payment lifecycle status

`REQUIRES_SCA` | `PENDING` | `REJECTED` | `ACCEPTED_SETTLEMENT_IN_PROCESS` | `ACCEPTED_SETTLEMENT_COMPLETED` | `CANCELLED`

#### `ReconciliationStatus`

| Value             |
| ----------------- |
| `PENDING_PUBLISH` |
| `PUBLISHED`       |
| `KAFKA_ACKED`     |
| `CONSUMED`        |
| `PROCESSED`       |
| `COMPLETED`       |
| `TIMEOUT`         |
| `ERROR`           |

#### `DiscrepancyType`

| Value                     |
| ------------------------- |
| `PUBLISH_TIMEOUT`         |
| `KAFKA_DELIVERY_TIMEOUT`  |
| `CONSUME_TIMEOUT`         |
| `PROCESSING_TIMEOUT`      |
| `WORKFLOW_SIGNAL_TIMEOUT` |
| `WORKFLOW_SIGNAL_FAILED`  |
| `WORKFLOW_NOT_FOUND`      |
| `DUPLICATE_EVENT`         |
| `KAFKA_METADATA_MISMATCH` |
| `PAYLOAD_ERROR`           |
| `UNKNOWN`                 |

#### `ResolutionStatus`

`UNRESOLVED` | `RESOLVED` | `IGNORED` | `ESCALATED` | `AUTO_RESOLVED`

#### `BatchType`

`REALTIME` | `HOURLY` | `DAILY` | `MANUAL`

#### `BatchStatus`

`RUNNING` | `COMPLETED` | `FAILED`

---

[< Back to Integration Design](06-integration-design.md) | [OpenAPI YAML Source](openapi.yaml)
