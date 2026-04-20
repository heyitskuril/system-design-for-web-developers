<div align="center">

# System Design for Web Developers

**A practical system design reference for developers who build real products — not for FAANG interviews.**

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](./LICENSE)
[![Contributions Welcome](https://img.shields.io/badge/contributions-welcome-brightgreen.svg)](./CONTRIBUTING.md)
[![PRs Welcome](https://img.shields.io/badge/PRs-welcome-brightgreen.svg)](./CONTRIBUTING.md)
![Last Updated](https://img.shields.io/badge/last%20updated-2026-blue)

[Read the Guide](./docs/01-what-is-system-design.md) · [Report an Issue](https://github.com/heyitskuril/system-design-for-web-developers/issues) · [Contribute](./CONTRIBUTING.md)

</div>

---

## What this is — and what it is not

Most system design content online is written for one of two audiences: senior infrastructure engineers at large tech companies, or developers preparing for FAANG-style interview loops. Both are legitimate, but neither is useful if you are a working web developer trying to make good architectural decisions for a real product you are actually building.

This guide is for the third group. It covers system design from a web developer's perspective — the decisions you will actually face, the trade-offs that matter at the scale you are actually working at, and the concepts you need to understand without requiring a background in distributed systems or site reliability engineering.

It is not a textbook. It is not an interview prep guide. It does not cover Kafka, Kubernetes, or how to design a system for 10 billion requests per day. It covers the things that matter when you are building a SaaS product, an e-commerce platform, or a B2B application — and when you need to make architectural decisions that will hold up as your product grows.

Every recommendation in this guide is backed by a citation. If a claim does not have a source, it should not be here.

---

## Who this is for

| Audience | How to use this guide |
|----------|----------------------|
| **Full-stack / backend web developer** | Read end-to-end to build a mental model for architectural decisions on your current or next product |
| **Tech lead or senior developer** | Use individual sections as reference material when making or reviewing architectural decisions with your team |
| **Indie hacker / solo developer** | Focus on Sections 1–5 to understand when your current approach is appropriate and when it is not |
| **Junior developer** | This guide assumes basic web development knowledge — read it alongside building something real |

---

## What is covered

| # | Section | Summary |
|---|---------|---------|
| 01 | [What Is System Design](./docs/01-what-is-system-design.md) | What system design actually means for a working developer, and why you need it |
| 02 | [Requirements and Constraints](./docs/02-requirements-and-constraints.md) | Defining functional and non-functional requirements, estimating scale, CAP theorem, and SLOs |
| 03 | [The C4 Model](./docs/03-C4-model.md) | A practical architecture diagramming approach you can actually use and maintain |
| 04 | [Monolith vs Modular Monolith](./docs/04-monolith-vs-modular-monolith.md) | The case for starting with a monolith and how to keep it from becoming a mess |
| 05 | [When Microservices](./docs/05-when-microservices.md) | The real cost of microservices and the actual conditions that justify them |
| 06 | [API Design and Communication](./docs/06-api-design-and-communication.md) | REST vs GraphQL, sync vs async, message queues, and when each pattern applies |
| 07 | [Database Decisions](./docs/07-database-decisions.md) | Relational vs document vs key-value, read replicas, CQRS, and avoiding premature switches |
| 08 | [Caching Strategy](./docs/08-caching-strategy.md) | The five types of caching, cache invalidation, Redis patterns, and what not to cache |
| 09 | [Scalability Patterns](./docs/09-scalability-patterns.md) | Vertical vs horizontal scaling, stateless services, load balancing, and database scaling |
| 10 | [Reliability and Failure](./docs/10-reliability-and-failure.md) | Circuit breakers, timeouts, idempotency, observability, and designing for the inevitable |
| 11 | [Real-World Examples](./docs/11-real-world-examples.md) | Three concrete scenarios with architectural decisions applied end-to-end |
| 12 | [References](./docs/12-references.md) | Every source cited in this guide, organized by category |

---

## How to use this guide

Read it in order the first time. The sections build on each other — the discussion of microservices in Section 5 assumes you have read the monolith section first, and the caching section assumes you have read the database section. A first read gives you a coherent mental model.

After that, use it as a reference. When you are making a database decision, go to Section 7. When you are thinking about whether to extract a service, go to Section 5. When you need to diagram your architecture for a team discussion, Section 3 has what you need.

The real-world examples in Section 11 are worth reading more than once. They are where the abstract concepts become concrete decisions. If you are building something new, read the scenario that most resembles what you are building before you make any significant architectural choices.

---

## Companion repositories

This is the third in a series of open engineering reference repositories:

- [Product Development Playbook](https://github.com/heyitskuril/product-development-playbook) — A 17-phase guide for building tech products from idea to launch
- [PERN Stack Srchitecture Guide](https://github.com/heyitskuril/pern-stack-architecture-guide) — Clean architecture, layering, and project structure for PERN apps
- [API Design Playbook](https://github.com/heyitskuril/api-design-playbook) — A practical reference for designing production-grade REST APIs

The three repositories are designed to complement each other. The playbook covers the full product development process, this guide covers architecture and system design decisions within that process, and the PERN guide covers the implementation details for a specific stack.

---

## References and standards

This guide draws from the following primary sources, among others. Full citations are in [docs/12-references.md](./docs/12-references.md).

- *Designing Data-Intensive Applications* — Martin Kleppmann (2017)
- *Building Microservices* (2nd Ed) — Sam Newman (2021)
- *Fundamentals of Software Architecture* — Richards & Ford (2020)
- *Release It!* (2nd Ed) — Michael Nygard (2018)
- *The Design of Web APIs* — Arnaud Lauret (2019)
- [Google Site Reliability Engineering Book](https://sre.google/sre-book/) — Google (2016, free online)
- [C4 Model](https://c4model.com/) — Simon Brown
- [The 12-Factor App](https://12factor.net/) — Heroku (2011)
- [AWS Well-Architected Framework](https://aws.amazon.com/architecture/well-architected/)
- Martin Fowler's architecture catalog at [martinfowler.com](https://martinfowler.com)

---

## Contributing

Found an error? Know a better source? Want to add a missing concept?

Read [CONTRIBUTING.md](./CONTRIBUTING.md) before opening a pull request. Contributions must be general, technology-agnostic where possible, and backed by a credible source. The standards in this guide are not negotiable — an unsourced opinion does not belong here, even if it is correct.

---

## License

This project is licensed under the [MIT License](./LICENSE). You are free to use, adapt, and distribute this guide for any purpose.

---

## About the author

<div align="center">

<img src="https://avatars.githubusercontent.com/heyitskuril" width="80" style="border-radius: 50%;" />

**Kuril**

A dreamer… who still working on it to make it real.

[![LinkedIn](https://img.shields.io/badge/LinkedIn-0A66C2?style=flat-square&logo=linkedin&logoColor=white)](https://linkedin.com/in/heyitskuril)
[![Instagram](https://img.shields.io/badge/Instagram-E4405F?style=flat-square&logo=instagram&logoColor=white)](https://instagram.com/heyitskuril)
[![GitHub](https://img.shields.io/badge/GitHub-181717?style=flat-square&logo=github&logoColor=white)](https://github.com/heyitskuril)
[![Email](https://img.shields.io/badge/Email-EA4335?style=flat-square&logo=gmail&logoColor=white)](mailto:kuril.dev@gmail.com)

</div>

---

<div align="center">

If this guide helped you, consider giving it a ⭐ — it helps others find it too.

</div>
