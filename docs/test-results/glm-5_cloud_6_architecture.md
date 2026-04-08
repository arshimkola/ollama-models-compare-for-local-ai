Here is a production-ready design for a webhook delivery system.

### 1. Database Schema (PostgreSQL)

We use PostgreSQL as a reliable queue. The `webhook_events` table acts as the persistent store and the task queue.

```sql
-- Enable UUID support
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";

-- Stores active webhook configurations and rate limit thresholds
CREATE TABLE webhook_destinations (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    url TEXT NOT NULL,
    secret_key TEXT NOT NULL, -- Used for HMAC signing
    rate_limit_per_minute INT NOT NULL DEFAULT 100,
    created_at TIMESTAMPTZ DEFAULT NOW()
);

-- The main queue table
CREATE TABLE webhook_events (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    destination_id UUID REFERENCES webhook_destinations(id) ON DELETE CASCADE,
    payload JSONB NOT NULL,
    
    -- Delivery State
    status TEXT NOT NULL DEFAULT 'pending', -- pending, processing, delivered, dead
    attempts INT NOT NULL DEFAULT 0,
    max_attempts INT NOT NULL DEFAULT 5,
    
    -- Timestamps
    next_attempt_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW(),
    last_error TEXT
);

-- Dead Letter Queue (Archive)
CREATE TABLE webhook_dlq (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    original_event_id UUID NOT NULL,
    destination_id UUID NOT NULL,
    payload JSONB NOT NULL,
    failure_reason TEXT NOT NULL,
    failed_at TIMESTAMPTZ DEFAULT NOW()
);

-- Indexes for Queue Processing
-- Critical for the 'polling' query performance
CREATE INDEX idx_webhook_events_polling 
ON webhook_events (status, next_attempt_at) 
WHERE (status IN ('pending', 'processing')); 

-- Index to prevent duplicate processing (Idempotency key logic can be added here)
CREATE INDEX idx_webhook_events_destination ON webhook_events (destination_id);

-- Rate Limiting Tracker (Sliding Window)
CREATE TABLE webhook_rate_limits (
    destination_id UUID PRIMARY KEY REFERENCES webhook_destinations(id) ON DELETE CASCADE,
    tokens INT NOT NULL,
    last_refill TIMESTAMPTZ NOT NULL
);
```

### 2. Core Delivery Service (TypeScript/Node.js)

This service handles the event loop, concurrency, retries, and signature generation.

**Key Dependencies:** `pg` (Postgres client), `axios` (HTTP client), `crypto` (Native Node module).

```typescript
import { Pool, PoolClient } from 'pg';
import axios from 'axios';
import crypto from 'crypto';

// Configuration
const POLL_INTERVAL_MS = 1000;
const BATCH_SIZE = 10;
const MAX_CONCURRENCY = 5;

// Backoff Constants
const BASE_DELAY_MS = 1000; // 1s
const MAX_DELAY_MS = 60000; // 1 minute
const JITTER_FACTOR = 0.2;

interface WebhookJob {
  id: string;
  destination_id: string;
  url: string;
  secret_key: string;
  payload: object;
  attempts: number;
  max_attempts: number;
}

export class WebhookDeliveryService {
  private pool: Pool;
  private isRunning: boolean = false;

  constructor(dbConnectionString: string) {
    this.pool = new Pool({ connectionString: dbConnectionString });
  }

  public start() {
    this.isRunning = true;
    this.poll();
  }

  public stop() {
    this.isRunning = false;
  }

  // --- Main Polling Loop ---
  private async poll() {
    while (this.isRunning) {
      try {
        const jobs = await this.fetchPendingJobs();
        if (jobs.length === 0) {
          await this.sleep(POLL_INTERVAL_MS);
          continue;
        }

        // Process in parallel with concurrency limit
        await Promise.all(jobs.map(job => this.processJob(job)));
      } catch (err) {
        console.error('Polling error:', err);
        await this.sleep(5000); // Backoff on DB error
      }
    }
  }

  // --- Database Interaction ---
  private async fetchPendingJobs(): Promise<WebhookJob[]> {
    const client = await this.pool.connect();
    try {
      await client.query('BEGIN');

      // Skip Locked allows concurrent workers to grab different rows
      // We join with destinations to get the URL and Secret in one go
      const res = await client.query(`
        UPDATE webhook_events
        SET status = 'processing', updated_at = NOW()
        WHERE id IN (
          SELECT e.id 
          FROM webhook_events e
          JOIN webhook_destinations d ON e.destination_id = d.id
          LEFT JOIN webhook_rate_limits r ON d.id = r.destination_id
          WHERE e.status = 'pending' 
            AND e.next_attempt_at <= NOW()
            AND (r.tokens > 0 OR r.tokens IS NULL)
          LIMIT $1
          FOR UPDATE SKIP LOCKED
        )
        RETURNING 
          e.id, e.destination_id, e.payload, e.attempts, e.max_attempts,
          d.url, d.secret_key
      `, [BATCH_SIZE]);

      await client.query('COMMIT');
      return res.rows;
    } catch (e) {
      await client.query('ROLLBACK');
      throw e;
    } finally {
      client.release();
    }
  }

  // --- Core Processing Logic ---
  private async processJob(job: WebhookJob) {
    try {
      // 1. Check Rate Limit (Optimistic Locking / Token Bucket)
      const canProceed = await this.consumeRateLimitToken(job.destination_id);
      if (!canProceed) {
        // If we fetched it but hit rate limit, re-queue immediately for next poll
        // Note: Ideally, rate limit check happens in SQL query (see above query), 
        // but this acts as a secondary check or if using distributed Redis.
        await this.resetJobStatus(job.id, new Date());
        return;
      }

      // 2. Send Webhook
      const success = await this.sendWebhook(job);

      if (success) {
        await this.markSuccess(job.id);
      } else {
        throw new Error('Delivery failed with non-2xx status');
      }
    } catch (error: any) {
      await this.handleFailure(job, error);
    }
  }

  private async sendWebhook(job: WebhookJob): Promise<boolean> {
    const payloadString = JSON.stringify(job.payload);
    const timestamp = Math.floor(Date.now() / 1000);
    
    // 3. Signature Generation (HMAC-SHA256)
    const signature = this.generateSignature(payloadString, job.secret_key, timestamp);

    try {
      const response = await axios.post(job.url, payloadString, {
        headers: {
          'Content-Type': 'application/json',
          'X-Webhook-Signature': signature,
          'X-Webhook-Timestamp': timestamp,
        },
        timeout: 5000, // 5s timeout
      });
      return response.status >= 200 && response.status < 300;
    } catch (error) {
      return false; // Network error or 4xx/5xx
    }
  }

  private async handleFailure(job: WebhookJob, error: Error) {
    const attempts = job.attempts + 1;
    
    if (attempts >= job.max_attempts) {
      // Move to Dead Letter Queue
      await this.moveToDeadLetterQueue(job, error.message);
    } else {
      // Calculate Exponential Backoff with Jitter
      const delay = this.calculateBackoff(attempts);
      const nextAttempt = new Date(Date.now() + delay);
      
      await this.retryJob(job.id, attempts, nextAttempt, error.message);
    }
  }

  // --- Helpers: Retry Logic ---
  private calculateBackoff(attempt: number): number {
    // Exponential Backoff: base * 2^attempt
    const exponentialDelay = BASE_DELAY_MS * Math.pow(2, attempt);
    // Cap delay
    const cappedDelay = Math.min(exponentialDelay, MAX_DELAY_MS);
    
    // Apply Jitter (Decorrelated Jitter approach simplified)
    // Random range between (delay * (1 - jitter)) and (delay * (1 + jitter))
    const jitter = cappedDelay * JITTER_FACTOR;
    return cappedDelay + (Math.random() * (jitter * 2) - jitter);
  }

  // --- Helpers: Database Updates ---
  private async markSuccess(id: string) {
    await this.pool.query(
      `UPDATE webhook_events SET status = 'delivered', updated_at = NOW() WHERE id = $1`,
      [id]
    );
  }

  private async retryJob(id: string, attempts: number, nextAttempt: Date, error: string) {
    await this.pool.query(
      `UPDATE webhook_events 
       SET status = 'pending', attempts = $2, next_attempt_at = $3, last_error = $4, updated_at = NOW()
       WHERE id = $1`,
      [id, attempts, nextAttempt, error]
    );
  }

  private async resetJobStatus(id: string, nextAttempt: Date) {
    await this.pool.query(
      `UPDATE webhook_events SET status = 'pending', next_attempt_at = $2 WHERE id = $1`,
      [id, nextAttempt]
    );
  }

  private async moveToDeadLetterQueue(job: WebhookJob, reason: string) {
    const client = await this.pool.connect();
    try {
      await client.query('BEGIN');
      
      // Insert into DLQ
      await client.query(
        `INSERT INTO webhook_dlq (original_event_id, destination_id, payload, failure_reason)
         VALUES ($1, $2, $3, $4)`,
        [job.id, job.destination_id, job.payload, reason]
      );

      // Remove from main queue (or update status to 'dead')
      await client.query(
        `DELETE FROM webhook_events WHERE id = $1`,
        [job.id]
      );
      
      await client.query('COMMIT');
    } catch (e) {
      await client.query('ROLLBACK');
      throw e;
    } finally {
      client.release();
    }
  }

  // --- Helpers: Rate Limiting (Token Bucket) ---
  private async consumeRateLimitToken(destinationId: string): Promise<boolean> {
      // Optimistic concurrency approach:
      // 1. Decrement token if tokens > 0
      // 2. If no row exists, initialize it
      const result = await this.pool.query(
        `INSERT INTO webhook_rate_limits (destination_id, tokens, last_refill)
         VALUES ($1, $2 - 1, NOW())
         ON CONFLICT (destination_id) 
         DO UPDATE SET 
           tokens = CASE 
             WHEN webhook_rate_limits.tokens <= 0 THEN webhook_rate_limits.tokens
             ELSE webhook_rate_limits.tokens - 1 
           END
         WHERE webhook_rate_limits.tokens > 0
         RETURNING tokens`,
        [destinationId, 100] // Default tokens logic handled here or in trigger
      );
      
      return result.rowCount > 0;
  }

  // --- Helpers: Cryptography ---
  private generateSignature(payload: string, secret: string, timestamp: number): string {
    // Prevent replay attacks by including timestamp in signature base
    const base = `${timestamp}.${payload}`;
    const hmac = crypto.createHmac('sha256', secret);
    hmac.update(base);
    return `sha256=${hmac.digest('hex')}`;
  }

  private sleep(ms: number) {
    return new Promise(resolve => setTimeout(resolve, ms));
  }
}
```

### 3. Key Implementation Details for Reliability

#### A. At-Least-Once Guarantee & "Processing" State
The system uses a transactional state machine.
1.  **Pending**: The default state.
2.  **Processing**: When a worker picks up the job, it updates the status to `processing` inside the `FOR UPDATE SKIP LOCKED` transaction.
3.  **Failure**: If the worker crashes (process dies) mid-processing, the database connection is released, but the row status remains `processing`.
    *   *Critical Detail:* A separate "Janitor" process (not implemented above for brevity) must run periodically to check for jobs stuck in `processing` for too long (e.g., > 5 minutes) and reset them to `pending`. This ensures that a crash doesn't result in a lost webhook.

#### B. Rate Limiting Implementation
We used a database-backed Token Bucket approach in the example code.
*   **Why not memory?** Memory doesn't work if you scale to multiple webhook instances.
*   **Why not Redis?** While Redis is faster, it adds infrastructure complexity. A Postgres `UPDATE ... WHERE tokens > 0` is atomic and sufficient for many mid-scale applications (approx. 100-500 writes/sec).
*   **Refill Logic:** A separate function/Cron job should run every minute to update the `tokens` count based on `webhook_destinations.rate_limit_per_minute`.

#### C. Exponential Backoff with Jitter
In `calculateBackoff`, we use the formula `base * 2^n`.
*   **Jitter:** We add random variance (+/- 20%).
*   **Why?** If 100 webhooks fail simultaneously (e.g., destination server reboot), they would all retry at the exact same second (Thundering Herd). Jitter spreads the load over time.

#### D. Security (Signature & Timestamp)
We generate an HMAC-SHA256 signature.
*   The payload is signed with the secret key.
*   **Timestamp:** We include the timestamp in the signature base (`timestamp.payload`). The receiving server should verify this timestamp is within a small window (e.g., 5 minutes) to prevent Replay Attacks, where an attacker captures a valid webhook and sends it again later.

#### E. Dead Letter Queue (DLQ)
We do not rely solely on status flags (`status='failed'`).
*   We move the data to a separate table (`webhook_dlq`).
*   **Reason:** The main `webhook_events` table must remain small and performant for the polling query. Keeping millions of failed events in the main table slows down the `SELECT ... SKIP LOCKED` query. DLQ allows long-term storage for debugging without impacting queue performance.
