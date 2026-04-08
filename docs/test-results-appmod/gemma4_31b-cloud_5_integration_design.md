To decouple this monolith, we must shift from **shared-database integration** to **API and Event-driven integration**. The biggest risk in this transition is "distributed monolith" syndrome, where services call each other synchronously in long chains.

We will implement an **Event-Driven Architecture (EDA)** using a message broker (e.g., Kafka or RabbitMQ) and a **BFF (Backend-for-Frontend)** pattern to abstract the complexity from the portal.

---

### Integration 1: Policy $\to$ Claims (Policy Change Notification)
**Scenario:** When a policy is cancelled or updated, the Claims service must know if active claims are still valid.
- **Pattern:** Async Event (Publish/Subscribe)
- **Consistency:** Eventual Consistency
- **Strategy:** The Policy Service publishes a "Domain Event." The Claims service consumes it and updates its local read-model of policy statuses.

**Event Schema (JSON):**
```json
{
  "eventId": "evt_98765",
  "eventType": "PolicyStatusChanged",
  "timestamp": "2023-10-27T10:00:00Z",
  "payload": {
    "policyId": "POL-12345",
    "oldStatus": "ACTIVE",
    "newStatus": "CANCELLED",
    "effectiveDate": "2023-10-27"
  }
}
```

**Error Handling & Compensation:**
- **Idempotency:** Claims Service tracks `eventId` to prevent processing the same update twice.
- **Dead Letter Queue (DLQ):** If the Claims service fails to process the event (e.g., DB down), the message moves to a DLQ for manual replay or automated retry.

---

### Integration 2: Claims $\to$ Payment (Payment Initiation)
**Scenario:** A claim is approved and needs to trigger a payment.
- **Pattern:** Async Event $\to$ Saga Pattern (Choreography)
- **Consistency:** Eventual Consistency via Compensating Transactions.
- **Strategy:** Claims Service emits `ClaimApproved`. Payment Service listens, processes the payment, and emits `PaymentProcessed` or `PaymentFailed`.

**API/Event Contracts:**
1. **ClaimApproved Event (Claims $\to$ Payment):**
```json
{
  "claimId": "CLM-555",
  "amount": 1200.00,
  "currency": "USD",
  "payeeAccountId": "ACC-99",
  "paymentMethod": "ACH"
}
```
2. **PaymentProcessed Event (Payment $\to$ Claims):**
```json
{
  "paymentId": "PAY-777",
  "claimId": "CLM-555",
  "status": "SUCCESS",
  "transactionReference": "BANK-REF-001"
}
```

**Compensation Logic (Code Example - TypeScript/Node.js):**
```typescript
// Claims Service: Handling Payment Failure
async function handlePaymentFailed(event: PaymentFailedEvent) {
  const claim = await claimRepo.findById(event.claimId);
  
  // COMPENSATING ACTION: Revert claim status from 'Pending Payment' back to 'Approved' 
  // or 'Payment Error' to allow manual intervention.
  await claimRepo.updateStatus(claim.id, 'PAYMENT_FAILED');
  await notificationService.alertAdjuster(claim.adjusterId, "Payment failed for claim " + claim.id);
}
```

---

### Integration 3: Customer BFF $\to$ All Services (The Aggregator)
**Scenario:** The portal needs to show a "Customer Dashboard" (Policy details, open claims, payment status).
- **Pattern:** Sync API (REST/GraphQL) + CQRS Query
- **Consistency:** Read-consistency (Aggregated view).
- **Strategy:** The BFF acts as an API Gateway. Instead of calling 3 services for every page load, the Reporting Service maintains a **Read Model** (Materialized View) optimized for the dashboard.

**API Contract (BFF $\to$ Reporting Service):**
`GET /api/v1/customer-summary/{customerId}`

**Response Schema:**
```json
{
  "customerId": "CUST-001",
  "activePolicies": [
    { "policyId": "POL-123", "type": "Auto", "expiry": "2024-01-01" }
  ],
  "recentClaims": [
    { "claimId": "CLM-555", "status": "Paid", "amount": 1200.00 }
  ],
  "outstandingBalance": 0.00
}
```

**Error Handling:**
- **Circuit Breaker:** If the Reporting Service is down, the BFF returns a cached version of the dashboard or a "Service Temporarily Unavailable" message for that specific component rather than crashing the whole page.
- **Timeout:** Strict 2-second timeout for sync calls to prevent thread exhaustion in the Node.js BFF.

---

### Summary Architecture Mapping

| Interaction | Legacy Method | Modern Pattern | Tooling | Logic |
| :--- | :--- | :--- | :--- | :--- |
| **Policy $\to$ Claim** | Direct DB Call | Async Event | Kafka/RabbitMQ | Eventual Consistency |
| **Claim $\to$ Payment** | Batch DB Read | Saga (Choreography) | Message Broker | Compensating Tx |
| **Portal $\to$ Data** | WCF/SQL | BFF $\to$ CQRS Query | REST/gRPC | Optimized Read Model |
| **Reporting** | Crystal $\to$ SQL | Python Service | Materialized View | Async Projection |
