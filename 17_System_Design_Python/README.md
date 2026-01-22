# System Design Python - Complete Reference

## 📚 Overview

This folder contains comprehensive system design guides for mid-senior backend Python engineers. Each document covers a complete system design problem with Python (Django/FastAPI) implementations, capacity estimations, scalability patterns, and interview preparation.

## 🗂️ Complete File Index

### Foundational Concepts

1. **[01_System_Design_Basics.md](01_System_Design_Basics.md)** ✅
   - 4-step interview framework
   - CAP theorem, ACID vs BASE
   - Horizontal vs vertical scaling
   - Load balancing strategies
   - Database scaling (sharding, replication)
   - Caching layers
   - Message queues
   - Trade-offs and best practices

### Core System Designs

2. **[02_URL_Shortener.md](02_URL_Shortener.md)** ✅ (bit.ly, TinyURL)
   - Base62 encoding vs hashing
   - Django & FastAPI implementations
   - Redis caching strategy
   - Database sharding
   - Rate limiting
   - Analytics tracking

3. **[01_Rate_Limiter.md](01_Rate_Limiter.md) / [01_API_Rate_Limiter.md](01_API_Rate_Limiter.md)** ✅
   - Token bucket algorithm
   - Sliding window
   - Redis-based implementation
   - Distributed rate limiting
   - DDoS protection

4. **[02_Distributed_Cache.md](02_Distributed_Cache.md)** ✅
   - Redis cluster architecture
   - Cache eviction policies (LRU, LFU)
   - Cache-aside vs write-through
   - Consistent hashing
   - Cache stampede prevention

5. **[03_Message_Queue.md](03_Message_Queue.md)** ✅
   - RabbitMQ / Celery patterns
   - At-least-once delivery
   - Dead letter queues
   - Priority queues
   - Async task processing

6. **[04_Load_Balancer.md](04_Load_Balancer.md)** ✅
   - Round robin, least connections
   - Health checks
   - Session persistence
   - Layer 4 vs Layer 7
   - Nginx / HAProxy

7. **[05_Search_Engine.md](05_Search_Engine.md)** ✅
   - Elasticsearch integration
   - Inverted index
   - Full-text search
   - Ranking algorithms
   - Autocomplete

8. **[02_Chat_System.md](02_Chat_System.md) / [06_Chat_System.md](06_Chat_System.md)** ✅
   - WebSocket architecture
   - Message persistence
   - Online status
   - Read receipts
   - Group chat

9. **[07_News_Feed.md](07_News_Feed.md)** ✅ (Twitter/Facebook)
   - Fanout service (push vs pull vs hybrid)
   - Feed generation algorithms
   - Celebrity user problem
   - Real-time updates
   - Ranking algorithms
   - Redis sorted sets

10. **[08_Notification_System.md](08_Notification_System.md)** ✅
    - Multi-channel (email, SMS, push, in-app)
    - Priority queues
    - Template management
    - Retry mechanisms with exponential backoff
    - User preferences
    - Delivery tracking

### Advanced System Designs

9. **09_Video_Streaming.md** (YouTube/Netflix) - Create next
10. **10_E_Commerce_Platform.md** - Planned
11. **11_Ride_Sharing.md** (Uber/Lyft) - Planned
12. **12_File_Storage.md** (Dropbox/Google Drive) - Create next
13. **13_Payment_System.md** - Create next
14. **14_Social_Network.md** - Planned

## 🎯 How to Use This Guide

### For Interview Preparation

1. **Start with Basics** (01): Master the fundamental framework
2. **Practice calculations**: Do back-of-envelope estimates for each system
3. **Understand trade-offs**: Why choose one approach over another
4. **Draw diagrams**: Practice architecture diagrams on whiteboard
5. **Code implementations**: Understand Django/FastAPI patterns

### Study Path by Experience Level

**Mid-Level (4-5 years):**

- Focus on: 01, 02, 03, 04, 07, 08
- Practice: Capacity estimation, basic scalability
- Goal: Explain clear architecture and trade-offs

**Senior Level (6-7 years):**

- Study: All documents
- Focus on: Distributed systems, data consistency, scalability
- Practice: Handling edge cases, failure scenarios
- Goal: Deep technical discussions and optimization strategies

**Staff/Principal (8+ years):**

- Master: All systems + variations
- Focus on: Multi-region, disaster recovery, cost optimization
- Practice: System evolution, migration strategies
- Goal: Guide technical direction and mentor others

## 📊 Common Patterns Across Systems

### Pattern 1: Layered Architecture

```
Client → CDN → Load Balancer → App Servers → Cache → Database
```

Used in: URL Shortener, News Feed, E-Commerce

### Pattern 2: Event-Driven Architecture

```
Service → Message Queue → Workers → Database → Webhooks
```

Used in: Notification System, Payment Processing, File Upload

### Pattern 3: Microservices Pattern

```
API Gateway → Auth Service → Business Services → Data Services
```

Used in: E-Commerce, Social Network, Video Platform

### Pattern 4: Real-Time Communication

```
Client ↔ WebSocket Server ↔ Message Broker ↔ Backend Services
```

Used in: Chat, Live Updates, Gaming

## 🔑 Key Design Principles

### 1. Scalability

- **Horizontal scaling**: Add more servers (stateless design)
- **Vertical scaling**: Upgrade existing servers (quick fix)
- **Database scaling**: Sharding, replication, caching
- **Load balancing**: Distribute traffic evenly

### 2. Reliability

- **Redundancy**: No single point of failure
- **Replication**: Data across multiple nodes
- **Health checks**: Detect and remove failed nodes
- **Circuit breakers**: Prevent cascade failures

### 3. Performance

- **Caching**: Redis, CDN, application cache
- **Async processing**: Message queues, background tasks
- **Database optimization**: Indexing, query optimization
- **Connection pooling**: Reuse database connections

### 4. Security

- **Authentication**: JWT, OAuth2, session management
- **Authorization**: RBAC, permission checks
- **Input validation**: Prevent injection attacks
- **Rate limiting**: Prevent abuse and DDoS
- **Encryption**: TLS, data encryption at rest

### 5. Observability

- **Logging**: Structured logs, centralized logging
- **Monitoring**: Metrics, dashboards, alerts
- **Tracing**: Distributed tracing for debugging
- **Health checks**: Endpoint monitoring

## 💻 Technology Stack

### Languages & Frameworks

- **Python 3.11+**
- **Django 5.0** - Full-featured web framework
- **FastAPI 0.110** - Modern async framework
- **Celery 5.3** - Distributed task queue

### Databases

- **PostgreSQL** - Primary relational database (ACID)
- **MongoDB** - Document store (flexible schema)
- **Redis** - Cache, session store, message broker
- **Elasticsearch** - Full-text search
- **Cassandra** - Wide-column store (high write throughput)

### Message Queues

- **RabbitMQ** - Reliable message broker
- **Apache Kafka** - High-throughput event streaming
- **AWS SQS** - Managed queue service

### Infrastructure

- **Docker** - Containerization
- **Kubernetes** - Container orchestration
- **Nginx** - Web server, load balancer
- **AWS/GCP/Azure** - Cloud providers

### Monitoring & Observability

- **Prometheus** - Metrics collection
- **Grafana** - Visualization
- **ELK Stack** - Elasticsearch, Logstash, Kibana
- **Sentry** - Error tracking

## 📈 Capacity Planning Reference

### Traffic Patterns

| System Type     | Typical Read:Write | Peak Traffic Multiplier |
| --------------- | ------------------ | ----------------------- |
| Social Media    | 100:1              | 3-5x                    |
| E-Commerce      | 10:1               | 10x (Black Friday)      |
| Banking         | 20:1               | 2x                      |
| Video Streaming | 1000:1             | 5x                      |
| Chat/Messaging  | 5:1                | 3x                      |

### Database Sizing

| Users | QPS  | Storage | Recommended Setup             |
| ----- | ---- | ------- | ----------------------------- |
| 1M    | 1K   | 100GB   | Single PostgreSQL             |
| 10M   | 10K  | 1TB     | Master + 2 Replicas           |
| 100M  | 100K | 10TB    | Sharded (4 shards) + replicas |
| 1B    | 1M   | 100TB   | Distributed DB (Cassandra)    |

### Cache Sizing (80/20 Rule)

Typically cache **20% of hot data** that serves **80% of requests**:

- 1M users: 2GB Redis
- 10M users: 20GB Redis
- 100M users: 200GB Redis (cluster)

## 🎓 Interview Question Categories

### 1. System Design Questions

- Design Twitter / Facebook feed
- Design URL shortener like bit.ly
- Design Netflix / YouTube
- Design Uber / Lyft
- Design Instagram / TikTok
- Design Dropbox / Google Drive
- Design payment gateway like Stripe
- Design notification system
- Design search engine like Google
- Design rate limiter / API gateway

### 2. Scalability Questions

- How do you scale to 1M users?
- How do you handle read-heavy vs write-heavy workloads?
- How do you shard a database?
- How do you design for high availability?
- How do you handle celebrity/hot user problem?

### 3. Data Consistency Questions

- Strong vs eventual consistency trade-offs
- How do you handle distributed transactions?
- CAP theorem in practice
- Database replication lag
- Cache invalidation strategies

### 4. Performance Questions

- How do you optimize slow API endpoints?
- How do you reduce database query time?
- When to use caching vs database?
- How do you handle N+1 query problem?
- How do you optimize for mobile vs web?

### 5. Reliability Questions

- How do you handle service failures?
- What is circuit breaker pattern?
- How do you implement retry logic?
- How do you design for disaster recovery?
- How do you prevent cascading failures?

## 🚀 Quick Reference Cheat Sheet

### When to use What?

**PostgreSQL:**

- Strong consistency needed (payments, inventory)
- Complex queries with joins
- ACID transactions required

**MongoDB:**

- Flexible schema
- Document-oriented data
- Rapid prototyping

**Redis:**

- Caching
- Session management
- Rate limiting counters
- Real-time leaderboards

**Cassandra:**

- High write throughput
- Time-series data
- Eventual consistency acceptable

**Elasticsearch:**

- Full-text search
- Log analytics
- Complex filtering and aggregation

**RabbitMQ:**

- Reliable message delivery
- Complex routing
- Priority queues

**Kafka:**

- Event streaming
- High throughput
- Event sourcing

## 📝 System Design Template

Use this template for any system design interview:

```markdown
## 1. Requirements (5 min)

### Functional:

- Feature 1
- Feature 2

### Non-Functional:

- QPS:
- Users:
- Latency:
- Availability:

## 2. Capacity Estimation (5 min)

- Traffic: X QPS
- Storage: Y TB
- Bandwidth: Z MB/s

## 3. High-Level Design (10 min)

[Draw architecture diagram]

## 4. Deep Dive (20 min)

- Database schema
- API design
- Scalability bottlenecks
- Trade-offs

## 5. Optimizations (10 min)

- Caching strategy
- Sharding approach
- Failure handling
```

## 🏆 Success Criteria

After studying these documents, you should be able to:

✅ Explain any system from scratch in 45 minutes
✅ Make accurate capacity estimations quickly
✅ Identify bottlenecks and propose solutions
✅ Discuss trade-offs confidently
✅ Implement core components in Python
✅ Handle follow-up questions about edge cases
✅ Design for scale from day 1
✅ Consider security and reliability from the start

## 📚 Additional Resources

### Books

- "Designing Data-Intensive Applications" by Martin Kleppmann
- "System Design Interview" by Alex Xu (Volumes 1 & 2)
- "Building Microservices" by Sam Newman

### Online Resources

- [System Design Primer](https://github.com/donnemartin/system-design-primer)
- [ByteByteGo Newsletter](https://bytebytego.com/)
- [High Scalability Blog](http://highscalability.com/)

### Python-Specific

- [Django Documentation](https://docs.djangoproject.com/)
- [FastAPI Documentation](https://fastapi.tiangolo.com/)
- [Real Python Tutorials](https://realpython.com/)

---

## 🤝 Contributing

These documents are continuously updated with:

- Real interview questions from FAANG companies
- Production patterns from 4+ years experience
- Latest Python best practices
- Updated capacity estimations

**Last Updated**: January 2026

**Python Version**: 3.11+
**Django Version**: 5.0+
**FastAPI Version**: 0.110+

---

**Remember**: System design is about trade-offs. There's no perfect solution, only solutions that fit specific requirements and constraints. Always ask clarifying questions and discuss alternatives!
