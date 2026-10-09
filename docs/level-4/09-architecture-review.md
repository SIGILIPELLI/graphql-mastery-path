---
description: "When GraphQL is the right architecture: what it solves and costs, REST, gRPC and tRPC compared, BFF and federation, a review checklist."
---

# 09 · Architecture Review: When GraphQL Fits and When It Doesn't

You've now built GraphQL servers, secured them, made them fast, federated them and evolved them.
The most senior skill left is judgement: knowing when *not* to do any of that. This lesson is an
architecture review in writing — what GraphQL actually buys you, what it costs, how it compares
with the alternatives, and a checklist you can apply to a real proposal. It draws on what earlier
lessons measured rather than on slogans.

## What GraphQL actually solves

Strip away the marketing and GraphQL solves three concrete problems:

1. **Many clients, many screens, one backend.** When a web app, two mobile apps and a partner
   integration each need different slices of the same data, a fixed set of REST endpoints either
   over-fetches for some or multiplies into per-screen endpoints. GraphQL lets each client ask for
   its own shape from one schema ([Level 1 · 01](../level-1/01-what-graphql-is.md)).
2. **Round trips for related data.** "Order, its lines, their products, their stock" is one request
   instead of a waterfall — valuable on mobile networks where latency, not bandwidth, dominates.
3. **A typed, introspectable contract.** The schema is documentation, validation and the source of
   generated client types at once ([Level 3 · 08](../level-3/08-codegen.md)), and schema diffs make
   breaking changes detectable ([lesson 05](05-schema-evolution.md)).

If none of these is a real problem for you, GraphQL's benefits are mostly theoretical.

## What it costs — with evidence from this course

| Cost | Where you saw it |
|---|---|
| N+1 data access is the default, not the exception | 1,001 statements for one query ([Level 2 · 05](../level-2/05-n-plus-one.md)) |
| Clients control server work; abuse needs cost analysis | 176,821 resolver calls from 127 characters ([Level 3 · 05](../level-3/05-security-limits.md)) |
| HTTP caching needs deliberate work (GET, hints, persisted queries) | [Level 3 · 06](../level-3/06-caching-persisted-queries.md) |
| Authorization must hold on every path to an object | the second-path bug ([Level 3 · 02](../level-3/02-authorization.md)) |
| Status codes and errors are less uniform than REST | Apollo vs Yoga table ([Level 3 · 09](../level-3/09-graphql-over-http.md)) |
| Observability needs field-level tooling; summed timings mislead | [lesson 02](02-plugins-performance.md) |
| Federation adds hops, planning and composition | query plans ([lesson 04](04-federation-gateway.md)) |

None of these is a reason to avoid GraphQL; all of them are work. A team adopting GraphQL should
budget for DataLoaders, cost limits, field-level auth discipline, tracing and schema governance from
the start — not discover them in production.

## Comparing the alternatives

| | GraphQL | REST (+ OpenAPI) | gRPC | tRPC |
|---|---|---|---|---|
| Client chooses response shape | yes | no (sparse fieldsets are possible but rare) | no | no |
| Contract | SDL, introspection | OpenAPI document | `.proto` files | TypeScript types |
| Cross-language clients | yes | yes | yes (codegen) | TypeScript only |
| HTTP caching | needs GET + persisted queries | natural | not applicable | limited |
| Streaming | subscriptions, `@defer`/`@stream` | SSE/WebSockets ad hoc | first-class bidirectional | subscriptions |
| Browser support | yes | yes | needs gRPC-Web/proxy | yes |
| Best fit | product APIs for varied clients | public resource APIs, simple CRUD, file transfer | service-to-service, low latency | full-stack TypeScript monorepos |

These aren't exclusive. A common, healthy architecture is **gRPC or REST between services, GraphQL at
the edge** where product clients meet the backend.

## Patterns at the edge

**Backend for Frontend (BFF).** One GraphQL server per client family (web, mobile), owned by that
client team, calling shared services. It keeps client-specific shaping out of the core services.
Cost: duplicated logic across BFFs if you're not careful.

**One shared graph.** A single schema for all clients — what this course mostly built. It shines when
clients overlap heavily; it needs schema governance as more teams contribute.

**Federated supergraph.** The shared graph split by domain ownership ([lessons 03](03-federation-concepts.md)
and [04](04-federation-gateway.md)). Worth it when many teams would otherwise queue behind one schema
owner; overkill for one or two teams.

**GraphQL as a thin layer over REST.** Resolvers call existing REST services (with DataLoader-style
batching). A pragmatic migration path — but the GraphQL layer inherits every limitation of the APIs
beneath it, including their N+1 behaviour if they have no batch endpoints.

## Where GraphQL is usually the wrong choice

- **One client, one team, simple CRUD.** The flexibility goes unused; the costs don't.
- **Public APIs whose consumers mostly want bulk export or webhooks.** REST endpoints, files and
  events serve them better.
- **Large binary transfer** (uploads, downloads, media). Use signed URLs and plain HTTP
  ([Level 3 · 09](../level-3/09-graphql-over-http.md)).
- **High-throughput internal RPC** where every microsecond counts and schemas are stable — gRPC is
  built for it.
- **Teams without capacity for the operational work** above. A well-run REST API beats a neglected
  GraphQL one.

## A review checklist

Use this on a real proposal. Each "no" is a risk to plan for, not an automatic veto.

**Fit**
- [ ] Are there several clients (or screens) with genuinely different data needs?
- [ ] Is round-trip latency a real problem for at least one client?
- [ ] Will client teams use the schema directly (types, docs, tooling)?

**Data access**
- [ ] Is there a batching strategy (DataLoader or batch endpoints) for every relationship?
- [ ] Are list fields paginated with enforced maximums?

**Security**
- [ ] Is authorization enforced in a data layer that every path goes through?
- [ ] Is there a cost limit (public) or a trusted-documents allowlist (first-party)?
- [ ] Are unexpected errors masked and logged?

**Operations**
- [ ] Are field-level metrics/tracing planned, with sampling?
- [ ] Is caching strategy defined (client cache, CDN with persisted GETs, data-layer caches)?
- [ ] Is there CI for breaking changes and a deprecation policy with usage data?

**Organisation**
- [ ] Who owns the schema? Is there a review process for changes?
- [ ] If multiple teams contribute, is federation (or modular ownership) planned before contention starts?

## How It Actually Works

Underneath the debate, the trade-off is about **where query planning happens**. In REST, the server
author plans every data access in advance: each endpoint is a fixed query, easy to cache, easy to
optimise, impossible for clients to abuse — and impossible for clients to reshape. In GraphQL, the
**client composes** the query at request time from a typed vocabulary, and the server executes it by
recursively calling resolvers ([lesson 01](01-execution-internals.md)). Every GraphQL cost in the table
above follows from that one shift: the server no longer knows the queries in advance, so batching has
to be dynamic (DataLoader), limits have to be computed (cost analysis), caching has to be keyed on
operations (persisted queries), and authorization has to work regardless of the path a query takes.
Trusted documents ([Level 3 · 06](../level-3/06-caching-persisted-queries.md)) are interesting precisely
because they move planning back to build time — keeping GraphQL's developer experience while
restoring much of REST's predictability.

## Common mistakes

- **Choosing GraphQL for its popularity** rather than for a problem from the first section.
- **Exposing the database schema as the GraphQL schema** ([Level 2 · 01](../level-2/01-schema-design.md)),
  which keeps the costs and loses the benefits.
- **Adopting federation before having multiple teams.**
- **Treating GraphQL as a replacement for service-to-service APIs** everywhere.
- **Skipping the operational budget** — DataLoaders, limits, tracing, governance — and blaming
  GraphQL when production suffers.

## Exercise

1. Apply the checklist to a system you know (or to the bookstore from Level 2) and write a one-page
   recommendation: GraphQL, REST, or both — and where each sits.
2. Sketch a migration from a REST API with 30 endpoints to GraphQL using a thin layer over the
   existing services. Which three relationships would need batch endpoints first?
3. For a mobile app on a high-latency network, estimate the number of sequential round trips for a
   screen with REST, then with one GraphQL query. What changes if the GraphQL server is federated
   and the query crosses three subgraphs in sequence?
4. Write the "decision record" for adopting trusted documents for your first-party apps: what you
   gain, what you lose, and what the client build pipeline must do.
