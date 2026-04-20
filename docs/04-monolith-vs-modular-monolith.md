# 04 — Monolith vs Modular Monolith

"Monolith" has acquired a pejorative meaning in software engineering that it does not deserve. When developers use the word, they usually mean something like: large, unmaintainable, impossible to change, with everything tangled together. What they are actually describing is a big ball of mud — an architecture with no internal structure, where any piece of code can depend on any other piece of code and nothing has a clear boundary. That is a real problem. But it is a code quality problem, not an architectural style problem, and conflating the two causes teams to make expensive mistakes.

A monolith, properly understood, is simply an application that is deployed as a single unit. There is nothing inherently wrong with it. The question is whether it has internal structure — and that question is entirely separate from whether it is a monolith.

## The case for the monolith

The monolith has genuine advantages that compound in significance for small teams and early-stage products. They are worth stating plainly because the industry tends to talk about them apologetically, as concessions to practicality rather than legitimate architectural virtues.

**Deployment is simple.** A single deployable unit means one thing to build, one thing to test end-to-end, one thing to roll back if something goes wrong. With microservices, every service has its own deployment pipeline, its own versioning, its own rollback procedure. The operational overhead is real and significant, and it starts on day one.

**Debugging is straightforward.** When a transaction fails in a monolith, the stack trace tells you exactly what happened. When a request fails in a microservices system, it may span five services, three databases, and two message queues — debugging it requires distributed tracing infrastructure and the expertise to use it. This is not a hypothetical difficulty; it is a documented reason why teams regret adopting microservices prematurely.

**Transactions are reliable.** When your code runs in a single process connected to a single database, you can wrap multiple operations in a database transaction and be certain that either all of them succeed or none of them do. In a distributed system, maintaining consistency across services is one of the hardest problems in distributed computing. The saga pattern, two-phase commit, and eventual consistency are all approaches to this problem, and none of them are simple.

**The feedback loop is fast.** Running the application locally, making a change, and seeing the result immediately is straightforward in a monolith. In a microservices system, running the full system locally often requires Docker Compose with a dozen services, several gigabytes of container images, and enough compute to slow down a developer laptop noticeably.

None of this means that monoliths are always the right choice. It means they have real advantages that should not be dismissed because they are unfashionable.

## The scale evidence

Shopify processed $61 billion in gross merchandise volume in 2019 while running a monolithic Ruby on Rails application. Their engineering team has published extensively about the tradeoffs. When they did eventually restructure, it was toward a modular monolith architecture that they call "component-based Rails" — not microservices.

Stack Overflow serves millions of developers daily. Their architecture, documented publicly by Nick Craver, uses nine web servers and four SQL servers. The application is a monolith. The team's position, stated directly in their engineering blog, is that they chose to scale up (better hardware, better optimization) rather than scale out (more services, more complexity) because vertical scaling was cheaper, simpler, and more than adequate for their actual load.

These are not examples of teams that lacked the sophistication to build microservices. They are examples of teams that evaluated the trade-offs and made deliberate choices.

## When the monolith starts to hurt

A monolith without internal structure does develop real problems as it grows. These problems are predictable, and they are worth understanding because they are the motivation for the modular monolith.

**Deployment coupling** is the most common. When every feature is deployed together, a bug in one feature blocks the release of every other feature. Teams that deploy infrequently because of this coupling experience longer feedback cycles and more difficult debugging.

**Build time growth** is a related problem. A large codebase with no module boundaries means every change triggers a full rebuild and a full test suite run. At a certain size, this is genuinely painful.

**Cognitive load** grows as the codebase grows, if there are no clear module boundaries. Developers who need to work on the billing system should not need to understand the notification system to do it safely. Without structure, those systems become entangled over time.

**Scaling inflexibility** emerges when different parts of the application have genuinely different resource requirements. If the video transcoding logic is in the same deployable unit as the user authentication logic, you cannot scale them independently.

These problems are real. But they are the problems of a monolith without structure — not of a monolith as such. The modular monolith addresses all of them without introducing the operational overhead of a distributed system.

## What a modular monolith is

A modular monolith is a monolith with explicit, enforced internal module boundaries. It is deployed as a single unit — one process, one build — but internally it is organized into modules that have clear, limited interfaces between them. A module owns its data structures, its business logic, and its database tables. Other modules interact with it only through its public interface, never by reaching into its internals.

The key word is "enforced." Every large codebase has modules in the sense of directories. What distinguishes a modular monolith from a ball of mud is that the module boundaries are enforced by tooling, convention, or both — so that a developer working on the payments module cannot accidentally import a private function from the notifications module without someone noticing.

The concept is not new. It is essentially what good object-oriented design has always recommended — high cohesion within a module, low coupling between modules — applied at the level of the application's major domains rather than individual classes.

## How to build one

The foundation of a modular monolith is feature-based directory structure. Instead of grouping code by technical layer (all controllers in one directory, all services in another), you group by domain:

```
src/
  modules/
    users/
      users.controller.ts
      users.service.ts
      users.repository.ts
      users.types.ts
      users.routes.ts
      index.ts             <- Public interface: only this file is imported by other modules
    billing/
      billing.controller.ts
      billing.service.ts
      billing.repository.ts
      billing.types.ts
      billing.routes.ts
      index.ts
    notifications/
      ...
      index.ts
  shared/
    database.ts            <- Database client, shared by all modules
    types.ts               <- Shared types used across modules
    errors.ts              <- Shared error types
```

The rule is simple: modules import from each other only through `index.ts`. If the billing module needs to send a notification, it calls `notifications.sendEmail(...)` through the notifications module's public interface — it does not import `notifications.service.ts` directly.

This rule gives you two important properties. First, circular dependencies become visible and preventable. If billing imports notifications and notifications imports billing, that is a circular dependency that your import checker will catch. It is also a signal that something is wrong with your module boundaries. Second, you can enforce the rule with static analysis tools. ESLint's `import/no-restricted-paths` rule, or TypeScript project references, can be configured to fail the build when a module imports from another module's internals.

Shared infrastructure — database client, logger, configuration — lives in a `shared/` directory that all modules can import freely. The shared kernel should contain only cross-cutting concerns, not business logic. Business logic that is shared across modules is usually a sign that the module boundaries are wrong and need to be reconsidered.

## The difference between a modular monolith and a big ball of mud

The ball of mud is the default outcome of a growing codebase without deliberate structure. Every file can import every other file. Business logic leaks into controllers. Data access logic leaks into business logic. The billing service queries the user table directly because it is faster. The notification service has a copy of the user formatting logic because the dependency would be "circular." Over time, the codebase becomes unmaintainable not because it is large but because it has no structure.

The modular monolith differs in one critical dimension: intentionality. The module boundaries are defined, documented, and enforced. The rule "modules only communicate through public interfaces" is a rule that the team upholds. When someone violates it — and someone will — the review process catches it and the violation is reverted.

The discipline is not difficult. It requires a shared understanding of the boundaries and tooling that enforces them. The payoff is a codebase that can grow significantly without becoming impossible to navigate.

## When to choose which

This is the decision framework, stated plainly:

**New product or MVP, any team size:** Start with a monolith. You do not know your domain well enough to define stable service boundaries. You cannot afford the operational overhead of a distributed system. You need fast iteration. The monolith is correct.

**Growing product, team of 3–15 engineers, domain boundaries becoming clear:** Evolve toward a modular monolith. Introduce module structure as you understand your domain. This is not a rewrite — it is a progressive refactoring. Add the `modules/` structure to new code first, then migrate existing code incrementally.

**Large team, proven domain boundaries, independent scaling requirements:** You may be ready to extract services. Section 5 covers when that is actually justified and what the decision process looks like.

The path is sequential: monolith first, modular monolith second, selective service extraction third. Skipping steps is expensive. The teams that succeed with microservices almost always went through the monolith phase — because that phase is how you learn your domain well enough to define stable boundaries.

Martin Fowler's "MonolithFirst" argument, published at martinfowler.com, makes this case directly: "Almost all the successful microservice stories have started with a monolith that got too big and was broken up." The implication is clear. The teams that tried to start with microservices from scratch encountered serious problems — most often, they got the service boundaries wrong because they did not yet understand their domain, and repartitioning a distributed system is much harder than repartitioning code in a monolith.

---

## Sources

- Newman, S. (2021). *Building Microservices* (2nd Ed). O'Reilly Media. — Chapter 1: Microservices and SOA; monolith-first argument; when not to use microservices.
- Fowler, M. (2015). "MonolithFirst." https://martinfowler.com/bliki/MonolithFirst.html — The case for starting with a monolith.
- Richards, M. & Ford, N. (2020). *Fundamentals of Software Architecture*. O'Reilly Media. — Architecture styles chapter; modular monolith pattern.
- Shopify Engineering Blog. (2019). "Deconstructing the Monolith: Designing Software that Maximizes Developer Productivity." https://shopify.engineering/deconstructing-the-monolith-designing-software-that-maximizes-developer-productivity
- Craver, N. (2016). "Stack Overflow: The Architecture - 2016 Edition." https://nickcraver.com/blog/2016/02/17/stack-overflow-the-architecture-2016-edition/
- Newman, S. (2019). *Monolith to Microservices*. O'Reilly Media. — Migration patterns, strangler fig, module extraction.