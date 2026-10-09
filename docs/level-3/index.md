---
description: "Level 3 of GraphQL: JWT auth, authorization on every path, directives, subscriptions, cost limits, caching, clients, codegen and HTTP."
---

# Level 3 · Advanced <span class="level-badge">Production Concerns</span>

Goal: take a working GraphQL API and make it safe to put in front of real users — authenticated,
authorized on every path, resistant to expensive queries, cacheable, real-time where it needs to
be, and pleasant to consume from typed clients.

Three ideas carry this level:

1. **Clients choose the work; the server must bound it.** Depth, cost, page sizes and — for
   first-party apps — an allowlist of trusted operations.
2. **Authorization belongs in the data layer.** A graph has many paths to the same object; only a
   rule every path goes through is safe.
3. **Conventions are features.** Stable ids, `__typename`, payloads that return what changed —
   they're what make normalized client caches, codegen and HTTP caching work.

## Modules

1. [Authentication](01-authentication.md) — issuing and verifying JWTs, context, four attacks rejected
2. [Authorization](02-authorization.md) — operation, object and field rules; hidden equals missing; the second-path bug
3. [Custom Schema Directives](03-directives.md) — `@auth`, `@lower`, `@truncate` with `mapSchema`; what introspection doesn't show
4. [Subscriptions over WebSockets](04-subscriptions.md) — `graphql-ws`, async iterators, a pub/sub, cleanup and scaling limits
5. [Securing a Public Graph](05-security-limits.md) — 176,821 resolver calls from 127 characters; depth limits, cost analysis, batching
6. [Caching & Persisted Queries](06-caching-persisted-queries.md) — `@cacheControl`, GET, APQ, trusted documents
7. [Clients](07-apollo-client-cache.md) — `fetch`, Apollo Client 4's normalized cache, cache redirects
8. [Type Safety with GraphQL Code Generator](08-codegen.md) — typed documents and resolvers; five mistakes caught
9. [GraphQL over HTTP in Depth](09-graphql-over-http.md) — media types and status codes compared across servers, uploads and CSRF
10. [Project — A Real-Time Task Board with Auth](10-project-task-board.md) — HTTP and WebSocket auth, per-event authorization, revocation

## What you need before starting

- **Levels 1 and 2 of this course**, especially context, DataLoader and mutation payloads.
- **Web security basics**: what CSRF and XSS are, how cookies and bearer tokens differ. The
  [Cybersecurity Mastery Path](https://sigilipelli.github.io/cybersecurity-mastery-path/) covers them.
- **TypeScript basics** for lesson 8 — see the
  [TypeScript Mastery Path](https://sigilipelli.github.io/typescript-mastery-path/).

## How the examples were checked

Everything was run on **Node.js 26.3** with **graphql 16.14.2**, **Apollo Server 5.5.1**,
**jose 6.2.12**, **graphql-ws 6.3.0** with **ws 8.22.0**, **@graphql-tools/utils 12.0.3**,
**graphql-query-complexity 2.0.0**, **Apollo Client 4.3.3**, **GraphQL Yoga 5.24.4**, and — for
lesson 8 — **@graphql-codegen/cli 7.4.5** with **TypeScript 7.0.2**. Servers were real processes on
localhost ports, exercised with `curl`, `fetch` and real WebSocket clients. Several first attempts
failed and are shown: a plugin hook that turned an error into a crash, generated imports that a
NodeNext project rejected, a stray process holding a port.

!!! note "What wasn't run"
    No external identity provider, Redis or message broker, CDN, or cloud object storage was used.
    Scaling subscriptions across instances, CDN caching behaviour and signed-URL uploads are
    described from documented behaviour and marked as not run.
