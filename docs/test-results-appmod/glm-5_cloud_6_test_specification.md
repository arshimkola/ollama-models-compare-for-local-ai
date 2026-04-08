# Test Specification: Insurance Claims Processing API

## 1. Introduction
This document outlines the test specification for validating the modernized Insurance Claims Processing API against the extracted business rules from the legacy PL/SQL procedure. The goal is to ensure functional parity and identical behaviour under all conditions.

---

## 2. Test Scenarios Grouped by Business Rule

### Group 1: Claim Status Validation (Rule 1)
**Rule:** Claims can only be processed when status is SUBMITTED, UNDER_REVIEW, or ESCALATED.

| Test ID | Scenario | Preconditions | Input Data | Expected Outcome |
| :--- | :--- | :--- | :--- | :--- |
| **TC_01_01** | Valid Status: SUBMITTED | Claim exists in DB | `claim_status: "SUBMITTED"` | Processing continues (Success). |
| **TC_01_02** | Valid Status: UNDER_REVIEW | Claim exists in DB | `claim_status: "UNDER_REVIEW"` | Processing continues (Success). |
| **TC_01_03** | Valid Status: ESCALATED | Claim exists in DB | `claim_status: "ESCALATED"` | Processing continues (Success). |
| **TC_01_04** | Invalid Status: REJECTED | Claim exists in DB | `claim_status: "REJECTED"` | **Error:** Processing blocked. Error message: "Invalid status for processing." |
| **TC_01_05** | Invalid Status: CLOSED | Claim exists in DB | `claim_status: "CLOSED"` | **Error:** Processing blocked. Error message: "Invalid status for processing." |

---

### Group 2: Policy Status Validation (Rule 2)
**Rule:** Associated policy must be ACTIVE.

| Test ID | Scenario | Preconditions | Input Data | Expected Outcome |
| :--- | :--- | :--- | :--- | :--- |
| **TC_02_01** | Active Policy | Policy linked to claim | `policy_status: "ACTIVE"` | Processing continues (Success). |
| **TC_02_02** | Inactive Policy (Lapsed) | Policy linked to claim | `policy_status: "LAPSED"` | **Error:** Processing blocked. Error message: "Policy is not active." |
| **TC_02_03** | Inactive Policy (Cancelled) | Policy linked to claim | `policy_status: "CANCELLED"` | **Error:** Processing blocked. Error message: "Policy is not active." |

---

### Group 3: Fraud Check (Rule 3)
**Rule:** Fraud score > 70 blocks approval and creates audit entry.

| Test ID | Scenario | Preconditions | Input Data | Expected Outcome |
| :--- | :--- | :--- | :--- | :--- |
| **TC_03_01** | Fraud Score Exceeds Limit | Valid Claim & Policy | `fraud_score: 71` | **Outcome:** Claim blocked from approval. <br>**Audit:** Entry created with reason "Fraud Score Exceeded". |
| **TC_03_02** | Fraud Score at Limit (Boundary) | Valid Claim & Policy | `fraud_score: 70` | Processing continues (Approval allowed if other rules pass). |
| **TC_03_03** | Fraud Score Acceptable | Valid Claim & Policy | `fraud_score: 50` | Processing continues. |

---

### Group 4: Auto-Approval Thresholds (Rule 4 & 5)
**Rules:** Threshold logic defined by Policy Type/Tier. Exceeding threshold requires Assessor Level >= 3 or triggers escalation.

| Test ID | Scenario | Input Data (Type/Tier/Amount) | Assessor Level | Expected Outcome |
| :--- | :--- | :--- | :--- | :--- |
| **TC_04_01** | Premium Gold Auto-Approve | Premium, Gold, $24,999 | N/A | **Auto-Approved.** |
| **TC_04_02** | Premium Platinum Threshold | Premium, Platinum, $25,000 | N/A | **Auto-Approved.** (Assuming <= threshold is auto). |
| **TC_04_03** | Premium Gold Exceeds Threshold | Premium, Gold, $25,001 | Level 3 | **Manual Approval.** (Assessor level sufficient). |
| **TC_04_04** | Premium Gold Exceeds Threshold (Low Level) | Premium, Gold, $25,001 | Level 1 | **Auto-Escalated.** (Assessor level insufficient). |
| **TC_04_05** | Standard Platinum Auto-Approve | Standard, Platinum, $12,000 | N/A | **Auto-Approved.** |
| **TC_04_06** | Standard Bronze Exceeds Threshold | Standard, Bronze, $8,001 | Level 2 | **Auto-Escalated.** (Assessor level < 3). |
| **TC_04_07** | Basic Policy Auto-Approve | Basic, N/A, $3,000 | N/A | **Auto-Approved.** |
| **TC_04_08** | Default Policy (Unknown Type) | Unknown, N/A, $1,000 | N/A | **Auto-Approved.** |
| **TC_04_09** | Default Policy Exceeds Threshold | Unknown, N/A, $1,001 | Level 3 | **Manual Approval.** |

---

### Group 5: Claim Frequency (Rule 6)
**Rule:** > 3 claims in 12 months requires Assessor Level >= 2, otherwise escalates.

| Test ID | Scenario | Preconditions | Input Data | Expected Outcome |
| :--- | :--- | :--- | :--- | :--- |
| **TC_05_01** | Low Frequency | 2 claims in last 12 months | Assessor: Level 1 | Processing continues normally. |
| **TC_05_02** | High Frequency Boundary | 4 claims in last 12 months | Assessor: Level 2 | **Manual Approval.** (Assessor level sufficient). |
| **TC_05_03** | High Frequency Escalation | 4 claims in last 12 months | Assessor: Level 1 | **Auto-Escalated.** |
| **TC_05_04** | Boundary Check | 3 claims in last 12 months | Assessor: Level 1 | Processing continues normally (Rule triggers on >3). |

---

### Group 6: Claim Age & Priority (Rule 7)
**Rule:** Claims filed > 30 days get HIGH priority and SLA breach flag on approval.

| Test ID | Scenario | Input Data (Filed Date) | Expected Outcome |
| :--- | :--- | :--- | :--- |
| **TC_06_01** | Fresh Claim | Filed 10 days ago | Priority: Normal. No SLA flag. |
| **TC_06_02** | Aged Claim Boundary | Filed 31 days ago | Priority: HIGH. SLA Breach Flag: TRUE. |
| **TC_06_03** | Exact Boundary | Filed 30 days ago | Priority: Normal. No SLA flag. |

---

### Group 7: Auto-Payment Queue (Rules 8 & 9)
**Rules:** Auto-payment for approved claims <= $5,000 and fraud < 30. Platinum customers get Priority 1.

| Test ID | Scenario | Input Data | Expected Outcome |
| :--- | :--- | :--- | :--- |
| **TC_07_01** | Auto-Payment Eligible | Amount: $5,000, Fraud: 25 | **Queued for Auto-Payment.** |
| **TC_07_02** | Amount Exceeded | Amount: $5,001, Fraud: 25 | **Not Queued.** (Manual payout required). |
| **TC_07_03** | Fraud Score Exceeded | Amount: $4,000, Fraud: 30 | **Not Queued.** (Threshold is < 30). |
| **TC_07_04** | Platinum Payment Priority | Tier: Platinum, Amount: $3,000 | **Payment Queue Priority:** 1. |
| **TC_07_05** | Standard Payment Priority | Tier: Gold, Amount: $3,000 | **Payment Queue Priority:** 2. |

---

### Group 8: Rejections & Actions (Rules 10, 11, 12)
**Rules:** Rejections require notes. REQUEST_INFO sends notification. All actions audited.

| Test ID | Scenario | Input Data | Expected Outcome |
| :--- | :--- | :--- | :--- |
| **TC_08_01** | Rejection with Notes | Action: REJECT, Notes: "Invalid docs" | **Success.** Audit log updated. |
| **TC_08_02** | Rejection without Notes | Action: REJECT, Notes: NULL | **Error:** "Notes are mandatory for rejection." |
| **TC_08_03** | Request Info Action | Action: REQUEST_INFO | Notification sent to customer. Audit log updated. |
| **TC_08_04** | Audit Log Verification | Any Action | Verify database entry in `AUDIT_LOG` table. |

---

### Group 9: Error Handling (Rule 13)
**Rule:** System errors trigger rollback and error logging.

| Test ID | Scenario | Preconditions | Expected Outcome |
| :--- | :--- | :--- | :--- |
| **TC_09_01** | System Failure | Simulate DB connection loss during processing | Transaction rolled back (no partial updates). Error logged to system error log. |

---

## 3. Boundary Value Analysis (Summary)

This section highlights specific edge cases derived from numerical boundaries in the rules.

| Rule | Boundary Point | Test Values | Expectation |
| :--- | :--- | :--- | :--- |
| **Fraud Score** | 70 (Block Threshold) | 69, 70, 71 | 69/70 Pass; 71 Block. |
| **Fraud Score (Pay)** | 30 (Auto-Pay Threshold) | 29, 30, 31 | 29 Auto-Pay; 30/31 No Auto-Pay. |
| **Claim Age** | 30 Days (Priority) | 29, 30, 31 days | 29/30 Normal; 31 High Priority. |
| **Claim Count** | 3 (Frequency) | 2, 3, 4 claims | 2/3 Normal; 4 Frequency Check triggers. |
| **Amounts** | Various Thresholds | $Limit - 0.01, $Limit, $Limit + 0.01 | Below/At = Auto; Above = Manual/Escalate. |

---

## 4. Cross-Rule Interaction Tests

These tests validate logic where multiple rules interact simultaneously.

| Test ID | Scenario Description | Setup Data | Expected Behaviour |
| :--- | :--- | :--- | :--- |
| **XC_01** | **Fraud Block Overrides All** <br>*(Rule 3 + Rule 4)* | Fraud Score: 75 <br> Amount: $1,000 (Auto-approve range) | **Outcome:** Claim BLOCKED. <br> *Reasoning:* Fraud check fails before approval logic is evaluated. |
| **XC_02** | **High Frequency + High Amount** <br>*(Rule 5 + Rule 6)* | Amount: > Threshold <br> Claim Count: 4 <br> Assessor Level: 2 | **Outcome:** Auto-Escalate. <br> *Reasoning:* Rule 5 requires Level >= 3. Rule 6 requires Level >= 2. The stricter requirement (Level 3) applies. Since Assessor is Level 2, it escalates. |
| **XC_03** | **Aged Claim + Platinum + Auto-Pay** <br>*(Rule 7 + Rule 9)* | Filed: 35 days ago <br> Tier: Platinum <br> Amount: $4,000 <br> Fraud: 10 | **Outcome:** Approved, Priority HIGH, SLA Flag TRUE. <br> Payment Queued with Priority 1. |
| **XC_04** | **Assessor Override on Fraud Block?** | Fraud Score: 75 <br> Assessor Level: 5 | **Outcome:** Claim BLOCKED. <br> *Reasoning:* Rule 3 is a hard block; Assessor level does not override Fraud Score > 70 (based on strict interpretation of "blocks approval"). |
| **XC_05** | **Threshold Boundary + Frequency Escalation** | Amount: $25,001 (Premium Gold) <br> Claim Count: 5 <br> Assessor: Level 3 | **Outcome:** Manual Approval Allowed. <br> *Reasoning:* Amount requires Level 3 (Met). Frequency requires Level 2 (Met). Higher requirement satisfied. |
| **XC_06** | **Rejection of Aged Claim** | Action: REJECT <br> Filed: 40 days ago | **Outcome:** Rejection processed. <br> *Note:* Rules 7 (Priority) do not prevent rejection. Rule 10 (Notes) applies. |

---

## 5. Glossary & Assumptions
*   **Assessor Level:** Integer value 1-5 representing authority.
*   **Thresholds:** Monetary limits are inclusive for auto-approval (e.g., <= $25,000) unless specified otherwise.
*   **Status Flow:** It is assumed that processing moves the claim to 'PROCESSED' or similar, but this specification focuses on the validation logic provided.
*   **Timestamps:** For Claim Age tests, server time is compared against `claim_filed_date`.
