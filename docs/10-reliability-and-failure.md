# 10 — Reliability and Failure

The fundamental assumption of systems design — stated plainly in Michael Nygard's *Release It!* and repeated throughout the Google SRE book — is that everything fails. Disks fail. Network connections time out. Third-party APIs return unexpected errors. Databases run out of connections. Memory is exhausted. Bugs are deployed. These are not extraordinary events; they are the normal operating conditions of production software.

A reliable system is not one that never fails. It is one that fails gracefully — degrading in a controlled way rather than collapsing entirely, recovering quickly when components come back online, and protecting users from the consequences of failures they cannot control.

## Availability math

Availability is expressed as a percentage of time a system is operational. The numbers matter because they translate directly into maximum allowed downtime:

| Availability | Annual downtime | Monthly downtime |
|-------------|-----------------|------------------|
| 99% (two nines) | 87.6 hours | 7.3 hours |
| 99.9% (three nines) | 8.76 hours | 43.8 minutes |
| 99.99% (four nines) | 52.6 minutes | 4.4 minutes |
| 99.999% (five nines) | 5.26 minutes | 26.3 seconds |

The cost of each additional nine is not linear. Going from 99% to 99.9% requires eliminating most planned downtime and having a reliable deployment process. Going from 99.9% to 99.99% requires redundant infrastructure, automatic failover, and incident response processes that bring systems back online in minutes. Going from 99.99% to 99.999% requires multi-region redundancy, chaos engineering, and a dedicated reliability engineering function. Each step up is an order of magnitude more expensive than the previous one.

Know what availability your application actually needs before investing in it. Most internal tools are fine at 99%. Most SaaS products need 99.9%. Payment processing needs to be higher. Do not chase five nines because it sounds impressive.

## Single points of failure

A single point of failure (SPOF) is any component whose failure causes the entire system to fail. Identifying and eliminating SPOFs is the first step in improving availability.

Common SPOFs in web applications:

- A single database server with no replica (if it fails, all writes fail)
- A single application server with no load balancer (if it fails, all traffic fails)
- A single job worker (if it fails, background work stops processing)
- A third-party API that is called synchronously on every request (if it is slow or unavailable, every request is slow or fails)

Eliminating SPOFs typically means adding redundancy: a primary and at least one replica for the database, multiple application instances behind a load balancer, multiple job workers. The goal is not zero SPOFs — some are acceptable — but understanding which ones exist and making a deliberate decision about each.

Third-party dependencies deserve specific attention. An authentication provider, a payment gateway, an email service — each is a potential SPOF if it is called synchronously without a fallback. Design for the case where each external dependency is unavailable. At minimum, implement timeouts. Where possible, implement graceful degradation.

## Graceful degradation

Graceful degradation means serving partial functionality when a dependency is unavailable, rather than failing completely. The principle is that a degraded experience is better than no experience.

Practical examples: if your recommendation service is unavailable, show popular products instead of personalized recommendations. If your search index is unavailable, show a message that search is temporarily unavailable rather than returning a server error. If the email service is down, accept the registration and queue the welcome email rather than rejecting the registration.

Not all functionality can degrade gracefully. You cannot process a payment if the payment provider is unavailable. You can, however, ensure that the payment failure is handled cleanly — a clear error message, no charge, no partial order creation — rather than leaving the system in an inconsistent state.

## The Circuit Breaker pattern

The Circuit Breaker pattern, described by Michael Nygard in *Release It!* and documented by Martin Fowler at martinfowler.com, prevents repeated calls to a failing dependency. Without it, a slow or failing downstream service causes every request that depends on it to wait until timeout, holding resources and potentially cascading the failure.

A circuit breaker wraps calls to an external dependency with a state machine that has three states:

- **Closed** (normal operation): requests pass through to the dependency. Failures are counted.
- **Open** (dependency is failing): requests fail immediately without calling the dependency. An error is returned from the circuit breaker itself.
- **Half-open** (recovery probe): after a timeout, a test request is allowed through. If it succeeds, the circuit closes. If it fails, the circuit stays open.

A simplified implementation in Node.js:

```typescript
class CircuitBreaker {
  private failures = 0;
  private lastFailure = 0;
  private state: 'closed' | 'open' | 'half-open' = 'closed';

  constructor(
    private readonly threshold: number = 5,
    private readonly timeout: number = 30_000
  ) {}

  async call<T>(fn: () => Promise<T>): Promise<T> {
    if (this.state === 'open') {
      if (Date.now() - this.lastFailure > this.timeout) {
        this.state = 'half-open';
      } else {
        throw new Error('Circuit breaker is open — dependency unavailable');
      }
    }

    try {
      const result = await fn();
      this.onSuccess();
      return result;
    } catch (error) {
      this.onFailure();
      throw error;
    }
  }

  private onSuccess(): void {
    this.failures = 0;
    this.state = 'closed';
  }

  private onFailure(): void {
    this.failures++;
    this.lastFailure = Date.now();
    if (this.failures >= this.threshold) {
      this.state = 'open';
    }
  }
}

// Usage
const paymentCircuitBreaker = new CircuitBreaker(5, 30_000);

async function processPayment(data: PaymentData) {
  return paymentCircuitBreaker.call(() => paymentProvider.charge(data));
}
```

For production use, prefer a library that handles edge cases: `opossum` is the most mature Node.js circuit breaker library.

## Timeouts and retry strategy

Every call to an external system must have a timeout. Without a timeout, a slow external call can hold a connection indefinitely, exhausting the connection pool and blocking all requests that need that resource. This is one of the most common causes of cascading failures in production.

Set timeouts at the right level: not so short that normal variations in response time cause false failures, not so long that they allow slow failures to cascade. A reasonable default for most web service calls is 5 seconds. For operations that are known to be slow (generating a complex report, processing a large file), set higher timeouts explicitly.

When a call fails with a transient error — network timeout, 503 Service Unavailable, connection refused — it is often worth retrying. Retry strategy matters:

**Do not retry immediately.** The system that failed probably needs a moment to recover. Immediate retries add load to a system that is already struggling.

**Use exponential backoff.** Wait progressively longer between retries: 1 second, 2 seconds, 4 seconds, 8 seconds. This reduces load on the recovering system while still attempting recovery.

**Add jitter.** Randomize the retry delay slightly so that multiple clients do not all retry at exactly the same time. Without jitter, exponential backoff produces synchronized retry storms. The AWS Architecture Blog calls this pattern "exponential backoff and jitter" and recommends it explicitly.

**Do not retry non-idempotent operations.** Retrying a GET request is safe — you are only reading data. Retrying a POST request that creates a record may create duplicates. Only retry operations where retrying is safe, or implement idempotency keys so that retries are safe.

```typescript
async function withRetry<T>(
  fn: () => Promise<T>,
  maxAttempts = 3,
  baseDelayMs = 1000
): Promise<T> {
  let lastError: Error;

  for (let attempt = 1; attempt <= maxAttempts; attempt++) {
    try {
      return await fn();
    } catch (error) {
      lastError = error as Error;

      if (attempt < maxAttempts) {
        // Exponential backoff with jitter
        const delay = baseDelayMs * Math.pow(2, attempt - 1);
        const jitter = Math.random() * 0.3 * delay;
        await new Promise(resolve => setTimeout(resolve, delay + jitter));
      }
    }
  }

  throw lastError!;
}
```

## Health checks

Health check endpoints allow the infrastructure (load balancers, container orchestration, monitoring systems) to determine whether an application instance is working correctly.

There are two types: liveness and readiness.

A **liveness check** answers the question "is the process alive?" If the process is deadlocked, out of memory, or stuck in an infinite loop, the liveness check will fail and the infrastructure will restart the instance. A liveness check should be simple and fast — it should not call the database or any external service, because if those are unavailable, you do not want to trigger a restart loop.

A **readiness check** answers the question "is this instance ready to accept traffic?" A newly started instance may be alive but still initializing — running database migrations, warming caches, establishing connection pools. A readiness check verifies that the instance can serve requests correctly. Traffic should not be routed to an instance until its readiness check passes.

```typescript
// Liveness — is the process alive?
app.get('/health/live', (_req, res) => {
  res.status(200).json({ status: 'ok' });
});

// Readiness — can this instance serve traffic?
app.get('/health/ready', async (_req, res) => {
  try {
    // Check database connectivity
    await db.$queryRaw`SELECT 1`;
    // Check Redis connectivity
    await redis.ping();
    res.status(200).json({ status: 'ready' });
  } catch (error) {
    res.status(503).json({ status: 'not ready', error: (error as Error).message });
  }
});
```

## Idempotency

An operation is idempotent if performing it multiple times produces the same result as performing it once. GET requests are idempotent by definition. PUT requests are designed to be idempotent (setting a resource to a specific state is the same whether done once or ten times). POST requests are not idempotent by default.

Idempotency matters for reliability because retries — network retries, client retries, infrastructure retries — can cause the same operation to be received multiple times. For operations that have side effects (creating a charge, sending an email, creating a record), duplicate execution is a bug.

The standard solution is idempotency keys: the client generates a unique key for each operation and includes it in the request. The server checks whether it has already seen this key before processing the operation. If it has, it returns the result of the previous execution rather than executing again. This is the pattern Stripe uses for all payment API calls.

```typescript
app.post('/api/orders', async (req, res) => {
  const idempotencyKey = req.headers['idempotency-key'] as string;

  if (!idempotencyKey) {
    return res.status(400).json({ error: 'Idempotency-Key header is required' });
  }

  // Check if this operation was already processed
  const cached = await redis.get(`idempotency:${idempotencyKey}`);
  if (cached) {
    return res.status(200).json(JSON.parse(cached));
  }

  const order = await createOrder(req.body);

  // Cache the result for 24 hours
  await redis.setex(`idempotency:${idempotencyKey}`, 86400, JSON.stringify(order));

  res.status(201).json(order);
});
```

## Observability basics

Observability is the ability to understand what is happening inside a system from its external outputs. The three signals are logs, metrics, and traces.

**Logs** record discrete events: a request was received, an error occurred, a job was processed. Structured logs (JSON format) are searchable and filterable in log aggregation systems. Every log entry should include a timestamp, a severity level, a correlation ID that can be used to trace a request across log lines, and relevant context.

**Metrics** are numeric measurements over time: request count per second, error rate, P99 latency, database connection pool usage. Metrics are aggregated — you do not store a metric for every request, you store counts and percentiles. They answer "what is happening" questions at a system level.

**Traces** show the lifecycle of a request through the system — which functions were called, in what order, and how long each took. For a monolith, a stack trace is often sufficient. For a distributed system, distributed tracing (OpenTelemetry, Jaeger, Datadog APM) correlates spans across services.

The minimum useful observability setup for a web application: structured application logs aggregated into a searchable system (Logtail, Papertrail, CloudWatch Logs), error tracking that groups and alerts on application errors (Sentry), and uptime monitoring that alerts within minutes of a complete outage (Better Uptime, UptimeRobot). Add metrics and distributed tracing when the complexity of your system justifies the investment.

## Incident response basics

When production fails, the speed and quality of the response matter. Two practices significantly improve both:

**Runbooks** are documented procedures for known failure scenarios. When the database connection pool is exhausted, the runbook says: check PgBouncer stats, look for long-running transactions, identify the source, restart the connection pool if necessary. A developer who has never seen this failure before can follow the runbook without needing to debug from scratch in a high-stress situation.

**Rollback procedure** is the single most important operational capability. Every deploy should be reversible. Know exactly how to revert to the previous version — what command to run, what migration to reverse, who needs to be notified. Practice it before you need it. A rollback that takes 2 minutes in practice takes 20 in a crisis if it has never been practiced.

---

## Sources

- Nygard, M. (2018). *Release It!* (2nd Ed). Pragmatic Bookshelf. — Stability patterns: Circuit Breaker, Timeout, Bulkhead, Steady State.
- Beyer, B. et al. (2016). *Site Reliability Engineering*. Google / O'Reilly. https://sre.google/sre-book/ — Error budgets, incident management, availability math.
- Burns, B. (2018). *Designing Distributed Systems*. O'Reilly Media. — Reliability patterns for distributed systems.
- Fowler, M. "CircuitBreaker." https://martinfowler.com/bliki/CircuitBreaker.html — Circuit breaker pattern definition and state machine.
- AWS Architecture Blog. (2015). "Exponential Backoff and Jitter." https://aws.amazon.com/blogs/architecture/exponential-backoff-and-jitter/ — Retry strategy with jitter.
- Stripe API Docs. "Idempotent Requests." https://stripe.com/docs/api/idempotent_requests — Idempotency key implementation reference.
- AWS Well-Architected Framework, Reliability Pillar. https://docs.aws.amazon.com/wellarchitected/latest/reliability-pillar/ — Failure management patterns.
- OpenTelemetry Documentation. https://opentelemetry.io/docs/ — Vendor-neutral observability standard.