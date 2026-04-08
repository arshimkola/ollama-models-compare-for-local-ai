Here is the analysis of the `PROCESS_INSURANCE_CLAIM` stored procedure. The business rules have been extracted and categorized by validation, processing logic, and action-specific workflows.

### **Category 1: Pre-Processing & Validation Rules**

| Rule ID | Description | Conditions | Actions | Edge Cases |
| :--- | :--- | :--- | :--- | :--- |
| **BR-VAL-001** | **Claim State Validation** | Claim status is **NOT** in `('SUBMITTED', 'UNDER_REVIEW', 'ESCALATED')`. | Set Output Status to `ERROR`. <br> Set Output Message indicating current status. <br> Terminate processing. | None. |
| **BR-VAL-002** | **Policy Active Status** | Associated Policy status is **NOT** `'ACTIVE'`. | Set Output Status to `ERROR`. <br> Set Output Message: 'Associated policy is not active'. <br> Terminate processing. | None. |
| **BR-VAL-003** | **Entity Existence** | Claim, Policy, Customer, or Assessor records do not exist in the database. | Catch `NO_DATA_FOUND` exception. <br> Set Output Status to `ERROR`. <br> Set Output Message: 'Claim or assessor not found'. | The code groups Claim/Policy/Customer/Assessor existence into a single exception handler, making specific root-cause diagnosis difficult. |
| **BR-VAL-004** | **Action Validity** | `p_action` is **NOT** in `('APPROVE', 'REJECT', 'REQUEST_INFO')`. | Set Output Status to `ERROR`. <br> Set Output Message: 'Invalid action: [value]'. <br> Proceed to Audit Log (does not return early). | Case sensitivity of `p_action` is not handled (e.g., 'approve' vs 'APPROVE'). |

---

### **Category 2: Financial & Risk Logic Rules**

| Rule ID | Description | Conditions | Actions | Edge Cases |
| :--- | :--- | :--- | :--- | :--- |
| **BR-FIN-001** | **Auto-Approval Threshold Matrix** | Determined by `v_policy_type` AND `v_customer_tier`. | Set `v_auto_threshold` variable:<br>• PREMIUM + (GOLD/PLATINUM): **25,000**<br>• PREMIUM (other): **15,000**<br>• STANDARD + PLATINUM: **12,000**<br>• STANDARD (other): **8,000**<br>• BASIC: **3,000**<br>• **ELSE**: **1,000** | If Policy Type or Tier is NULL or unexpected value, threshold defaults to **1,000**. |
| **BR-RISK-001** | **Fraud Score Blocking** | `v_fraud_score` **> 70**. | Set Output Status to `BLOCKED`. <br> Insert `audit_log` with reason 'FRAUD_BLOCK'. <br> Terminate processing (Claim status not updated). | `v_fraud_score` defaults to **0** if NULL in DB (`NVL`). <br> Score of exactly **70** is allowed. |
| **BR-RISK-002** | **High Claim Frequency Review** | `v_prev_claims_12m` **> 3** (in last 12 months, excluding current claim). | **If Assessor Level < 2:**<br>Set Output Status to `ESCALATED`. <br> Update Claim Status to `ESCALATED`. <br> Terminate processing.<br>**If Assessor Level >= 2:**<br>Proceed normally. | Count excludes current claim (`claim_id != p_claim_id`). <br> Exactly **3** claims in 12 months does not trigger escalation. |
| **BR-RISK-003** | **Auto-Payment Eligibility** | `v_claim_amount` **<= 5,000** AND `v_fraud_score` **< 30**. | Insert record into `payment_queue`. <br> **Priority:** 1 if Tier='PLATINUM', else 2. | Amount of exactly **5,000** is eligible. <br> Fraud score of exactly **30** is **not** eligible. |

---

### **Category 3: Authorization & Workflow Rules**

| Rule ID | Description | Conditions | Actions | Edge Cases |
| :--- | :--- | :--- | :--- | :--- |
| **BR-AUTH-001** | **Assessor Authority Limit (Amount)** | `v_claim_amount` **>** `v_auto_threshold` AND `v_assessor_level` **< 3**. | Set Output Status to `ESCALATED`. <br> Update Claim Status to `ESCALATED`. <br> Clear `assigned_assessor` (set to NULL). <br> Insert `audit_log` with reason 'ESCALATE'. <br> Terminate processing. | Assessor Level **3** or higher can approve amounts exceeding the threshold. |
| **BR-SLA-001** | **SLA Breach Prioritization** | `v_days_since_filed` **> 30**. | Update Claim `priority` to `'HIGH'`. <br> Update Claim `sla_breached` to `'Y'`. | Claim filed exactly **30** days ago is not flagged. <br> This update occurs even if the claim is later escalated (before return). |
| **BR-WF-001** | **Rejection Mandatory Notes** | `p_action` = `'REJECT'` AND `p_notes` IS `NULL`. | Set Output Status to `ERROR`. <br> Set Output Message: 'Rejection requires notes'. <br> Terminate processing. | Empty string `''` is treated as valid (not NULL) in PL/SQL unless explicitly checked. |
| **BR-WF-002** | **Information Request Notification** | `p_action` = `'REQUEST_INFO'`. | Update Claim Status to `INFO_REQUESTED`. <br> Insert record into `customer_notifications`. | Notification message is taken directly from `p_notes`. |

---

### **Category 4: System & Audit Rules**

| Rule ID | Description | Conditions | Actions | Edge Cases |
| :--- | :--- | :--- | :--- | :--- |
| **BR-SYS-001** | **General Audit Logging** | Process completes without early `RETURN` (i.e., Success, Reject, Request Info, Invalid Action). | Insert record into `audit_log` with action, notes, and assessor ID. | **Defect Identified:** If escalated due to *Frequency* (BR-RISK-002), the code returns early *without* inserting an audit log, unlike Fraud or Amount Escalation. |
| **BR-SYS-002** | **Transaction Integrity** | Any `OTHERS` exception occurs. | `ROLLBACK` transaction. <br> Insert into `error_log`. <br> `COMMIT` the error log. <br> Set Output Status to `ERROR`. | The error log commit is independent of the main transaction rollback. |
| **BR-SYS-003** | **Claim Status Update (Approval)** | `p_action` = `'APPROVE'` AND all checks pass. | Update Claim Status to `APPROVED`. <br> Set `approved_by`, `approved_date`, and `notes`. | `p_notes` can be NULL for approvals (unlike rejections). |

### **Summary of Critical Findings for Modernisation**

1.  **Inconsistent Audit Logging:** There is a logic gap in the "High Claim Frequency" escalation path (Line 93-97). Unlike the Fraud Block and Amount Escalation paths, this path performs an early `RETURN` without writing to the `audit_log` table. This should be fixed in the new system.
2.  **Hardcoded Thresholds:** Financial thresholds (e.g., 25000, 5000, 70) are hardcoded in the PL/SQL. These should be externalized to a configuration table or rules engine in the modernised application.
3.  **NULL Handling:** `fraud_score` uses `NVL(..., 0)`, treating missing data as "low risk". This assumption should be validated with business stakeholders.
4.  **Transaction Scope:** The `error_log` insert is committed even if the main transaction rolls back. Ensure this pattern is intentional for the new architecture (usually desired for debugging).
5.  **Assessor Unassignment:** When escalating due to amount authority (Line 78), the `assigned_assessor` is set to `NULL`. This implies the claim goes back into a pool for a senior assessor to pick up. This workflow needs to be preserved.
