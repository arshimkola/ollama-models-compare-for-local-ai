# Test Specification: Insurance Claims API Modernization (Parity Testing)

**Document Version:** 1.0  
**Objective:** Validate that the modernized Claims API produces behavior identical to the legacy PL/SQL procedure based on extracted business rules.  
**Scope:** Functional parity, data integrity, audit logging, and error handling.  
**Strategy:** Black-box testing with database state verification. All tests assume the legacy system behavior is the source of truth.

---

## 1. Test Environment & Prerequisites

*   **API Endpoint:** `POST /api/v1/claims/{claimId}/process`
*   **Test Data:** A seeded database containing policies, customers, and claims mirroring legacy production data distributions.
*   **Mock Services:** 
    *   Payment Gateway (to verify queueing).
    *   Notification Service (to verify emails/SMS).
    *   Audit Log Service (or direct DB access to `AUDIT_LOGS` table).
*   **User Roles:** Test accounts for Assessors with Levels 1 through 5.

---

## 2. Test Scenarios Grouped by Rule

### 2.1 Claim Status & Policy Validity (Rules 1, 2)
**Objective:** Ensure claims are only processed under valid state conditions.

| Test ID | Preconditions | Input Data | Expected Outcome | Rule Ref |
| :--- | :--- | :--- | :--- | :--- |
| **TS-01** | Claim Status: `SUBMITTED`<br>Policy Status: `ACTIVE` | Action: `APPROVE` | **200 OK**. Claim moves to `APPROVED`. | 1, 2 |
| **TS-02** | Claim Status: `UNDER_REVIEW`<br>Policy Status: `ACTIVE` | Action: `APPROVE` | **200 OK**. Claim moves to `APPROVED`. | 1, 2 |
| **TS-03** | Claim Status: `ESCALATED`<br>Policy Status: `ACTIVE` | Action: `APPROVE` | **200 OK**. Claim moves to `APPROVED`. | 1, 2 |
| **TS-04** | Claim Status: `PENDING_INFO`<br>Policy Status: `ACTIVE` | Action: `APPROVE` | **400 Bad Request**. Error: "Invalid Claim Status". | 1 |
| **TS-05** | Claim Status: `CLOSED`<br>Policy Status: `ACTIVE` | Action: `APPROVE` | **400 Bad Request**. Error: "Invalid Claim Status". | 1 |
| **TS-06** | Claim Status: `SUBMITTED`<br>Policy Status: `LAPSED` | Action: `APPROVE` | **400 Bad Request**. Error: "Policy Not Active". | 2 |
| **TS-07** | Claim Status: `SUBMITTED`<br>Policy Status: `CANCELLED` | Action: `APPROVE` | **400 Bad Request**. Error: "Policy Not Active". | 2 |

**Boundary Values:**
*   Policy Expiry Date = Today 00:00:00 (Should be Active).
*   Policy Expiry Date = Yesterday 23:59:59 (Should be Inactive).

---

### 2.2 Fraud Detection Logic (Rule 3)
**Objective:** Verify fraud score blocking and audit requirements.

| Test ID | Preconditions | Input Data | Expected Outcome | Rule Ref |
| :--- | :--- | :--- | :--- | :--- |
| **TS-08** | Fraud Score: `71`<br>Amount: `$1,000` | Action: `APPROVE` | **403 Forbidden**. Claim remains `SUBMITTED`. Audit Entry Created (Type: `FRAUD_BLOCK`). | 3, 12 |
| **TS-09** | Fraud Score: `70`<br>Amount: `$1,000` | Action: `APPROVE` | **200 OK**. Processing continues (not blocked by fraud). | 3 |
| **TS-10** | Fraud Score: `0`<br>Amount: `$1,000` | Action: `APPROVE` | **200 OK**. Processing continues. | 3 |

**Negative Tests:**
*   Fraud Score NULL (Should default to 0 or throw error depending on legacy behavior; assume 0 for parity).
*   Fraud Score > 100 (Verify system handles out-of-range data gracefully).

---

### 2.3 Auto-Approval Thresholds (Rule 4)
**Objective:** Validate the matrix of Policy Type and Tier against Claim Amount.

| Test ID | Policy Type | Tier | Claim Amount | Assessor Level | Expected Outcome | Rule Ref |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| **TS-11** | PREMIUM | GOLD | $25,000 | 1 | **200 OK** (Auto-Approved). | 4 |
| **TS-12** | PREMIUM | GOLD | $25,001 | 1 | **200 OK** (State: `ESCALATED`). | 4, 5 |
| **TS-13** | PREMIUM | SILVER | $15,000 | 1 | **200 OK** (Auto-Approved). | 4 |
| **TS-14** | PREMIUM | SILVER | $15,001 | 1 | **200 OK** (State: `ESCALATED`). | 4, 5 |
| **TS-15** | STANDARD | PLATINUM | $12,000 | 1 | **200 OK** (Auto-Approved). | 4 |
| **TS-16** | STANDARD | PLATINUM | $12,001 | 1 | **200 OK** (State: `ESCALATED`). | 4, 5 |
| **TS-17** | STANDARD | BRONZE | $8,000 | 1 | **200 OK** (Auto-Approved). | 4 |
| **TS-18** | STANDARD | BRONZE | $8,001 | 1 | **200 OK** (State: `ESCALATED`). | 4, 5 |
| **TS-19** | BASIC | ANY | $3,000 | 1 | **200 OK** (Auto-Approved). | 4 |
| **TS-20** | BASIC | ANY | $3,001 | 1 | **200 OK** (State: `ESCALATED`). | 4, 5 |
| **TS-21** | UNKNOWN | NONE | $1,000 | 1 | **200 OK** (Auto-Approved). | 4 |
| **TS-22** | UNKNOWN | NONE | $1,001 | 1 | **200 OK** (State: `ESCALATED`). | 4, 5 |

**Boundary Values:**
*   Test exact threshold amounts (e.g., $25,000.00 vs $25,000.01).
*   Test currency precision (ensure cents are evaluated correctly).

---

### 2.4 Assessor Levels & Escalation (Rules 5, 6)
**Objective:** Verify manual override capabilities and frequency checks.

| Test ID | Preconditions | Input Data | Expected Outcome | Rule Ref |
| :--- | :--- | :--- | :--- | :--- |
| **TS-23** | Amount > Threshold<br>Assessor Level: `3` | Action: `APPROVE` | **200 OK**. Claim `APPROVED`. | 5 |
| **TS-24** | Amount > Threshold<br>Assessor Level: `2` | Action: `APPROVE` | **200 OK**. Claim `ESCALATED`. | 5 |
| **TS-25** | Claims (12mo): `4`<br>Assessor Level: `2` | Action: `APPROVE` | **200 OK**. Claim `APPROVED`. | 6 |
| **TS-26** | Claims (12mo): `4`<br>Assessor Level: `1` | Action: `APPROVE` | **200 OK**. Claim `ESCALATED`. | 6 |
| **TS-27** | Claims (12mo): `3`<br>Assessor Level: `1` | Action: `APPROVE` | **200 OK**. Claim `APPROVED` (if amount allows). | 6 |

**Boundary Values:**
*   Claims in 12 months = Exactly 3 (Rule does not trigger).
*   Claims in 12 months = Exactly 4 (Rule triggers).
*   Claim date = 365 days ago (Included in 12 months?).
*   Claim date = 366 days ago (Excluded from 12 months?).

---

### 2.5 Priority & SLA Handling (Rule 7)
**Objective:** Verify priority assignment and SLA flagging on old claims.

| Test ID | Preconditions | Input Data | Expected Outcome | Rule Ref |
| :--- | :--- | :--- | :--- | :--- |
| **TS-28** | Claim Filed: `29 days ago` | Action: `APPROVE` | **200 OK**. Priority: `NORMAL`. SLA Flag: `FALSE`. | 7 |
| **TS-29** | Claim Filed: `30 days ago` | Action: `APPROVE` | **200 OK**. Priority: `NORMAL`. SLA Flag: `FALSE`. | 7 |
| **TS-30** | Claim Filed: `31 days ago` | Action: `APPROVE` | **200 OK**. Priority: `HIGH`. SLA Flag: `TRUE`. | 7 |
| **TS-31** | Claim Filed: `31 days ago` | Action: `REJECT` | **200 OK**. Priority: `HIGH`. SLA Flag: `FALSE` (Rule says "on approval"). | 7 |

---

### 2.6 Payment Queuing & Priority (Rules 8, 9)
**Objective:** Validate auto-payment logic and queue prioritization.

| Test ID | Preconditions | Input Data | Expected Outcome | Rule Ref |
| :--- | :--- | :--- | :--- | :--- |
| **TS-32** | Approved<br>Amount: `$5,000`<br>Fraud: `29`<br>Tier: `PLATINUM` | Action: `APPROVE` | **200 OK**. Payment Queued. Queue Priority: `1`. | 8, 9 |
| **TS-33** | Approved<br>Amount: `$5,000`<br>Fraud: `29`<br>Tier: `GOLD` | Action: `APPROVE` | **200 OK**. Payment Queued. Queue Priority: `2`. | 8, 9 |
| **TS-34** | Approved<br>Amount: `$5,001`<br>Fraud: `29` | Action: `APPROVE` | **200 OK**. Payment **NOT** Queued (Manual Pay). | 8 |
| **TS-35** | Approved<br>Amount: `$5,000`<br>Fraud: `30` | Action: `APPROVE` | **200 OK**. Payment **NOT** Queued (Fraud Check). | 8 |
| **TS-36** | Rejected<br>Amount: `$1,000`<br>Fraud: `0` | Action: `REJECT` | **200 OK**. Payment **NOT** Queued. | 8 |

**Boundary Values:**
*   Fraud Score 29 (Queue) vs 30 (No Queue).
*   Amount 5000.00 (Queue) vs 5000.01 (No Queue).

---

### 2.7 Action Specifics (Rules 10, 11)
**Objective:** Validate mandatory fields and side-effects for specific actions.

| Test ID | Preconditions | Input Data | Expected Outcome | Rule Ref |
| :--- | :--- | :--- | :--- | :--- |
| **TS-37** | Status: `SUBMITTED` | Action: `REJECT`<br>Notes: `NULL` | **400 Bad Request**. Error: "Notes Mandatory". | 10 |
| **TS-38** | Status: `SUBMITTED` | Action: `REJECT`<br>Notes: `"Damaged"` | **200 OK**. Claim `REJECTED`. | 10 |
| **TS-39** | Status: `SUBMITTED` | Action: `REQUEST_INFO` | **200 OK**. Claim `PENDING_INFO`. Notification Sent. | 11 |
| **TS-40** | Status: `SUBMITTED` | Action: `APPROVE` | **200 OK**. No Notification Sent. | 11 |

---

### 2.8 System Integrity (Rules 12, 13)
**Objective:** Ensure auditability and transactional safety.

| Test ID | Preconditions | Input Data | Expected Outcome | Rule Ref |
| :--- | :--- | :--- | :--- | :--- |
| **TS-41** | Any Valid Scenario | Action: `APPROVE` | **Audit Log**: Entry created with Timestamp, User, Action, Old/New State. | 12 |
| **TS-42** | DB Connection Lost (Simulated) | Action: `APPROVE` | **500 Error**. Claim Status **UNCHANGED**. Error Log Entry Created. | 13 |
| **TS-43** | Payment Service Timeout (Simulated) | Action: `APPROVE` | **500 Error**. Claim Status **UNCHANGED** (Rollback). Error Log Entry Created. | 13 |

---

## 3. Cross-Rule Interaction Tests
**Objective:** Validate behavior when multiple rules trigger simultaneously. Legacy systems often have implicit precedence (e.g., Fraud check usually happens before Threshold check).

| Test ID | Scenario Description | Preconditions | Input Data | Expected Outcome | Rules Involved |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **INT-01** | **Fraud Block vs. Auto-Approval**<br>Does fraud block override auto-approval logic? | Fraud: `75`<br>Amount: `$100` (Under Threshold) | Action: `APPROVE` | **403 Forbidden**. Claim NOT approved. Audit: `FRAUD_BLOCK`. (Fraud takes precedence). | 3, 4, 12 |
| **INT-02** | **Frequency vs. Threshold Escalation**<br>Which escalation reason is logged? | Claims (12mo): `4`<br>Amount: `$30,000` (Over Threshold)<br>Assessor: `2` | Action: `APPROVE` | **200 OK**. Claim `ESCALATED`. <br>*Verify Audit Log indicates both triggers or highest precedence.* | 5, 6, 12 |
| **INT-03** | **SLA + Payment Priority**<br>Old Platinum claim low amount. | Filed: `40 days ago`<br>Tier: `PLATINUM`<br>Amount: `$1,000`<br>Fraud: `10` | Action: `APPROVE` | **200 OK**. <br>1. Priority: `HIGH`.<br>2. SLA Flag: `TRUE`.<br>3. Payment Queued Priority: `1`. | 7, 8, 9 |
| **INT-04** | **Fraud Borderline + Payment**<br>Ensure payment doesn't queue if fraud is borderline. | Fraud: `29`<br>Amount: `$5,000` | Action: `APPROVE` | **200 OK**. Payment Queued. | 3, 8 |
| **INT-05** | **Fraud Borderline + Payment (Fail)** | Fraud: `30`<br>Amount: `$5,000` | Action: `APPROVE` | **200 OK**. Payment **NOT** Queued. | 3, 8 |
| **INT-06** | **Rejection with SLA Breach**<br>Does SLA flag set on rejection? | Filed: `40 days ago` | Action: `REJECT`<br>Notes: `"Invalid"` | **200 OK**. Claim `REJECTED`. SLA Flag: `FALSE` (Rule 7 specifies "on approval"). | 7, 10 |
| **INT-07** | **Complex Escalation**<br>High Amount + High Frequency + Low Assessor. | Amount: `$50,000`<br>Claims (12mo): `5`<br>Assessor: `1` | Action: `APPROVE` | **200 OK**. Claim `ESCALATED`. | 5, 6 |

---

## 4. Boundary Value Analysis Summary
*   **Fraud Score:** 29 (Pass Pay), 30 (Fail Pay), 70 (Pass Proc), 71 (Block Proc).
*   **Claim Amount:** $1,000 (Default Limit), $3,000 (Basic Limit), $8,000, $12,000, $15,000, $25,000. Test `Limit` and `Limit + 0.01`.
*   **Time:** 29 Days (Normal), 30 Days (Normal), 31 Days (SLA Breach).
*   **Frequency:** 3 Claims (Pass), 4 Claims (Fail/Escalate).
*   **Assessor Level:** 1, 2, 3 (Critical thresholds for escalation rules).

## 5. Negative Test Summary
*   **Invalid Payload:** Missing `claimId`, malformed JSON.
*   **Unauthorized User:** Assessor ID not found or no permissions.
*   **State Mismatch:** Approving an already Approved claim.
*   **Data Integrity:** Policy ID does not exist.
*   **Concurrency:** Two assessors processing the same claim simultaneously (Optimistic locking check).

## 6. Execution Notes
1.  **Audit Verification:** Every test case resulting in a 200 OK or a business logic block (403) must verify a corresponding row in the `AUDIT_LOGS` table.
2.  **Rollback Verification:** For Rule 13 tests, verify the database transaction isolation level ensures no partial updates occur.
3.  **Legacy Comparison:** Where possible, run the same input data against the legacy PL/SQL procedure (via a test harness) and compare the API response payload and final database state byte-for-byte.
