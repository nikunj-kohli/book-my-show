# Design Decisions

## Decision: Redis-based seat locking with fallback safety

**Context:** High-concurrency ticket sales require a distributed lock to prevent double-booking and reduce DB contention.

**Options considered:**
1. PostgreSQL row-level `FOR UPDATE` locking - safe but blocks many concurrent seat checks and raises contention.
2. Redis distributed locks only - fast but unsafe if Redis fully fails and application proceeds without protection.
3. Redis locks + DB optimistic guard (chosen) - fast common path plus a secondary DB-level check for correctness.

**Why chosen:**
- Redis SETNX locks are low-latency and support rapid seat hold releases.
- The primary DB remains the source of truth; if Redis is unavailable, we fail the hold request instead of risking a stale lock.
- A versioned seat row or unique booking constraint on the DB prevents double-booking even under high concurrency.

**Tradeoffs accepted:**
- If Redis fails, throughput drops because requests must return 503 instead of proceeding.
- This design does not eliminate all race conditions if the DB schema lacks proper seat constraints.

**Revision trigger:**
- If Redis reliability drops or a ticket sale requires guaranteed lockless operation, move to a stronger DB-based pessimistic lock or a consensus-based lock store.

## Decision: Replica reads for non-critical browsing data

**Context:** Scale event browsing without overloading the primary database.

**Options considered:**
1. All reads from primary - consistent, but poor scalability under event traffic.
2. Strongly-consistent synchronous replicas - lower lag but much higher write latency.
3. Read replicas for non-transactional data only (chosen).

**Why chosen:**
- Event details, seat maps, and historical bookings can tolerate eventual consistency.
- Availability shown to users can be stale, but final booking validation occurs on the primary.

**Tradeoffs accepted:**
- Users may see temporary stale availability counts.
- Replica lag can cause UX mismatch, but not booking correctness.

**Revision trigger:**
- If the product requires strong read-after-write semantics for the same session, add a read-through cache or session-affinity primary reads for recently changed seats.

## Decision: Per-user seat hold limit in Redis

**Context:** Prevent a single user from holding too many seats during a flash sale.

**Options considered:**
1. Global rate limiting per IP - easy but bypassable with multiple tabs or proxies.
2. Business logic enforcement only at checkout - too late and still blocks inventory.
3. Redis counter keyed by user ID before lock acquisition (chosen).

**Why chosen:**
- Enforces a hard cap per user before expensive lock or DB operations.
- Works on the fast path with minimal overhead.

**Tradeoffs accepted:**
- A user may still open multiple accounts and bypass the limit.
- Requires authenticated user context for enforcement.

**Revision trigger:**
- If seat hoarding grows more sophisticated, add user verification, deposit holds, or payment method preauthorization.

## Decision: SQS queue with DLQ and fallback detection

**Context:** Payment processing is asynchronous and may fail due to third-party gateway issues.

**Options considered:**
1. Synchronous payment from the API - higher latency and lower throughput.
2. SQS queue without fallback monitoring - durable, but opaque during outages.
3. SQS queue with DLQ and operational alarms (chosen).

**Why chosen:**
- Decouples booking submission from payment execution.
- Provides durable retry and visibility timeout semantics.
- Enables post-raft recovery with a dead-letter queue for poison messages.

**Tradeoffs accepted:**
- Users see eventual confirmation rather than instant booking success.
- If SQS is unavailable, the API must surface degraded mode instead of silently accepting requests.

**Revision trigger:**
- If the product needs truly synchronous purchase guarantees, move to a hybrid flow with optional immediate payment processing.

## Decision: Cost-aware peak scaling with CloudFront optimization

**Context:** Peak ticket sales should scale without blowing steady-state budget.

**Options considered:**
1. Provisioned capacity only - predictable cost, poor burst handling.
2. Unlimited auto-scaling without budget guard - risky for event spikes.
3. Auto-scaling with Spot fallback and cache optimization (chosen).

**Why chosen:**
- Spot instances and CDN cache tuning reduce event costs.
- Treat peak event traffic as a separate operational cost rather than a monthly steady-state budget.

**Tradeoffs accepted:**
- Spot instances add eviction risk and complexity.
- CloudFront cache rules require careful invalidation for time-sensitive event pages.

**Revision trigger:**
- If monthly budgets tighten, invest more in cache hit ratio, pre-warming, and serverless edge functions.
