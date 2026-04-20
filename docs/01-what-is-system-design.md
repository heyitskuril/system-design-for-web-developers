# 01 — What Is System Design

System design is the practice of defining the structure of a software system — how it is divided into components, how those components communicate, where data lives, and how the whole thing behaves under load, partial failure, or unexpected usage. That definition sounds abstract, so here is a more grounded version: system design is the set of decisions you make before writing code that determine whether your application holds together or falls apart once real users are using it.

Most developers encounter system design in one of two contexts: interview preparation or job descriptions that mention it as a required skill. Both are somewhat misleading. The interview version — design Twitter, design Uber, design a URL shortener for a million requests per second — teaches you to talk about distributed systems in a structured way, but it does not teach you how to make good decisions for the actual products you build. The scale is wrong, the constraints are wrong, and the scenarios are artificial.

The version that matters is quieter. It is the conversation you have when you are deciding whether to store user sessions in a cookie or in Redis. It is the question you ask when a page is slow and you need to decide whether to add a cache, add an index, or rethink the query. It is the judgment call about whether to split a growing service into two or whether that split will create more problems than it solves. These are system design decisions. They happen every week on a working engineering team, and they rarely announce themselves.

## Architecture versus system design

The terms are related but not the same, and the distinction is worth making explicit before going further. Martin Fowler describes software architecture as the decisions that are hard to change — the choices that become the load-bearing walls of the system. System design is broader: it includes architecture, but it also includes the operational and structural decisions that shape how a system works in practice, many of which can be changed with less ceremony.

Richards and Ford, in *Fundamentals of Software Architecture*, define architecture as the combination of structure (the architectural style — monolith, microservices, event-driven), characteristics (the quality attributes — scalability, reliability, availability), decisions (the rules that constrain how the system is built), and design principles (the guidelines that inform how it is built without being rules. System design encompasses all of that and adds the operational layer: where things run, how they communicate, where bottlenecks will be, and what breaks first under load.

For working developers, the practical takeaway is this: architecture decisions are the ones that require significant effort to undo. System design decisions are a superset — some are architectural, many are not. Both require deliberate thought, and both are poorly served by ad-hoc decision-making.

## Why web developers need this

There is a persistent idea that system design is a concern for senior engineers or platform teams — that it is something you graduate into after years of writing features. This is wrong, and it causes real damage to products.

Every full-stack developer makes system design decisions constantly. When you choose to store something in `localStorage` instead of a server-side session, that is a system design decision with security and scalability implications. When you write a synchronous API endpoint that triggers an email, you have made a decision about coupling that will cause problems the first time the email provider goes down. When you put everything in one table because it is simpler, you have made a data modeling decision that may require a painful migration later.

The engineers who make these decisions well are not necessarily more experienced. They are the ones who have developed a mental model for thinking about trade-offs. That mental model is what this guide is trying to give you.

## The three questions system design helps you answer

Every meaningful system design exercise is really trying to answer three questions:

**What do we build?** This is the requirements phase — defining functional requirements (what the system must do) and non-functional requirements (how well it must do it — availability, latency, throughput, consistency). Getting this wrong means building the right architecture for the wrong problem. Section 2 covers this in detail.

**How does it fit together?** This is the structural phase — defining components, their boundaries, and how they communicate. This is where you choose between a monolith and a distributed system, between synchronous and asynchronous communication, between one database and multiple. Sections 3 through 8 cover this.

**What happens when it breaks?** This is the failure planning phase — accounting for the fact that every component will fail eventually, that networks are unreliable, and that users will do unexpected things. A system that works perfectly in development but fails unpredictably in production is a poorly designed system. Sections 10 and 11 cover this.

These three questions should be answered in order. The most common mistake in system design is jumping to the second question — immediately debating monolith versus microservices, PostgreSQL versus MongoDB — without first establishing a clear picture of what the system needs to do and how it will be used.

## The trap of premature optimization for scale

The most expensive mistake in system design is building for a scale you will never reach. The industry has an incentive to overcomplicate — there are more blog posts, conference talks, and YouTube videos about distributed systems and event-driven architectures than there are about well-designed monoliths, because the former is more interesting to write about.

The data, however, tells a different story. Stack Overflow, one of the most visited websites in the world, runs on a handful of on-premises servers and a largely monolithic architecture. They have published their infrastructure in detail: in 2016, they were serving 200 million HTTP requests per day on nine web servers and four SQL servers. The architecture is deliberately simple. Their engineering team has written extensively about why they resist the pressure to distribute.

Shopify ran as a Ruby on Rails monolith for years while processing billions of dollars in transactions. When they eventually moved to a modular monolith architecture, the work was driven by team structure and deployment independence, not by the need to handle more traffic.

Sam Newman, in *Building Microservices*, is direct about this: "I've spoken to a lot of teams who have convinced themselves they need microservices at the start of a project, and the vast majority of them end up wishing they had started with something simpler." The reasons developers reach for complex architectures prematurely are usually social — it seems more serious, more scalable, more like what real companies do — not technical.

The principle to carry into the rest of this guide is this: the right architecture is the simplest one that meets your actual requirements. Complexity is a liability. Every additional component, every network hop, every distributed transaction is something that can go wrong and something that requires operational effort to maintain. You earn that complexity by having problems that require it, not by anticipating problems you might someday have.

## A mental model for the rest of this guide

Think of this guide as a sequence of decisions. The earlier decisions constrain the later ones, which is why requirements come before architecture, and architecture comes before specific patterns like caching or circuit breakers.

Start with what the system must do and what constraints it must operate within (Section 2). Decide how to represent and communicate its structure (Section 3). Choose the right architectural style for your team and product stage (Sections 4 and 5). Design the communication patterns between components (Section 6). Choose your data stores and understand their trade-offs (Section 7). Add caching where it improves performance without creating consistency problems (Section 8). Plan for growth and load (Section 9). Build in failure handling from the start (Section 10). Finally, see how all of this applies in concrete scenarios (Section 11).

The goal is not to memorize patterns. The goal is to build the judgment to know which pattern fits which problem — and the discipline to choose simplicity when simplicity is enough.

---

## Sources

- Richards, M. & Ford, N. (2020). *Fundamentals of Software Architecture*. O'Reilly Media. — Architecture definition framework, quality characteristics.
- Newman, S. (2021). *Building Microservices* (2nd Ed). O'Reilly Media. — Monolith-first argument, microservices caution.
- Fowler, M. (2003). *Patterns of Enterprise Application Architecture*. Addison-Wesley. — Architecture as hard-to-change decisions.
- Stack Overflow Blog. (2016). "Stack Overflow: The Architecture - 2016 Edition." https://nickcraver.com/blog/2016/02/17/stack-overflow-the-architecture-2016-edition/
- Fowler, M. "Software Architecture Guide." https://martinfowler.com/architecture/