Here is the complete OpenAPI 3.1 specification in YAML format.

```yaml
openapi: 3.1.0
info:
  title: Claims Processing API
  description: |
    API for managing insurance claims, modernizing the legacy PL/SQL system. 
    This API handles the end-to-end lifecycle of a claim, from submission to payment, 
    including fraud checks, policy verification, and workflow management.
  version: 1.0.0
  contact:
    name: Claims Platform Team
    email: platform-team@insurance.com

servers:
  - url: https://api.insurance.com/v1
    description: Production Server
  - url: https://sandbox.insurance.com/v1
    description: Sandbox Environment

tags:
  - name: Claims
    description: Claim lifecycle management
  - name: Policies
    description: Policy lookup and verification
  - name: Assessors
    description: Claims assessor management
  - name: Payments
    description: Payment processing and queuing
  - name: Audit
    description: Immutable audit logs

security:
  - bearerAuth: []

paths:
  # ==========================================
  # Claims
  # ==========================================
  /claims:
    get:
      tags: [Claims]
      summary: List claims
      description: Retrieve a paginated list of claims. Filterable by status.
      parameters:
        - $ref: '#/components/parameters/PageParam'
        - $ref: '#/components/parameters/LimitParam'
        - name: status
          in: query
          schema:
            $ref: '#/components/schemas/ClaimStatus'
        - name: policyId
          in: query
          schema:
            type: string
            format: uuid
      responses:
        '200':
          description: Pagated list of claims
          headers:
            X-RateLimit-Limit:
              $ref: '#/components/headers/RateLimitLimit'
            X-RateLimit-Remaining:
              $ref: '#/components/headers/RateLimitRemaining'
            Link:
              description: Pagination links
              schema:
                type: string
                example: </claims?page=2>; rel="next", </claims?page=5>; rel="last"
          content:
            application/json:
              schema:
                type: object
                properties:
                  data:
                    type: array
                    items:
                      $ref: '#/components/schemas/ClaimSummary'
                  pagination:
                    $ref: '#/components/schemas/PaginationMeta'
    post:
      tags: [Claims]
      summary: Create a new claim
      description: Creates a claim in DRAFT status.
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/ClaimCreateRequest'
      responses:
        '201':
          description: Claim created successfully
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/Claim'
        '400':
          $ref: '#/components/responses/BadRequest'
        '401':
          $ref: '#/components/responses/Unauthorized'

  /claims/{claimId}:
    parameters:
      - name: claimId
        in: path
        required: true
        schema:
          type: string
          format: uuid
    get:
      tags: [Claims]
      summary: Get claim details
      responses:
        '200':
          description: Claim details
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/Claim'
        '404':
          $ref: '#/components/responses/NotFound'
    put:
      tags: [Claims]
      summary: Update claim details
      description: Update claim data. Only allowed while in DRAFT or PENDING_INFO status.
      requestBody:
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/ClaimCreateRequest'
      responses:
        '200':
          description: Claim updated
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/Claim'
        '409':
          description: Conflict - Claim cannot be edited in current state

  /claims/{claimId}/actions/submit:
    post:
      tags: [Claims]
      summary: Submit a claim
      description: |
        Transitions claim to SUBMITTED status. 
        System performs initial validation, fraud scoring, and assessor assignment.
        Auto-escalates if amount exceeds assessor threshold.
      parameters:
        - name: claimId
          in: path
          required: true
          schema:
            type: string
            format: uuid
      responses:
        '200':
          description: Claim submitted successfully
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/Claim'
        '422':
          description: Business rule violation
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ProblemDetails'

  /claims/{claimId}/actions/review:
    post:
      tags: [Claims]
      summary: Begin review process
      description: Transitions claim to UNDER_REVIEW. Only valid from SUBMITTED or ESCALATED states.
      parameters:
        - name: claimId
          in: path
          required: true
          schema:
            type: string
            format: uuid
      responses:
        '200':
          description: Claim under review
        '409':
          description: Invalid status transition

  /claims/{claimId}/actions/approve:
    post:
      tags: [Claims]
      summary: Approve a claim
      description: |
        Approves the claim and triggers payment processing.
        **Business Rules Enforced:**
        * Policy must be active.
        * Fraud score must be <= 70.
        * Claim amount must be within assessor threshold (unless escalated).
      parameters:
        - name: claimId
          in: path
          required: true
          schema:
            type: string
            format: uuid
      responses:
        '200':
          description: Claim approved
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/Claim'
        '422':
          description: Approval criteria not met (e.g. Fraud score too high, Policy inactive)
          content:
            application/json:
              example:
                type: /errors/claim-approval-violation
                title: Approval Failed
                status: 422
                detail: "Claim cannot be approved due to fraud score of 85."

  /claims/{claimId}/actions/reject:
    post:
      tags: [Claims]
      summary: Reject a claim
      parameters:
        - name: claimId
          in: path
          required: true
          schema:
            type: string
            format: uuid
      requestBody:
        required: true
        description: Rejection requires mandatory notes.
        content:
          application/json:
            schema:
              type: object
              required: [notes]
              properties:
                notes:
                  type: string
                  minLength: 10
      responses:
        '200':
          description: Claim rejected

  /claims/{claimId}/actions/request-info:
    post:
      tags: [Claims]
      summary: Request additional information
      description: Sets status to PENDING_INFO.
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
              type: object
              properties:
                message:
                  type: string
      responses:
        '200':
          description: Information requested

  # ==========================================
  # Policies
  # ==========================================
  /policies/{policyId}:
    get:
      tags: [Policies]
      summary: Get policy details
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

  /policies/{policyId}/verify:
    get:
      tags: [Policies]
      summary: Verify policy status
      description: Lightweight check to verify if a policy is active for claim approval.
      parameters:
        - name: policyId
          in: path
          required: true
          schema:
            type: string
      responses:
        '200':
          description: Policy verification result
          content:
            application/json:
              schema:
                type: object
                properties:
                  isActive:
                    type: boolean
                  policyId:
                    type: string
                  holderName:
                    type: string

  # ==========================================
  # Assessors
  # ==========================================
  /assessors/me:
    get:
      tags: [Assessors]
      summary: Get current assessor profile
      description: Returns the authenticated assessor's details, including authorization level and threshold limits.
      responses:
        '200':
          description: Assessor profile
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/Assessor'

  # ==========================================
  # Payments
  # ==========================================
  /payments:
    get:
      tags: [Payments]
      summary: List payments
      description: Retrieve a list of payments, optionally filtered by claim ID or status.
      parameters:
        - name: claimId
          in: query
          schema:
            type: string
            format: uuid
        - name: status
          in: query
          schema:
            $ref: '#/components/schemas/PaymentStatus'
      responses:
        '200':
          description: List of payments
          content:
            application/json:
              schema:
                type: array
                items:
                  $ref: '#/components/schemas/Payment'

  # ==========================================
  # Audit
  # ==========================================
  /audit-log:
    get:
      tags: [Audit]
      summary: Get audit trail
      description: Retrieve an immutable log of actions for a specific claim.
      parameters:
        - name: claimId
          in: query
          required: true
          schema:
            type: string
            format: uuid
        - $ref: '#/components/parameters/PageParam'
        - $ref: '#/components/parameters/LimitParam'
      responses:
        '200':
          description: Audit log entries
          content:
            application/json:
              schema:
                type: array
                items:
                  $ref: '#/components/schemas/AuditEntry'

components:
  securitySchemes:
    bearerAuth:
      type: http
      scheme: bearer
      bearerFormat: JWT
      description: JWT Bearer token issued by the Identity Provider.

  parameters:
    PageParam:
      name: page
      in: query
      description: Page number for pagination
      schema:
        type: integer
        default: 1
    LimitParam:
      name: limit
      in: query
      description: Number of items per page
      schema:
        type: integer
        default: 20
        maximum: 100

  headers:
    RateLimitLimit:
      description: The number of allowed requests in the current period
      schema:
        type: integer
    RateLimitRemaining:
      description: The number of remaining requests in the current period
      schema:
        type: integer
    RateLimitReset:
      description: The time at which the rate limit resets (UTC epoch seconds)
      schema:
        type: integer

  responses:
    BadRequest:
      description: Bad Request
      content:
        application/json:
          schema:
            $ref: '#/components/schemas/ProblemDetails'
    Unauthorized:
      description: Unauthorized
      content:
        application/json:
          schema:
            $ref: '#/components/schemas/ProblemDetails'
    NotFound:
      description: Resource not found
      content:
        application/json:
          schema:
            $ref: '#/components/schemas/ProblemDetails'

  schemas:
    # --- Enums ---
    ClaimStatus:
      type: string
      enum: [DRAFT, SUBMITTED, UNDER_REVIEW, ESCALATED, PENDING_INFO, APPROVED, REJECTED, PAID]
      description: |
        Lifecycle status of the claim.
        * DRAFT: Initial state.
        * SUBMITTED: Awaiting initial review.
        * ESCALATED: Amount exceeds assessor threshold.
        
    AssessorLevel:
      type: string
      enum: [BASIC, STANDARD, PREMIUM, GOLD, PLATINUM]
      description: |
        Authorization levels determining claim approval limits.
        * BASIC: $3,000
        * STANDARD: $8,000
        * PREMIUM: $15,000
        * GOLD/PLATINUM: $25,000

    PaymentStatus:
      type: string
      enum: [QUEUED, PROCESSING, COMPLETED, FAILED]

    # --- Core Objects ---
    Money:
      type: object
      description: Monetary value object
      properties:
        amount:
          type: number
          format: double
          example: 1500.00
        currency:
          type: string
          default: USD

    ClaimCreateRequest:
      type: object
      required:
        - policyId
        - claimantId
        - incidentDate
        - amount
      properties:
        policyId:
          type: string
          format: uuid
        claimantId:
          type: string
        incidentDate:
          type: string
          format: date-time
        amount:
          $ref: '#/components/schemas/Money'
        description:
          type: string
        documents:
          type: array
          items:
            type: string
            format: uri

    ClaimSummary:
      type: object
      properties:
        id:
          type: string
          format: uuid
        status:
          $ref: '#/components/schemas/ClaimStatus'
        amount:
          $ref: '#/components/schemas/Money'
        createdDate:
          type: string
          format: date-time
        slaBreached:
          type: boolean
          description: True if claim is over 30 days old.

    Claim:
      type: object
      allOf:
        - $ref: '#/components/schemas/ClaimSummary'
        - type: object
          properties:
            policyId:
              type: string
              format: uuid
            assessorId:
              type: string
            fraudScore:
              type: integer
              minimum: 0
              maximum: 100
              description: Calculated fraud risk score.
            isEscalated:
              type: boolean
            notes:
              type: array
              items:
                type: object
                properties:
                  createdAt:
                    type: string
                    format: date-time
                  author:
                    type: string
                  content:
                    type: string

    Policy:
      type: object
      description: Insurance policy details
      properties:
        id:
          type: string
        holderName:
          type: string
        status:
          type: string
          enum: [ACTIVE, LAPSED, CANCELLED]
        startDate:
          type: string
          format: date
        endDate:
          type: string
          format: date
        claimHistoryCount:
          type: integer
          description: Count of claims in the last 12 months.

    Assessor:
      type: object
      properties:
        id:
          type: string
        username:
          type: string
        level:
          $ref: '#/components/schemas/AssessorLevel'
        approvalThreshold:
          type: number
          description: Maximum claim amount this assessor can approve without escalation.
          example: 8000.00

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
          $ref: '#/components/schemas/Money'
        status:
          $ref: '#/components/schemas/PaymentStatus'
        processedDate:
          type: string
          format: date-time
        isAutoApproved:
          type: boolean
          description: True if automatically queued for small claims (<= $5k and fraud score < 30).

    AuditEntry:
      type: object
      description: Immutable record of a claim action.
      properties:
        id:
          type: string
          format: uuid
        claimId:
          type: string
          format: uuid
        action:
          type: string
          enum: [CREATED, SUBMITTED, REVIEWED, APPROVED, REJECTED, ESCALATED, INFO_REQUESTED]
        performedBy:
          type: string
        timestamp:
          type: string
          format: date-time
        metadata:
          type: object
          additionalProperties: true

    PaginationMeta:
      type: object
      properties:
        currentPage:
          type: integer
        totalPages:
          type: integer
        totalItems:
          type: integer
        itemsPerPage:
          type: integer

    ProblemDetails:
      type: object
      description: RFC 9457 Problem Details for HTTP APIs
      properties:
        type:
          type: string
          format: uri
          description: A URI reference that identifies the problem type.
          example: "/errors/validation-failed"
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
          description: A URI reference that identifies the specific occurrence of the problem.
        errors:
          type: array
          description: Optional list of specific validation errors
          items:
            type: object
            properties:
              field:
                type: string
              message:
                type: string
```
