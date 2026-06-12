# BookMyShow Architecture

## Overview
This document describes the BookMyShow ticketing architecture for Part B of the design roast. It covers the full system flow, component responsibilities, and the key data paths.

## System Components

- **User-facing layer**: Browser / Mobile App → CloudFront CDN
- **Load balancer**: Application Load Balancer (ALB) with SSL termination, health checks, and routing for `/api/*` traffic
- **Application layer**: Node.js API servers behind auto-scaling
- **Cache and lock layer**: Redis cluster used for availability cache and distributed seat locks
- **Queue layer**: SQS payment queue with dead-letter queue (DLQ)
- **Payment worker layer**: ECS Fargate workers processing payment messages
- **Database layer**: PostgreSQL primary with read replicas
- **Notification layer**: SNS publishing to email and SMS channels

## Architecture Diagram

```
Mobile / Browser
      │
      ▼
CloudFront CDN ──── [static assets, event pages: cache hit → user]
      │ (API requests only)
      ▼
Application Load Balancer (ALB)
  - SSL termination
  - Health checks every 10s
  - Rate limit: 200 req/IP/min
      │
      ├──── Node.js API ×1
      ├──── Node.js API ×2    ← Auto-scale group (4–20 instances)
      ├──── Node.js API ×3      Scale-out: CPU > 70% for 2 min
      └──── Node.js API ×N
              │
              ├── READ ──► Redis Cluster (3 nodes, ElastiCache)
              │             ├── Cache: availability:event:cat (TTL 30s)
              │             ├── Cache: event:[id] (TTL 3600s)
              │             └── Locks: seat_lock:[id] (TTL 30s, SETNX)
              │
              ├── WRITE ──► PostgreSQL Primary (RDS db.r6g.xlarge)
              │             └── Replicates to ──► Read Replica 1
              │                                   Read Replica 2
              │
              └── PUBLISH ──► SQS Payment Queue
                               Visibility timeout: 20s
                               Max receive count: 3
                               DLQ: payment-dlq
                                    │
                             ┌──────▼──────────────────┐
                             │ Payment Worker (ECS) ×10 │
                             │ 1. Read from SQS         │
                             │ 2. Call Razorpay API      │
                             │ 3. Update DB             │
                             │ 4. Publish SNS           │
                             │ 5. Delete SQS message    │
                             └──────────────────────────┘
                                    │
                             ┌──────▼────────┐
                             │   AWS SNS     │
                             ├──► SES Email  │
                             └──► SNS SMS    │
                                  (Twilio)   │
                             └──────────────┘
```

## Key data flow labels

- Browser/Mobile → CloudFront: static asset delivery and dynamic API pass-through
- ALB → Node.js APIs: HTTP requests for booking flows and event queries
- Node.js APIs → Redis: availability cache reads and seat lock acquisition via SETNX
- Node.js APIs → PostgreSQL primary: booking writes, payment metadata, hold confirmation
- Node.js APIs → SQS: enqueue payment processing message after seat hold
- Payment workers → SNS: publish booking confirmation notifications
- PostgreSQL primary → read replicas: asynchronous replication for read scaling

## Design emphasis

- Availability count reads may be stale from cache/replica, but final booking consistency is enforced on the primary.
- Redis lock failure must fail fast: no seat hold without a lock.
- SQS provides durable retry and DLQ behavior for payment jobs.
- Notifications are decoupled from booking completion using SNS.
