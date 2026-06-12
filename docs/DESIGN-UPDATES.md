# Post-Roast Design Updates

## Update 1: Added per-user seat hold limit

**Triggered by:** Panel Question 3 - seat holding abuse scenario

**What changed:**
- Added a Redis counter keyed by `seat_hold_user:[userId]` with a TTL of 15 minutes.
- The API checks this counter before attempting any seat lock acquisition.
- If the counter is `>= 8`, the request returns `429 Too Many Requests` immediately.
- The counter increments on successful hold acquisition and decrements when the hold expires or booking confirms.

**Why this is necessary:**
Without this limit, a single authenticated user can open multiple tabs and hold hundreds of seats during the same sale window, starving legitimate buyers of inventory.

**What it costs:**
One additional Redis GET/INCR per seat hold request, which is negligible compared to the existing lock acquisition.

**What it doesn't solve:**
A determined attacker can still use multiple accounts or bots. This is a rate-limit defense, not a full fraud prevention system.

## Update 2: Added SQS outage detection and fallback transparency

**Triggered by:** Panel Question 4 - SQS outage and pending booking visibility

**What changed:**
- Added a `GET /bookings/:id` polling endpoint that returns booking status: pending, confirmed, failed.
- Added CloudWatch alarms for SQS send/receive errors and queue depth changes during sale windows.
- Designed a circuit breaker: if SQS publish fails repeatedly for 60 seconds, the API enters degraded mode and returns a clear error rather than accepting new bookings silently.

**Why this is necessary:**
If SQS becomes unavailable, the user must not be left with a phantom ``202`` response. Explicit status polling and operational alarms make the outage detectable and recoverable.

**What it costs:**
A modest API extension and a small amount of additional application state tracking.

**What it doesn't solve:**
This does not convert an SQS outage into a normal booking flow. It only makes the failure mode visible and bounds the damage.
