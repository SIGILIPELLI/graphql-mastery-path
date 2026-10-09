---
description: "Level 4 of GraphQL: graphql-js internals, tracing, Apollo Federation, schema evolution, error masking, code-first, graphql-js 17, capstone."
---

# Level 4 · Master <span class="level-badge">Architecture</span>

Goal: understand GraphQL down to the executor and up to the organisation — how a query runs
inside `graphql-js`, how to observe and evolve a live graph, how to split one across teams with
federation, and when GraphQL is the wrong tool.

Three ideas carry this level:

1. **Measure the right thing.** Parsing is microseconds; resolvers are milliseconds; summed
   resolver time isn't latency. Look at phases, critical paths and query plans.
2. **Evolve, don't version.** Expand, deprecate, measure who still uses a field, then contract —
   with checks against real client operations.
3. **Federation is an organisational tool.** It pays off when many teams share one graph, and it
   costs hops, planning and governance.

## Modules

1. [Inside graphql-js](01-execution-internals.md) — lexer, AST, visitors, the 27 validation rules, measured phase costs, a toy executor
2. [Server Plugins, Tracing & Performance](02-plugins-performance.md) — the hook order, per-field timing, why summed time misleads
3. [Federation Concepts](03-federation-concepts.md) — subgraphs, entities, `@key`, `_service` and `_entities` queried directly
4. [Running a Federated Graph Locally](04-federation-gateway.md) — `@apollo/gateway`, query plans, subgraph traffic
5. [Schema Evolution](05-schema-evolution.md) — expand–contract, per-client field usage, operation checks before removal
6. [Errors, Masking & Logging in Production](06-production-errors.md) — what leaks by default, allow-list masking with error ids, Yoga's defaults
7. [Code-First Schemas](07-code-first.md) — graphql-js type objects and Python's Strawberry, and where nullability comes from
8. [Beyond Apollo Server: GraphQL Yoga & graphql-js 17](08-yoga-graphql17.md) — a verified diff of 16 vs 17, `@defer`/`@stream`, abort signals
9. [Architecture Review](09-architecture-review.md) — what GraphQL solves and costs, alternatives, and a review checklist
10. [Capstone — A Federated Storefront Graph](10-capstone.md) — three subgraphs, `@requires`, batched entities, identity forwarding, cost limits, tests

## What you need before starting

- **Levels 1–3 of this course.**
- Comfort reading library source code — several lessons explain behaviour by pointing at it.
- For lesson 7's Python half: Python 3.10+ (the
  [Python Mastery Path](https://sigilipelli.github.io/python-mastery-path/) covers what's needed).
- For architecture context, the [System Design Mastery Path](https://sigilipelli.github.io/system-design-mastery-path/).

## How the examples were checked

Everything was run on **Node.js 26.3** with **graphql 16.14.2** and **Apollo Server 5.5.1**, plus
**@apollo/subgraph 2.15.1**, **@apollo/gateway 2.14.4**, **DataLoader 2.2.3**,
**graphql-query-complexity 2.0.0** and **GraphQL Yoga 5.24.4**; lesson 8 used **graphql 17.0.2**
in a separate project with **@graphql-yoga/plugin-defer-stream 3.24.4**, and lesson 7 used
**Strawberry 0.332.0** on **Python 3.14.7** (graphql-core 3.3.0). Federated examples ran real
subgraph and router processes on localhost. Timings are from one laptop and show proportions, not
benchmarks. Where a library behaved differently from common examples — `buildSubgraphSchema`
rejecting the single-object form, Yoga leaking original errors when `NODE_ENV=development` — the
lesson shows what actually happened.

!!! note "What wasn't run"
    The Rust Apollo Router, `rover` composition, Apollo GraphOS (usage reporting, schema checks),
    OpenTelemetry collectors and any cloud deployment need accounts or infrastructure that weren't
    available. They're described from their documentation and marked as not run.
