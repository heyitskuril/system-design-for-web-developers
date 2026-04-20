# 07 — Database Decisions

The database decision is the one architects get asked about most often and get wrong most frequently. The wrong answer is almost always driven by one of two things: following a trend without evaluating fit, or switching databases to solve a problem that is actually a query design or schema problem. The right answer is almost always simpler than the alternatives being considered.

## The correct default: a relational database

For most web applications, PostgreSQL is the right database and the argument for choosing something else needs to clear a high bar. This is not a tribal position — it is the conclusion you reach when you evaluate what web applications actually need.

Web applications store structured data with relationships. Users have orders. Orders have line items. Products belong to categories. These relationships are the natural domain of a relational model, and SQL is a mature, expressive language for querying it. Foreign keys enforce referential integrity at the database level — a deleted user cannot leave orphaned orders. Transactions ensure that a set of related writes either all succeed or all fail. Constraints prevent invalid data from entering the system.

ACID properties — Atomicity, Consistency, Isolation, Durability — are the foundation of database reliability. A relational database with ACID transactions gives you strong guarantees about data integrity that you would otherwise have to implement yourself, poorly, in application code. PostgreSQL specifically implements these correctly and has done so reliably for decades.

Kleppmann's *Designing Data-Intensive Applications* is the essential reference for this topic. His analysis of database trade-offs is thorough and honest about the cases where each type of database is genuinely appropriate. His conclusion, distilled: relational databases have held up better than critics expected because the relational model is genuinely well-suited to most data that organizations care about.

The argument for PostgreSQL over MySQL or other relational databases is mostly about features: JSON support (PostgreSQL's JSONB type provides document store capabilities when you occasionally need them without switching databases), full-text search, advanced indexing (GIN, GiST, partial indexes), window functions, and CTEs (common table expressions). These are not hypothetical benefits — they come up in real web applications.

## Document databases: when they are actually appropriate

MongoDB and other document databases store data as JSON-like documents, without a fixed schema and without the relational model. This is appropriate in specific situations:

The data genuinely has a document structure. If you are storing product catalog data where each product has a completely different set of attributes — a shoe has size and width, a laptop has RAM and processor, a book has ISBN and author — and you never need to query across those attributes relationally, a document database fits the data model naturally. The shoe schema and the laptop schema do not conflict; each document is self-describing.

Schema flexibility is a genuine requirement, not a preference. If the structure of your data changes frequently during development, or if you are building a system where users define the schema (a CMS with custom fields, for example), a document database can simplify the data layer. This is a legitimate use case.

What document databases are not suited for: data with complex relationships that require joins, operations that require transactions across multiple documents (MongoDB added multi-document transactions in version 4.0, but they are more limited than PostgreSQL's), and data where strong consistency is required for correctness.

The most common mistake with document databases is choosing MongoDB because it is "more flexible" and then gradually recreating the relational model inside it: storing arrays of related IDs, writing application-level join logic, dealing with referential integrity bugs because the database does not enforce it. If you find yourself doing this, you chose the wrong database.

## Key-value stores: what Redis is actually for

Redis is an in-memory data store with persistence options. It is extraordinarily fast for what it does well, and frequently misused for things it does not do well.

What Redis is for: caching, session storage, job queues, rate limiting, real-time leaderboards and counters, pub/sub messaging, and any use case where you need very fast access to relatively small amounts of data and can tolerate the data living in memory.

What Redis is not for: primary application data storage. Redis is not a replacement for your relational database. Its memory-based design means data is expensive to store at scale. Its persistence model (either RDB snapshots or AOF append-only file) provides durability guarantees that are weaker than a relational database's write-ahead log. Using Redis as your primary database puts your application data at risk.

Redis's data structures make it useful in specific patterns beyond simple key-value caching:

- **Sorted sets** are ideal for leaderboards, rate limiting with sliding windows, and ordered queues.
- **Hashes** map to session objects naturally — a user session is a hash of session properties.
- **Lists** support simple FIFO queues and are the underlying structure for BullMQ's job queue.
- **Pub/Sub** enables real-time messaging between application instances.

The decision is straightforward: if you need a cache or a job queue, use Redis. If you need to store primary application data, use PostgreSQL.

## Polyglot persistence

Polyglot persistence is the pattern of using multiple databases with different characteristics for different purposes within the same application. A typical configuration: PostgreSQL for primary application data, Redis for caching and sessions, Elasticsearch or PostgreSQL full-text search for search functionality.

This is a reasonable pattern when each database genuinely earns its place. The cost is operational: every database you run is another thing to back up, monitor, upgrade, and understand. For a small team, operational overhead is a real constraint. Do not add a database for a problem you could solve with a PostgreSQL feature you have not yet explored.

The correct approach: start with PostgreSQL for everything. Add Redis when you need a cache or a job queue — the operational cost is low and the benefit is high. Add a specialized search index (Elasticsearch, Typesense, Meilisearch) only when PostgreSQL's full-text search is genuinely insufficient for your search requirements. Add other stores only when you have a documented need that your existing stores cannot meet.

## Read replicas

A read replica is a copy of your primary database that receives all writes from the primary and allows read queries to run without affecting the primary's resources. The replica follows the primary with a small amount of lag — typically milliseconds.

Read replicas address a specific problem: a database that is being read so frequently that read queries are competing with write queries for resources, degrading performance for both. This problem does not exist at small scale. It starts to appear when read load is high enough that the primary database's CPU or I/O is constrained.

Before adding a read replica, exhaust two cheaper alternatives. First, make sure your queries are optimized and your indexes are correct. A slow query that touches a million rows when it should touch ten is a query design problem, not a scaling problem. `EXPLAIN ANALYZE` in PostgreSQL will show you the query plan — if it says "Seq Scan" on a large table, you probably need an index. Second, add a caching layer (Section 8). If the same data is being read repeatedly, serving it from Redis eliminates the database reads entirely.

If you have done both and the database is still under pressure, a read replica is the right next step. Direct all read-only queries (listing queries, reporting queries, dashboard queries) to the replica. Keep all write queries and anything that requires reading freshly-written data on the primary.

The consistency implication: replica lag means a user who writes data and then immediately reads it may read stale data from the replica. The pattern to handle this is "read your own writes" — for a brief window after a write, direct that user's reads to the primary. After the window expires, the replica has caught up.

## Database per service and why it matters

If you adopt a microservices architecture (Section 5), the rule is: each service owns its own database. No service queries another service's database directly. All data access goes through the owning service's API.

This rule is not aesthetic. It is the boundary that makes independent deployment possible. If services A and B share a database, changing the schema for service A's data requires careful coordination with service B to ensure it still works. You have reintroduced the coupling that microservices were supposed to eliminate.

In practice, this rule is violated constantly, usually for performance reasons: "it's faster if service A just reads directly from service B's database instead of making an API call." This is true in the short term and false in the long term. The performance saved is real; the coupling created is expensive.

## CQRS

CQRS (Command Query Responsibility Segregation) is a pattern that separates the data model used for writes (commands) from the data model used for reads (queries). In its simplest form, a write model represents the current state of the domain and enforces business rules, while a read model is optimized for specific query patterns.

Martin Fowler's article on CQRS at martinfowler.com is the essential reference. His observation is direct: CQRS is appropriate for a small subset of systems — specifically those with complex business domains where the write model and the read model have meaningfully different structures. It is not a general-purpose pattern.

A concrete example where CQRS makes sense: an e-commerce system where orders go through complex state transitions with many business rules (the write model), and the order history page needs to display a denormalized summary including computed fields from multiple related tables (the read model). Maintaining a separate, pre-computed read model avoids complex joins on every page load.

A practical, simplified version of CQRS: use your primary database for writes and use a materialized view or a separate reporting table for complex read queries. This is much simpler than a full CQRS implementation with separate event stores and projections, and it handles the same class of problem for most web applications.

Do not reach for CQRS because a query is slow. A slow query is almost always a problem of missing indexes, poor schema design, or excessive data being read. Fix those first.

---

## Sources

- Kleppmann, M. (2017). *Designing Data-Intensive Applications*. O'Reilly Media. — The essential reference for understanding database trade-offs, replication, consistency, ACID vs BASE, polyglot persistence.
- Hernandez, M. (2020). *Database Design for Mere Mortals* (4th Ed). Addison-Wesley. — Relational modeling, normalization, practical schema design.
- Fowler, M. (2011). "PolyglotPersistence." https://martinfowler.com/bliki/PolyglotPersistence.html — When multiple databases are appropriate.
- Fowler, M. (2011). "CQRS." https://martinfowler.com/bliki/CQRS.html — CQRS definition, when to use it.
- Redis Documentation. https://redis.io/docs/ — Data structures, persistence options, use case guidance.
- PostgreSQL Documentation. https://www.postgresql.org/docs/ — JSONB, full-text search, indexing types, EXPLAIN ANALYZE.