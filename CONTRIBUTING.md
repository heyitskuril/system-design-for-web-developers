# Contributing to System Design for Web Developers

Contributions that improve the accuracy, clarity, or coverage of this guide are welcome. This document explains what kinds of contributions are accepted, what the standards are, and how to submit them.

---

## What can be contributed

| Type | Examples |
|------|---------|
| Fix a factual error | A claim is incorrect or outdated; a cited source has been updated |
| Fix a broken link | A URL no longer resolves or has moved |
| Add a missing reference | A credible source that supports or extends a claim in an existing section |
| Improve clarity | A paragraph that is unclear or confusing; a concept that needs a better explanation |
| Fix a code example | Incorrect syntax, an unrealistic example, a missing type annotation |
| Add a missing concept | A relevant pattern or concept that belongs in an existing section |
| Propose a new section | A topic that is genuinely absent and fits the guide's scope |

---

## What will not be accepted

- Unsourced opinions, regardless of how correct they seem. Every substantive claim in this guide has a citation. New claims require citations.
- Stack-specific content. This guide is intentionally technology-agnostic. Content that only applies to one framework or cloud provider does not belong here.
- Interview preparation content. This is not a FAANG interview prep guide. Content that is useful for interview scenarios but not for real product development is out of scope.
- Vendor promotion. Links to commercial products are acceptable only as implementation examples where no neutral alternative exists. Sales-oriented framing is not acceptable.
- Large structural changes without prior discussion. Open an issue first.

---

## Standards for references

A reference must meet all of the following criteria to be accepted:

- It is a published book, a formal specification, a peer-reviewed paper, or an article from a recognized engineering organization (Google Engineering, Stripe Engineering, Netflix Tech Blog, Martin Fowler's blog, OWASP, W3C, IETF, etc.).
- It is publicly accessible. Paywalled content is acceptable if the full citation is provided and the source is genuinely authoritative (a specific textbook, for example). Links must be free.
- It is relevant to the specific claim it supports. A general reference to a long book is not a citation — the specific chapter or section should be noted where possible.
- It is not primarily a product landing page or a tutorial from an unknown source.

---

## How to contribute

For small fixes (typos, broken links, minor wording improvements):

1. Fork the repository
2. Make the change
3. Open a pull request with a clear title describing what was changed

For larger contributions (new concepts, new sections, code example changes):

1. Open an issue describing what you want to add or change and why
2. Wait for a maintainer response before writing
3. Fork the repository and create a branch: `fix/broken-link-section-8`, `add/bulkhead-pattern-section-10`, `improve/caching-write-through-explanation`
4. Make the change
5. Open a pull request referencing the issue

---

## Pull request requirements

- One change per pull request
- PR title must describe the change: `fix: broken Redis docs link in Section 8` or `add: probabilistic early expiration reference to Section 8`
- Any new reference must include Title, Author, Year, and the specific section it supports
- Code examples must be syntactically correct TypeScript/Node.js
- Changes must not introduce vendor-specific or stack-specific assumptions

---

## Commit message format

```
fix: broken link to Redis docs in Section 8
add: Fowler CQRS reference to Section 7
improve: clarify circuit breaker state machine explanation
remove: outdated Node.js version reference in Section 9
```

---

Thank you for helping keep this guide accurate and useful.

— **Kuril** ([@heyitskuril](https://github.com/heyitskuril))