## Integration Architecture Design: Insurance Modernization

This design replaces direct database coupling with explicit contracts. We will use an **Event-Driven Architecture (EDA)** for state changes and **Synchronous APIs** for real-time queries.

**Infrastructure Backbone:**
*   **Message Broker:** Apache Kafka (for durability and replayability).
*   **API Gateway:** Kong or AWS API Gateway (for auth, rate limiting, routing).
*   **Service Mesh:** Istio (for mTLS and observability between services).
*   **Schema Registry:** Confluent Schema Registry (to enforce event contract versioning).

---

## Integration 1: Policy Lifecycle $\rightarrow$ Claims Eligibility
**Criticality:** High. A claim cannot exist without an active policy. Legacy relied on foreign key constraints and direct joins.
**Pattern:** **Async Event (State Replication) + Sync API (Validation)**

### 1.1 Architecture Strategy
The Claims Service cannot query the Policy DB. Instead, the Policy Service publishes lifecycle events. The Claims Service maintains a **read-replica snapshot** of relevant policy data (Policy ID, Status, Expiry) to make fast local decisions. For submission, a synchronous check ensures the latest state.

### 1.2 Event Schema (CloudEvents v1.0)
**Topic:** `policy.lifecycle.v1`
**Producer:** Policy Service (.NET 8)
**Consumer:** Claims Service (Node.js)

```json
{
  "specversion": "1.0",
  "type": "com.insurance.policy.status_changed",
  "source": "policy-service",
  "id": "5368a9b2-8d33-413f-9933-298a1c355d12",
  "time": "2023-10-27T10:00:00Z",
  "datacontenttype": "application/json",
  "data": {
    "policyId": "POL-123456",
    "customerId": "CUST-9876",
    "status": "ACTIVE", 
    "previousStatus": "PENDING",
    "effectiveDate": "2023-10-27T00:00:00Z",
    "expiryDate": "2024-10-27T00:00:00Z",
    "coverageLimit": 50000.00,
    "currency": "USD"
  }
}
```

### 1.3 API Contract (Sync Validation)
When a user submits a claim, the Claims Service calls the Policy Service to ensure no race conditions occurred between the event arrival and the submission.

**Endpoint:** `GET /api/v1/policies/{policyId}/validation`
**Response:**
```json
{
  "isValid": true,
  "policyId": "POL-123456",
  "status": "ACTIVE",
  "remainingLimit": 45000.00,
  "blockReason": null
}
```

### 1.4 Data Consistency & Error Handling
*   **Consistency:** **Eventual Consistency** for the local snapshot. **Strong Consistency** for the submission validation API call.
*   **Error Handling (Consumer):**
    *   If Claims Service fails to process the event (e.g., DB lock), Kafka offsets are not committed.
    *   After 3 retries, the message moves to a **Dead Letter Queue (DLQ)**.
    *   An alert triggers a manual reconciliation script.
*   **Code Example (Claims Service Consumer - TypeScript):**

```typescript
// claims-service/src/consumers/policy.consumer.ts
async function handlePolicyEvent(message: KafkaMessage) {
  const event = JSON.parse(message.value.toString());
  
  try {
    await db.transaction(async (tx) => {
      // Upsert policy snapshot for fast local lookup
      await tx.query(
        `INSERT INTO policy_snapshots (policy_id, status, expiry, coverage) 
         VALUES ($1, $2, $3, $4) 
         ON CONFLICT (policy_id) DO UPDATE SET status = $2, expiry = $3`,
        [event.data.policyId, event.data.status, event.data.expiryDate, event.data.coverageLimit]
      );
    });
    await consumer.commitMessage(message);
  } catch (error) {
    logger.error('Failed to sync policy snapshot', error);
    throw error; // Trigger Kafka retry mechanism
  }
}
```

---

## Integration 2: Claim Approval $\rightarrow$ Payment Processing
**Criticality:** Critical. Financial transaction. Legacy had Java batch jobs reading Claims tables directly.
**Pattern:** **Saga Pattern (Choreography) with Compensation**

### 1.1 Architecture Strategy
When a claim is approved, a `ClaimApproved` event is emitted. The Payment Service consumes this and initiates a bank transfer. If the bank transfer fails, the Payment Service emits a `PaymentFailed` event, which the Claims Service consumes to reverse the approval status (Compensation).

### 1.2 Event Schema
**Topic:** `claims.decision.v1`
**Producer:** Claims Service (Node.js)
**Consumer:** Payment Service (Java/Spring)

```json
{
  "specversion": "1.0",
  "type": "com.insurance.claim.approved",
  "source": "claims-service",
  "id": "claim-evt-998877",
  "time": "2023-10-27T11:30:00Z",
  "data": {
    "claimId": "CLM-554433",
    "policyId": "POL-123456",
    "amount": 5000.00,
    "currency": "USD",
    "payeeBankAccount": "****1234",
    "idempotencyKey": "CLM-554433-PAY-001" 
  }
}
```

### 1.3 Data Consistency & Compensation
*   **Consistency:** **Transactional Outbox Pattern**. The Claims Service saves the `ClaimApproved` state and the `Event` in the same DB transaction. A separate relay process publishes the event to Kafka. This guarantees "At-Least-Once" delivery without distributed 2PC.
*   **Idempotency:** The `idempotencyKey` ensures the Payment Service doesn't pay twice if the event is redelivered.
*   **Compensation:**
    1.  Payment Service fails to transfer funds.
    2.  Payment Service publishes `payment.processing_failed`.
    3.  Claims Service consumes event and sets Claim Status to `PAYMENT_FAILED` (requires manual review).
    4.  **Alert:** Operations team notified.

### 1.4 Code Example (Payment Service - Java/Spring Boot)

```java
// payment-service/src/main/java/com/insurance/payment/service/PaymentSagaHandler.java

@Service
public class PaymentSagaHandler {

    @Autowired
    private BankGateway bankGateway;
    
    @Autowired
    private KafkaTemplate<String, PaymentEvent> kafkaTemplate;

    @KafkaListener(topics = "claims.decision.v1", groupId = "payment-service-group")
    public void processClaimApproval(ClaimApprovedEvent event) {
        String idempotencyKey = event.getData().getIdempotencyKey();

        // 1. Check Idempotency
        if (paymentRepository.existsByIdempotencyKey(idempotencyKey)) {
            log.warn("Duplicate payment request received: {}", idempotencyKey);
            return; 
        }

        try {
            // 2. Execute Payment
            bankGateway.transfer(event.getData().getAmount(), event.getData().getPayeeBankAccount());
            
            // 3. Record Success
            paymentRepository.save(new PaymentRecord(idempotencyKey, "SUCCESS"));
            
            // 4. Emit Success Event (for Reporting/BFF)
            kafkaTemplate.send("payment.completed.v1", new PaymentCompletedEvent(event.getData().getClaimId()));
            
        } catch (BankException e) {
            // 5. Compensation Trigger
            paymentRepository.save(new PaymentRecord(idempotencyKey, "FAILED"));
            kafkaTemplate.send("payment.processing_failed.v1", 
                new PaymentFailedEvent(event.getData().getClaimId(), e.getMessage()));
        }
    }
}
```

---

## Integration 3: Customer BFF $\rightarrow$ Policy & Claims (Read Aggregation)
**Criticality:** High. Legacy Portal did direct SQL joins across tables. This creates tight coupling and performance bottlenecks.
**Pattern:** **API Composition (BFF) + Caching**

### 1.1 Architecture Strategy
The Customer BFF (Backend for Frontend) acts as an aggregator. It calls the Policy and Claims services in parallel. It does **not** join data in the database; it joins data in memory. For high-load dashboards, the BFF implements a caching layer (Redis) to reduce load on core services.

### 1.2 API Contract (BFF Endpoint)
**Endpoint:** `GET /bff/v1/customer/{customerId}/dashboard`
**Response:**
```json
{
  "customerId": "CUST-9876",
  "policies": [
    {
      "policyId": "POL-123456",
      "type": "AUTO",
      "status": "ACTIVE",
      "expiry": "2024-10-27"
    }
  ],
  "recentClaims": [
    {
      "claimId": "CLM-554433",
      "policyId": "POL-123456",
      "amount": 5000.00,
      "status": "PAYMENT_PROCESSING"
    }
  ]
}
```

### 1.3 Data Consistency
*   **Read-Your-Own-Write:** If a user just updated a policy, the BFF might show stale data from Redis.
    *   *Solution:* The BFF checks a `version` header or bypasses cache if a `Cache-Control: no-store` header is sent from the client (e.g., immediately after a form submission).
*   **Partial Failure:** If the Claims Service is down, the BFF should not fail the whole request.
    *   *Solution:* **Circuit Breaker**. Return policy data with a `claims: null` and a warning message.

### 1.4 Code Example (BFF - Node.js/TypeScript)

```typescript
// customer-bff/src/routes/dashboard.ts
import { getPolicySummary } from '../clients/policy-service';
import { getClaimsHistory } from '../clients/claims-service';
import { cache } from '../middleware/redis';

export const getDashboard = async (req: Request, res: Response) => {
  const { customerId } = req.params;
  const cacheKey = `dash:${customerId}`;

  // 1. Check Cache
  const cached = await cache.get(cacheKey);
  if (cached) return res.json(JSON.parse(cached));

  try {
    // 2. Parallel Execution (Promise.all)
    const [policies, claims] = await Promise.allSettled([
      getPolicySummary(customerId),
      getClaimsHistory(customerId)
    ]);

    // 3. Handle Partial Failures gracefully
    const dashboardData = {
      customerId,
      policies: policies.status === 'fulfilled' ? policies.value : [],
      claims: claims.status === 'fulfilled' ? claims.value : [],
      warnings: []
    };

    if (policies.status === 'rejected') dashboardData.warnings.push('Policy data unavailable');
    if (claims.status === 'rejected') dashboardData.warnings.push('Claims data unavailable');

    // 4. Cache for 5 minutes
    await cache.set(cacheKey, JSON.stringify(dashboardData), 300);

    res.json(dashboardData);

  } catch (error) {
    // 5. Global Fallback
    res.status(503).json({ error: 'Service temporarily unavailable' });
  }
};
```

---

## Cross-Cutting Concerns

### 1. Security & Authentication
*   **Protocol:** mTLS between all microservices.
*   **Identity:** OAuth2 / OIDC. The BFF holds the user session. Internal services trust the BFF via JWT propagation (`Authorization: Bearer <token>`).
*   **Authorization:** Policies are enforced at the API Gateway (Scope: `read:policies`) and within services (Claim ownership checks).

### 2. Observability
*   **Distributed Tracing:** OpenTelemetry. Every request (HTTP or Kafka) carries a `traceparent` header.
*   **Correlation IDs:** Every Kafka event and HTTP request must have a `X-Correlation-ID`. This allows tracing a Claim from submission $\rightarrow$ Approval $\rightarrow$ Payment across 3 different technology stacks.

### 3. Reporting Service Integration (Note on Target State)
While not one of the top 3 operational integrations, the **Reporting Service (Python)** replaces Crystal Reports.
*   **Pattern:** **CDC (Change Data Capture)**.
*   **Implementation:** Use Debezium to stream DB transaction logs from Policy/Claims DBs into a Data Lake (S3/Snowflake).
*   **Reasoning:** This completely removes the "Direct SQL against all tables" anti-pattern from the legacy state without adding load to the operational microservices.

## Summary of Decoupling

| Legacy Coupling | Modern Integration | Benefit |
| :--- | :--- | :--- |
| **Claims $\rightarrow$ Policy DB** | **Kafka Events + Snapshot Table** | Claims service doesn't crash if Policy DB is locked. |
| **Payment $\rightarrow$ Claims DB** | **Saga (ClaimApproved Event)** | Payment can retry without locking Claim records. |
| **Portal $\rightarrow$ All Tables** | **BFF API Composition** | Security boundary enforced; DB schema changes hidden from frontend. |
| **Reporting $\rightarrow$ Prod DB** | **CDC to Data Lake** | Analytical queries don't slow down transactional processing. |
