# Java Backend Engineering: System Design & Situational Scenarios Master Curriculum

A comprehensive, production-grade guide covering distributed system design foundations, 24 high-level distributed architectures, 11 low-level machine coding systems, and 16 engineering leadership and incident triage situational scenarios tailored specifically for Senior, Staff, and Principal Java Backend Engineers.

---

## Master Architecture & Preparation Roadmap

```mermaid
flowchart TD
    subgraph Pillar1 ["Phase 1: Distributed Foundations (01)"]
        direction TB
        FND["01. System Design Foundations<br/>(Scalability Primitives, CAP/PACELC, Raft Consensus, 2PC, B+ vs LSM Trees, Consistent Hashing, Caching Topologies, Resilience4j)"]
    end

    subgraph Pillar2 ["Phase 2: High-Level Architecture HLD (02)"]
        direction TB
        HLD_Scale["Web Scale & Edge (URL Shortener, CDN, Snowflake ID, Rate Limiter)"]
        HLD_Realtime["Real-Time & Events (Chat, Notifications, Video Streaming)"]
        HLD_Fintech["Transactional & Commerce (Stripe Payments, Flash Sale, Uber Ride-Hailing, Uber Eats)"]
        HLD_Systems["Distributed Storage & Workflows (Dropbox, Job Scheduler, Temporal Engine, Graph Search)"]
        HLD_MissionCritical["Mission-Critical Enterprise (UK Faster Payments 24/7, Card Fraud <100ms, Open Banking API, Mainframe Migration)"]
        
        HLD_Scale --> HLD_Realtime --> HLD_Fintech --> HLD_Systems --> HLD_MissionCritical
    end

    subgraph Pillar3 ["Phase 3: Low-Level Design & Machine Coding LLD (03)"]
        direction TB
        LLD_Concurrency["Concurrency Primitives (Custom Bounded Queue, ThreadPool, ReadWriteLock List)"]
        LLD_Storage["Storage & Scheduling (Segmented File Manager, DAG Job Scheduler, Train Scheduler)"]
        LLD_Apps["Application Machine Coding (In-Memory Twitter, In-Memory Message Queue, In-Memory Filesystem, Rule Engine AST, Spotify)"]
        
        LLD_Concurrency --> LLD_Storage --> LLD_Apps
    end

    subgraph Pillar4 ["Phase 4: Situational & Engineering Leadership (04)"]
        direction TB
        SIT_Incidents["Sev-1 Outage Command (Cascading Failures, JVM Memory Leaks, HikariCP Deadlocks)"]
        SIT_Architecture["Technical Pushback (Monolith vs Microservices, REST vs Events, Premature Optimization)"]
        SIT_Execution["Execution & Governance (Unrealistic Deadlines, Technical Debt, Breaking API Contracts)"]
        SIT_Leadership["People & Strategy (Mentorship, Code Review Deadlocks, Project Failures, Radical Ownership, Speed vs Accuracy)"]
        
        SIT_Incidents --> SIT_Architecture --> SIT_Execution --> SIT_Leadership
    end

    Pillar1 --> Pillar2
    Pillar2 --> Pillar3
    Pillar3 --> Pillar4

    style Pillar1 fill:#0f172a,stroke:#3b82f6,stroke-width:1.5px,color:#93c5fd
    style Pillar2 fill:#0f172a,stroke:#10b981,stroke-width:1.5px,color:#a7f3d0
    style Pillar3 fill:#0f172a,stroke:#f59e0b,stroke-width:1.5px,color:#fde68a
    style Pillar4 fill:#0f172a,stroke:#8b5cf6,stroke-width:1.5px,color:#c4b5fd
    style FND fill:#1e293b,stroke:#3b82f6,stroke-width:2px,color:#fff
    style HLD_Scale fill:#1e293b,stroke:#10b981,stroke-width:2px,color:#fff
    style HLD_Realtime fill:#1e293b,stroke:#10b981,stroke-width:2px,color:#fff
    style HLD_Fintech fill:#1e293b,stroke:#10b981,stroke-width:2px,color:#fff
    style HLD_Systems fill:#1e293b,stroke:#10b981,stroke-width:2px,color:#fff
    style HLD_MissionCritical fill:#1e293b,stroke:#10b981,stroke-width:2px,color:#fff
    style LLD_Concurrency fill:#1e293b,stroke:#f59e0b,stroke-width:2px,color:#fff
    style LLD_Storage fill:#1e293b,stroke:#f59e0b,stroke-width:2px,color:#fff
    style LLD_Apps fill:#1e293b,stroke:#f59e0b,stroke-width:2px,color:#fff
    style SIT_Incidents fill:#1e293b,stroke:#8b5cf6,stroke-width:2px,color:#fff
    style SIT_Architecture fill:#1e293b,stroke:#8b5cf6,stroke-width:2px,color:#fff
    style SIT_Execution fill:#1e293b,stroke:#8b5cf6,stroke-width:2px,color:#fff
    style SIT_Leadership fill:#1e293b,stroke:#8b5cf6,stroke-width:2px,color:#fff
```

---

## Curriculum Coverage & Module Index

The index below details the 4 core pillars of the consolidated System Design & Situational Scenarios curriculum for Java Backend Engineers:

| Module File | Scope & Domain | Key Architectures & Scenarios Covered |
| :--- | :--- | :--- |
| [`01_system_design_foundations.md`](01_system_design_foundations.md) | Distributed Foundations | Scalability primitives, L4 vs L7 load balancing, Consistent Hashing with V-Nodes, CAP & PACELC theorems, Raft consensus, 2PC, B+ Tree vs LSM Tree, database sharding, caching topologies, cache stampede defenses, gRPC vs REST, circuit breakers, and rate limiting algorithms. |
| [`02_high_level_system_design.md`](02_high_level_system_design.md) | 24 High-Level Architectures | URL Shortener, Multi-Region Active-Active Redirects, API Rate Limiter, Global CDN, 64-bit Snowflake ID Generator, Real-Time Chat (WhatsApp/Slack), Notification Engine, Video Streaming (YouTube/Netflix), Payments (Stripe), Flash Sale Checkout, Ride-Hailing (Uber), Uber Eats Homepage, Distributed Cache (Redis), File Storage (Dropbox), Autocomplete, Collaborative Docs, Job Scheduler, Workflow Engine, Feed System, Billion-Scale Graph Search, **UK Faster Payments 24/7 Settlement**, **Real-Time Card Fraud Detection (<100ms)**, **Open Banking API Platform with Instant Revocation**, and **Mainframe to Microservices Zero-Downtime Migration**. |
| [`03_low_level_system_design.md`](03_low_level_system_design.md) | 11 Machine Coding Systems | Custom Bounded Queue & ThreadPool from scratch (no `ExecutorService`), Concurrent Sorted Linked List (ReadWriteLock), Segmented File Manager with byte-range locking, DAG Job Scheduler with dependency resolution, Playground Player Assignment with anti-starvation, Train & Platform Scheduler, In-Memory Twitter, In-Memory Pull-Based Message Queue, In-Memory Filesystem (`mkdir`, `pwd`, `cd` with wildcards), Rule Engine (AST/Composite), and Spotify LLD. |
| [`04_situational_scenarios.md`](04_situational_scenarios.md) | 16 Situational Engineering Scenarios | Incident Command during cascading outages, JVM memory leak & GC pause triage, HikariCP pool exhaustion, Pushing back on management (Monolith vs Microservices), Sync REST vs Event-driven trade-offs, Delivering under unrealistic deadlines, Deliberate technical debt governance, Managing breaking API changes across 15 teams, Turning around underperforming engineers, Resolving bitter code review stalemates, **Project failure postmortems & learnings**, **Radical cross-cutting system ownership**, **Balancing speed vs accuracy in financial launches**, and **Handling conflicting business vs technical requirements**. |

---

## References & Further Reading
- Kleppmann, Martin. *Designing Data-Intensive Applications*. O'Reilly Media.
- Xu, Alex. *System Design Interview – An Insider's Guide* (Vols. 1 & 2).
- Bloch, Joshua. *Effective Java* (3rd Edition). Addison-Wesley.
- Goetz, Brian, et al. *Java Concurrency in Practice*. Addison-Wesley.
- Nygard, Michael T. *Release It! Design and Deploy Production-Ready Software*. Pragmatic Bookshelf.
- Larson, Will. *Staff Engineer: Leadership Beyond the Management Track*.

