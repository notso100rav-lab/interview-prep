# Chapter 1: 15-Day System Design Strategy

> **Estimated study time:** Reference guide | **Priority:** 🟡 Planning tool

---

## Overview

This chapter provides a structured, day-by-day plan to build system design interview skills from the ground up in 15 days. Each day balances theory, practice problems, and review.

---

## Prerequisites

Before starting, ensure you are comfortable with:
- HTTP, TCP/IP basics
- Relational databases and basic SQL
- One backend language (Java / Python / Go)
- Basic understanding of REST APIs

---

## Day-by-Day Plan

### Day 1 — Foundations & Framework

**Goal:** Understand the interview format and the structured approach.

**Topics:**
- System design interview format (45–60 min)
- The RESHADED framework:
  - **R**equirements (functional + non-functional)
  - **E**stimations (QPS, storage, bandwidth)
  - **S**ervice design (API design)
  - **H**igh-level design (components and data flow)
  - **A**rchitecture deep dive (one component in detail)
  - **D**ata design (schema, storage choice)
  - **E**volution (scaling, failure handling)
  - **D**iscussion (trade-offs, alternatives)

**Practice:**
- Design a URL shortener at a conceptual level (no details yet)
- Focus only on requirements gathering and capacity estimation

**Resources:**
- Read: "Grokking the System Design Interview" — Introduction chapter
- Watch: 1 mock system design interview on YouTube

---

### Day 2 — Scalability Basics

**Goal:** Understand horizontal vs vertical scaling and stateless architecture.

**Topics:**
- Vertical vs horizontal scaling
- Stateless services — why services must be stateless to scale
- Load balancers: L4 (TCP) vs L7 (HTTP), round-robin vs least-connections vs IP hash
- CDN: pull vs push, cache-control headers
- DNS and GeoDNS

**Practice:**
- Design a scalable static content delivery system
- Add load balancing and CDN to the URL shortener

**Key questions to answer:**
- How do I ensure my service can scale from 1,000 to 1,000,000 requests/second?
- What makes a service stateless and why does it matter?

---

### Day 3 — Databases: SQL vs NoSQL

**Goal:** Know when to use relational vs non-relational storage.

**Topics:**
- SQL strengths: ACID, JOINs, complex queries, mature tooling
- NoSQL categories: key-value (Redis, DynamoDB), document (MongoDB), column-family (Cassandra), graph (Neo4j)
- CAP theorem (practical interpretation)
- Replication: leader-follower, multi-leader, leaderless
- Sharding: range, hash, directory-based

**Practice:**
- Decide storage for: user profiles, product catalog, session store, activity feed, financial ledger
- Add database layer to URL shortener (justify choice)

**Key trade-off:**
- SQL: consistency + flexibility at the cost of scale
- NoSQL: scale + availability at the cost of consistency / query power

---

### Day 4 — Caching

**Goal:** Know how, what, and when to cache.

**Topics:**
- Cache-aside, write-through, write-behind
- Cache invalidation strategies (TTL, event-driven)
- Redis data structures and use cases
- Cache stampede / thundering herd prevention
- CDN as a cache layer
- When NOT to cache (frequently updated data, user-specific data)

**Practice:**
- Add a caching layer to the URL shortener
- Design a leaderboard service using Redis sorted sets

---

### Day 5 — Asynchronous Communication

**Goal:** Understand message queues and when to use async patterns.

**Topics:**
- Synchronous vs asynchronous communication
- Message queues: Kafka, RabbitMQ, AWS SQS — trade-offs
- Kafka architecture: topics, partitions, consumer groups, offsets
- At-least-once, at-most-once, exactly-once delivery semantics
- Event-driven architecture patterns

**Practice:**
- Design a notification service (email + push + SMS) using message queues
- Add async processing to URL shortener analytics

---

### Day 6 — API Design

**Goal:** Design APIs that are consistent, versioned, and developer-friendly.

**Topics:**
- REST principles: resource naming, HTTP verbs, status codes, HATEOAS
- GraphQL — when it adds value and when it's overkill
- gRPC — when to choose it over REST
- API versioning strategies: URL path, header, query param
- Rate limiting: token bucket, leaky bucket, sliding window
- Pagination: offset, cursor-based (keyset)

**Practice:**
- Design the complete API for the URL shortener
- Design a paginated product search API

---

### Day 7 — Review & Practice (Days 1–6)

**Goal:** Consolidate learning from the first week.

**Activities:**
- [ ] Draw the full architecture diagram for the URL shortener
- [ ] Write a 1-page design document covering all RESHADED components
- [ ] Practice explaining your design out loud (record yourself if possible)
- [ ] Identify 2–3 weakest areas from days 1–6 and review them

---

### Day 8 — Distributed System Patterns

**Goal:** Learn the core patterns for building reliable distributed systems.

**Topics:**
- Circuit breaker pattern (Hystrix, Resilience4j)
- Retry with exponential backoff and jitter
- Bulkhead isolation
- Saga pattern for distributed transactions (choreography vs orchestration)
- Idempotency: idempotency keys, conditional writes
- Distributed locks (Redis SETNX, ZooKeeper)

**Practice:**
- Design a payment processing flow with saga pattern
- Identify failure modes in the URL shortener and add resilience

---

### Day 9 — Data Consistency & Transactions

**Goal:** Understand consistency models and how to maintain data integrity at scale.

**Topics:**
- ACID vs BASE
- Consistency models: strong, eventual, causal, monotonic read
- Two-Phase Commit (2PC) — why it's rarely used in modern systems
- Idempotent writes and optimistic locking
- Event sourcing as an alternative to mutable state
- CQRS (Command Query Responsibility Segregation)

**Practice:**
- Design an e-commerce order system with inventory management
- Discuss consistency trade-offs for each component

---

### Day 10 — Search & Analytics

**Goal:** Know when and how to add search and analytical capabilities.

**Topics:**
- Elasticsearch: inverted index, query DSL, relevance scoring
- Full-text search vs exact-match filtering
- Clickstream analytics: write-heavy append-only logs → Kafka → data warehouse
- Time-series data: InfluxDB, TimescaleDB, Prometheus
- Data pipelines: ETL vs ELT

**Practice:**
- Add a product search feature using Elasticsearch to the e-commerce system
- Design a real-time analytics dashboard for URL shortener (clicks by hour/country)

---

### Day 11 — Microservices Architecture

**Goal:** Understand when microservices are appropriate and how to design service boundaries.

**Topics:**
- Microservices vs monolith — when to split
- Domain-driven design: bounded contexts, aggregates
- Service discovery (Consul, Kubernetes DNS)
- API gateway patterns (Kong, AWS API Gateway)
- Service mesh (Istio, Linkerd)
- Data ownership: each service owns its own database

**Practice:**
- Split the e-commerce system into microservices
- Design the service boundary between Order, Inventory, and Payment services

---

### Day 12 — Real-World Design Problem #1

**Goal:** End-to-end practice under timed conditions.

**Problem:** Design a **Rate Limiter Service** (used by an API gateway)

**Requirements:**
- Functional: limit requests per user/IP per time window
- Non-functional: < 5ms latency, 99.99% availability, 1M QPS

**Time yourself:** 45 minutes

**Review checklist:**
- [ ] Defined functional and non-functional requirements
- [ ] Estimated QPS and storage
- [ ] Chose appropriate algorithm (token bucket / sliding window)
- [ ] Designed for distributed deployment (Redis cluster)
- [ ] Discussed edge cases (Redis failure, clock skew)
- [ ] Mentioned monitoring and alerting

---

### Day 13 — Real-World Design Problem #2

**Goal:** Practice a complex, multi-service design.

**Problem:** Design a **Chat System** (WhatsApp-level scale)

**Requirements:**
- 1:1 and group messaging
- Message delivery receipt (sent, delivered, read)
- Presence (online/offline)
- Push notifications for offline users

**Time yourself:** 50 minutes

**Key decisions to justify:**
- WebSocket vs long polling vs SSE for real-time
- How to route messages to the correct server
- Storage: messages in Cassandra (append-heavy)
- Fanout on write vs read for group messages

---

### Day 14 — Low-Level Design Practice

**Goal:** Practice object-oriented design problems.

**Problems (pick 2, 30 minutes each):**
1. Design a parking lot system
2. Design a library management system
3. Design a vending machine
4. Design an elevator controller

**OOP design checklist:**
- [ ] Identified entities and their relationships
- [ ] Applied SOLID principles
- [ ] Used appropriate design patterns (Strategy, Observer, Factory, etc.)
- [ ] Wrote clean Java interfaces and class skeletons
- [ ] Discussed extensibility

---

### Day 15 — Final Review & Weak Area Deep-Dive

**Goal:** Solidify understanding and prepare mentally for interviews.

**Activities:**
- [ ] Flashcard review: all key concepts, patterns, and trade-offs
- [ ] Redo your weakest Day 1–14 topic without notes
- [ ] Practice the "Trade-offs" section for each design you built
- [ ] Review behavioral questions: system failures you've handled, technical decisions, performance wins
- [ ] Prepare 2 questions to ask your interviewer about their system

**Mental preparation:**
- It's OK to say "I'm not sure, but here's my thinking..."
- The process matters more than the perfect answer
- Ask clarifying questions — interviewers reward structured thinking
- Think aloud throughout the interview

---

## Quick Reference: Capacity Estimation

```
Daily active users (DAU): given in problem
Requests per second: DAU × avg_requests_per_day / 86,400

Storage per day: write_QPS × avg_payload_size × 86,400
Storage for 5 years: storage_per_day × 365 × 5

Read/write ratio: given or assumed (e.g. 100:1 for social media)
Peak traffic: 2–3× average

Bandwidth: RPS × avg_response_size
Cache memory: hot 20% of data (80/20 rule)
```

---

## Common Patterns Cheat Sheet

| Pattern | Use when |
|---------|---------|
| **Cache-aside** | Read-heavy, tolerate stale data |
| **Write-through** | Write-heavy, need fresh cache |
| **Message queue** | Decouple producers/consumers, handle spikes |
| **CQRS** | Read and write models have different shapes |
| **Event sourcing** | Full audit log, time travel, replay |
| **Saga** | Distributed transaction across microservices |
| **Circuit breaker** | Prevent cascade failures |
| **Sharding** | DB too large for one node |
| **Leader election** | Exactly one node performs a task |
| **Rate limiting** | Protect APIs from abuse |
