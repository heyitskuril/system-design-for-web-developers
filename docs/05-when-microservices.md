# 05 — When Microservices

This section is not about how to build microservices. It is about whether you should. The distinction matters because microservices architecture has accumulated a significant body of enthusiastic literature, conference talks, and vendor marketing — almost all of which assumes that the decision to adopt microservices has already been made correctly. The harder question, which gets far less attention, is whether the decision was correct in the first place.

The honest answer for most web developers building most products is: not yet, possibly never, and almost certainly not now.

## What microservices actually are

The term is used loosely, and the loose usage leads to confusion. Microservices are not simply small services, or services in general, or any architecture that involves multiple processes. The defining characteristics of a microservices architecture are specific:

Each service owns its own data. It has its own database, and no other service queries that database directly. If service A needs data that service B owns, it requests it through service B's API. This is the rule that distinguishes microservices from a distributed monolith — and it is the rule that makes microservices genuinely complex.

Each service deploys independently. Releasing a change to the payments service does not require deploying the user service. This independence is the primary operational benefit of microservices, and it only exists if services are genuinely decoupled — if changing service A's API does not require simultaneous changes to services B, C, and D.

Services communicate over the network. This is not an implementation detail — it is a fundamental architectural property with significant implications. Network calls are slower than in-process function calls by several orders of magnitude. Networks are unreliable in ways that in-process calls are not. Serialization and deserialization add overhead. These costs are real and constant.

Sam Newman's *Building Microservices* is the industry's most comprehensive treatment of this architecture. Newman defines microservices as "small, autonomous services that work together" — with the emphasis on autonomous as much as small.

## The real cost

The enthusiasm for microservices in the industry consistently underestimates the operational overhead they introduce. This is not a matter of opinion — it is a pattern that has been documented repeatedly by teams that adopted microservices and later wrote honestly about the experience.

**Distributed tracing.** When a user reports that a checkout failed, debugging it in a monolith means reading a stack trace. In a microservices system, the request may have touched the API gateway, the cart service, the inventory service, the pricing service, and the payments service before failing. Without distributed tracing infrastructure — OpenTelemetry, Jaeger, Datadog APM — you cannot see what happened. Setting this up is not trivial. Maintaining it is ongoing work.

**Distributed transactions.** Database transactions are one of the most powerful tools for maintaining data consistency. In a monolith connected to one database, wrapping multiple writes in a transaction is one line of code and a fundamental reliability guarantee. In a microservices system, if creating an order requires writing to both the orders database and the inventory database, you cannot use a transaction. You have to implement a saga — a pattern where each service performs its local transaction and publishes an event, and compensating transactions undo previous steps if a later step fails. This is substantially more complex, and getting it wrong causes data inconsistency.

**Service discovery.** Services need to know where other services are. In a monolith, you call a function. In a microservices system, service A needs to know service B's current address, which may change if service B is redeployed or scaled. Service discovery — whether via DNS, a service mesh, or a registry like Consul — adds infrastructure that needs to run reliably.

**Testing complexity.** Testing a feature that spans multiple services requires either spinning up all those services locally (complex, slow, resource-intensive) or mocking the services you do not own (fast, but mocks that diverge from reality produce false confidence). Contract testing — using tools like Pact to verify that service expectations match service implementations — is the right answer, but it requires discipline and tooling to maintain.

**Network reliability.** Every inter-service call can fail. Every inter-service call needs a timeout. Every inter-service call needs retry logic. Every retry introduces the possibility of duplicate operations. Every duplicate operation that is not idempotent introduces a bug. These concerns exist in every service boundary in your system. They compound.

None of this is unsolvable. Teams at Amazon, Netflix, and Uber have solved these problems. But those teams have hundreds or thousands of engineers. The overhead that is manageable with a dedicated platform engineering team is a serious burden for a team of five.

## Conway's Law

Melvin Conway observed in 1968 that "any organization that designs a system will produce a design whose structure is a copy of the organization's communication structure." This became known as Conway's Law, and its implications for microservices architecture are significant.

Microservices work best when the service boundaries align with team boundaries. The payments service is owned by the payments team, who deploy it, monitor it, and are responsible for it. When service boundaries do not align with team boundaries, you get distributed ownership — multiple teams sharing responsibility for a service — which produces exactly the coupling and coordination overhead that microservices were supposed to eliminate.

This means that microservices are not just a technical decision. They are an organizational decision. Before adopting microservices, you need teams that are large enough to staff independent ownership of independent services. The "two-pizza team" heuristic from Amazon — that a team should be small enough to be fed by two pizzas — is really about optimal team size for service ownership, not about service size per se. A team of four engineers can own one or two services effectively. A team of 10 engineers trying to share ownership of 15 services cannot.

For a team of five developers building a product, the organizational preconditions for microservices probably do not exist. This is not a criticism — it is a statement about scale.

## When microservices are actually justified

Genuine justification for microservices requires a combination of conditions, not just one. If you can check all of the following boxes, microservices may be the right choice:

**Different scaling requirements exist for different parts of the system.** The video transcoding pipeline needs 50x the compute of the user authentication system. Deploying them as separate services lets you scale them independently. If all parts of your system have roughly similar load characteristics, this benefit does not exist.

**Independent deployment is genuinely needed.** Different teams own different features and need to release them on independent schedules without coordinating with other teams. This is a real benefit — but it requires teams large enough and stable enough to own services independently.

**Technology heterogeneity is required.** One service genuinely needs a different language or runtime — a Python-based machine learning inference service alongside a Node.js API, for example. This is a legitimate reason to deploy separately. It is not a reason to decompose the entire system; deploy only the part that requires a different technology.

**Organizational scale supports it.** Roughly, you need enough engineers to staff independent teams for independent services, with the engineering leadership and platform tooling to support them. Below a certain team size, the overhead is not worth it.

If you have two of these four conditions but not all of them, the right answer is probably a modular monolith with good internal boundaries, not microservices.

## The specific antipatterns to avoid

**The distributed monolith** is the worst outcome: a system that looks like microservices — multiple services, network communication, separate deployments — but behaves like a tightly coupled monolith. Services are tightly coupled at the data level (service A queries service B's database directly), or tightly coupled at the deployment level (deploying service A requires deploying services B, C, and D at the same time). You get all the complexity of microservices with none of the independence benefits.

**Nanoservices** are services that have been decomposed so finely that the overhead of the service boundary — network calls, serialization, deployment, monitoring — is greater than the logic inside the service. A service that contains one function is almost certainly too small.

**Premature decomposition** is splitting a monolith into services before the domain boundaries are stable. Domain boundaries are stable when the model of the business has settled — when you know, with some confidence, that the inventory domain and the pricing domain will not need to be merged because it turns out they are really one domain. This knowledge comes from building and shipping the monolith. It rarely exists at the start of a project.

## What to do instead when you feel the pain

When a monolith starts to feel painful, the instinct is often to reach for microservices. Most of the time, there are less expensive solutions:

Better module boundaries eliminate coupling without introducing distributed systems complexity. If the monolith feels tangled, apply the modular monolith patterns from Section 4 before splitting services.

Asynchronous background jobs handle work that does not need to happen synchronously within a request, without requiring a separate service. If the user service is slow because it sends email synchronously, the solution is a job queue — not a separate email microservice.

Read replicas handle read-heavy load on the database without requiring service decomposition. If the reporting queries are degrading the performance of the transactional API, adding a read replica solves the problem at the data layer.

Caching handles the case where expensive computations or database reads are repeated frequently. This is covered in Section 8.

Extract a service only when you have exhausted these alternatives and the problem genuinely requires independent deployment or independent scaling. And when you do extract a service, extract one — the smallest possible service that solves the specific problem. Do not decompose the whole system at once.

---

## Sources

- Newman, S. (2021). *Building Microservices* (2nd Ed). O'Reilly Media. — Service ownership, data isolation, distributed transactions, antipatterns.
- Newman, S. (2019). *Monolith to Microservices*. O'Reilly Media. — Migration patterns, when extraction is justified.
- Fowler, M. (2014). "Microservices." https://martinfowler.com/articles/microservices.html — Original definitional article with Newman.
- Fowler, M. (2015). "MicroservicePremium." https://martinfowler.com/bliki/MicroservicePremium.html — The overhead cost argument.
- Conway, M. (1968). "How Do Committees Invent?" *Datamation.* — Original Conway's Law paper.
- Richards, M. & Ford, N. (2020). *Fundamentals of Software Architecture*. O'Reilly Media. — Microservices architecture style, trade-offs.
- Richardson, C. (2018). *Microservices Patterns*. Manning. — Saga pattern, distributed data management.