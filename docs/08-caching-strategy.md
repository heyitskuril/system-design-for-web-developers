# 08 — Caching Strategy

Caching is one of the highest-leverage performance tools available to a web developer. It is also the source of a class of bugs that are difficult to reproduce, hard to debug, and caused by one of the two hard problems in computer science that Phil Karlton identified: "There are only two hard things in computer science: cache invalidation and naming things." This is not a joke. Cache invalidation failures in production — showing a user stale data after an update, serving an old version of a file after a deploy, returning incorrect counts after a deletion — are a predictable consequence of adding caching without thinking through the invalidation strategy.

This section covers the five types of caching a web developer needs to understand, the common patterns for each, and the specific failure modes to plan for.

## Why caching works

Caching works because of the principle of locality: recently accessed data is likely to be accessed again soon (temporal locality), and data near recently accessed data is likely to be accessed soon (spatial locality). Computing a result is more expensive than returning a stored copy of that result, and the cost difference is often large — a database query that takes 50ms returns in microseconds from an in-memory cache.

The principle applies at every level of the system: the CPU has caches for memory access, the operating system has a page cache for disk access, the browser has a cache for HTTP responses, the CDN has a cache for static assets, and the application has a cache for expensive computations. Every level of caching is an application of the same underlying principle.

## The five types of caching

**CDN caching** stores static assets — JavaScript bundles, CSS files, images, fonts — at edge locations close to users. The CDN serves these files from the nearest point of presence rather than from your origin server, reducing latency and eliminating load on your infrastructure for requests that do not require dynamic content. Cloudflare, AWS CloudFront, and Fastly are common CDN providers.

CDN caching is controlled by HTTP Cache-Control headers set by your origin server. `Cache-Control: public, max-age=31536000, immutable` tells the CDN (and browsers) to cache this file for one year and that it will never change — appropriate for content-hashed static assets where the filename changes when the content changes. The `immutable` directive tells the browser not to revalidate during the max-age window even on a hard refresh.

**HTTP caching** is the browser's built-in mechanism for caching responses from HTTP servers. It is controlled by Cache-Control headers and, for conditional requests, by ETags and Last-Modified headers. A proper HTTP caching setup reduces the number of requests that reach your server and improves perceived performance for repeat visitors.

The `stale-while-revalidate` directive is particularly useful for dynamic content: `Cache-Control: max-age=60, stale-while-revalidate=3600` tells the browser to serve the cached response for up to 60 seconds, and if the response is stale, to serve it anyway while fetching a fresh version in the background. The user sees a fast response even after the cache expires, and the fresh version is ready for the next request.

ETags provide conditional caching: the server sends an ETag header (a hash of the response content) with each response. On subsequent requests, the browser sends `If-None-Match: <etag>` and the server returns either a 304 Not Modified (if the content has not changed, saving bandwidth) or a fresh 200 response with the new ETag.

**Application-level caching** stores the results of expensive operations — database queries, external API calls, computed aggregates — in a fast in-memory store like Redis. When the application needs data, it first checks the cache; if the data is there and not expired (a cache hit), it returns the cached value. If the data is not in the cache (a cache miss), it fetches from the source, stores the result in the cache, and returns it.

This is the cache-aside pattern (also called lazy loading), the most common caching pattern in web applications. In Node.js with `ioredis`:

```typescript
import { Redis } from 'ioredis';

const redis = new Redis(process.env.REDIS_URL);

async function getProduct(productId: string) {
  const cacheKey = `product:${productId}`;

  // Check cache first
  const cached = await redis.get(cacheKey);
  if (cached) {
    return JSON.parse(cached);
  }

  // Cache miss — fetch from database
  const product = await db.product.findUnique({ where: { id: productId } });

  if (product) {
    // Store in cache for 5 minutes
    await redis.setex(cacheKey, 300, JSON.stringify(product));
  }

  return product;
}
```

The pattern is straightforward. The invalidation question is not. When a product is updated, the cached version becomes stale. The cache entry must be explicitly deleted so the next read fetches the fresh version:

```typescript
async function updateProduct(productId: string, data: Partial<Product>) {
  const updated = await db.product.update({
    where: { id: productId },
    data,
  });

  // Invalidate the cache entry
  await redis.del(`product:${productId}`);

  return updated;
}
```

The failure mode: if the invalidation is forgotten, or if it fails silently, users see stale data after updates. Every cached value must have a corresponding invalidation path.

**Database query caching** refers to the database engine's own internal query plan cache. PostgreSQL caches query execution plans, which means a repeated identical query does not need to be parsed and planned every time. This is automatic and transparent — you do not configure it directly. Understanding it matters because it has implications for parameterized queries: a query with parameters that change the execution plan may behave differently than expected if the cached plan is suboptimal for the new parameter values. For most web developers, this is background knowledge rather than something to act on, but it explains occasional counterintuitive performance behavior.

**In-memory application cache** is a cache stored directly in the application process's memory — a JavaScript `Map`, a module-level object, or a library like `node-cache`. It is faster than Redis (no network round trip) and appropriate for values that are expensive to compute, change rarely, and are the same for all users: configuration values loaded from the database, lookup tables, feature flags, translated content.

```typescript
import NodeCache from 'node-cache';

const cache = new NodeCache({ stdTTL: 600 }); // 10 minutes default TTL

async function getConfigurationValue(key: string): Promise<string> {
  const cached = cache.get<string>(key);
  if (cached !== undefined) {
    return cached;
  }

  const value = await db.configuration.findUnique({ where: { key } });
  cache.set(key, value?.value ?? '');
  return value?.value ?? '';
}
```

The limitation of in-memory caching in a multi-instance application: each instance has its own cache. If instance A updates a value and invalidates its local cache, instance B still serves the stale cached version until its own TTL expires. For values that change infrequently, this is acceptable. For values where consistency across instances matters, use Redis.

## Write strategies

The cache-aside pattern (read-through lazy loading) is correct for most use cases. There are two additional write strategies worth knowing.

**Write-through caching** updates the cache every time a value is written to the primary store. Writes go to both the cache and the database. The advantage: the cache is always warm and always consistent with the database. The disadvantage: you pay the cache write cost on every database write, including writes for values that will never be read from the cache.

**Write-behind (write-back) caching** writes to the cache first and to the database asynchronously. Writes are fast because they only hit the in-memory cache. The disadvantage: if the system fails before the write is persisted to the database, data is lost. This strategy is appropriate for high-write scenarios where some data loss is acceptable (telemetry, counters) and inappropriate for anything that must be durable (orders, payments, user accounts).

## Cache warming

A cold cache — a cache that has just started, with no entries — causes all reads to go to the database until entries are populated. For systems with predictable access patterns, cache warming pre-populates the cache before traffic arrives. A scheduled job runs before a deployment or traffic spike and populates the cache with the most frequently accessed data.

This is worth implementing for high-traffic scenarios like a flash sale (pre-warm the product catalog) or a marketing campaign (pre-warm the landing page data). For normal operations, the cache warms itself naturally within a few minutes of startup.

## The cache stampede problem

When a cached value expires, many concurrent requests may find an empty cache simultaneously and each send a query to the database. If the query is expensive and the request volume is high, the resulting database load can cause a cascade failure. This is the thundering herd problem, also known as cache stampede.

Three solutions address it:

**Mutex lock on cache miss.** When a thread finds an empty cache, it acquires a lock before querying the database. Other threads that also find an empty cache wait for the lock. Only one thread queries the database; all threads receive the result.

```typescript
async function getWithLock(key: string): Promise<Product | null> {
  const cached = await redis.get(key);
  if (cached) return JSON.parse(cached);

  const lockKey = `lock:${key}`;
  const acquired = await redis.set(lockKey, '1', 'EX', 5, 'NX');

  if (!acquired) {
    // Another request is populating the cache — wait and retry
    await new Promise(resolve => setTimeout(resolve, 100));
    return getWithLock(key);
  }

  try {
    const value = await db.product.findUnique({ where: { id: key } });
    await redis.setex(key, 300, JSON.stringify(value));
    return value;
  } finally {
    await redis.del(lockKey);
  }
}
```

**Probabilistic early expiration.** Before the TTL expires, randomly decide (with increasing probability as the expiration approaches) to refresh the value. This spreads the refresh work across multiple requests rather than concentrating it at the exact expiration moment.

**Stale-while-revalidate at the application level.** Serve the stale cached value while a single background process refreshes it. Users see slightly stale data for a brief window, but no request is ever blocked by an empty cache.

## TTL strategy

The TTL (time-to-live) for a cached value should reflect how often the data changes and how stale it is acceptable to be.

- Static reference data (country list, product categories, configuration values): 10–60 minutes or longer
- User profile data: 5–15 minutes, with explicit invalidation on update
- Product catalog: 1–10 minutes depending on update frequency, with explicit invalidation on update
- Shopping cart: do not cache; reads should always be fresh
- Session data: set to the session timeout duration
- Real-time counts, inventory levels, prices at checkout: do not cache at the application level; these must be accurate

## What not to cache

Some data must not be cached, or requires careful isolation if it is:

- User-specific sensitive data (access tokens, payment methods) should never be cached without strict key isolation per user — `user:123:payment-methods` not `payment-methods`. If you accidentally return user A's cached data to user B, you have a serious security incident.
- Financial totals and inventory levels at the point of transaction must be read from the primary database. Cache them for display purposes (showing available stock), but re-read at the moment of the actual operation (checkout, stock reservation).
- Data where stale values cause incorrect behavior. A product price shown on a listing page can be slightly stale. The price charged at checkout must be fresh.

---

## Sources

- Kleppmann, M. (2017). *Designing Data-Intensive Applications*. O'Reilly Media. — Caching in distributed systems, consistency implications.
- Xu, A. (2020). *System Design Interview – An Insider's Guide*. ByteByteGo. — Caching patterns, write strategies, cache stampede.
- Fowler, M. "Cache." https://martinfowler.com/eaaCatalog/ — Cache pattern in the enterprise application patterns catalog.
- Redis Documentation. https://redis.io/docs/ — Data structures, persistence, eviction policies.
- Cloudflare. "Cache-Control directives." https://developers.cloudflare.com/cache/concepts/cache-control/ — CDN caching behavior.
- MDN Web Docs. "HTTP caching." https://developer.mozilla.org/en-US/docs/Web/HTTP/Caching — Browser cache behavior, Cache-Control, ETags.
- Marcotte, B. et al. (2019). "Caching Best Practices." https://web.dev/http-cache/ — Web.dev guidance on HTTP cache headers.