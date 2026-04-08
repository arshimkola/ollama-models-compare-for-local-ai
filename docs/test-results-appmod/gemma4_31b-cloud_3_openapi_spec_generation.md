```yaml
openapi: 3.1.0
info:
  title: Insurance Claims Processing API
  description: >
    Modernized API for insurance claims processing, replacing legacy PL/SQL systems.
    Handles claim lifecycles, policy verification, assessor authorizations, and automated payments.
  version: 1.0.0
  contact:
    name: API Governance Team
    email: api-gov@insurance-corp.com

servers:
  - url: https://api.insurance-corp.com/v1
    description: Production server

components:
  securitySchemes:
    BearerAuth:
      type: http
      scheme: bearer
      bearerFormat: JWT
      description: JWT token containing assessor identity and authorization level (1-3).

  headers:
    X-RateLimit-Limit:
      description: The maximum number of requests allowed in the current window.
      schema:
        type: integer
    X-RateLimit-Remaining:
      description: The number of requests remaining in the current window.
      schema:
        type: integer
    X-RateLimit-Reset:
      description: The time at which the current rate limit window resets (UTC epoch).
      schema:
        type: integer

  schemas:
    # --- Common Schemas ---
    Error:
      type: object
      description: RFC 9457 Problem Details for HTTP APIs
      properties:
        type:
          type: string
          format: uri
          example: "https://api.insurance-corp.com/probs/fraud-score-too-high"
        title:
          type: string
          example: "Approval Blocked"
        status:
          type: integer
          example: 403
        detail:
          type: string
          example: "Claim cannot be approved because the fraud score (85) exceeds the threshold of 70."
        instance:
          type: string
          format: uri

    PaginationParams:
      type: object
      properties:
        page:
          type: integer
          default: 1
        pageSize:
          type: integer
          default: 20
          maximum: 100

    # --- Domain Schemas ---
    ClaimStatus:
      type: string
      enum: [SUBMITTED, UNDER_REVIEW, ESCALATED, APPROVED, REJECTED, INFO_REQUESTED]

    Claim:
      type: object
      properties:
        claimId:
          type: string
          format: uuid
          readOnly: true
        policyNumber:
          type: string
          pattern: '^[A-Z]{2}-\d{6}$'
        amount:
          type: number
          format: float
          minimum: 0
        status:
          $ref: '#/components/schemas/ClaimStatus'
        fraudScore:
          type: integer
          minimum: 0
          maximum: 100
        description:
          type: string
        submittedAt:
          type: string
          format: date-time
        slaBreached:
          type: boolean
          description: True if claim has been open for > 30 days.
          readOnly: true
        assessorId:
          type: string
          format: uuid
          readOnly: true

    ClaimTransitionRequest:
      type: object
      required: [action]
      properties:
        action:
          type: string
          enum: [REVIEW, APPROVE, REJECT, REQUEST_INFO]
        notes:
          type: string
          description: Required if action is REJECT.
        assessorId:
          type: string
          format: uuid

    Policy:
      type: object
      properties:
        policyNumber:
          type: string
          readOnly: true
        holderName:
          type: string
        isActive:
          type: boolean
        tier:
          type: string
          enum: [BASIC, STANDARD, PREMIUM, GOLD, PLATINUM]
        expiryDate:
          type: string
          format: date

    PaymentQueueItem:
      type: object
      properties:
        paymentId:
          type: string
          format: uuid
        claimId:
          type: string
          format: uuid
        amount:
          type: number
        status:
          type: string
          enum: [QUEUED, PROCESSED, FAILED]
        queuedAt:
          type: string
          format: date-time

    AuditLog:
      type: object
      properties:
        logId:
          type: string
          format: uuid
        claimId:
          type: string
          format: uuid
        action:
          type: string
        performedBy:
          type: string
        timestamp:
          type: string
          format: date-time
        previousStatus:
          $ref: '#/components/schemas/ClaimStatus'
        newStatus:
          $ref: '#/components/schemas/ClaimStatus'
        metadata:
          type: object

security:
  - BearerAuth: []

paths:
  /claims:
    get:
      summary: List claims
      parameters:
        - $ref: '#/components/parameters/PageParam'
        - $ref: '#/components/parameters/SizeParam'
      responses:
        '200':
          description: List of claims
          content:
            application/json:
              schema:
                type: array
                items:
                  $ref: '#/components/schemas/Claim'
    post:
      summary: Submit a new claim
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/Claim'
      responses:
        '201':
          description: Claim created
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/Claim'
        '400':
          description: Validation error
          content:
            application/problem+json:
              schema:
                $ref: '#/components/schemas/Error'

  /claims/{claimId}:
    parameters:
      - name: claimId
        in: path
        required: true
        schema:
          type: string
          format: uuid
    get:
      summary: Get claim details
      responses:
        '200':
          description: Claim found
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/Claim'
        '404':
          description: Claim not found
    put:
      summary: Update claim metadata
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/Claim'
      responses:
        '200':
          description: Updated successfully
    delete:
      summary: Remove claim (soft delete)
      responses:
        '204':
          description: Deleted

  /claims/{claimId}/transition:
    parameters:
      - name: claimId
        in: path
        required: true
        schema:
          type: string
          format: uuid
    post:
      summary: Trigger workflow transition
      description: >
        Moves claim through states. 
        Logic applied: 
        - Only allowed if status is SUBMITTED, UNDER_REVIEW, or ESCALATED.
        - Approvals check Policy status and Fraud Score (< 70).
        - Rejections require notes.
        - High-value claims trigger auto-escalation based on Assessor Level.
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/ClaimTransitionRequest'
      responses:
        '200':
          description: Transition successful
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/Claim'
        '403':
          description: Business rule violation (e.g., Fraud Score, Policy Inactive, or Insufficient Assessor Level)
          content:
            application/problem+json:
              schema:
                $ref: '#/components/schemas/Error'
        '422':
          description: Unprocessable entity (e.g., Missing rejection notes)
          content:
            application/problem+json:
              schema:
                $ref: '#/components/schemas/Error'

  /policies/{policyNumber}:
    get:
      summary: Lookup policy and verify active status
      parameters:
        - name: policyNumber
          in: path
          required: true
          schema:
            type: string
      responses:
        '200':
          description: Policy details
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/Policy'
        '404':
          description: Policy not found

  /payments/queue:
    get:
      summary: View auto-payment queue
      description: Returns claims <= $5K with fraud score < 30 that are approved.
      responses:
        '200':
          description: Queue list
          content:
            application/json:
              schema:
                type: array
                items:
                  $ref: '#/components/schemas/PaymentQueueItem'

  /audit/logs:
    get:
      summary: Retrieve immutable audit logs
      parameters:
        - name: claimId
          in: query
          schema:
            type: string
            format: uuid
      responses:
        '200':
          description: Audit history
          content:
            application/json:
              schema:
                type: array
                items:
                  $ref: '#/components/schemas/AuditLog'

components:
  parameters:
    PageParam:
      name: page
      in: query
      schema:
        type: integer
    SizeParam:
      name: pageSize
      in: query
      schema:
        type: integer
```
