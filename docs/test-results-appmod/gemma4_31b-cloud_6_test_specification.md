# Test Specification: Insurance Claims Processing API Validation

## 1. Introduction
This document defines the test specification to ensure the modernized API maintains parity with the legacy PL/SQL business logic. The goal is to validate that every business rule is enforced identically, including edge cases and complex interactions.

---

## 2. Test Scenarios by Business Rule

### Group A: State & Policy Validation (Rules 1, 2)
| Scenario ID | Rule | Preconditions | Input Data | Expected Outcome |
| :--- | :--- | :--- | :--- | :--- |
| **TS_01.1** | 1 | Claim exists | Status: `SUBMITTED` | Process continues |
| **TS_01.2** | 1 | Claim exists | Status: `UNDER_REVIEW` | Process continues |
| **TS_01.3** | 1 | Claim exists | Status: `ESCALATED` | Process continues |
| **TS_01.4** | 1 | Claim exists | Status: `CLOSED` or `PAID` | Error: Invalid status for processing |
| **TS_02.1** | 2 | Valid Policy | Policy Status: `ACTIVE` | Process continues |
| **TS_02.2** | 2 | Valid Policy | Policy Status: `LAPSED` or `EXPIRED` | Error: Associated policy not active |

### Group B: Fraud & Security (Rule 3)
| Scenario ID | Rule | Preconditions | Input Data | Expected Outcome |
| :--- | :--- | :--- | :--- | :--- |
| **TS_03.1** | 3 | Valid Claim | Fraud Score: 71 | Approval blocked; Audit entry created |
| **TS_03.2** | 3 | Valid Claim | Fraud Score: 70 | Approval allowed (unless other rules block) |

### Group C: Auto-Approval Thresholds (Rule 4)
| Scenario ID | Rule | Preconditions | Input Data | Expected Outcome |
| :--- | :--- | :--- | :--- | :--- |
| **TS_04.1** | 4 | Policy: PREMIUM, Tier: GOLD | Claim Amt: $25,000 | Auto-Approval eligible |
| **TS_04.2** | 4 | Policy: PREMIUM, Tier: SILVER | Claim Amt: $15,000 | Auto-Approval eligible |
| **TS_04.3** | 4 | Policy: STANDARD, Tier: PLATINUM | Claim Amt: $12,000 | Auto-Approval eligible |
| **TS_04.4** | 4 | Policy: STANDARD, Tier: BRONZE | Claim Amt: $8,000 | Auto-Approval eligible |
| **TS_04.5** | 4 | Policy: BASIC, Tier: Any | Claim Amt: $3,000 | Auto-Approval eligible |
| **TS_04.6** | 4 | Policy: UNKNOWN, Tier: Any | Claim Amt: $1,000 | Auto-Approval eligible |

### Group D: Assessor Levels & Escalation (Rules 5, 6)
| Scenario ID | Rule | Preconditions | Input Data | Expected Outcome |
| :--- | :--- | :--- | :--- | :--- |
| **TS_05.1** | 5 | Claim > Threshold | Assessor Level: 3 | Approval processed |
| **TS_05.2** | 5 | Claim > Threshold | Assessor Level: 2 | Status $\rightarrow$ `ESCALATED` |
| **TS_06.1** | 6 | Cust. Claims (12mo): 4 | Assessor Level: 2 | Approval processed |
| **TS_06.2** | 6 | Cust. Claims (12mo): 4 | Assessor Level: 1 | Status $\rightarrow$ `ESCALATED` |

### Group E: Priority & SLA (Rule 7)
| Scenario ID | Rule | Preconditions | Input Data | Expected Outcome |
| :--- | :--- | :--- | :--- | :--- |
| **TS_07.1** | 7 | Date: 31 days ago | Approval Action | Priority: `HIGH`, SLA Breach: `TRUE` |
| **TS_07.2** | 7 | Date: 30 days ago | Approval Action | Priority: `NORMAL`, SLA Breach: `FALSE` |

### Group F: Payment Queue Logic (Rules 8, 9)
| Scenario ID | Rule | Preconditions | Input Data | Expected Outcome |
| :--- | :--- | :--- | :--- | :--- |
| **TS_08.1** | 8 | Approved | Amt: $5,000, Fraud: 29 | Queued for Auto-payment |
| **TS_08.2** | 8 | Approved | Amt: $5,001, Fraud: 10 | Not queued for Auto-payment |
| **TS_08.3** | 8 | Approved | Amt: $1,000, Fraud: 30 | Not queued for Auto-payment |
| **TS_09.1** | 9 | Tier: PLATINUM | Auto-payment trigger | Payment Priority: 1 |
| **TS_09.2** | 9 | Tier: GOLD/SILVER | Auto-payment trigger | Payment Priority: 2 |

### Group G: Actions & Auditing (Rules 10, 11, 12, 13)
| Scenario ID | Rule | Preconditions | Input Data | Expected Outcome |
| :--- | :--- | :--- | :--- | :--- |
| **TS_10.1** | 10 | Rejection Action | Notes: `NULL` | Error: Notes mandatory |
| **TS_11.1** | 11 | `REQUEST_INFO` | Action trigger | Notification sent to customer |
| **TS_12.1** | 12 | Any Action | Any Input | Audit log entry created with timestamp/user |
| **TS_13.1** | 13 | Database Timeout | Processing call | Transaction rolled back; Error log entry created |

---

## 3. Boundary Value Analysis (BVA)

| Feature | Lower Boundary | Mid Point | Upper Boundary |
| :--- | :--- | :--- | :--- |
| **Fraud Score** | 0 (Safe) | 70 (Threshold) | 71 (Block) |
| **Premium Gold Amt** | $24,999.99 (Auto) | $25,000.00 (Auto) | $25,000.01 (Manual) |
| **Claim Frequency** | 3 claims (Normal) | 4 claims (Escalate) | 10 claims (Escalate) |
| **SLA Days** | 30 days (Normal) | 31 days (High) | 90 days (High) |
| **Payment Amt** | $0.01 (Auto-pay) | $5,000.00 (Auto-pay) | $5,000.01 (Manual) |

---

## 4. Negative Testing

- **Invalid Status Transition:** Attempt to process a claim that is already `PAID` to ensure it cannot be re-approved.
- **Policy Mismatch:** Attempt to process a claim where the `policy_id` provided does not exist in the system.
- **SQL Injection/Payload Stress:** Passing extremely large strings in the "Notes" field for rejections to ensure the API handles it without crashing (unlike legacy PL/SQL buffer limits).
- **Unauthorized User:** Attempting to approve a claim with an Assessor Level of 0 or NULL.

---

## 5. Cross-Rule Interaction Tests (Complex Scenarios)

These tests ensure that the logic doesn't "short-circuit" incorrectly.

**Scenario INT_01: The High-Risk High-Value Claim**
- **Inputs:** Policy: `PREMIUM/GOLD`, Amount: `$30,000`, Fraud Score: `75`, Claims in 12mo: `5`.
- **Expected Result:** 
    1. Rule 3 (Fraud > 70) must trigger **first** and block approval.
    2. Audit entry created for fraud block.
    3. Rule 5 & 6 (Assessor level) should not be evaluated if Fraud blocks the process.

**Scenario INT_02: The "Perfect Storm" Approval**
- **Inputs:** Policy: `STANDARD/PLATINUM`, Amount: `$12,000`, Fraud Score: `20`, Claims in 12mo: `4`, Date: `35 days ago`, Assessor Level: `2`.
- **Expected Result:**
    1. Rule 4: Threshold is $12k $\rightarrow$ Eligible for auto-approval.
    2. Rule 6: > 3 claims requires Assessor $\ge 2 \rightarrow$ Pass.
    3. Rule 7: > 30 days $\rightarrow$ Priority: `HIGH`, SLA Breach: `TRUE`.
    4. Rule 8: Amount > $5k $\rightarrow$ No auto-payment queue.
    5. Rule 12: Audit log created.

**Scenario INT_03: The Fast-Track Auto-Pay**
- **Inputs:** Policy: `BASIC`, Amount: `$2,000`, Fraud Score: `10`, Tier: `PLATINUM` (if possible in business logic), Claims in 12mo: `1`.
- **Expected Result:**
    1. Rule 4: Within $3k threshold $\rightarrow$ Auto-Approved.
    2. Rule 8: $\le \$5k$ and Fraud $< 30 \rightarrow$ Queued for auto-payment.
    3. Rule 9: Platinum tier $\rightarrow$ Payment Priority 1.
