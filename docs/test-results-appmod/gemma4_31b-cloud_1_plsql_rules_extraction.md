Based on the provided PL/SQL code, here is the extraction of all business rules.

### 1. Workflow & State Validation Rules
These rules ensure the claim is in a valid state to be processed.

| Rule ID | Description | Conditions | Actions | Edge Cases |
| :--- | :--- | :--- | :--- | :--- |
| **BR-01** | Claim State Validation | `v_claim_status` is NOT one of: 'SUBMITTED', 'UNDER_REVIEW', 'ESCALATED' | Stop processing; return `ERROR` status and message. | Claims already 'APPROVED' or 'REJECTED' are blocked. |
| **BR-02** | Policy Activity Validation | `v_policy_status` != 'ACTIVE' | Stop processing; return `ERROR` status and message. | Expired or cancelled policies cannot have claims processed. |
| **BR-03** | Action Validation | `p_action` is not 'APPROVE', 'REJECT', or 'REQUEST_INFO' | Stop processing; return `ERROR` status. | Case sensitivity of `p_action` may lead to unexpected errors. |

---

### 2. Financial Threshold Rules (Auto-Approval)
These rules define the monetary limits based on policy and customer profiles.

| Rule ID | Description | Conditions | Actions | Edge Cases |
| :--- | :--- | :--- | :--- | :--- |
| **BR-04** | Approval Threshold Matrix | Based on `v_policy_type` and `v_customer_tier`:<br>1. Premium + Gold/Platinum $\rightarrow$ \$25k<br>2. Premium $\rightarrow$ \$15k<br>3. Standard + Platinum $\rightarrow$ \$12k<br>4. Standard $\rightarrow$ \$8k<br>5. Basic $\rightarrow$ \$3k<br>6. Other $\rightarrow$ \$1k | Assigns `v_auto_threshold` value for use in authority checks. | Any combination not explicitly listed defaults to \$1,000. |

---

### 3. Approval Logic & Risk Controls
These rules govern the "APPROVE" action and a set of "gates" the claim must pass.

| Rule ID | Description | Conditions | Actions | Edge Cases |
| :--- | :--- | :--- | :--- | :--- |
| **BR-05** | Fraud Block Gate | `p_action` = 'APPROVE' AND `v_fraud_score` > 70 | Stop processing; Status $\rightarrow$ 'BLOCKED'; Log 'FRAUD_BLOCK' in `audit_log`. | Fraud score is treated as 0 if NULL. |
| **BR-06** | Assessor Authority Check | `p_action` = 'APPROVE' AND `v_claim_amount` > `v_auto_threshold` AND `v_assessor_level` < 3 | Status $\rightarrow$ 'ESCALATED'; Unassign assessor; Log 'ESCALATE' in `audit_log`. | Level 3+ assessors can approve any amount regardless of threshold. |
| **BR-07** | Frequency Risk Check | `p_action` = 'APPROVE' AND `v_prev_claims_12m` > 3 AND `v_assessor_level` < 2 | Status $\rightarrow$ 'ESCALATED'. | Level 2+ assessors can ignore high-frequency triggers. |
| **BR-08** | SLA Priority Trigger | `p_action` = 'APPROVE' AND `v_days_since_filed` > 30 | Set `priority` = 'HIGH' and `sla_breached` = 'Y'. | Occurs only upon approval, not during the review phase. |
| **BR-09** | Auto-Payment Trigger | `p_action` = 'APPROVE' AND `v_claim_amount` $\le$ 5,000 AND `v_fraud_score` < 30 | Insert into `payment_queue`. | Payment priority is set to '1' for Platinum customers, otherwise '2'. |

---

### 4. Rejection & Information Request Rules
These rules govern the "REJECT" and "REQUEST_INFO" actions.

| Rule ID | Description | Conditions | Actions | Edge Cases |
| :--- | :--- | :--- | :--- | :--- |
| **BR-10** | Mandatory Rejection Notes | `p_action` = 'REJECT' AND `p_notes` IS NULL | Stop processing; return `ERROR` status. | Rejection is impossible without providing a reason. |
| **BR-11** | Information Request | `p_action` = 'REQUEST_INFO' | Status $\rightarrow$ 'INFO_REQUESTED'; Create record in `customer_notifications`. | Not validated for notes; null notes may be sent to customers. |

---

### 5. System & Audit Rules
General system behaviors and data integrity rules.

| Rule ID | Description | Conditions | Actions | Edge Cases |
| :--- | :--- | :--- | :--- | :--- |
| **BR-12** | Audit Trail | Any successful action or block | Insert into `audit_log` with `p_action` and `p_notes`. | If `p_notes` is null, 'No notes' is logged. |
| **BR-13** | Global Error Handling | Any unhandled exception | Rollback transaction; log to `error_log`; return `ERROR` status. | Specific `NO_DATA_FOUND` handler returns a a custom "not found" message. |
