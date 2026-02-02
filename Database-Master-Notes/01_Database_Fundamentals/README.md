# 📁 01_Database_Fundamentals

> Master the foundational concepts that underpin all database systems. This section covers the core principles, trade-offs, and architectural patterns that every database engineer must understand deeply.

---

## 📋 Overview

This folder contains comprehensive notes on fundamental database concepts that form the bedrock of database engineering. Whether you're designing a new system, debugging performance issues, or making architectural decisions, these fundamentals will guide your thinking.

**Target Audience:** Junior to Staff engineers (1-10+ years experience)

**Time Investment:** 2-3 weeks to deeply understand all concepts

---

## 📚 Files in This Section

### 1. [ACID Properties](01_ACID_Properties.md) 🔒

**Core Concept:** Atomicity, Consistency, Isolation, Durability

**What You'll Learn:**

- How ACID guarantees data correctness
- MVCC and transaction isolation levels
- Write-ahead logging (WAL) and durability mechanisms
- Real-world disasters from ACID violations

**Key Topics:**

- Two-phase commit (2PC)
- Deadlock detection and prevention
- Optimistic vs pessimistic locking
- Transaction logs and recovery

**When to Read:** Before designing any transactional system

**Real-World Applications:** Banking, e-commerce checkout, inventory management

---

### 2. [BASE Properties](02_BASE_Properties.md) 🌐

**Core Concept:** Basically Available, Soft State, Eventual Consistency

**What You'll Learn:**

- Trade-offs of eventual consistency
- Conflict resolution strategies (vector clocks, CRDTs)
- Tunable consistency models
- Anti-entropy and repair mechanisms

**Key Topics:**

- Quorum reads/writes
- Hinted handoff
- Read repair
- Gossip protocols

**When to Read:** Before working with distributed NoSQL systems

**Real-World Applications:** Social media feeds, shopping carts, DNS, content delivery

---

### 3. [OLTP vs OLAP](03_OLTP_vs_OLAP.md) ⚡

**Core Concept:** Transactional vs Analytical workloads

**What You'll Learn:**

- Row-oriented vs column-oriented storage
- B-Tree vs bitmap indexes
- Star schema and dimensional modeling
- Lambda architecture patterns

**Key Topics:**

- Data warehouse design
- ETL vs ELT pipelines
- Columnar compression techniques
- Partition pruning

**When to Read:** Before separating operational and analytical databases

**Real-World Applications:** Business intelligence, data warehousing, reporting systems

---

### 4. [CAP Theorem](04_CAP_Theorem.md) 🔺

**Core Concept:** Consistency, Availability, Partition Tolerance trade-offs

**What You'll Learn:**

- Why you can't have all three (the proof)
- CP vs AP system design choices
- Partition tolerance in practice
- PACELC theorem extension

**Key Topics:**

- Network partitions (Byzantine failures)
- Split-brain scenarios
- Quorum consensus
- Graceful degradation

**When to Read:** Before designing any distributed system

**Real-World Applications:** Multi-region databases, microservices, cloud architectures

---

### 5. [Database Normalization](05_Normalization.md) 📐

**Core Concept:** Organizing data to reduce redundancy

**What You'll Learn:**

- Normal forms (1NF through 5NF)
- When to normalize vs denormalize
- Update/deletion/insertion anomalies
- Denormalization patterns for performance

**Key Topics:**

- Functional dependencies
- BCNF (Boyce-Codd Normal Form)
- Multi-valued dependencies
- Materialized views

**When to Read:** Before designing database schemas

**Real-World Applications:** OLTP systems, relational database design, schema migrations

---

### 6. [Database Types Overview](06_Database_Types_Overview.md) 🗄️

**Core Concept:** Choosing the right database for the job

**What You'll Learn:**

- Relational, document, key-value, column-family, graph, time-series, NewSQL
- Internal architecture of each database type
- Polyglot persistence patterns
- When to use which database

**Key Topics:**

- Database internals (storage engines, indexes)
- Consistency models per database type
- Scaling characteristics
- Use case mapping

**When to Read:** Before selecting databases for new projects

**Real-World Applications:** System design, architecture decisions, technology evaluation

---

### 7. [OLAP vs HTAP](07_OLAP_vs_HTAP.md) ⚡

**Core Concept:** Hybrid Transactional/Analytical Processing

**What You'll Learn:**

- HTAP architecture (dual storage engines)
- Real-time analytics without ETL
- TiDB, CockroachDB, SingleStore comparison
- When HTAP is better than Lambda architecture

**Key Topics:**

- Row store + column store in one database
- Replication lag considerations
- Resource isolation (OLTP vs OLAP)
- Hot/cold data management

**When to Read:** When you need real-time analytics on transactional data

**Real-World Applications:** Real-time dashboards, fraud detection, dynamic pricing

---

## 🎯 Learning Path

### For Junior Engineers (1-3 years)

**Focus:** Understand the basics deeply

1. Start with **ACID Properties** (understand transactions)
2. Read **Database Types Overview** (survey the landscape)
3. Study **Normalization** (learn schema design)
4. Review **OLTP vs OLAP** (understand workload differences)

**Goal:** Make informed decisions about schema design and database choice

---

### For Mid-Level Engineers (3-6 years)

**Focus:** Master distributed systems concepts

1. Deep dive into **CAP Theorem** (distributed trade-offs)
2. Study **BASE Properties** (eventual consistency patterns)
3. Compare **OLTP vs OLAP** (architectural patterns)
4. Explore **HTAP Systems** (modern unified systems)

**Goal:** Design distributed systems with appropriate trade-offs

---

### For Senior Engineers (6-10 years)

**Focus:** Architect complex systems

1. Master **CAP Theorem** + **BASE Properties** (make conscious trade-offs)
2. Study **Normalization** + denormalization patterns (performance vs integrity)
3. Deep dive into **Database Types** internals (understand storage engines)
4. Evaluate **HTAP** vs Lambda/Kappa architectures (simplify systems)

**Goal:** Make architecture decisions that balance performance, consistency, and operational complexity

---

### For Staff/Principal Engineers (10+ years)

**Focus:** Strategic technology choices

1. Use **CAP Theorem** to guide multi-region design
2. Apply **BASE Properties** for eventually consistent systems at scale
3. Choose **Database Types** based on access patterns and scale requirements
4. Evaluate **HTAP** for real-time analytics use cases

**Goal:** Drive technology strategy, mentor teams, design systems for 10x scale

---

## 🔗 Connections to Other Sections

### Leads to:

- **02_Relational_Deep_Dive** → Apply ACID and normalization principles
- **06_NoSQL_Deep_Dive** → Apply BASE and CAP principles
- **20_Distributed_Systems** → Apply CAP and partition tolerance patterns
- **08_System_Design_Patterns** → Use these fundamentals in real architectures

### Prerequisites from:

- None! This is the foundation section.

---

## 🎓 Practical Exercises

### Exercise 1: ACID vs BASE Trade-offs

**Goal:** Understand when to choose each

**Task:**

1. Design a banking system (ACID required)
2. Design a social media feed (BASE acceptable)
3. Identify the trade-offs in each design

**Deliverable:** Architecture diagram with consistency guarantees

---

### Exercise 2: Normalize a Schema

**Goal:** Practice normalization

**Task:**

1. Start with an unnormalized e-commerce schema
2. Normalize to 1NF, 2NF, 3NF
3. Identify performance bottlenecks
4. Denormalize strategically

**Deliverable:** Schema evolution with rationale at each step

---

### Exercise 3: Database Selection

**Goal:** Choose appropriate databases

**Task:**

1. Given a system with multiple workloads:
   - User authentication
   - Social graph
   - Activity feed
   - Real-time analytics
   - Search
2. Choose database types for each
3. Justify your choices

**Deliverable:** Polyglot persistence architecture with justifications

---

### Exercise 4: CAP Theorem Analysis

**Goal:** Analyze real-world systems

**Task:**

1. Analyze DynamoDB, Spanner, Cassandra
2. Classify as CP or AP
3. Identify consistency/availability trade-offs
4. Design partition handling

**Deliverable:** Comparison matrix with failure scenarios

---

## 📊 Concept Comparison Matrix

| Concept             | Use When                            | Avoid When                          | Complexity | Impact                  |
| ------------------- | ----------------------------------- | ----------------------------------- | ---------- | ----------------------- |
| **ACID**            | Money, inventory, critical data     | High scale, eventual consistency OK | Medium     | High correctness        |
| **BASE**            | Social media, caching, feeds        | Financial, critical consistency     | Medium     | High availability       |
| **OLTP**            | Transactions, real-time updates     | Analytics, reporting                | Low        | Fast writes             |
| **OLAP**            | Analytics, reporting, BI            | Real-time transactions              | Medium     | Fast reads              |
| **Normalization**   | Data integrity, avoid redundancy    | Performance bottleneck (joins)      | Low        | Data correctness        |
| **Denormalization** | Read-heavy, performance critical    | Write-heavy, complex updates        | Medium     | Read speed              |
| **CAP (CP)**        | Banking, strong consistency         | High availability critical          | High       | Strong consistency      |
| **CAP (AP)**        | Social media, high availability     | Financial, strong consistency       | High       | High availability       |
| **HTAP**            | Real-time analytics on transactions | Massive historical data (PB+)       | High       | Simplified architecture |

---

## 🔧 Tools & Technologies Mentioned

### Relational Databases (ACID):

- PostgreSQL
- MySQL
- Oracle
- SQL Server

### NoSQL Databases (BASE):

- Cassandra (AP)
- MongoDB (CP/AP tunable)
- DynamoDB (AP)
- Redis (Cache)

### OLAP Systems:

- Redshift
- Snowflake
- ClickHouse
- BigQuery

### HTAP Systems:

- TiDB
- CockroachDB
- SingleStore (MemSQL)

### Graph Databases:

- Neo4j
- ArangoDB

### Time-Series:

- InfluxDB
- TimescaleDB
- Prometheus

---

## 💡 Key Takeaways

1. **ACID vs BASE:** Choose based on consistency requirements, not hype
2. **OLTP vs OLAP:** Separate workloads have different storage needs
3. **CAP Theorem:** You can't have all three; choose consciously
4. **Normalization:** Start normalized, denormalize for proven bottlenecks
5. **Database Types:** One size does NOT fit all; use polyglot persistence
6. **HTAP:** Simplifies architecture when real-time analytics needed

---

## 📖 Further Reading

### Books:

- **"Designing Data-Intensive Applications"** by Martin Kleppmann (chapters 1-9)
- **"Database Internals"** by Alex Petrov (chapters 1-5)
- **"Database Design for Mere Mortals"** by Michael Hernandez

### Papers:

- **"CAP Theorem"** by Eric Brewer (2000)
- **"Spanner: Google's Globally-Distributed Database"** (2012)
- **"Dynamo: Amazon's Highly Available Key-value Store"** (2007)

### Online Courses:

- **CMU 15-445: Database Systems** (Andy Pavlo)
- **MIT 6.824: Distributed Systems** (Robert Morris)

---

## ❓ Common Questions

### Q: Should I always normalize to 3NF?

**A:** Start with 3NF for data integrity. Denormalize only after measuring performance bottlenecks. Premature optimization is dangerous!

### Q: Can I use a NoSQL database for everything?

**A:** No! NoSQL databases sacrifice ACID for scale/availability. Use relational databases when ACID transactions are required (e.g., banking).

### Q: Is HTAP a replacement for data warehouses?

**A:** No. HTAP is for real-time analytics on recent data (TB scale). Data warehouses are for deep historical analytics (PB+ scale). Use both!

### Q: How do I choose between CP and AP in CAP theorem?

**A:** Ask: "During a network partition, is it worse to be unavailable or inconsistent?" If inconsistency is unacceptable (banking), choose CP. If unavailability is unacceptable (social media), choose AP.

---

## 🚀 Next Steps

After mastering these fundamentals:

1. **Apply to projects:** Use these concepts in system design
2. **Deep dive:** Explore specific database types (Relational, NoSQL, etc.)
3. **Practice:** Implement toy databases to understand internals
4. **Teach others:** Best way to solidify understanding

Continue to **[02_Relational_Deep_Dive](../02_Relational_Deep_Dive)** to apply ACID and normalization principles in depth.

---

## 📝 Progress Tracking

Track your progress:

- [ ] Completed ACID Properties notes
- [ ] Completed BASE Properties notes
- [ ] Completed OLTP vs OLAP notes
- [ ] Completed CAP Theorem notes
- [ ] Completed Normalization notes
- [ ] Completed Database Types Overview notes
- [ ] Completed OLAP vs HTAP notes
- [ ] Completed all practical exercises
- [ ] Ready to move to next section

**Estimated Time:** 2-3 weeks (reading + exercises)

---

## 🤝 Contributing

Found errors or want to add examples? Contributions welcome!

---

**Last Updated:** February 2, 2026
**Section Status:** ✅ Complete (7/7 files)
**Next Section:** [02_Relational_Deep_Dive](../02_Relational_Deep_Dive)
