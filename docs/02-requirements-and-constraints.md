# 02 — Requirements and Constraints

No architectural decision is correct in isolation. A choice that is appropriate for one set of requirements is actively harmful for another. PostgreSQL with synchronous replication is a great choice for a financial application and unnecessary overhead for a content blog. A caching layer in front of your database is a good idea when you have 10,000 users and a bad idea when you have 100 and your data changes every few seconds. The architecture follows from the requirements — which means that if you do not define your requirements clearly before making architectural choices, you are essentially guessing.

This section is about how to define requirements well enough to make good decisions. It is not about producing a comprehensive specification document. It is about getting clear on the handful of dimensions that constrain your architecture.

## Functional versus non-functional requirements

The distinction is straightforward but important enough to be explicit. Functional requirements describe what the system does — the features, the user stories, the business logic. "Users can register, log in, create a project, and invite collaborators" is a functional requirement. "Admins can export a CSV of all users created in the last 30 days" is a functional requirement.

Non-functional requirements describe how well the system does it — the quality attributes, the performance envelope, the constraints. "The API should respond in under 200ms for 95% of requests" is a non-functional requirement. "The system should be available 99.9% of the time" is a non-functional requirement. "Data must not be lost even if the primary database server fails" is a non-functional requirement.

The reason the distinction matters is that non-functional requirements have a much larger impact on architecture than functional requirements do. Adding a new feature usually means writing new code. Changing the availability target from 99% to 99.99% might require a completely different infrastructure approach. The functional requirements tell you what to build. The non-functional requirements tell you how it needs to behave — and they are the ones that constrain your architectural options.

## The non-functional requirements that matter for web applications

There are many quality attributes a system can have, but most web applications need to reason seriously about a specific subset:

**Availability** is the percentage of time the system is operational and accessible to users. The important thing to understand about availability targets is not the percentage — it is what that percentage means in downtime. 99% availability means roughly 87 hours of downtime per year. 99.9% means about 8.7 hours. 99.99% means about 52 minutes. The jump from 99.9% to 99.99% is not just a decimal point — it typically requires redundancy, automated failover, and a significantly more complex operational setup. Chasing availability you do not need is one of the most common ways teams waste engineering effort.

**Latency** is how long a user waits for a response. The metric that matters is not average latency — it is the tail. P99 latency (the response time experienced by the slowest 1% of requests) is what determines whether your application feels slow. The 99th percentile is not an edge case; at 100 requests per second, it is one user per second experiencing the slow path. Google's research on web performance found that a 100ms increase in load time decreases conversion by 7%. For most web applications, a P99 below 2 seconds for API responses is a reasonable starting target.

**Throughput** is how much work the system can handle per unit of time — typically measured in requests per second (RPS) or transactions per second (TPS). Throughput and latency interact: a system under load will see latency increase as throughput approaches its capacity. The relationship is not linear. Understanding your expected throughput helps you size infrastructure and identify bottlenecks before they occur in production.

**Consistency** is how quickly all parts of the system agree on the current state of data. A user who updates their profile photo should see the new photo immediately; consistency here is critical. A dashboard showing aggregate analytics from the last hour can tolerate being a few seconds behind; eventual consistency is fine. The consistency requirement varies significantly across different parts of the same application, and treating it as a binary (consistent or not) leads to over-engineering in some places and bugs in others.

**Durability** is the guarantee that data, once written, survives failures. Durability is usually provided by your database (PostgreSQL's write-ahead log, for example, ensures that committed transactions are durable even if the server crashes immediately after), but it requires deliberate configuration. A database without backups has durability guarantees that hold right up until the disk fails.

**Scalability** is the ability to handle growth — more users, more data, more requests — without requiring a fundamental re-architecture. Scalability is not the same as performance. A system can perform well at current load and be completely non-scalable. Planning for scalability is worthwhile; obsessing over it at MVP stage is usually a mistake.

## Estimating scale honestly

Back-of-envelope calculations are a tool for sanity-checking assumptions, not for precision. Alex Xu's *System Design Interview* has a good treatment of this — the point is not to get the exact number but to understand the order of magnitude so you can make proportionate decisions.

Start with daily active users (DAU). If you expect 10,000 DAU and each user makes about 20 requests per day, you are looking at 200,000 requests per day, which is roughly 2.3 requests per second on average and perhaps 10 requests per second at peak (accounting for the fact that usage is not uniformly distributed). That is a load that a single well-configured server can handle comfortably. If you expect 1 million DAU, you are at 230 requests per second average and potentially 1,000+ at peak — a different problem entirely.

Storage estimation follows the same logic. If each user generates roughly 1MB of data per month and you expect 100,000 users after year one, you are looking at about 100GB of data. That is trivially manageable. If each user generates 100MB per month (video, high-resolution images), you are at 10TB, which changes your storage and CDN strategy significantly.

The ratio of reads to writes matters for database design. A content platform might have 100:1 reads to writes — caching and read replicas become high-leverage. A chat application might be closer to 1:1 — different trade-offs apply. Write-heavy systems have different architectural needs than read-heavy systems, and making this explicit early saves painful realization later.

None of these calculations need to be precise. Being off by 2x is irrelevant. Being off by 100x is a problem. The goal is to know whether you are designing for dozens, thousands, or millions — because the right architecture for each is genuinely different.

## CAP theorem: what it actually means for database decisions

The CAP theorem, formally proven by Eric Brewer and refined by Gilbert and Lynch, states that a distributed system cannot simultaneously guarantee all three of the following: Consistency (every read returns the most recent write), Availability (every request receives a response), and Partition tolerance (the system continues to operate when network partitions occur). In the presence of a network partition, you must choose between consistency and availability.

The reason this matters for web developers is not the theoretical computer science — it is the practical implication for choosing data stores. Kleppmann's *Designing Data-Intensive Applications* is the best treatment of this; his key observation is that partition tolerance is not actually optional for distributed systems (network failures happen; you must tolerate them), so the real choice is between consistency and availability when a partition occurs.

A traditional relational database running on a single node does not face this trade-off — it is not distributed, so there is no partition to worry about. But once you add replication, or use a database like Cassandra or DynamoDB that is designed to be distributed by default, the trade-off becomes real. Cassandra and DynamoDB favor availability — they will continue to accept writes during a partition and reconcile conflicts later. This gives you a highly available system at the cost of possible reads of stale data. A system running PostgreSQL with synchronous replication favors consistency — a write is not acknowledged until it has been written to the replica, and if the replica is unavailable, the write may fail.

The practical decision for most web developers: use PostgreSQL as your primary database. It gives you strong consistency, ACID transactions, and a mature ecosystem. Its availability characteristics are good enough for applications that do not require a globally distributed presence. If you later encounter specific data access patterns that PostgreSQL handles poorly — high-volume time-series data, for example — that is the time to evaluate adding a specialized store. Not before.

## Service Level Objectives

SLOs (Service Level Objectives) are the internal targets you set for the reliability and performance of your system. They are not the same as SLAs (Service Level Agreements), which are contractual commitments to customers with commercial consequences. An SLO is what you use to make engineering decisions.

The Google SRE book defines SLOs clearly: they are a target value or range for a service level measured by a service level indicator (SLI). An SLI might be request latency — the fraction of requests served within some threshold. An SLO derived from that SLI might be: "95% of homepage requests should complete within 200ms." An error budget is the maximum allowed rate of failures — if your SLO is 99.9% availability, your error budget is 0.1%, which is 43.8 minutes per month.

The reason SLOs matter even on small teams is that they force you to be specific about what reliability you actually need, which prevents two failure modes: building brittle systems because you never articulated that reliability matters, and over-engineering systems in pursuit of reliability targets that nobody actually needs. A personal project does not need five-nines availability. A payments processing component probably does. Making this explicit, even informally, is worthwhile before making the architectural decisions that affect it.

## Documenting requirements as a single page

Before making any significant architectural decision, write down your requirements on a single page. Not a formal specification — a working document that captures the functional scope (what the MVP does), the non-functional targets (what availability, latency, and consistency you need), the scale estimates (how many users, how much data, what read/write ratio), and the constraints (budget, team size, timeline, technology choices already made).

This document does not need to be precise. It needs to be explicit. The value is not in the document itself but in the forcing function: decisions that were previously implicit become visible, assumptions that were unstated get challenged, and trade-offs that were being avoided have to be confronted.

Make this page the first thing you produce when starting a new system or a significant new component. Refer to it when you are making architectural decisions. Update it when your understanding changes. It costs an hour to write and can prevent weeks of rework.

---

## Sources

- Kleppmann, M. (2017). *Designing Data-Intensive Applications*. O'Reilly Media. — CAP theorem, consistency models, replication trade-offs.
- Xu, A. (2020). *System Design Interview – An Insider's Guide*. ByteByteGo. — Back-of-envelope estimation techniques.
- Beyer, B., Jones, C., Petoff, J., & Murphy, R. (Eds.). (2016). *Site Reliability Engineering*. Google / O'Reilly. https://sre.google/sre-book/ — SLOs, SLIs, error budgets.
- Brewer, E. (2000). "Towards Robust Distributed Systems." PODC Keynote. — Original CAP conjecture.
- Gilbert, S. & Lynch, N. (2002). "Brewer's Conjecture and the Feasibility of Consistent, Available, Partition-Tolerant Web Services." *ACM SIGACT News.* — Formal proof of CAP.
- DORA Research. (2023). "Accelerate State of DevOps Report." https://dora.dev/ — Engineering performance metrics and reliability research.
- Google. (2012). "Speed Matters." https://research.google/pubs/the-effect-of-latency-on-user-experience/ — Latency and conversion impact.