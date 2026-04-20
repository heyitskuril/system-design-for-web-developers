# 11 — Real-World Examples

The previous sections introduced patterns and principles. This section applies them to concrete scenarios — not "design Twitter" or "design YouTube," but the kind of products web developers actually build. Each scenario walks through the architectural decisions that matter at a specific stage, explains why those decisions are correct, and identifies what you should not build yet.

---

## Scenario 1: A SaaS dashboard product, 0 to 10,000 users

You are building a B2B SaaS product. Companies subscribe, their employees log in, and they see data, manage workflows, and collaborate. You are starting from zero. You expect to reach 10,000 registered users — maybe 2,000–3,000 DAU — within the first year if things go well.

### What architecture to start with

Start with a monolith. There is no legitimate technical argument for anything else at this stage. You do not know your domain well enough to define stable service boundaries. The team is small. The operational overhead of a distributed system would slow you down without providing any benefit.

The application is a Node.js API server and a React SPA. They are separate deployable units (the API deploys independently of the frontend build), but the API is a single process. There is one PostgreSQL database and one Redis instance for sessions and caching.

```
[Browser] React SPA
    |
    | HTTPS
    |
[Server] Node.js API (Express or Fastify)
    |          |
    |          | Redis (sessions, cache)
    |
[Database] PostgreSQL
```

This architecture handles 10,000 users without breaking a sweat. A single well-configured application server can handle thousands of concurrent connections. PostgreSQL scales to millions of rows on a modest server. Redis on the same instance adds negligible overhead.

### Database design at this scale

Organize tables around your core entities. For a typical SaaS product: `users`, `organizations`, `memberships` (the join table between users and organizations), and whatever your product's core entities are. Add `created_at` and `updated_at` to every table. Add `deleted_at` for soft deletes where you need to preserve history.

Multi-tenancy at this scale is row-level: every table that belongs to a tenant has an `organization_id` column, and every query includes `WHERE organization_id = $1`. This is simple and correct at this scale. It is also the design that is easiest to migrate from if you ever need schema-level or database-level tenant isolation.

Enforce tenant isolation in the service layer, not the controller layer. A function like `getProducts(organizationId)` that takes the organization ID as a required parameter is harder to get wrong than a function that reads the organization ID from a global request context.

### What you do not need yet

You do not need a message queue. If you are sending a welcome email after registration, call the email provider synchronously. If it fails, log the failure and move on. The complexity of a job queue is not justified until you have high-volume async work or need guaranteed delivery with retry logic.

You do not need Redis caching. You need Redis for sessions. Cache-aside caching is a performance optimization — add it when you have measured a specific performance problem, not before. A query that takes 20ms does not need to be cached.

You do not need read replicas. Your write load and read load are both minimal. A single PostgreSQL instance handles both.

You do not need a CDN for API responses. A CDN is appropriate for static assets — your JavaScript bundle, images, CSS. API responses are dynamic and should not be cached at the CDN level unless you have specific, cacheable endpoints that serve the same data to many users.

You do not need microservices. You do not need a service mesh. You do not need Kubernetes. You need a well-structured monolith running on a managed platform (Fly.io, Railway, Render, a single EC2 or Droplet) with a managed database (Supabase, Neon, RDS, or similar).

The entire architecture — API server, PostgreSQL, Redis — runs on infrastructure that costs $50–100 per month and is adequate for your first 10,000 users and probably your first 100,000.

---

## Scenario 2: An e-commerce platform with a flash sale

You are running an e-commerce platform. Normal daily traffic is manageable on your existing infrastructure. You are planning a flash sale — a limited-time discount on high-demand products — and you expect 10x–50x normal traffic for a period of 2–4 hours. You have had issues in the past with slow page loads and inventory overselling.

### The bottlenecks under load

Flash sales stress three specific parts of an e-commerce system:

**Product catalog reads.** The product listing and product detail pages are read by everyone who visits. At 50x traffic, every page load hitting the database is not sustainable. These pages are largely static — the product name, description, and images do not change during the sale.

**Inventory queries.** "Is this item in stock?" is asked constantly during a sale. The answer changes (stock decreases with each purchase), but not on every single request — it changes at most a few times per second. Checking the live database for every request is expensive and unnecessary.

**Order creation writes.** This is the critical path — the place where you must not lose data, must not double-charge, must not oversell. This part cannot be cached or approximated. It must be correct.

### Caching strategy for the product catalog

The product catalog is an excellent candidate for caching. Cache the full product object (name, description, images, base price) for each product with a 5–10 minute TTL. At 50x traffic, this reduces database reads for product data from thousands per minute to a handful of cache refresh operations.

For the product listing pages, consider server-side rendering with ISR (Incremental Static Regeneration) or a CDN cache with a short TTL. The listing page with 20 products does not change every second — caching it for 60 seconds at the CDN reduces origin load by orders of magnitude.

Inventory display is handled separately from inventory actuals. Show a cached "available / low stock / out of stock" status on the listing page. This status is refreshed every 30–60 seconds from the actual inventory count. Do not show the exact inventory number if you want to avoid the thundering herd of users refreshing to monitor it.

### Queue-based order processing

Order creation during a flash sale should not attempt to process everything synchronously. The pattern that prevents overselling and handles traffic spikes:

1. User submits order. The API validates the request (complete data, valid payment method) and places the order into a queue. Returns immediately with 202 Accepted and an order reference number.
2. The order worker pulls orders from the queue and processes them one at a time (or with controlled concurrency). For each order: check inventory with a database transaction, decrement inventory if available, process payment, confirm or reject the order, send confirmation email via another queue.
3. The user polls a status endpoint (or receives a WebSocket push) to learn when their order is confirmed.

The queue serializes the high-concurrency write load. Instead of 500 simultaneous database transactions competing for inventory row locks, the queue processes orders sequentially, preventing race conditions and overselling. BullMQ with Redis handles this cleanly.

The pessimistic locking strategy for inventory deduction:

```typescript
async function reserveInventory(
  productId: string,
  quantity: number,
  orderId: string
): Promise<boolean> {
  return db.$transaction(async (tx) => {
    // Lock the inventory row for this transaction
    const inventory = await tx.$queryRaw<Array<{ available: number }>>`
      SELECT available FROM inventory
      WHERE product_id = ${productId}
      FOR UPDATE
    `;

    if (!inventory[0] || inventory[0].available < quantity) {
      return false; // Out of stock
    }

    await tx.$executeRaw`
      UPDATE inventory
      SET available = available - ${quantity},
          reserved = reserved + ${quantity}
      WHERE product_id = ${productId}
    `;

    await tx.orderItem.update({
      where: { orderId_productId: { orderId, productId } },
      data: { status: 'reserved' },
    });

    return true;
  });
}
```

The `FOR UPDATE` lock ensures that only one transaction at a time can decrement inventory for a given product. Other transactions wait for the lock to be released. This prevents overselling without the complexity of optimistic locking retry loops.

### Infrastructure changes versus application code

Before modifying infrastructure, exhaust application-level options. In this scenario: caching the product catalog, queuing order processing, and adding a database index on the inventory table's `product_id` column (if it is missing) probably handles the flash sale without any infrastructure changes.

If the application server becomes the bottleneck — CPU is maxing out, response times are climbing — add a second application instance behind a load balancer. Because the application is stateless (sessions in Redis), this is a one-command operation on any managed platform. The database should not be the bottleneck if caching is in place.

---

## Scenario 3: A multi-tenant B2B SaaS scaling from 10 to 500 business customers

You have 10 enterprise customers. Each has dozens to hundreds of users. You are growing and expect to reach 500 customers. Your current architecture is a monolith with row-level multi-tenancy. You need to think about what changes as you scale.

### Tenancy architecture decision

Row-level multi-tenancy (every table has an `organization_id` column) works well up to a few hundred tenants in a single database. Its advantages are simplicity — one database, one schema, easy queries across tenants for reporting. Its disadvantages are data isolation (a bug can theoretically expose one tenant's data to another) and performance isolation (a large tenant's queries can slow down a small tenant's experience).

Schema-level tenancy (one PostgreSQL schema per tenant, each with the same table structure) provides better isolation without the complexity of separate databases. Queries within a tenant execute within that tenant's schema. Cross-tenant operations require explicit schema switching. PostgreSQL supports this natively and it is a reasonable choice for hundreds of customers.

Database-level tenancy (separate database per tenant) provides the strongest isolation and is appropriate when enterprise contracts require data residency guarantees or when regulatory compliance demands it. The operational cost is high: hundreds of databases to back up, monitor, and migrate. Use it when the contractual or regulatory requirement is real, not speculatively.

For 10–500 customers, row-level tenancy with careful query isolation is correct. The migration path to schema-level tenancy, if you need it later, is a database-level operation that does not require changing application logic if the service layer already enforces tenant isolation correctly.

### When to extract the first service

At some point, the modular monolith benefits from extracting a service. The trigger is usually one of two things: a part of the system has genuinely different scaling requirements (the data processing pipeline that runs large reports needs significantly more CPU than the web API), or a team is large enough and a domain stable enough to warrant independent deployment.

The first service to extract is almost never a user-facing feature. It is usually a background processing function: report generation, data export, bulk email sending, analytics computation. These have three properties that make extraction safe: they communicate with the rest of the system asynchronously (via queue), they have no user-facing latency requirement, and they are self-contained enough to have clear boundaries.

The strangler fig pattern, from Martin Fowler, describes the correct migration: build the new service alongside the existing monolith, gradually route traffic or work to the new service, and eventually remove the corresponding code from the monolith. Never do a big-bang rewrite. Extract the specific function, verify it works in production, then remove the monolith code.

### Authorization complexity at scale

Enterprise B2B SaaS has complex authorization requirements: roles at the organization level, roles at the workspace or project level, resource-level permissions, custom roles defined by enterprise admins. This complexity grows with the customer count and should be designed for, not bolted on.

The pattern that scales is attribute-based access control (ABAC) or policy-based authorization: instead of hard-coding role checks (`if user.role === 'admin'`), the authorization system evaluates a policy against a set of attributes (user attributes, resource attributes, environment attributes). Open Policy Agent (OPA) is a widely adopted policy engine for this pattern.

For most SaaS products below the complexity threshold, a simple RBAC model (roles with permissions, enforced in the service layer) is sufficient and much simpler to build and understand. Build the simplest authorization model that meets your actual requirements. Migrate to a more sophisticated model when you have customers with requirements that the simple model cannot meet.

---

These three scenarios are not exhaustive, but they cover the decision space that most web developers encounter. The patterns are the same regardless of the specific domain: start simple, measure, add complexity only when the problem justifies it, and ensure that every added component earns its operational cost.

---

## Sources

- Newman, S. (2021). *Building Microservices* (2nd Ed). O'Reilly Media. — Strangler fig pattern, service extraction strategy.
- Fowler, M. "StranglerFigApplication." https://martinfowler.com/bliki/StranglerFigApplication.html — Incremental migration pattern.
- Kleppmann, M. (2017). *Designing Data-Intensive Applications*. O'Reilly Media. — Database transaction patterns, row locking.
- Microsoft Azure. "Multitenancy architecture approaches." https://learn.microsoft.com/en-us/azure/architecture/guide/multitenant/overview — Row, schema, and database tenancy trade-offs.
- Heroku. (2011). *The 12-Factor App.* https://12factor.net/ — Stateless processes, backing services.
- Open Policy Agent Documentation. https://www.openpolicyagent.org/docs/ — Policy-based authorization reference.