# System Design Notes

> Use this directory to store your notes after studying each system design topic from Hello Interview.

## 10 Core Designs to Master

Complete these in order. After studying each one, create a markdown file with your notes.

| # | Design | Week | Notes File | Completed? |
|---|--------|------|------------|------------|
| 1 | URL Shortener (bit.ly) | 3 | `01-url-shortener.md` | [ ] |
| 2 | Rate Limiter | 4 | `02-rate-limiter.md` | [ ] |
| 3 | Twitter / News Feed | 5 | `03-news-feed.md` | [ ] |
| 4 | Chat / Messaging System | 6 | `04-chat-system.md` | [ ] |
| 5 | Notification System | 7 | `05-notification-system.md` | [ ] |
| 6 | File Storage (S3/Dropbox) | 8 | `06-file-storage.md` | [ ] |
| 7 | Payment System | 9 | `07-payment-system.md` | [ ] |
| 8 | E-commerce / Order System | 10 | `08-order-system.md` | [ ] |
| 9 | Search Autocomplete | 11 | `09-search-autocomplete.md` | [ ] |
| 10 | Location-Based Service | 12 | `10-location-service.md` | [ ] |

## Note Template

When studying each design, create a file with this structure:

```markdown
# Design: [System Name]

## Requirements
### Functional
- ...
### Non-Functional
- Scale: X users, Y requests/sec
- Latency: < Xms
- Availability: 99.9%

## Capacity Estimation
- DAU: ...
- QPS: ...
- Storage: ...

## High-Level Design
- [Draw/describe the architecture]
- Key components and their interactions

## API Design
- Endpoint 1: ...
- Endpoint 2: ...

## Data Model
- Table/Collection 1: ...
- Table/Collection 2: ...

## Deep Dives
- Scaling strategy
- Caching layer
- Database choice and why
- How to handle failures

## Trade-offs Discussed
- [Trade-off 1]
- [Trade-off 2]

## My PayU Experience Connection
- [How I can relate this to my real work]
```

## System Design Framework (use this in every interview)

### Step 1: Requirements (3-5 min)
- Clarify functional requirements (what does the system do?)
- Clarify non-functional requirements (scale, latency, consistency)
- Ask about constraints (budget, timeline, existing infrastructure)

### Step 2: Capacity Estimation (3-5 min)
- DAU, QPS (read and write separately)
- Storage per day/year
- Bandwidth

### Step 3: High-Level Design (10-15 min)
- Draw the main components
- Show data flow
- Identify the core service

### Step 4: Deep Dive (15-20 min)
- Pick 2-3 areas to go deep
- Database schema design
- Caching strategy
- Scaling approach
- Handle edge cases

### Step 5: Wrap-up (3-5 min)
- Summarize trade-offs
- Discuss monitoring and alerting
- Mention what you'd do with more time

## Key Concepts Quick Reference

| Concept | When to Use |
|---------|-------------|
| Load Balancer | Distribute traffic, high availability |
| CDN | Static content, reduce latency globally |
| Redis Cache | Frequently read data, session storage, rate limiting |
| Message Queue (Kafka) | Async processing, event-driven, decoupling |
| Database Sharding | Horizontal scaling for large datasets |
| Read Replicas | Read-heavy workloads, reduce primary DB load |
| Consistent Hashing | Distributed cache, minimize redistribution |
| WebSockets | Real-time bidirectional communication |
| API Gateway | Single entry point, rate limiting, auth |
| Circuit Breaker | Fault tolerance, prevent cascade failures |

## Your Secret Weapons (real experience to reference)

1. **Kafka** — Used in NACH Service for async webhook delivery with guaranteed processing
2. **Redis** — Implemented in orchestration service, 20% latency reduction, cache-aside pattern
3. **State Machine** — Built for loan workflows and partner onboarding
4. **Rate Limiting** — Implemented for batch processing to protect downstream services
5. **Read-Write Separation** — Achieved 10x query improvement in Digital Lending Suite
6. **Factory + Strategy Pattern** — Multi-vendor insurance service, multi-NACH-type service
7. **Webhook Reliability** — Dynamic configs, retry workflows, HMAC validation
8. **mTLS** — Configured for Google Pay integration
