# 03 — The C4 Model

Architecture diagrams are almost always useful in theory and almost always a mess in practice. Walk into any engineering team and you will find diagrams on whiteboards that do not match the codebase, Confluence pages with unlabeled boxes connected by arrows nobody can explain, and half-finished Lucidchart documents from the last onboarding session. The problem is not that developers are bad at diagramming — it is that there is no shared standard for what a diagram should contain, who it is for, and what level of detail is appropriate.

The C4 model, created by Simon Brown, solves this problem by giving diagrams a specific structure: four levels of abstraction, each with a defined audience and a defined purpose. It is not a formal notation — you do not need special software or a certification. It is a vocabulary and a discipline. Once you adopt it, you can look at any C4 diagram and immediately know what you are looking at and why.

## What the C4 model is

C4 stands for Context, Containers, Components, and Code — four levels of diagram, each drilling one level deeper into the system. The insight behind the model is that different audiences need different levels of detail, and trying to show everything to everyone in a single diagram produces something that is simultaneously too detailed for a business stakeholder and not detailed enough for a developer trying to understand a component.

Simon Brown introduced the model in *Software Architecture for Developers* (2018) and maintains it at c4model.com. ThoughtWorks included it in their Technology Radar as a technique worth adopting, noting that it provides a simple, consistent approach to communicating software architecture at different levels of abstraction.

The model is intentionally notation-agnostic. You can draw C4 diagrams in Figma, in draw.io using the C4 shape library, in Mermaid inside a markdown file, or using the Structurizr DSL which was designed specifically for the model. The notation matters less than the discipline of thinking at the right level for the right audience.

## Level 1 — The Context diagram

The Context diagram is the highest level. It shows your system as a single box — no internal detail — surrounded by the people and systems that interact with it. The audience is anyone who needs to understand what the system does and how it fits into its environment: business stakeholders, new team members, and non-technical audiences.

A good Context diagram answers three questions: Who uses this system (the actors)? What external systems does this system depend on or interact with? What is the boundary of the system itself?

For a typical SaaS web application, the Context diagram might show: a web application (your system), an end user (a person), an admin user (a person), a payment provider like Stripe (an external system), an email service like SendGrid (an external system), and perhaps an analytics platform. Nothing inside your system is visible at this level — that is the point.

Keep the Context diagram simple. The mistake is including too much. If a diagram has more than 8–10 elements at the Context level, you are probably including things that belong at the Container level.

## Level 2 — The Container diagram

The Container diagram is the most useful level for most engineering conversations. It opens up your system and shows its high-level technical components — the applications, services, databases, and other deployable units that make up the system. Each container is something that runs: a web server, a single-page application, a background worker, a database.

The audience for Container diagrams is technical: developers, tech leads, and devops engineers. This is the diagram you draw when you are planning a new system, onboarding a developer, or discussing deployment architecture.

For the same SaaS application, the Container diagram would show: a React single-page application (running in the browser), a Node.js API server (running on a server), a PostgreSQL database (running on a database server), a Redis instance (for sessions and caching), and a background job worker (for email, reports, and other async tasks). Relationships between containers include the protocol — "HTTPS/JSON", "TCP/5432", "Redis protocol" — and a brief description of why the relationship exists.

The Container diagram is where most system design conversations happen. When you are deciding whether to add a service, you are making a Container-level decision. When you are discussing where to put the authentication logic, you are talking about Container-level architecture.

## Level 3 — The Component diagram

The Component diagram goes one level deeper, opening up a single container and showing its internal structure — the logical components and their relationships. This is the level at which you document the internal organization of a service: the controllers, services, repositories, and other significant pieces of code.

The audience is the development team working on that specific container. This diagram is not useful for stakeholders, and it is not useful for understanding the system as a whole — it is for understanding one container in detail.

Not every container needs a Component diagram. Focus on containers that are large or complex enough that their internal structure is not obvious. A simple CRUD API with three endpoints probably does not need a Component diagram. An API that handles complex business logic across multiple domains probably does.

Component diagrams tend to go stale fastest, because internal code structure changes more frequently than system structure. If you draw them, plan to update them. An outdated Component diagram that contradicts the code is worse than no diagram at all.

## Level 4 — The Code diagram

The Code level is optional and rarely needed. It maps to standard UML class diagrams or sequence diagrams and shows the implementation details of a specific component. In practice, you only need this level for genuinely complex algorithmic or structural problems where a diagram communicates something that code comments cannot.

For most web applications, you will never use Level 4. The exception is when you are designing a complex domain model, a non-trivial state machine, or an integration pattern where the sequence of calls and their timing is important.

## A worked example

Here is how to draw a C4 Context and Container diagram for a typical SaaS web application using text. In a real tool, these would be visual diagrams; this notation illustrates the structure.

**Level 1 — Context**

```
[Person] End User
  - Uses the application to manage their projects

[Person] Admin
  - Manages users and monitors system health

[Software System] SaaS Application
  - The system being described

[External System] Stripe
  - Handles payment processing and subscription billing

[External System] SendGrid
  - Handles transactional email delivery

[External System] Google OAuth
  - Handles user authentication via social login

Relationships:
  End User  --> SaaS Application : uses, via web browser
  Admin     --> SaaS Application : administers, via web browser
  SaaS Application --> Stripe    : processes payments via HTTPS API
  SaaS Application --> SendGrid  : sends emails via HTTPS API
  SaaS Application --> Google OAuth : authenticates users via OAuth2
```

**Level 2 — Container**

```
[Web Browser] React SPA
  - Single-page application delivering the user interface
  - Technology: React, TypeScript

[Container: API Server] Node.js REST API
  - Handles all application logic and data access
  - Technology: Node.js, Express, TypeScript

[Container: Database] PostgreSQL
  - Primary data store for all application data
  - Technology: PostgreSQL 16

[Container: Cache] Redis
  - Session storage, application-level caching, job queues
  - Technology: Redis 7

[Container: Worker] Background Job Worker
  - Processes async jobs: email, reports, webhooks
  - Technology: Node.js, BullMQ

Relationships:
  React SPA      --> Node.js REST API : HTTPS/JSON (API calls)
  Node.js REST API --> PostgreSQL     : TCP (queries via ORM)
  Node.js REST API --> Redis          : Redis protocol (cache, sessions)
  Node.js REST API --> Background Worker : via Redis job queue
  Background Worker --> SendGrid      : HTTPS API (email sending)
  Background Worker --> PostgreSQL    : TCP (job data access)
```

This is the kind of diagram that belongs in your repository's `docs/` folder. It takes 30 minutes to draw, it answers the most common questions a new developer will have about the system, and it communicates to anyone who reads it exactly what the system is made of and how it fits together.

## What not to draw

The most common mistake with C4 is over-documentation. You do not need all four levels for every system. A simple application with a single API server, one database, and a React frontend does not need Component diagrams. A solo developer project does not need a Context diagram — the context is obvious.

Draw the diagrams that answer questions you are actually being asked, or questions that will be asked by the next developer who joins the project. Do not draw diagrams for their own sake. The return on investment for a Context and Container diagram is high; the return on a Component diagram for a straightforward REST API is probably not worth the maintenance cost.

## Keeping diagrams alive

An outdated diagram is worse than no diagram. When a diagram says the system uses MongoDB and the system actually uses PostgreSQL, it actively misleads anyone who reads it. The only value of a diagram is its accuracy.

The most effective approach for keeping diagrams current is the "architecture as code" model: define your architecture in a text file that lives in your repository alongside your code, and generate diagrams from it. Changes to the architecture require changing the definition file, which is tracked in version control like any other change.

Structurizr, the tool created by Simon Brown specifically for C4, supports this via the Structurizr DSL — a domain-specific language for describing systems in text. A diagram defined in DSL can be rendered automatically, versioned with git, and reviewed in pull requests. The alternative — maintaining diagram files in Figma or draw.io — requires manual discipline that teams reliably fail to maintain over time.

For teams that want to stay in Markdown, Mermaid diagrams can be embedded directly in `.md` files and rendered natively by GitHub. They do not support the full C4 notation, but they are good enough for Container diagrams and they stay in version control with the code.

---

## Sources

- Brown, S. (2018). *Software Architecture for Developers*. Leanpub. — C4 model origin, level definitions, worked examples.
- Brown, S. C4 Model official site. https://c4model.com/ — Canonical reference for the model.
- ThoughtWorks. (2020). *Technology Radar: C4 model.* https://www.thoughtworks.com/radar/techniques/c4-model — Industry adoption note.
- Structurizr DSL Documentation. https://structurizr.com/help/dsl — Architecture-as-code implementation.
- Mermaid Documentation. https://mermaid.js.org/syntax/block.html — Code-based diagram generation supported natively by GitHub.