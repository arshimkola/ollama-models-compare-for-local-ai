Here is the extraction of business rules from the `PROCESS_INSURANCE_CLAIM` stored procedure.

### Business Rules Extraction

| Rule ID | Description | Conditions | Actions | Edge Cases / Notes |
| :--- | :--- | :--- | :--- | :--- |
| **BR-01** | **Claim State Validation** | Claim status is NOT IN ('SUBMITTED', 'UNDER_REVIEW', 'ESCALATED'). | Halt processing. Set OUT status to 'ERROR'. Return message: "Claim not in reviewable state." | Prevents processing of claims that are already closed, rejected, or in an invalid workflow state. |
| **BR-02** | **Policy Active Validation** | Associated Policy status is NOT 'ACTIVE'. | Halt processing. Set OUT status to 'ERROR'. Return message: "Associated policy is not active". | Ensures claims are only processed against valid, in-force policies. |
| **BR-03** | **Dynamic Approval Threshold Calculation** | Determine monetary limit for auto-approval based on Policy Type and Customer Tier. | Calculate `v_auto_threshold`:<br>• PREMIUM + (GOLD/PLATINUM) = $25k<br>• PREMIUM (Other) = $15k<br>• STANDARD + PLATINUM = $12k<br>• STANDARD (Other) = $8k<br>• BASIC = $3k<br>• Unknown Type = $1k | The logic uses a `CASE` statement. The `ELSE 1000` acts as a safe default for unmapped policy types. |
| **BR-04** | **Fraud Score Gate** | Action = 'APPROVE' AND Fraud Score > 70. | Halt approval. Set status 'BLOCKED'. Insert Audit Log ('FRAUD_BLOCK'). | Hard stop. Even senior assessors cannot bypass this via this procedure. |
| **BR-05** | **Assessor Authority Limit Check** | Action = 'APPROVE' AND Claim Amount > `v_auto_threshold` AND Assessor Level < 3. | Halt approval. Update Claim Status to 'ESCALATED'. Insert Audit Log ('ESCALATE'). Return message regarding authority limits. | If Assessor Level is >= 3, this check is passed (they have authority). |
| **BR-06** | **High Frequency Review Check** | Action = 'APPROVE' AND Previous Claims (12 months) > 3 AND Assessor Level < 2. | Halt approval. Update Claim Status to 'ESCALATED'. Return message regarding frequency. | **Gap Identified:** Unlike other escalations, this block does **not** insert an entry into `audit_log` before returning. |
| **BR-07** | **SLA Breach Flagging** | Action = 'APPROVE' AND Days Since Filed > 30. | Update Claim: Set Priority = 'HIGH', SLA_Breached = 'Y'. Processing continues (does not stop approval). | This logic applies only if the claim proceeds to approval status. |
| **BR-08** | **Claim Approval Execution** | Action = 'APPROVE' AND all previous checks passed. | Update Claim: Status = 'APPROVED', set Approved_By/Date. Insert Audit Log. | |
| **BR-09** | **Auto-Payment Queueing** | Action = 'APPROVE' AND Claim Amount <= 5000 AND Fraud Score < 30. | Insert record into `payment_queue`. Priority is 1 for PLATINUM customers, 2 for others. | This happens immediately after approval logic within the same transaction. |
| **BR-10** | **Rejection Mandatory Notes** | Action = 'REJECT' AND `p_notes` is NULL. | Halt processing. Set OUT status to 'ERROR'. Return message: "Rejection requires notes". | Data integrity check to ensure reasons are recorded for rejections. |
| **BR-11** | **Claim Rejection Execution** | Action = 'REJECT' AND `p_notes` provided. | Update Claim: Status = 'REJECTED', set Rejected_By/Date. Insert Audit Log. | |
| **BR-12** | **Information Request Workflow** | Action = 'REQUEST_INFO'. | Update Claim: Status = 'INFO_REQUESTED'. Insert record into `customer_notifications`. Insert Audit Log. | Does not require assessor authority level checks. |
| **BR-13** | **Action Validation** | `p_action` NOT IN ('APPROVE', 'REJECT', 'REQUEST_INFO'). | Halt processing. Set OUT status to 'ERROR'. | Handles invalid inputs for the action parameter. |
| **BR-14** | **General Exception Handling** | Any database error (e.g., `NO_DATA_FOUND` or others). | Rollback transaction. Log to `error_log`. Set OUT status to 'ERROR'. | Ensures partial updates are not committed during system failures. |
