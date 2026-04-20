# 09 — Scalability Patterns

Scalability is the ability to handle more load without requiring a fundamental change to the system's architecture. It is not the same as performance — a fast, well-optimized system can be completely non-scalable if its design prevents it from handling growth. But scalability should not be chased before it is needed. Every scalability mechanism adds complexity, and complexity has a cost that is paid continuously.

The principle to carry through this section: scale in order of cost. Apply the cheapest interventions first, measure whether they are sufficient, and only move to more expensive interventions when the cheaper ones have been exhausted.

## Vertical versus horizontal scaling

Vertical scaling means adding resources to an existing server: more CPU cores, more RAM, faster disks. It is simple to execute — you upgrade the instance in your cloud provider's console. The application does not change. The deployment does not change. The debugging story does not change.

Horizontal scaling means adding more servers and distributing load across them. It requires the application to be stateless (discussed below), a load balancer to distribute requests, and a shared backing store for any state. It is more complex but removes the ceiling that vertical scaling eventually hits.

The case for vertical scaling first is stronger than it is usually presented. AWS, GCP, and Azure offer instances with 96 cores and 384GB of RAM. A single server at this scale can handle a very large web application. The cost is not negligible, but it is often less than the engineering cost of making a system horizontally scalable. Stack Overflow's architecture, discussed in Section 1, is the canonical example: they chose to scale up rather than out, and a small number of powerful servers handle millions of daily users.

Vertical scaling has a real ceiling: the largest available instance type on your cloud provider. It also has an availability ceiling: a single server is a single point of failure. If your application needs to survive a server failure — if your availability requirements are above roughly 99.5% — you need horizontal scaling so that traffic can shift to other servers when one fails.

The decision framework: start vertical. Scale up before you scale out. Move to horizontal scaling when you have evidence that vertical scaling is no longer sufficient, or when your availability requirements demand redundancy.

## Stateless services

Statelessness is the prerequisite for horizontal scaling. If your application stores state in the process's memory — in a module-level variable, in a local file, in an in-process session store — then sending requests to multiple instances produces inconsistent behavior. User A logs into instance 1 and their session is stored in instance 1's memory. Their next request goes to instance 2, which has no session for them, and they are logged out.

Making an Express application stateless means moving all state out of the process:

- Sessions go to Redis, not the process's memory
- Uploaded files go to object storage (S3, GCS, R2), not the local filesystem
- Any state that must survive a process restart or be shared across instances goes to an external store
- Configuration comes from environment variables, not hard-coded values

A stateless application instance can be replaced, duplicated, or terminated at any time without affecting correctness. This is what The 12-Factor App means by "processes" — "Twelve-factor processes are stateless and share-nothing." It is what makes horizontal scaling, rolling deployments, and auto-scaling possible.

## Load balancing

A load balancer distributes incoming requests across multiple application instances. It is the component that makes horizontal scaling work from the client's perspective — the client connects to one address, and the load balancer decides which instance handles each request.

The common load-balancing algorithms are:

**Round robin** distributes requests in sequence: request 1 goes to instance 1, request 2 to instance 2, request 3 to instance 3, request 4 back to instance 1. Simple and effective when instances are homogeneous and requests are roughly similar in cost.

**Least connections** sends each new request to the instance with the fewest active connections. Better than round robin when request handling times vary significantly — long-running requests do not accumulate on one instance.

**IP hash** routes requests from the same client IP to the same instance. This provides sticky sessions without requiring shared session storage, but it reduces the load-balancing effect because clients are locked to specific instances. In most cases, moving sessions to Redis is a better solution than sticky sessions.

Modern cloud load balancers (AWS ALB, GCP Load Balancer, Cloudflare) handle the algorithmic choices transparently and add SSL termination, health checking, and DDoS protection. You configure routing rules and health check endpoints; the algorithm is an implementation detail.

## Database scaling

Database scaling is the most constrained part of application scaling. Unlike application servers, which can be added horizontally with relative ease, databases have more fundamental limitations. The order of interventions matters significantly:

**Query optimization and indexing** should always come first. The majority of database performance problems are caused by missing indexes or poorly written queries. A query that takes 2 seconds on a 1 million row table with a sequential scan takes 2 milliseconds with the right index. `EXPLAIN ANALYZE` in PostgreSQL shows the query plan; if you see `Seq Scan` on a large table where you expected an index scan, you need an index. This costs nothing and should be exhausted before any other intervention.

**Connection pooling** (PgBouncer for PostgreSQL) manages a pool of database connections and multiplexes many application connections onto a smaller number of actual database connections. PostgreSQL's memory overhead per connection is significant; without connection pooling, a large number of application instances can exhaust the database's connection limit. PgBouncer adds minimal latency and substantially reduces connection overhead.

**Read replicas** distribute read queries across multiple database instances. All writes go to the primary; read queries can go to replicas. This is the right intervention when the database is under sustained read load that cannot be addressed by caching or query optimization. The setup and maintenance cost is moderate. See Section 7 for consistency implications.

**Vertical scaling the database** — upgrading to a more powerful instance — is often the most cost-effective next step after optimization and connection pooling. A larger database instance with more CPU and RAM can handle significantly more load, and the operational cost is zero compared to the engineering effort of adding replicas or sharding.

**Partitioning and sharding** are the high-complexity options of last resort. Partitioning divides a single table into multiple physical partitions based on a partition key (often a date range or a tenant ID), allowing queries that filter on the partition key to scan only the relevant partition. Sharding divides data across multiple database servers, each owning a subset of the data. Both require careful application-level logic, complicate queries that need data across partitions, and add significant operational complexity. For most web applications, these are not decisions you will need to make.

## Async processing for throughput

Background job queues (Section 6) increase system throughput by decoupling work from HTTP request handling. An endpoint that synchronously sends email, generates a PDF, and updates three tables takes hundreds of milliseconds per request. Under load, this limits throughput because each request holds a connection for its full duration.

Moving this work to a background queue transforms the endpoint into one that enqueues a job and returns immediately. The endpoint response time drops from hundreds of milliseconds to single-digit milliseconds. Throughput increases dramatically because the bottleneck is no longer the per-request work — it is the throughput of the job workers, which can be scaled independently.

This is not a premature optimization. It is the correct design for any work that does not need to complete within the HTTP request cycle.

## Rate limiting

Rate limiting controls how many requests a client can make within a time window. It prevents abuse, protects against denial-of-service attacks, and ensures fair resource allocation among clients.

The two common algorithms are the token bucket and the sliding window. The **token bucket** algorithm gives each client a bucket that fills at a constant rate and empties when requests are made. The client can burst up to the bucket size and then is limited to the fill rate. The **sliding window** algorithm counts requests in a rolling time window — for example, a maximum of 100 requests in any 60-second window.

In Express with Redis:

```typescript
import rateLimit from 'express-rate-limit';
import RedisStore from 'rate-limit-redis';
import { Redis } from 'ioredis';

const redis = new Redis(process.env.REDIS_URL);

const apiLimiter = rateLimit({
  windowMs: 60 * 1000,       // 1 minute window
  max: 100,                   // 100 requests per window
  standardHeaders: true,      // Return rate limit info in headers
  legacyHeaders: false,
  store: new RedisStore({
    sendCommand: (...args: string[]) => redis.call(...args),
  }),
  message: {
    status: 'error',
    code: 'RATE_LIMIT_EXCEEDED',
    message: 'Too many requests. Please try again in a moment.',
  },
});

// Apply to all API routes
app.use('/api/', apiLimiter);

// Stricter limits for auth endpoints
const authLimiter = rateLimit({
  windowMs: 15 * 60 * 1000,  // 15 minutes
  max: 10,                    // 10 attempts per window
  store: new RedisStore({
    sendCommand: (...args: string[]) => redis.call(...args),
  }),
});

app.use('/api/auth/', authLimiter);
```

Using Redis as the store means rate limit state is shared across all application instances. Without a shared store, each instance counts independently and the limit is effectively multiplied by the number of instances.

## Back pressure

Back pressure is what happens when a system is overwhelmed: requests arrive faster than they can be processed, queues grow without bound, and eventually the system fails entirely. Handling it gracefully means refusing work before the system is overwhelmed rather than accepting all work and failing under load.

In HTTP terms, a server that is at capacity should return 429 Too Many Requests or 503 Service Unavailable rather than accepting requests and timing out. This gives clients a clear signal to retry later and prevents the server from taking on more work than it can handle.

In job queue terms, back pressure means limiting the queue depth. A queue with no depth limit will grow indefinitely under sustained overload, and when the system recovers, it processes a backlog of stale work that may no longer be relevant. A bounded queue that rejects new items when full forces the caller to handle the rejection and prevents unbounded state accumulation.

## Auto-scaling

Auto-scaling is the ability for your infrastructure to add and remove instances automatically based on measured load. Cloud platforms implement it as a policy: "when CPU utilization exceeds 70% for 5 minutes, add two instances; when it falls below 30% for 10 minutes, remove two instances."

Auto-scaling requires stateless application instances (they must be replaceable), fast instance startup (a new instance that takes 3 minutes to become ready is not helpful during a spike), and accurate metrics to trigger on. CPU utilization is the most common trigger but not always the right one — a request-rate trigger is often more appropriate for web applications where the CPU usage per request varies significantly.

For most web applications, auto-scaling is not a day-one concern. Design your application to be stateless (so it is ready for horizontal scaling), then add auto-scaling when you have evidence that manual scaling is insufficient.

---

## Sources

- Kleppmann, M. (2017). *Designing Data-Intensive Applications*. O'Reilly Media. — Database replication, partitioning, sharding trade-offs.
- Xu, A. (2020). *System Design Interview – An Insider's Guide*. ByteByteGo. — Horizontal vs vertical scaling, load balancing, back-of-envelope estimates.
- Abbott, M. & Fisher, M. (2015). *The Art of Scalability* (2nd Ed). Addison-Wesley. — Scale cube model, organizational and operational scalability.
- Heroku. (2011). *The 12-Factor App.* https://12factor.net/ — Stateless processes (Factor VI), process model for concurrency (Factor VIII).
- AWS. (2023). *AWS Well-Architected Framework — Performance Efficiency Pillar.* https://docs.aws.amazon.com/wellarchitected/latest/performance-efficiency-pillar/ — Scaling patterns, resource selection.
- PgBouncer Documentation. https://www.pgbouncer.org/usage.html — Connection pooling for PostgreSQL.
- express-rate-limit Documentation. https://express-rate-limit.mintlify.app/ — Rate limiting for Express.