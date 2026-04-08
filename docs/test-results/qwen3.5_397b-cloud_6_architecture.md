This design focuses on reliability, concurrency, and observability. It uses a **Worker Polling Pattern** with database-level locking to ensure horizontal scalability without external queue dependencies (like RabbitMQ/SQS), keeping the architecture simple yet robust.

### 1. Database Schema (PostgreSQL)

This schema prioritizes indexing for the worker polling loop and stores sufficient context for debugging failures.

```sql
-- Enable uuid extension for unique IDs
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";

-- 1. Subscriptions: Configuration per destination
CREATE TABLE webhook_subscriptions (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    target_url TEXT NOT NULL,
    secret_key TEXT NOT NULL, -- Used for HMAC signing
    is_active BOOLEAN DEFAULT TRUE,
    max_retries INT DEFAULT 5,
    rate_limit_requests INT DEFAULT 100, -- Requests allowed
    rate_limit_window_sec INT DEFAULT 60, -- Per window
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW()
);

-- 2. Events: The payload to be sent (decoupled from delivery attempts)
CREATE TABLE webhook_events (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    event_type TEXT NOT NULL,
    payload JSONB NOT NULL,
    created_at TIMESTAMPTZ DEFAULT NOW()
);

-- 3. Deliveries: The state machine for at-least-once delivery
CREATE TABLE webhook_deliveries (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    subscription_id UUID NOT NULL REFERENCES webhook_subscriptions(id),
    event_id UUID NOT NULL REFERENCES webhook_events(id),
    
    -- State Machine: 'pending', 'sending', 'success', 'retrying', 'failed_permanent'
    status TEXT NOT NULL DEFAULT 'pending', 
    
    attempt_count INT NOT NULL DEFAULT 0,
    last_attempt_at TIMESTAMPTZ,
    next_retry_at TIMESTAMPTZ, -- Critical for polling
    
    last_http_status INT,
    last_error_message TEXT,
    response_body_preview TEXT, -- Truncated for debugging
    
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW()
);

-- Indexes for Performance & Concurrency
-- Worker polls for 'pending' or 'retrying' where next_retry_at <= now
CREATE INDEX idx_deliveries_poll ON webhook_deliveries (status, next_retry_at) 
    WHERE status IN ('pending', 'retrying');

-- Foreign key lookup
CREATE INDEX idx_deliveries_subscription ON webhook_deliveries (subscription_id);

-- Rate Limit Tracking (Simple DB-based sliding window for moderate scale)
-- For high scale (>1k req/s), move this logic to Redis
CREATE TABLE webhook_rate_limits (
    subscription_id UUID PRIMARY KEY REFERENCES webhook_subscriptions(id),
    request_count INT DEFAULT 0,
    window_start_at TIMESTAMPTZ DEFAULT NOW()
);
```

### 2. Core Delivery Service (TypeScript/Node.js)

This service implements the worker loop, backoff logic, signing, and rate limiting.

```typescript
import { Pool, PoolClient } from 'pg';
import crypto from 'crypto';
import axios, { AxiosError } from 'axios';
import { EventEmitter } from 'events';

// --- Types ---
type DeliveryStatus = 'pending' | 'sending' | 'success' | 'retrying' | 'failed_permanent';

interface WebhookEvent {
  id: string;
  event_type: string;
  payload: any;
}

interface Subscription {
  id: string;
  target_url: string;
  secret_key: string;
  max_retries: number;
  rate_limit_requests: number;
  rate_limit_window_sec: number;
}

interface DeliveryJob {
  id: string;
  subscription_id: string;
  event_id: string;
  attempt_count: number;
  target_url: string;
  secret_key: string;
  max_retries: number;
  payload: any;
  event_type: string;
}

// --- Configuration ---
const CONFIG = {
  POLL_INTERVAL_MS: 1000,
  BATCH_SIZE: 20,
  REQUEST_TIMEOUT_MS: 5000,
  BASE_BACKOFF_MS: 1000,
  MAX_BACKOFF_MS: 300000, // 5 minutes
  JITTER_MS: 1000,
};

export class WebhookDeliveryService extends EventEmitter {
  private pool: Pool;
  private isRunning = false;

  constructor(pool: Pool) {
    super();
    this.pool = pool;
  }

  /**
   * Entry point to queue a new webhook event for all active subscriptions.
   * In production, you might filter subscriptions by event type.
   */
  async queueEvent(eventType: string, payload: any): Promise<void> {
    const client = await this.pool.connect();
    try {
      await client.query('BEGIN');
      
      // 1. Create Event
      const eventRes = await client.query(
        'INSERT INTO webhook_events (event_type, payload) VALUES ($1, $2) RETURNING id',
        [eventType, JSON.stringify(payload)]
      );
      const eventId = eventRes.rows[0].id;

      // 2. Create Delivery Jobs for all active subscriptions
      // We create one delivery row per subscription to track individual delivery state
      await client.query(
        `INSERT INTO webhook_deliveries (subscription_id, event_id, status, next_retry_at)
         SELECT id, $1, 'pending', NOW() 
         FROM webhook_subscriptions 
         WHERE is_active = TRUE`,
        [eventId]
      );

      await client.query('COMMIT');
    } catch (err) {
      await client.query('ROLLBACK');
      throw err;
    } finally {
      client.release();
    }
  }

  /**
   * Main Worker Loop
   */
  public start(): void {
    if (this.isRunning) return;
    this.isRunning = true;
    this.workerLoop().catch(console.error);
  }

  public stop(): void {
    this.isRunning = false;
  }

  private async workerLoop(): Promise<void> {
    while (this.isRunning) {
      try {
        const jobs = await this.fetchDueJobs();
        if (jobs.length === 0) {
          await this.sleep(CONFIG.POLL_INTERVAL_MS);
          continue;
        }

        // Process jobs in parallel (limited concurrency)
        await Promise.all(jobs.map((job) => this.processJob(job)));
      } catch (err) {
        console.error('Worker loop error:', err);
        await this.sleep(CONFIG.POLL_INTERVAL_MS);
      }
    }
  }

  /**
   * Fetches jobs that are due, locking them to prevent duplicate processing.
   * Uses SKIP LOCKED for high concurrency.
   */
  private async fetchDueJobs(): Promise<DeliveryJob[]> {
    const client = await this.pool.connect();
    try {
      await client.query('BEGIN');

      // Select due jobs and lock them
      const query = `
        SELECT 
          d.id, d.subscription_id, d.event_id, d.attempt_count,
          s.target_url, s.secret_key, s.max_retries,
          e.payload, e.event_type
        FROM webhook_deliveries d
        JOIN webhook_subscriptions s ON d.subscription_id = s.id
        JOIN webhook_events e ON d.event_id = e.id
        WHERE d.status IN ('pending', 'retrying')
          AND d.next_retry_at <= NOW()
        ORDER BY d.next_retry_at ASC
        LIMIT $1
        FOR UPDATE SKIP LOCKED
      `;
      
      const res = await client.query(query, [CONFIG.BATCH_SIZE]);
      
      // Update status to 'sending' immediately to hold lock and indicate work in progress
      if (res.rows.length > 0) {
        const ids = res.rows.map((r: any) => r.id);
        await client.query(
          `UPDATE webhook_deliveries SET status = 'sending' WHERE id = ANY($1)`,
          [ids]
        );
      }

      await client.query('COMMIT');
      return res.rows;
    } catch (err) {
      await client.query('ROLLBACK');
      throw err;
    } finally {
      client.release();
    }
  }

  private async processJob(job: DeliveryJob): Promise<void> {
    const client = await this.pool.connect();
    try {
      // 1. Rate Limit Check
      const isAllowed = await this.checkRateLimit(client, job.subscription_id, job.target_url);
      if (!isAllowed) {
        // Reschedule for later (e.g., +1 minute) without incrementing attempt count
        await this.rescheduleJob(client, job.id, 60000, false);
        return;
      }

      // 2. Send Request
      const startTime = Date.now();
      let httpStatus = 0;
      let responseBody = '';
      
      try {
        const signature = this.generateSignature(job.secret_key, job.payload, startTime);
        const response = await axios.post(job.target_url, job.payload, {
          headers: {
            'Content-Type': 'application/json',
            'X-Webhook-Signature': signature,
            'X-Webhook-Timestamp': startTime.toString(),
          },
          timeout: CONFIG.REQUEST_TIMEOUT_MS,
          validateStatus: () => true, // Accept any status code
        });
        httpStatus = response.status;
        responseBody = response.data ? String(response.data).substring(0, 500) : '';
      } catch (err: any) {
        // Network errors, timeouts, DNS failures
        responseBody = err.message || 'Unknown network error';
      }

      // 3. Handle Response
      const isSuccess = httpStatus >= 200 && httpStatus < 300;

      if (isSuccess) {
        await this.finalizeDelivery(client, job.id, 'success', httpStatus, responseBody);
      } else {
        await this.handleFailure(client, job, httpStatus, responseBody);
      }
    } catch (err) {
      console.error(`Job ${job.id} processing error:`, err);
      // Critical DB error, let the retry logic pick it up later or alert
    } finally {
      client.release();
    }
  }

  private async handleFailure(
    client: PoolClient, 
    job: DeliveryJob, 
    status: number, 
    error: string
  ): Promise<void> {
    const nextAttempt = job.attempt_count + 1;

    if (nextAttempt > job.max_retries) {
      // Move to Dead Letter Queue
      await this.finalizeDelivery(client, job.id, 'failed_permanent', status, error);
      this.emit('dlq', { jobId: job.id, reason: 'max_retries_exceeded' });
    } else {
      // Calculate Backoff
      const delayMs = this.calculateBackoff(nextAttempt);
      await this.rescheduleJob(client, job.id, delayMs, true, status, error);
    }
  }

  private async rescheduleJob(
    client: PoolClient,
    jobId: string,
    delayMs: number,
    incrementAttempt: boolean,
    status?: number,
    error?: string
  ): Promise<void> {
    const query = `
      UPDATE webhook_deliveries 
      SET status = 'retrying',
          next_retry_at = NOW() + INTERVAL '${delayMs} milliseconds',
          attempt_count = attempt_count + ${incrementAttempt ? 1 : 0},
          last_attempt_at = NOW(),
          last_http_status = $2,
          last_error_message = $3
      WHERE id = $1
    `;
    await client.query(query, [jobId, status || null, error || null]);
  }

  private async finalizeDelivery(
    client: PoolClient,
    jobId: string,
    status: DeliveryStatus,
    httpStatus?: number,
    responsePreview?: string
  ): Promise<void> {
    await client.query(
      `UPDATE webhook_deliveries 
       SET status = $2, last_http_status = $3, response_body_preview = $4, updated_at = NOW()
       WHERE id = $1`,
      [jobId, status, httpStatus || null, responsePreview || null]
    );
  }

  // --- Helpers ---

  private calculateBackoff(attempt: number): number {
    const base = CONFIG.BASE_BACKOFF_MS;
    const max = CONFIG.MAX_BACKOFF_MS;
    const jitter = Math.floor(Math.random() * CONFIG.JITTER_MS);
    
    // Exponential: 2^attempt * base
    const delay = Math.min(base * Math.pow(2, attempt), max);
    return delay + jitter;
  }

  private generateSignature(secret: string, payload: any, timestamp: number): string {
    const payloadStr = JSON.stringify(payload);
    const signedPayload = `${timestamp}.${payloadStr}`;
    return crypto
      .createHmac('sha256', secret)
      .update(signedPayload)
      .digest('hex');
  }

  /**
   * Simple DB-based Rate Limiter (Sliding Window approximation)
   * Note: For high throughput, replace this with Redis INCR/EXPIRE.
   */
  private async checkRateLimit(
    client: PoolClient, 
    subId: string, 
    url: string
  ): Promise<boolean> {
    // Fetch current window state
    const res = await client.query(
      `SELECT request_count, window_start_at FROM webhook_rate_limits 
       WHERE subscription_id = $1`,
      [subId]
    );

    const now = new Date();
    let count = 0;
    let windowStart = now;

    // Retrieve subscription config (simplified: assuming cached or fetched earlier)
    // In prod, fetch config once per batch to avoid N+1
    const subConfig = await client.query(
      `SELECT rate_limit_requests, rate_limit_window_sec FROM webhook_subscriptions WHERE id = $1`,
      [subId]
    );
    const limit = subConfig.rows[0].rate_limit_requests;
    const windowSec = subConfig.rows[0].rate_limit_window_sec;

    if (res.rows.length > 0) {
      count = res.rows[0].request_count;
      windowStart = new Date(res.rows[0].window_start_at);
      
      // Check if window expired
      const elapsedSec = (now.getTime() - windowStart.getTime()) / 1000;
      if (elapsedSec > windowSec) {
        count = 0;
        windowStart = now;
      }
    }

    if (count >= limit) {
      return false; // Rate limited
    }

    // Increment counter
    if (res.rows.length > 0) {
      await client.query(
        `UPDATE webhook_rate_limits SET request_count = request_count + 1 WHERE subscription_id = $1`,
        [subId]
      );
    } else {
      await client.query(
        `INSERT INTO webhook_rate_limits (subscription_id, request_count, window_start_at) 
         VALUES ($1, 1, $2)`,
        [subId, now]
      );
    }

    return true;
  }

  private sleep(ms: number): Promise<void> {
    return new Promise((resolve) => setTimeout(resolve, ms));
  }
}
```

### 3. Key Implementation Details for Reliability

#### 1. Concurrency & Locking (`SKIP LOCKED`)
The most critical part of scaling this worker is the `SELECT ... FOR UPDATE SKIP LOCKED` query in `fetchDueJobs`.
*   **Why:** If you run multiple worker instances (e.g., in Kubernetes), they will all poll the DB at the same time. Without locking, two workers might pick up the same job.
*   **Mechanism:** `FOR UPDATE` locks the rows. `SKIP LOCKED` tells Postgres to ignore rows locked by other workers and immediately return available rows. This creates a natural, contention-free queue without needing Redis/RabbitMQ.

#### 2. At-Least-Once Guarantee
*   **State Machine:** The `webhook_deliveries` table acts as the source of truth. A job is never deleted upon success; it is marked `success`.
*   **Crash Recovery:** If a worker crashes after picking a job (`status = 'sending'`) but before updating it to `success` or `retrying`, the job remains in `sending` state.
*   **Stuck Job Handling:** In a production system, you need a separate "watchdog" process that resets jobs stuck in `sending` state for too long (e.g., > 10 mins) back to `retrying` so they aren't lost forever.

#### 3. Signature Verification (HMAC-SHA256)
*   **Provider Side:** We sign `timestamp.payload` using the secret. Including the timestamp prevents replay attacks (the consumer should reject requests older than 5 minutes).
*   **Consumer Side:** The consumer recalculates the hash using their stored secret. If it matches, the payload is authentic and hasn't been tampered with.

#### 4. Rate Limiting Strategy
*   **Implementation:** The code above uses a DB-based sliding window.
*   **Production Warning:** Under high load, updating `webhook_rate_limits` on every request causes row contention on the `subscriptions` table.
*   **Recommendation:** For >500 requests/second, move rate limiting to **Redis**. Use the `INCR` and `EXPIRE` commands. The `checkRateLimit` function should check Redis first. If Redis says "GO", then proceed to DB locking for the job.

#### 5. Exponential Backoff with Jitter
*   **Formula:** `min(base * 2^attempt, max) + random_jitter`.
*   **Why Jitter:** Without jitter, if your system fails (e.g., downstream API down), all retries happen at the exact same millisecond on the next power-of-2 interval, creating a "thundering herd" that can crush the recovering service. Jitter spreads the load.

#### 6. Dead Letter Queue (DLQ)
*   **Logic:** When `attempt_count > max_retries`, status becomes `failed_permanent`.
*   **Action:** The system emits a `dlq` event. You should have a separate admin dashboard or listener that picks up `failed_permanent` jobs. This allows ops teams to inspect the payload/error and manually replay specific webhooks without re-queueing the entire event stream.

#### 7. Payload Storage
*   **Decoupling:** Note that `webhook_events` stores the payload, and `webhook_deliveries` references it.
*   **Benefit:** If you add a new subscription *after* an event was created, you can easily create new delivery rows referencing the existing event payload without duplicating the JSON data. It also keeps the delivery table narrow and fast for indexing.
