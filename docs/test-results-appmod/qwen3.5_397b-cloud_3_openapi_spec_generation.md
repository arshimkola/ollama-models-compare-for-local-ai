```yaml
openapi: 3.1.0
info:
  title: Claims Processing API
  description: |
    Modern API for Insurance Claims Processing. 
    Replaces legacy PL/SQL system with RESTful architecture.
    
    ## Business Rules Overview
    - **Workflow:** Claims move through SUBMITTED -> UNDER_REVIEW -> APPROVED/REJECTED.
    - **Escalation:** Auto-escalates based on Assessor Level and Policy Tier thresholds.
    - **Fraud:** Claims with Fraud Score > 70 cannot be approved.
    - **SLA:** Claims open > 30 days are flagged.
    - **Auto-Payment:** Claims <= $5K with Fraud Score < 30 are auto-queued upon approval.
    - **Frequency:** >3 claims in 12 months triggers senior review.
  version: 1.0.0
  contact:
    name: API Support
    email: api-support@insurance.com
  license:
    name: Proprietary
    url: https://insurance.com/licenses

servers:
  - url: https://api.insurance.com/v1
    description: Production
  - url: https://staging-api.insurance.com/v1
    description: Staging

security:
  - bearerAuth: []

paths:
  /claims:
    post:
      summary: Submit a new claim
      description: |
        Creates a new claim in SUBMITTED status.
        Triggers initial fraud scoring.
      operationId: submitClaim
      tags:
        - Claims
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/ClaimSubmission'
            examples:
              standardClaim:
                summary: Standard Motor Claim
                value:
                  policyId: "POL-123456"
                  amount: 2500.00
                  currency: "USD"
                  description: "Rear bumper damage in parking lot"
                  incidentDate: "2023-10-01"
      responses:
        '201':
          description: Claim created successfully
          headers:
            X-RateLimit-Limit:
              $ref: '#/components/headers/RateLimitLimit'
            X-RateLimit-Remaining:
              $ref: '#/components/headers/RateLimitRemaining'
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/Claim'
        '400':
          $ref: '#/components/responses/BadRequest'
        '401':
          $ref: '#/components/responses/Unauthorized'
        '409':
          description: Conflict (e.g., Policy inactive, duplicate claim)
          content:
            application/problem+json:
              schema:
                $ref: '#/components/schemas/ProblemDetails'
              example:
                type: "https://api.insurance.com/errors/policy-inactive"
                title: "Policy Not Active"
                status: 409
                detail: "Cannot submit claim for inactive policy POL-123456"
    get:
      summary: List claims
      description: Returns a paginated list of claims.
      operationId: listClaims
      tags:
        - Claims
      parameters:
        - $ref: '#/components/parameters/Page'
        - $ref: '#/components/parameters/PageSize'
        - name: status
          in: query
          schema:
            type: string
            enum:
              - SUBMITTED
              - UNDER_REVIEW
              - ESCALATED
              - APPROVED
              - REJECTED
              - PAYMENT_QUEUED
              - PAID
        - name: assessorId
          in: query
          schema:
            type: string
      responses:
        '200':
          description: List of claims
          headers:
            X-RateLimit-Limit:
              $ref: '#/components/headers/RateLimitLimit'
            X-RateLimit-Remaining:
              $ref: '#/components/headers/RateLimitRemaining'
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/PaginatedClaims'
        '401':
          $ref: '#/components/responses/Unauthorized'

  /claims/{claimId}:
    get:
      summary: Get claim details
      operationId: getClaim
      tags:
        - Claims
      parameters:
        - name: claimId
          in: path
          required: true
          schema:
            type: string
            format: uuid
      responses:
        '200':
          description: Claim details
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/Claim'
        '404':
          $ref: '#/components/responses/NotFound'
        '401':
          $ref: '#/components/responses/Unauthorized'

  /claims/{claimId}/transition:
    post:
      summary: Transition claim workflow
      description: |
        Executes workflow actions (REVIEW, APPROVE, REJECT, REQUEST_INFO, ESCALATE).
        
        **Business Rules Enforced:**
        - **REVIEW:** Only allowed on SUBMITTED, UNDER_REVIEW, ESCALATED.
        - **APPROVE:** Requires Active Policy, Fraud Score <= 70. Auto-queues payment if <= $5K & Fraud < 30.
        - **REJECT:** Requires `notes` in request body.
        - **ESCALATE:** Auto-triggered if amount exceeds assessor threshold.
      operationId: transitionClaim
      tags:
        - Claims
      parameters:
        - name: claimId
          in: path
          required: true
          schema:
            type: string
            format: uuid
      requestBody:
        required: true
        content:
          application/json:
            schema:
              oneOf:
                - $ref: '#/components/schemas/ApproveRequest'
                - $ref: '#/components/schemas/RejectRequest'
                - $ref: '#/components/schemas/GenericTransitionRequest'
            discriminator:
              propertyName: action
              mapping:
                APPROVE: '#/components/schemas/ApproveRequest'
                REJECT: '#/components/schemas/RejectRequest'
                REVIEW: '#/components/schemas/GenericTransitionRequest'
                REQUEST_INFO: '#/components/schemas/GenericTransitionRequest'
                ESCALATE: '#/components/schemas/GenericTransitionRequest'
      responses:
        '200':
          description: Transition successful
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/Claim'
        '400':
          $ref: '#/components/responses/BadRequest'
        '403':
          description: Forbidden (e.g., Assessor level insufficient, Fraud score too high)
          content:
            application/problem+json:
              schema:
                $ref: '#/components/schemas/ProblemDetails'
              example:
                type: "https://api.insurance.com/errors/fraud-threshold"
                title: "Fraud Threshold Exceeded"
                status: 403
                detail: "Cannot approve claim with fraud score 75. Max allowed is 70."
        '409':
          description: Conflict (Invalid state transition)
          content:
            application/problem+json:
              schema:
                $ref: '#/components/schemas/ProblemDetails'
        '401':
          $ref: '#/components/responses/Unauthorized'

  /policies/{policyId}:
    get:
      summary: Get policy details
      description: Read-only lookup to verify active status and tier.
      operationId: getPolicy
      tags:
        - Policies
      parameters:
        - name: policyId
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
          $ref: '#/components/responses/NotFound'
        '401':
          $ref: '#/components/responses/Unauthorized'

  /assessors/me:
    get:
      summary: Get current assessor profile
      description: Returns auth context, authorization level, and thresholds.
      operationId: getMyProfile
      tags:
        - Assessors
      responses:
        '200':
          description: Assessor profile
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/Assessor'
        '401':
          $ref: '#/components/responses/Unauthorized'

  /payments:
    get:
      summary: List payments
      description: Includes auto-queued and processed payments.
      operationId: listPayments
      tags:
        - Payments
      parameters:
        - $ref: '#/components/parameters/Page'
        - $ref: '#/components/parameters/PageSize'
        - name: status
          in: query
          schema:
            type: string
            enum:
              - QUEUED
              - PROCESSING
              - COMPLETED
              - FAILED
      responses:
        '200':
          description: List of payments
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/PaginatedPayments'
        '401':
          $ref: '#/components/responses/Unauthorized'

  /audit/claims/{claimId}:
    get:
      summary: Get claim audit log
      description: Immutable log of all actions on a specific claim.
      operationId: getClaimAudit
      tags:
        - Audit
      parameters:
        - name: claimId
          in: path
          required: true
          schema:
            type: string
            format: uuid
      responses:
        '200':
          description: Audit log
          content:
            application/json:
              schema:
                type: array
                items:
                  $ref: '#/components/schemas/AuditEntry'
        '404':
          $ref: '#/components/responses/NotFound'
        '401':
          $ref: '#/components/responses/Unauthorized'

components:
  securitySchemes:
    bearerAuth:
      type: http
      scheme: bearer
      bearerFormat: JWT
      description: JWT token containing assessor ID and roles.

  headers:
    RateLimitLimit:
      description: Request limit per window
      schema:
        type: integer
    RateLimitRemaining:
      description: Remaining requests in window
      schema:
        type: integer
    RateLimitReset:
      description: Unix timestamp for window reset
      schema:
        type: integer

  parameters:
    Page:
      name: page
      in: query
      description: Page number (0-indexed)
      schema:
        type: integer
        default: 0
    PageSize:
      name: pageSize
      in: query
      description: Items per page
      schema:
        type: integer
        default: 20
        maximum: 100

  responses:
    BadRequest:
      description: Bad Request
      content:
        application/problem+json:
          schema:
            $ref: '#/components/schemas/ProblemDetails'
    Unauthorized:
      description: Unauthorized
      content:
        application/problem+json:
          schema:
            $ref: '#/components/schemas/ProblemDetails'
    NotFound:
      description: Resource Not Found
      content:
        application/problem+json:
          schema:
            $ref: '#/components/schemas/ProblemDetails'

  schemas:
    ProblemDetails:
      type: object
      description: RFC 9457 Problem Details
      properties:
        type:
          type: string
          format: uri
          description: A URI reference that identifies the problem type.
        title:
          type: string
          description: A short, human-readable summary of the problem type.
        status:
          type: integer
          description: The HTTP status code.
        detail:
          type: string
          description: A human-readable explanation specific to this occurrence.
        instance:
          type: string
          format: uri
          description: A URI reference that identifies the specific occurrence.
      required:
        - type
        - title
        - status

    ClaimSubmission:
      type: object
      properties:
        policyId:
          type: string
        amount:
          type: number
          format: double
          minimum: 0.01
        currency:
          type: string
          enum: [USD, EUR, GBP]
        description:
          type: string
          maxLength: 1000
        incidentDate:
          type: string
          format: date
      required:
        - policyId
        - amount
        - currency
        - description
        - incidentDate

    Claim:
      type: object
      properties:
        id:
          type: string
          format: uuid
        policyId:
          type: string
        status:
          $ref: '#/components/schemas/ClaimStatus'
        amount:
          type: number
          format: double
        currency:
          type: string
        fraudScore:
          type: integer
          minimum: 0
          maximum: 100
          description: Calculated risk score. >70 blocks approval.
        assessorId:
          type: string
          description: Assigned assessor.
        createdAt:
          type: string
          format: date-time
        updatedAt:
          type: string
          format: date-time
        slaBreached:
          type: boolean
          description: True if claim open > 30 days.
        autoPaymentQueued:
          type: boolean
          description: True if auto-queued for payment (<= $5K, Fraud < 30).
        notes:
          type: string
          description: Public notes on the claim.
      required:
        - id
        - policyId
        - status
        - amount
        - createdAt

    ClaimStatus:
      type: string
      enum:
        - SUBMITTED
        - UNDER_REVIEW
        - ESCALATED
        - APPROVED
        - REJECTED
        - PAYMENT_QUEUED
        - PAID
        - REQUESTED_INFO

    Policy:
      type: object
      properties:
        id:
          type: string
        holderId:
          type: string
        status:
          type: string
          enum: [ACTIVE, INACTIVE, SUSPENDED]
        tier:
          $ref: '#/components/schemas/PolicyTier'
        startDate:
          type: string
          format: date
        endDate:
          type: string
          format: date
      required:
        - id
        - status
        - tier

    PolicyTier:
      type: string
      enum:
        - BASIC
        - STANDARD
        - PREMIUM
        - PREMIUM_GOLD
        - PREMIUM_PLATINUM
      description: Determines escalation thresholds.

    Assessor:
      type: object
      properties:
        id:
          type: string
        name:
          type: string
        level:
          $ref: '#/components/schemas/AssessorLevel'
        thresholds:
          type: object
          description: Max approval amount per Policy Tier without escalation.
          properties:
            BASIC:
              type: number
              example: 3000
            STANDARD:
              type: number
              example: 8000
            PREMIUM:
              type: number
              example: 15000
            PREMIUM_GOLD_PLATINUM:
              type: number
              example: 25000
      required:
        - id
        - level

    AssessorLevel:
      type: integer
      enum: [1, 2, 3]
      description: 1=Junior, 2=Senior, 3=Manager. Affects escalation thresholds.

    Payment:
      type: object
      properties:
        id:
          type: string
          format: uuid
        claimId:
          type: string
          format: uuid
        amount:
          type: number
          format: double
        status:
          $ref: '#/components/schemas/PaymentStatus'
        queuedAt:
          type: string
          format: date-time
        processedAt:
          type: string
          format: date-time
      required:
        - id
        - claimId
        - amount
        - status

    PaymentStatus:
      type: string
      enum:
        - QUEUED
        - PROCESSING
        - COMPLETED
        - FAILED

    AuditEntry:
      type: object
      description: Immutable record of action.
      properties:
        id:
          type: string
          format: uuid
        claimId:
          type: string
          format: uuid
        action:
          type: string
          description: e.g., SUBMIT, APPROVE, REJECT, SYSTEM_ESCALATE
        actorId:
          type: string
          description: User ID or 'SYSTEM'
        timestamp:
          type: string
          format: date-time
        details:
          type: object
          description: Contextual data (e.g., oldStatus, newStatus, notes)
      required:
        - id
        - claimId
        - action
        - timestamp

    # Transition Request Schemas
    GenericTransitionRequest:
      type: object
      properties:
        action:
          type: string
          enum: [REVIEW, REQUEST_INFO, ESCALATE]
        notes:
          type: string
          description: Optional notes for the action.
      required:
        - action

    ApproveRequest:
      type: object
      properties:
        action:
          type: string
          enum: [APPROVE]
        notes:
          type: string
      required:
        - action
      description: |
        Approval logic:
        1. Checks Policy Active.
        2. Checks Fraud Score <= 70.
        3. Checks Assessor Threshold (Auto-escalates if exceeded).
        4. Auto-queues payment if Amount <= 5000 AND Fraud < 30.

    RejectRequest:
      type: object
      properties:
        action:
          type: string
          enum: [REJECT]
        notes:
          type: string
          minLength: 1
          description: Rejection reason is mandatory.
      required:
        - action
        - notes

    PaginatedClaims:
      type: object
      properties:
        items:
          type: array
          items:
            $ref: '#/components/schemas/Claim'
        total:
          type: integer
        page:
          type: integer
        pageSize:
          type: integer
      required:
        - items
        - total
        - page
        - pageSize

    PaginatedPayments:
      type: object
      properties:
        items:
          type: array
          items:
            $ref: '#/components/schemas/Payment'
        total:
          type: integer
        page:
          type: integer
        pageSize:
          type: integer
      required:
        - items
        - total
        - page
        - pageSize
```
