This design focuses on decoupling the monolith by establishing clear bounded contexts. The architecture follows a **Hybrid Integration Pattern**: Synchronous API for real-time validation queries, and Asynchronous Event-Driven Architecture (EDA) for state changes and cross-domain updates to ensure data consistency and resilience.

We will use an **API Gateway** for ingress and an **Event Mesh/Broker** (e.g., Kafka/RabbitMQ) for internal decoupling.

### Critical Integration 1: Claim Submission & Validation
**Interaction:** Claims Service (Node.js) requires Policy data to validate coverage.
**Legacy Behavior:** PL/SQL procedure joins `Claims` and `Policies` tables directly.
**Target Pattern:** **Synchronous API Call (Backend for Frontend / Service-to-Service)**.

In a microservices architecture, we cannot allow cross-service database joins. The Claims Service must ask the Policy Service for policy details. Since the user is waiting for a "Claim Accepted" confirmation, this must be synchronous.

#### API Contract (OpenAPI Spec)
**Policy Service Endpoint:** `GET /api/v1/policies/{policyId}/coverage-status`

```yaml
paths:
  /api/v1/policies/{policyId}/coverage-status:
    get:
      summary: Validates if a policy is active and retrieves coverage limits
      parameters:
        - name: policyId
          in: path
          required: true
          schema:
            type: string
            format: uuid
      responses:
        '200':
          description: Valid Policy
          content:
            application/json:
              schema:
                type: object
                properties:
                  policyId:
                    type: string
                  status:
                    type: string
                    enum: [ACTIVE, LAPSED, CANCELLED]
                  coverageAmount:
                    type: number
                    format: decimal
                  deductible:
                    type: number
                    format: decimal
        '404':
          description: Policy not found
        '410':
          description: Policy Expired
```

#### Implementation (Claims Service - Node.js)
Using a Circuit Breaker pattern to handle Policy Service outages gracefully.

```typescript
// claims-service/src/services/policy-client.service.ts
import axios from 'axios';
import { CircuitBreaker } from '../utils/circuit-breaker'; // Hypothetical wrapper

const policyServiceUrl = process.env.POLICY_SERVICE_URL;
const breaker = new CircuitBreaker({ threshold: 5, timeout: 3000 });

export async function validatePolicyForClaim(policyId: string, claimAmount: number): Promise<boolean> {
  try {
    // Execute within Circuit Breaker
    const response = await breaker.execute(() => 
      axios.get(`${policyServiceUrl}/api/v1/policies/${policyId}/coverage-status`, {
        headers: { 'X-Internal-Request': 'true' } // Internal auth header
      })
    );

    const { status, coverageAmount, deductible } = response.data;

    if (status !== 'ACTIVE') {
      throw new Error('Policy is not active');
    }

    const maxCoverage = coverageAmount - deductible;
    if (claimAmount > maxCoverage) {
      throw new Error('Claim amount exceeds coverage limit');
    }

    return true;
  } catch (error) {
    // Fallback strategy: If Policy Service is down, queue claim for manual review
    if (error instanceof CircuitBreakerError) {
      console.warn('Policy Service unavailable. Queuing for manual review.');
      // Logic to push to 'pending-validation' queue
      return false; 
    }
    throw error;
  }
}
```

#### Data Consistency & Error Handling
*   **Consistency:** **Strong Consistency** (immediate) for the read. The Claims Service must know *now* if the policy is valid.
*   **Error Handling:**
    *   **Policy Not Found (404):** Reject claim immediately.
    *   **Policy Service Down:** The Circuit Breaker opens. The Claims Service accepts the claim but marks it as `PENDING_REVIEW` and creates a task for an admin user. This prevents the user from seeing a generic 500 error.

---

### Critical Integration 2: Payment Execution
**Interaction:** Claims Service approves a claim → Payment Service issues payment.
**Legacy Behavior:** Java batch job polls `Claims` table for status 'APPROVED' and writes bank files.
**Target Pattern:** **Asynchronous Choreography (Domain Event)**.

Polling is resource-intensive and introduces latency. We replace this with an event-driven push. When a claim is approved, the Claims Service emits an event. The Payment Service consumes it.

#### Event Schema (CloudEvents / AsyncAPI)
**Topic:** `domain.claims.events`
**Event Type:** `ClaimApproved`

```json
{
  "specversion": "1.0",
  "type": "com.insurance.claims.events.ClaimApproved",
  "source": "/services/claims-service",
  "id": "A234-1234-1234",
  "time": "2023-10-27T14:00:00Z",
  "datacontenttype": "application/json",
  "data": {
    "claimId": "clm_556677",
    "policyId": "pol_112233",
    "paymentDetails": {
      "payeeName": "John Doe",
      "bankAccount": "****1234", // Tokenized or reference ID
      "routingNumber": "****9876"
    },
    "amount": 1500.00,
    "currency": "USD",
    "idempotencyKey": "clm_556677_v1"
  }
}
```

#### Implementation (Payment Service - Java/Spring Boot)
The Payment Service listens to the topic and processes the payment.

```java
// payment-service/src/main/java/com/insurance/payment/ClaimEventListener.java
import org.springframework.kafka.annotation.KafkaListener;
import org.springframework.stereotype.Service;
import com.fasterxml.jackson.databind.ObjectMapper;

@Service
public class ClaimEventListener {

    private final PaymentProcessor paymentProcessor;
    private final PaymentRepository paymentRepo;
    private final ObjectMapper objectMapper;

    @KafkaListener(topics = "domain.claims.events", groupId = "payment-service-group")
    public void handleClaimApprovedEvent(String message) {
        try {
            // Deserialize
            ClaimApprovedEvent event = objectMapper.readValue(message, ClaimApprovedEvent.class);

            // Idempotency Check - Crucial for exactly-once processing
            if (paymentRepo.existsByClaimId(event.getData().getClaimId())) {
                log.info("Payment already processed for claim {}", event.getData().getClaimId());
                return;
            }

            // Process Payment
            PaymentInstruction instruction = mapToInstruction(event);
            paymentProcessor.initiateTransfer(instruction);

        } catch (Exception e) {
            // Send to Dead Letter Queue (DLQ) for manual inspection
            throw new PaymentProcessingException("Failed to process claim event", e);
        }
    }
}
```

#### Data Consistency & Error Handling
*   **Consistency:** **Eventual Consistency**. The Claims Service marks the claim `APPROVED`. The Payment Service eventually picks it up.
*   **Error Handling:**
    *   **Failure (DLQ):** If the Payment Service fails to process (e.g., bank API down), the message goes to a Dead Letter Queue. An alert triggers ops to replay the message.
    *   **Compensation:** If the payment fails permanently after being approved, the Payment Service publishes a `PaymentFailed` event. The Claims Service listens to this and reverts the claim status to `APPROVED_PAYMENT_FAILED` for manual intervention.

---

### Critical Integration 3: Policy Cancellation Propagation
**Interaction:** Policy Admin cancels a policy → Active Claims must be rejected/suspended.
**Legacy Behavior:** Policy Administration (.NET WCF) writes directly to `Claims` table (updates status).
**Target Pattern:** **Asynchronous Choreography (Domain Event)**.

This is the classic "Write propagation" problem. We cannot allow the Policy Service to write to the Claims DB. We invert control: Policy Service announces the change, Claims Service reacts.

#### Event Schema
**Topic:** `domain.policy.events`
**Event Type:** `PolicyCancelled`

```json
{
  "specversion": "1.0",
  "type": "com.insurance.policy.events.PolicyCancelled",
  "source": "/services/policy-service",
  "id": "B234-5678-9101",
  "time": "2023-10-27T15:30:00Z",
  "data": {
    "policyId": "pol_112233",
    "cancellationReason": "NON_PAYMENT",
    "effectiveDate": "2023-10-27",
    "existingClaimIds": ["clm_556677", "clm_998877"] 
  }
}
```

#### Implementation (Claims Service - Node.js)
The Claims Service consumes the event and updates its own DB.

```typescript
// claims-service/src/handlers/policy-cancelled.handler.ts
import { KafkaConsumer } from '../messaging/kafka';
import { ClaimsRepository } from '../db/repository';

const consumer = new KafkaConsumer('domain.policy.events', 'claims-service-group');

consumer.on('message', async (message) => {
  const event = JSON.parse(message.value.toString());

  if (event.type === 'PolicyCancelled') {
    const { policyId, existingClaimIds } = event.data;

    console.log(`Processing cancellation for Policy ${policyId}`);

    try {
      // Transactional update to local DB
      await ClaimsRepository.transaction(async (trx) => {
        // Find all open claims for this policy (defensive: don't just trust event data)
        const claims = await ClaimsRepository.findOpenClaimsByPolicyId(policyId, trx);

        for (const claim of claims) {
          await ClaimsRepository.updateStatus(
            claim.id, 
            'CANCELLED_POLICY_VOID', 
            trx
          );
          // Optional: Emit ClaimStatusUpdated event for notification service
        }
      });
    } catch (err) {
      console.error('Failed to process policy cancellation', err);
      // Throw to trigger consumer retry logic
      throw err; 
    }
  }
});
```

#### Data Consistency & Error Handling
*   **Consistency:** **Eventual Consistency**. There is a time window where the Policy is cancelled, but the Claim status is still "Open".
    *   *Mitigation:* The Claims Service validation logic (Integration #1) checks Policy status *before* processing. If a claim comes in during propagation, the validation API will reject it.
*   **Error Handling:**
    *   **Out of Order Events:** If `PolicyCancelled` arrives before `ClaimCreated`?
    *   *Solution:* The Claims Service stores the `PolicyId` with a version or state. If `ClaimCreated` happens after `PolicyCancelled`, the Claims Service can check Policy status via the Sync API (Integration #1) again upon creation.

---

### Summary of Architecture Decisions

| Integration | Pattern | Strategy | Why? |
| :--- | :--- | :--- | :--- |
| **Claim Validation** | **Sync API** | Strong Consistency | User waits for immediate validation response. |
| **Payment Execution** | **Async Event** | Eventual Consistency | Decouples financial batch processing; allows retries without locking UI. |
| **Policy Cancellation**| **Async Event** | Eventual Consistency | Decouples domains; prevents distributed transactions across DBs. |
| **Reporting** | **Data Replication** | Event-Carried State Transfer | Removes heavy query load from transactional DBs. |

**Reporting Note:** While not detailed in the "Top 3", the legacy Crystal Reports dependency is handled by **Event Sourcing projections**. The Python Reporting Service subscribes to all domain events (`ClaimCreated`, `PaymentIssued`, `PolicyCancelled`) and maintains a denormalized "Read Model" (e.g., Elasticsearch or Data Warehouse) optimized for SQL-like queries, effectively replicating the legacy reporting capability without hitting the operational databases.
