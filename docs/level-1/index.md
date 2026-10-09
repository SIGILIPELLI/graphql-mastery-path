---
description: "Level 1 of GraphQL: types, queries, resolvers, mutations, Apollo Server over HTTP, errors and null propagation, introspection and a project."
---

# Level 1 · Entry <span class="level-badge">Foundations</span>

Goal: understand GraphQL well enough to build a small, correct API from scratch — and to
explain exactly what happens between a query string arriving and a JSON response leaving.

Three ideas carry this level:

1. **The schema is the contract.** Types, nullability and arguments decide what clients can
   ask and what they can rely on. Everything is validated against it before any of your code
   runs.
2. **Resolvers chain through `parent`.** Every field has a resolver; each receives what its
   parent returned. You never write "the query" — you write small functions the executor
   composes.
3. **Errors are part of the response.** A GraphQL response can carry partial data and errors
   together, and `!` decides how far a failure spreads.

## Modules

1. [What GraphQL Is](01-what-graphql-is.md) — one endpoint, a typed schema, client-shaped responses; the same data via REST and GraphQL
2. [Your First Schema and Query with graphql-js](02-first-schema.md) — parse, validate and execute as separate phases; the AST; resolver maps
3. [The Type System](03-type-system.md) — scalars and their coercion rules, objects, lists, non-null, enums
4. [Writing Queries](04-query-language.md) — arguments, aliases, fragments, variables, directives, named operations
5. [Resolvers](05-resolvers.md) — `parent`, `args`, `context`, `info`; the default resolver; what a throw does
6. [Mutations and Input Types](06-mutations-inputs.md) — input objects, absent vs `null`, serial execution measured
7. [Serving GraphQL over HTTP with Apollo Server](07-apollo-server-http.md) — request shapes, context from headers, CSRF prevention, status codes, production defaults
8. [Errors, Partial Data & Null Propagation](08-errors.md) — request vs field errors, `extensions`, how `!` moves a null upward
9. [Introspection, SDL & Tooling](09-introspection-tooling.md) — `__schema`/`__type`, dumping a live schema, detecting breaking changes
10. [Project — A Library Catalog API with Tests](10-project-library-api.md) — loans with real rules, tested through `executeOperation`

## What you need before starting

- **JavaScript and Node.js basics**: modules, `async`/`await`, objects and arrays. The
  [JavaScript Mastery Path](https://sigilipelli.github.io/javascript-mastery-path/) and
  [Node.js Mastery Path](https://sigilipelli.github.io/nodejs-mastery-path/) cover them.
- **HTTP and JSON**: methods, headers, status codes. If REST is familiar, you're ready; the
  [REST API Mastery Path](https://sigilipelli.github.io/rest-api-mastery-path/) is a good companion.
- Node.js 22 or newer and `npm`.

## How the examples were checked

Every example in this level was run on **Node.js 26.3** with **graphql 16.14.2**,
**@graphql-tools/schema 10.1.3** and **Apollo Server 5.5.1**, using `node:test` for tests.
Outputs are pasted exactly as printed, except where a lesson says it trimmed stack traces or
timings. Apollo Server's error responses include stack traces with file paths in development
mode; the lessons shorten those and say so.

!!! note "What wasn't run"
    Nothing in Level 1 needs external services. Hosted tools mentioned in passing — Apollo
    Sandbox, editor plugins — are described, not demonstrated.
