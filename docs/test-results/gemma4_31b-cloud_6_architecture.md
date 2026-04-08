This system follows a **Transactional Outbox Pattern**. Instead of sending the webhook directly during a request, we persist the event to a database and use a worker to process it. This ensures that if the process crashes, the event is not lost.

### 1. Database Schema (PostgreSQL)

We use two tables: `webhook_subscriptions` to store destination URLs and secrets, and `webhook_events` to track the state and retry logic of each delivery.

```sql
-- Subscription settings for each customer/endpoint
CREATE TABLE webhook_subscriptions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    target_url TEXT NOT NULL,
    secret_key TEXT NOT NULL, -- Used for HMAC signing
    rate_limit_per_minute INTEGER DEFAULT 60,
    is_active BOOLEAN DEFAULT true,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT now()
);

-- The delivery queue/log
CREATE TABLE webhook_events (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    subscription_id UUID REFERENCES webhook_subscriptions(id),
    payload JSONB NOT NULL,
    status TEXT NOT NULL CHECK (status IN ('pending', 'delivered', 'failed', 'dlq')),
    retry_count INTEGER DEFAULT 0,
    next_attempt_at TIMESTAMP WITH TIME ZONE DEFAULT now(),
    last_http_status INTEGER,
    error_message TEXT,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT now(),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT now()
);

-- Index for the worker to find ready-to-send events efficiently
CREATE INDEX idx_webhook_pending ON webhook_events (next_attempt_at) 
WHERE status = 'pending';
```

---

### 2. Core Delivery Service (TypeScript)

This implementation assumes the use of `axios` for HTTP and `crypto` for signatures.

```typescript
import axios, { AxiosError } from 'axios';
import crypto from 'crypto';
import { Pool } from 'pg';

interface WebhookEvent {
  id: string;
  subscription_id: string;
  payload: object;
  retry_count: number;
}

interface Subscription {
  target_url: string;
  secret_key: string;
  rate_limit_per_minute: number;
}

class WebhookDeliveryService {
  private pool: Pool;
  private MAX_RETRIES = 10;

  constructor(dbPool: Pool) {
    this.pool = dbPool;
  }

  /**
   * Generates HMAC-SHA256 signature for the payload
   */
  private generateSignature(payload: string, secret: string): string {
    return crypto
      .createHmac('sha256', secret)
      .update(payload)
      .digest('hex');
  }

  /**
   * Calculates exponential backoff with jitter
   * formula: (2^retry * 1000ms) + random jitter
   */
  private calculateNextAttempt(retryCount: number): Date {
    const delayMs = Math.pow(2, retryCount) * 1000 + Math.random() * 1000;
    return new Date(Date.now() + delayMs);
  }

  async processEvent(event: WebhookEvent) {
    const subResult = await this.pool.query(
      'SELECT target_url, secret_key, rate_limit_per_minute FROM webhook_subscriptions WHERE id = $1',
      [event.subscription_id]
    );
    const sub: Subscription = subResult.rows[0];

    if (!sub) {
      await this.moveToDLQ(event.id, 'Subscription not found');
      return;
    }

    const body = JSON.stringify(event.payload);
    const signature = this.generateSignature(body, sub.secret_key);

    try {
      const response = await axios.post(sub.target_url, body, {
        headers: {
          'Content-Type': 'application/json',
          'X-Webhook-Signature': signature,
          'X-Webhook-Event-Id': event.id,
        },
        timeout: 5000, // 5s timeout to prevent hanging workers
      });

      await this.markAsDelivered(event.id, response.status);
    } catch (error) {
      await this.handleFailure(event, error as AxiosError);
    }
  }

  private async handleFailure(event: WebhookEvent, error: AxiosError) {
    const status = error.response?.status || 500;
    const isRetryable = status >= 500 || status === 429;

    if (isRetryable && event.retry_count < this.MAX_RETRIES) {
      const nextAttempt = this.calculateNextAttempt(event.retry_count);
      await this.pool.query(
        `UPDATE webhook_events 
         SET retry_count = retry_count + 1, 
             next_attempt_at = $1, 
             last_http_status = $2, 
             error_message = $3 
         WHERE id = $4`,
        [nextAttempt, status, error.message, event.id]
      );
    } else {
      await this.moveToDLQ(event.id, `Max retries reached or non-retryable status: ${status}`);
    }
  }

  private async markAsDelivered(id: string, status: number) {
    await this.pool.query(
      'UPDATE webhook_events SET status = $1, last_http_status = $2, updated_at = now() WHERE id = $3',
      ['delivered', status, id]
    );
  }

  private async moveToDLQ(id: string, reason: string) {
    await this.pool.query(
      'UPDATE webhook_events SET status = $1, error_message = $2, updated_at = now() WHERE id = $3',
      ['dlq', reason, id]
    );
  }
}
```

---

### 3. Key Implementation Details for Reliability

#### At-Least-Once Delivery
To prevent losing events, the system uses a **State Machine** in the database. 
- **Atomic Commit:** The event is saved to `webhook_events` in the same transaction as the business logic that triggered it.
- **Worker Locking:** When the worker fetches events, it should use `SELECT ... FOR UPDATE SKIP LOCKED`. This ensures multiple worker nodes don't process the same event simultaneously.

#### Rate Limiting
Implementing rate limiting strictly in the delivery service can be slow. The recommended approach is:
1. **Token Bucket:** Use Redis to store a bucket for each `subscription_id`.
2. **Pre-Check:** Before `processEvent` is called, check the Redis bucket. If empty, reschedule the event in the DB by incrementing `next_attempt_at` by 1 minute.

#### Signature Verification (Client Side)
The client receives the `X-Webhook-Signature`. They should:
1. Compute `HMAC-SHA256(payload, secret)`.
2. Use a **constant-time comparison** function (e.g., `crypto.timingSafeEqual`) to compare their computed hash with the header to prevent timing attacks.

#### Handling "Zombie" Deliveries
If a worker crashes mid-delivery, the event remains as `pending` but `next_attempt_at` will pass. 
- **Watchdog:** Implement a periodic cleanup job that finds events that have been `pending` for more than 10 minutes without an update and resets them to be retried.

#### Throughput Scaling
For high volume, move the `webhook_events` table to a partitioned table by `created_at` (daily) and use a distributed queue (like RabbitMQ or SQS) to trigger the `processEvent` logic, while keeping the DB as the "Source of Truth" for delivery status.
