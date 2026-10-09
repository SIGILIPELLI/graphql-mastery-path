---
description: "Level 2 of GraphQL: schema design, interfaces and unions, scalars, SQLite, N+1 and DataLoader, pagination, mutation payloads and testing."
---

# Level 2 · Intermediate <span class="level-badge">Real Data</span>

Goal: design a schema clients can depend on for years, back it with a real database without
drowning it in queries, and prove it all with tests that catch regressions you can't see by
eye.

Three ideas carry this level:

1. **Design from the screen backwards.** A schema shaped like your tables keeps every cost of
   GraphQL and loses its benefits. Start from what clients render.
2. **Count your queries.** GraphQL's execution model makes N+1 data access the default. A
   per-request statement counter turns an invisible problem into a failing test.
3. **Expected failures are data.** Invalid input and "already taken" belong in the schema as
   typed results; the `errors` array is for things that actually went wrong.

## Modules

1. [Schema Design](01-schema-design.md) — client-shaped vs database-shaped schemas, naming, money and time, a tested matrix of safe and breaking nullability changes
2. [Interfaces and Unions](02-interfaces-unions.md) — inline fragments, `__resolveType`, the `Node` pattern, every validation error you'll meet
3. [Custom Scalars](03-custom-scalars.md) — `serialize`, `parseValue`, `parseLiteral`; a February 30th bug; `graphql-scalars`
4. [Connecting a Database](04-database-context.md) — `node:sqlite`, per-request context, parameters, `LIKE` wildcards, constraint errors
5. [The N+1 Problem](05-n-plus-one.md) — statement counts for 10, 100 and 500 rows; why resolvers can't see it
6. [DataLoader](06-dataloader.md) — batching and caching, from 1,001 statements to 3; five pitfalls, each run
7. [Pagination](07-pagination.md) — the offset bug, keyset cursors, connections, query plans
8. [Mutation Design](08-mutation-design.md) — `userErrors` payloads vs result unions, constraint codes, evolving outcomes
9. [Testing a GraphQL Server](09-testing.md) — unit, `executeOperation`, HTTP and schema-snapshot tests; two regressions caught
10. [Project — A Bookstore API on SQLite](10-project-bookstore.md) — everything above, plus a cache-invalidation bug found by a test

## What you need before starting

- **Level 1 of this course**: resolvers, context, mutations, errors, Apollo Server basics.
- **Basic SQL**: `SELECT`, `JOIN`, `WHERE`, indexes and constraints. The
  [SQL Mastery Path](https://sigilipelli.github.io/sql-mastery-path/) covers them.
- Install: `npm install graphql @apollo/server @graphql-tools/schema dataloader graphql-scalars`.

## How the examples were checked

Everything in this level was run on **Node.js 26.3** (whose built-in `node:sqlite` bundles
**SQLite 3.53.4**) with **graphql 16.14.2**, **Apollo Server 5.5.1**, **@graphql-tools/schema
10.1.3**, **DataLoader 2.2.3** and **graphql-scalars 2.0.0**. Statement counts are exact.
Timings come from one laptop with in-memory databases; they show ratios (24 ms vs 4.7 ms for the
same query with and without batching), not benchmarks — a networked database would widen the gap.

!!! note "What wasn't run"
    No PostgreSQL or MySQL server was used. Where their behaviour differs from SQLite in a way
    that matters (async drivers, SQLSTATE codes, row-value support), the lesson says so and
    describes the documented behaviour rather than showing output.
