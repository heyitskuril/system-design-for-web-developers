# 06 — API Design and Communication

The communication patterns you choose between components of your system — and the APIs you design for those components — have a larger impact on the day-to-day experience of developing and operating that system than almost any other decision. Get them right, and adding features is straightforward, debugging is tractable, and the system behaves predictably. Get them wrong, and you spend engineering time working around the communication layer instead of building product.

This section covers three questions: what communication pattern to use for a given interaction, which API style fits which scenario, and how to design APIs that are easy to use and change over time.

## The three communication patterns

**Synchronous request/response** is the default and, for most operations, the correct choice. The client sends a request and waits for a response before proceeding. HTTP APIs — whether REST or GraphQL — are synchronous. The caller knows immediately whether the operation succeeded or failed, and can respond to the user accordingly.

The constraint of synchronous communication is that the caller is blocked while waiting. If the operation takes time — generating a report, processing a file, sending a batch of emails — the HTTP connection is held open and the user waits. More importantly, if the downstream system is slow or unavailable, the caller is blocked and potentially cascading failures can occur.

**Asynchronous messaging** decouples the sender from the receiver by introducing a queue. The sender puts a message on the queue and continues immediately. The receiver picks up the message and processes it when it is ready. Neither party is blocked waiting for the other. This pattern is correct for operations that are genuinely background work: sending email, generating PDF reports, processing uploaded files, syncing data to third-party systems.

The trade-off is that the user's immediate confirmation is limited to "your request was received" rather than "your operation is complete." This is fine for operations where the user does not need an immediate result — a confirmation email, a weekly report, a data export. It is wrong for operations where the user's next action depends on the result — adding an item to a cart, placing an order, logging in.

**Event streaming** is a pattern where events are published to a persistent log and consumed by multiple subscribers, potentially long after the event occurred. Kafka is the most common implementation. This pattern is appropriate when multiple systems need to react to the same event independently, or when you need a durable history of events that can be replayed. For most web applications, event streaming is not necessary. A simple job queue handles the async use cases that actually arise.

## REST versus GraphQL

This is a genuine trade-off, not a matter of preference, and the right answer depends on what you are building.

REST (Representational State Transfer) is the appropriate default for most web APIs. REST APIs map naturally to HTTP — resources have URLs, HTTP methods (GET, POST, PUT, PATCH, DELETE) express the type of operation, and HTTP status codes communicate the result. REST APIs are easy to cache — a GET to `/api/products/123` returns the same data every time until the product changes, and HTTP caching machinery (CDN, browser cache, reverse proxy) works transparently. REST is well-understood, well-documented, and supported by every HTTP client in existence. The tooling for testing, documenting, and monitoring REST APIs is mature.

The weakness of REST is over-fetching and under-fetching. A mobile client that needs just the product name and price for a list of 20 products will receive the full product object for each — everything the API returns, whether the client needs it or not. A screen that needs data from three different resources makes three separate requests. For complex, multi-client scenarios, this is genuinely inefficient.

GraphQL addresses both problems. The client specifies exactly which fields it needs, and the server returns exactly those fields. A single GraphQL query can fetch data from multiple resources in one request. This is valuable when you have multiple clients with different data requirements (a mobile app and a web app that need different subsets of the same data), or when the frontend is frequently modified by a team that wants independence from backend changes.

The trade-offs against REST are real. GraphQL responses are POST requests to a single endpoint — HTTP caching does not work without additional tooling. The server-side implementation is more complex: you need a GraphQL resolver layer that maps each field to a data source. The N+1 problem — where fetching a list of items and then the related data for each item causes N+1 database queries — is a known hazard that requires DataLoader or similar batching solutions. Error handling is non-standard: GraphQL returns 200 OK even when there are errors, which are embedded in the response body.

The decision rule: use REST as the default. Use GraphQL when you have multiple clients with genuinely different data requirements, or when a frontend team needs the flexibility to query for different data combinations without requiring backend changes. Do not use GraphQL to solve performance problems that are actually caused by missing indexes, poor query design, or excessive database calls — those problems will persist behind a GraphQL layer.

## Internal versus external API design

Public APIs — consumed by third parties, developers building on your platform, or clients you do not control — have different design requirements than internal APIs consumed only within your own system.

Public APIs must prioritize stability. Breaking changes affect users you cannot contact and will not be forgiven easily. Public APIs should be versioned from day one. Design for the fact that clients will not update immediately — or possibly ever. The resources you expose in a public API become a contract that is expensive to change.

Lauret's *The Design of Web APIs* covers public API design in depth. The principles that stand out for web developers: design for your users, not your database; use consistent naming conventions (pick snake_case or camelCase and stick to it everywhere); never expose internal implementation details (your database table names, your ORM's generated field names, your internal ID schemes); and design for the operation the user is performing, not the data model you happen to have.

Internal APIs — between services you control, or between your frontend and your own backend — can afford more pragmatism. You can make breaking changes by updating both sides simultaneously. You can use more compact representations. You can expose internal details that would be inappropriate in a public API. This pragmatism does not mean internal API design does not matter — it means different trade-offs apply.

## Synchronous versus asynchronous: making the call

The decision between synchronous and asynchronous communication is not primarily technical — it is about the semantics of the operation.

Ask two questions: Does the user's next action depend on knowing the result? And does the operation need to complete before you respond?

If both answers are yes, the operation is synchronous. Logging in, adding a product to a cart, placing an order — the user needs to know the outcome before they can proceed.

If the operation is work that happens in the background — work the user set in motion but does not need to wait for — it is asynchronous. Sending a welcome email after registration, generating a PDF report, processing an uploaded CSV file, sending a webhook to a third-party system.

The practical consequence: every API endpoint that triggers background work should accept the request, enqueue a job, and return immediately with a 202 Accepted response. The work happens asynchronously. The user is not blocked. If the job fails, it can be retried without the user being involved.

## Background job queues

Message queues for background jobs — BullMQ with Redis, RabbitMQ, or a database-backed queue — are one of the highest-value tools in a web application's architecture. They decouple the API layer from the work layer, provide automatic retry with exponential backoff, and give you a dashboard for monitoring the state of pending and failed jobs.

The pattern is straightforward:

```typescript
// API endpoint — accepts the request and enqueues immediately
app.post('/api/reports', async (req, res) => {
  const job = await reportQueue.add('generate-report', {
    userId: req.user.id,
    reportType: req.body.reportType,
    dateRange: req.body.dateRange,
  });

  res.status(202).json({
    message: 'Report generation started',
    jobId: job.id,
  });
});

// Worker process — picks up jobs and processes them
reportQueue.process('generate-report', async (job) => {
  const { userId, reportType, dateRange } = job.data;
  const reportData = await generateReport(reportType, dateRange);
  const pdf = await renderPDF(reportData);
  await uploadToStorage(pdf, userId);
  await sendEmailWithLink(userId, storageUrl);
});
```

The API response time is milliseconds — just long enough to write to the queue. The actual work happens in a separate process on whatever timeline it requires. If the report generation fails, BullMQ retries it automatically according to the configured retry strategy.

Common use cases that belong in a queue: email sending, SMS notifications, webhook delivery, PDF generation, image processing, data export, third-party sync, and any operation that takes more than a few hundred milliseconds.

## The API gateway pattern

An API gateway sits in front of multiple services or APIs and provides a single entry point for clients. It handles cross-cutting concerns: authentication, rate limiting, request logging, SSL termination, routing requests to the appropriate backend service.

For a monolith or modular monolith, you almost certainly do not need an API gateway. Your API server handles all of this directly. Add it when you have multiple services that each need these cross-cutting concerns, and when centralizing them reduces duplication and simplifies client integration.

For a microservices architecture, an API gateway is usually appropriate. Without it, clients need to know the address of every service and handle authentication independently with each one. That coupling is exactly what the API gateway eliminates.

Popular implementations include Kong, AWS API Gateway, and NGINX. For most teams, this is not a decision to make early — it is a decision to make when the need is clear.

## gRPC

gRPC is a remote procedure call framework developed by Google that uses HTTP/2 as its transport and Protocol Buffers as its serialization format. It generates client and server code from a schema definition, provides strong typing across service boundaries, and is significantly more performant than JSON over HTTP for high-throughput inter-service communication.

gRPC is appropriate for internal service-to-service communication where performance is critical and you control both the client and the server. It is not appropriate for browser-facing APIs — browsers do not support raw HTTP/2 gRPC calls without a proxy layer like grpc-web, which adds complexity.

For most web developers, gRPC is not a decision to make on a new project. If you have a services architecture where inter-service latency is a measured problem, it is worth evaluating. If you are building a typical web application, REST over HTTPS is simpler and adequate.

---

## Sources

- Kleppmann, M. (2017). *Designing Data-Intensive Applications*. O'Reilly Media. — Encoding and evolution, message passing, RPC trade-offs.
- Lauret, A. (2019). *The Design of Web APIs*. Manning. — REST API design principles, versioning, consumer-oriented design.
- Richardson, L. & Amundsen, M. (2013). *RESTful Web APIs*. O'Reilly Media. — REST constraints, hypermedia, resource design.
- GraphQL Specification. https://spec.graphql.org/ — Official GraphQL specification and rationale.
- Newman, S. (2021). *Building Microservices* (2nd Ed). O'Reilly Media. — Chapter on communication patterns, sync vs async, API gateways.
- BullMQ Documentation. https://docs.bullmq.io/ — Job queue implementation reference.
- gRPC Documentation. https://grpc.io/docs/ — Protocol specification and language support.
- Microsoft Azure. "API design best practices." https://learn.microsoft.com/en-us/azure/architecture/best-practices/api-design