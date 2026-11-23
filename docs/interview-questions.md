---
title: "GraphQL Interview Questions and Answers (178 Questions)"
description: "178 GraphQL interview questions with answers and explanations, from beginner to advanced, organised by level. Free, from the GraphQL Mastery Path."
---

# GraphQL Interview Questions and Answers

**178 questions** with the correct answer and a short explanation, organised from beginner (Level 1) to advanced (Level 4). Every question comes from a lesson in the free GraphQL Mastery Path, so you can open the lesson to go deeper.

!!! tip "Practise, don't just read"
    Each lesson also ends with an interactive quiz, and the [final quiz](quiz.md) tests all four levels. Try to answer each question below before reading the answer.

## Level 1 · Entry

### [01 · What GraphQL Is: One Endpoint, a Typed Schema, Client-Shaped Responses](level-1/01-what-graphql-is.md)

**Q1.** What is GraphQL?

**Answer:** A query language and runtime where clients ask for exactly the fields they need from a typed schema

Single endpoint, typed.

**Q2.** What problem does it address versus fixed REST endpoints?

**Answer:** Over-fetching and under-fetching: one request shapes the response

Client-driven.

**Q3.** How many endpoints does a typical GraphQL API expose?

**Answer:** One (e.g. /graphql)

POST queries.

**Q4.** Is GraphQL tied to a database?

**Answer:** No: resolvers fetch data from anywhere

Backend agnostic.

### [02 · Your First Schema and Query with graphql-js](level-1/02-first-schema.md)

**Q5.** What does this print?

```javascript
const schema = buildSchema(`
  type Author { id: ID! name: String! books: [Book!]! }
  type Book { id: ID! title: String! year: Int author: Author! }
  type Query { book(id: ID!): Book books: [Book!]! }
`);
const A = { id: '1', name: 'Ada', books: () => B.filter(b => b.author.id === '1') };
const B = [ { id: '1', title: 'Engines', year: 1843, author: A }, { id: '2', title: 'Notes', year: null, author: A } ];
const root = { book: ({ id }) => B.find(b => b.id === id), books: () => B };
show(graphqlSync({ schema, source: '{ books { title } }', rootValue: root }));
```

**Answer:** {"books":[{"title":"Engines"},{"title":"Notes"}]}

The response mirrors the query shape. Verified by running it with graphql-js.

**Q6.** What does `!` mean in a schema?

**Answer:** The field is non-null

Type safety.

**Q7.** What does `[Book!]!` mean?

**Answer:** A non-null list of non-null Books

Both list and items non-null.

**Q8.** What is the SDL?

**Answer:** Schema Definition Language used to describe types

type, input, enum.

### [03 · The Type System: Scalars, Objects, Lists, Non-Null &amp; Enums](level-1/03-type-system.md)

**Q9.** Which kinds of types exist?

**Answer:** Scalars, objects, interfaces, unions, enums and input objects

Type system.

**Q10.** What are built-in scalars?

**Answer:** Int, Float, String, Boolean and ID

Custom scalars can be added.

**Q11.** When use an enum?

**Answer:** For a fixed set of allowed values

Validated values.

**Q12.** What's an input type for?

**Answer:** Structured arguments to mutations/fields

Cannot have resolvers.

### [04 · Writing Queries: Arguments, Aliases, Fragments, Variables &amp; Directives](level-1/04-query-language.md)

**Q13.** What does this print?

```javascript
const schema = buildSchema(`
  type Author { id: ID! name: String! books: [Book!]! }
  type Book { id: ID! title: String! year: Int author: Author! }
  type Query { book(id: ID!): Book books: [Book!]! }
`);
const A = { id: '1', name: 'Ada', books: () => B.filter(b => b.author.id === '1') };
const B = [ { id: '1', title: 'Engines', year: 1843, author: A }, { id: '2', title: 'Notes', year: null, author: A } ];
const root = { book: ({ id }) => B.find(b => b.id === id), books: () => B };
show(graphqlSync({ schema, source: '{ a: book(id: "1") { title } b: book(id: "2") { title year } }', rootValue: root }));
```

**Answer:** {"a":{"title":"Engines"},"b":{"title":"Notes","year":null}}

Aliases rename result keys; nullable year is null. Verified by running it with graphql-js.

**Q14.** What does this print?

```javascript
const schema = buildSchema(`
  type Author { id: ID! name: String! books: [Book!]! }
  type Book { id: ID! title: String! year: Int author: Author! }
  type Query { book(id: ID!): Book books: [Book!]! }
`);
const A = { id: '1', name: 'Ada', books: () => B.filter(b => b.author.id === '1') };
const B = [ { id: '1', title: 'Engines', year: 1843, author: A }, { id: '2', title: 'Notes', year: null, author: A } ];
const root = { book: ({ id }) => B.find(b => b.id === id), books: () => B };
show(graphqlSync({ schema, source: 'query Q($id: ID!) { book(id: $id) { title } }', rootValue: root, variableValues: { id: '2' } }));
```

**Answer:** {"book":{"title":"Notes"}}

Variables keep queries static and safe. Verified by running it with graphql-js.

**Q15.** What are fragments for?

**Answer:** Reusing field selections

fragment on Type.

**Q16.** What does @include(if: $x) do?

**Answer:** Conditionally includes a field

Built-in directive.

### [05 · Resolvers: parent, args, context &amp; info](level-1/05-resolvers.md)

**Q17.** What does this print?

```javascript
const schema = buildSchema(`
  type Author { id: ID! name: String! books: [Book!]! }
  type Book { id: ID! title: String! year: Int author: Author! }
  type Query { book(id: ID!): Book books: [Book!]! }
`);
const A = { id: '1', name: 'Ada', books: () => B.filter(b => b.author.id === '1') };
const B = [ { id: '1', title: 'Engines', year: 1843, author: A }, { id: '2', title: 'Notes', year: null, author: A } ];
const root = { book: ({ id }) => B.find(b => b.id === id), books: () => B };
const withR = buildSchema(`type Query { greet(name: String = "world"): String }`);
show(graphqlSync({ schema: withR, source: '{ a: greet b: greet(name: "Ada") }', rootValue: { greet: ({ name }) => `hi ${name}` } }));
```

**Answer:** {"a":"hi world","b":"hi Ada"}

Default argument values apply when omitted. Verified by running it with graphql-js.

**Q18.** What arguments does a resolver receive?

**Answer:** (parent, args, context, info)

Parent is the object resolved above.

**Q19.** What does a default resolver do?

**Answer:** Reads the property with the field's name from the parent

Only write resolvers when needed.

**Q20.** Why are resolvers independent?

**Answer:** Each field resolves separately, enabling per-field logic (and N+1 risk)

Execution model.

### [06 · Mutations and Input Types](level-1/06-mutations-inputs.md)

**Q21.** What does this print?

```javascript
const schema = buildSchema(`
  type Author { id: ID! name: String! books: [Book!]! }
  type Book { id: ID! title: String! year: Int author: Author! }
  type Query { book(id: ID!): Book books: [Book!]! }
`);
const m = buildSchema(`input NewBook { title: String! year: Int } type Book { id: ID! title: String! year: Int } type Query { x: Int } type Mutation { addBook(input: NewBook!): Book! }`);
const db = [];
show(graphqlSync({ schema: m, source: 'mutation { addBook(input: { title: "T", year: 2024 }) { id title year } }', rootValue: { addBook: ({ input }) => { const b = { id: String(db.length + 1), ...input }; db.push(b); return b; } } }));
```

**Answer:** {"addBook":{"id":"1","title":"T","year":2024}}

Mutations take input objects and return the changed object. Verified by running it with graphql-js.

**Q22.** What distinguishes a mutation from a query?

**Answer:** Mutations are executed serially and are intended for writes

Top-level fields run in order.

**Q23.** Why use a single input argument?

**Answer:** Evolvable and easier to validate

Add fields without breaking.

**Q24.** What should a mutation return?

**Answer:** The affected object (or a payload with it and errors)

Let clients update caches.

### [07 · Serving GraphQL over HTTP with Apollo Server](level-1/07-apollo-server-http.md)

**Q25.** What does Apollo Server do?

**Answer:** Serves a GraphQL schema over HTTP with resolvers, plugins and tooling

Node implementation.

**Q26.** Which HTTP method carries queries by default?

**Answer:** POST with a JSON body {query, variables}

GET possible for queries.

**Q27.** What status code do GraphQL errors usually return?

**Answer:** 200 with an errors array (partial data possible)

Transport vs execution errors.

**Q28.** What does context carry?

**Answer:** Per-request values like the user and data loaders

Created each request.

### [08 · Errors, Partial Data &amp; Null Propagation](level-1/08-errors.md)

**Q29.** What does this print?

```javascript
const schema = buildSchema(`
  type Author { id: ID! name: String! books: [Book!]! }
  type Book { id: ID! title: String! year: Int author: Author! }
  type Query { book(id: ID!): Book books: [Book!]! }
`);
const A = { id: '1', name: 'Ada', books: () => B.filter(b => b.author.id === '1') };
const B = [ { id: '1', title: 'Engines', year: 1843, author: A }, { id: '2', title: 'Notes', year: null, author: A } ];
const root = { book: ({ id }) => B.find(b => b.id === id), books: () => B };
const bad = buildSchema(`type Query { a: String! b: String }`);
show(graphqlSync({ schema: bad, source: '{ a b }', rootValue: { a: () => { throw new Error('boom'); }, b: () => 'ok' } }));
```

**Answer:** {"errors":["boom"],"data":null}

A null in a non-null field propagates up and nulls the parent. Verified by running it with graphql-js.

**Q30.** What does this print?

```javascript
const schema = buildSchema(`
  type Author { id: ID! name: String! books: [Book!]! }
  type Book { id: ID! title: String! year: Int author: Author! }
  type Query { book(id: ID!): Book books: [Book!]! }
`);
const A = { id: '1', name: 'Ada', books: () => B.filter(b => b.author.id === '1') };
const B = [ { id: '1', title: 'Engines', year: 1843, author: A }, { id: '2', title: 'Notes', year: null, author: A } ];
const root = { book: ({ id }) => B.find(b => b.id === id), books: () => B };
const s2 = buildSchema(`type Query { a: String b: String }`);
show(graphqlSync({ schema: s2, source: '{ a b }', rootValue: { a: () => { throw new Error('boom'); }, b: () => 'ok' } }));
```

**Answer:** {"errors":["boom"],"data":{"a":null,"b":"ok"}}

Nullable fields isolate failures (partial data). Verified by running it with graphql-js.

**Q31.** What's a validation error?

**Answer:** The query doesn't match the schema; it's rejected before execution

Unknown field etc.

**Q32.** How should clients treat errors?

**Answer:** Check both data and errors since both can be present

Partial results.

### [09 · Introspection, SDL &amp; Tooling](level-1/09-introspection-tooling.md)

**Q33.** What does introspection return?

```javascript
const schema = buildSchema(`
  type Author { id: ID! name: String! books: [Book!]! }
  type Book { id: ID! title: String! year: Int author: Author! }
  type Query { book(id: ID!): Book books: [Book!]! }
`);
const A = { id: '1', name: 'Ada', books: () => B.filter(b => b.author.id === '1') };
const B = [ { id: '1', title: 'Engines', year: 1843, author: A }, { id: '2', title: 'Notes', year: null, author: A } ];
const root = { book: ({ id }) => B.find(b => b.id === id), books: () => B };
const r = graphqlSync({ schema, source: '{ __type(name: "Book") { kind fields { name } } }' });
console.log(r.data.__type.kind, r.data.__type.fields.map(f => f.name).join(','));
```

**Answer:** OBJECT id,title,year,author

Schemas describe themselves. Verified by running it with graphql-js.

**Q34.** Why disable introspection in production sometimes?

**Answer:** To reduce schema exposure for private APIs (security through obscurity isn't enough)

Combine with auth.

**Q35.** What tool explores schemas interactively?

**Answer:** GraphiQL/Apollo Sandbox

Docs from introspection.

**Q36.** What does `__typename` return?

**Answer:** The object type's name at runtime

Used by clients for caching.

### [10 · Project — A Library Catalog API with Tests](level-1/10-project-library-api.md)

**Q37.** What does the library API project combine?

**Answer:** Schema design, resolvers, mutations, errors and tests

Level 1 project.

**Q38.** How do you test the schema?

**Answer:** Execute queries in-process and assert on the result

graphql() function.

**Q39.** What proves a mutation works?

**Answer:** A follow-up query showing the change

Observable behavior.

**Q40.** What should errors teach clients?

**Answer:** What went wrong and what to do (codes and messages)

Design errors.

## Level 2 · Intermediate

### [01 · Schema Design: Nullability, Naming &amp; Client-Driven Shapes](level-2/01-schema-design.md)

**Q41.** What's a good schema design principle?

**Answer:** Model the domain and client needs, not your database tables

Demand-driven.

**Q42.** Why prefer nouns for types and verbs for mutations?

**Answer:** Clarity and consistency

createUser.

**Q43.** How should nullability be chosen?

**Answer:** Non-null only when you can always guarantee it

Errors propagate.

**Q44.** What does deprecating a field look like?

**Answer:** @deprecated(reason: "...") while keeping it working

Evolve without breaking.

### [02 · Interfaces and Unions](level-2/02-interfaces-unions.md)

**Q45.** What does this print?

```javascript
const s = buildSchema(`interface Animal { name: String! } type Dog implements Animal { name: String! bark: String } type Cat implements Animal { name: String! } type Query { pets: [Animal!]! }`);
const root = { pets: () => [{ __typename: 'Dog', name: 'Rex', bark: 'woof' }, { __typename: 'Cat', name: 'Tom' }] };
show(graphqlSync({ schema: s, source: '{ pets { name __typename ... on Dog { bark } } }', rootValue: root }));
```

**Answer:** {"pets":[{"name":"Rex","__typename":"Dog","bark":"woof"},{"name":"Tom","__typename":"Cat"}]}

Inline fragments select type-specific fields. Verified by running it with graphql-js.

**Q46.** When use a union instead of an interface?

**Answer:** When types share no common fields

SearchResult = A | B.

**Q47.** What must resolve abstract types?

**Answer:** __resolveType (or __typename) to pick the concrete type

Runtime type.

**Q48.** What can interfaces enforce?

**Answer:** Shared fields across implementing types

Contracts.

### [03 · Custom Scalars](level-2/03-custom-scalars.md)

**Q49.** What does a custom scalar do?

**Answer:** Defines serialization/validation for a value type like DateTime

serialize/parseValue.

**Q50.** Why validate custom scalars on input?

**Answer:** Reject malformed values early with clear errors

parseLiteral/parseValue.

**Q51.** Which format is good for DateTime?

**Answer:** ISO 8601 strings

Unambiguous.

**Q52.** What's a risk of a generic JSON scalar?

**Answer:** It bypasses the type system

Use sparingly.

### [04 · Connecting a Database: SQLite &amp; Per-Request Context](level-2/04-database-context.md)

**Q53.** What belongs in context?

**Answer:** The DB client, current user and loaders

Per request.

**Q54.** Why create DataLoaders per request?

**Answer:** Batching/caching must not leak data between users

Isolation.

**Q55.** What's the resolver layering best practice?

**Answer:** Resolvers thin, calling service/data layers

Testable.

**Q56.** Should resolvers open DB connections each call?

**Answer:** No: use a pool from context

Efficiency.

### [05 · The N+1 Problem: Seeing and Counting It](level-2/05-n-plus-one.md)

**Q57.** How many resolver calls happen (N+1)?

```javascript
const schema = buildSchema(`
  type Author { id: ID! name: String! books: [Book!]! }
  type Book { id: ID! title: String! year: Int author: Author! }
  type Query { book(id: ID!): Book books: [Book!]! }
`);
let calls = 0;
const s = buildSchema(`type Author { name: String! } type Book { title: String! author: Author! } type Query { books: [Book!]! }`);
const authors = { 1: 'Ada', 2: 'Bo' };
const root = { books: () => [1,2,1,2,1].map((a, i) => ({ title: 'b' + i, author: () => { calls++; return { name: authors[a] }; } })) };
graphqlSync({ schema: s, source: '{ books { title author { name } } }', rootValue: root });
console.log('author lookups:', calls);
```

**Answer:** author lookups: 5

One lookup per book: the N+1 pattern. Verified by running it with graphql-js.

**Q58.** What is the N+1 problem?

**Answer:** One query for a list plus one per item for related data

Batch them.

**Q59.** How detect it?

**Answer:** Query logging/tracing counting DB calls per request

Observability.

**Q60.** Is it unique to GraphQL?

**Answer:** No, but per-field resolvers make it easy to hit

Fix with batching.

### [06 · DataLoader: Batching &amp; Per-Request Caching](level-2/06-dataloader.md)

**Q61.** How many batch calls does the loader make?

```javascript
function makeLoader(batchFn) { const cache = new Map(); let queue = []; let scheduled = false; return { load(key) { if (cache.has(key)) return cache.get(key); const p = new Promise(resolve => { queue.push({ key, resolve }); if (!scheduled) { scheduled = true; Promise.resolve().then(() => { const keys = [...new Set(queue.map(q => q.key))]; const items = queue; queue = []; scheduled = false; const res = batchFn(keys); items.forEach(i => i.resolve(res[keys.indexOf(i.key)])); }); } }); cache.set(key, p); return p; } }; }
let batches = 0;
const loader = makeLoader(keys => { batches++; return keys.map(k => 'author' + k); });
(async () => { const r = await Promise.all([1,2,1,2,1].map(k => loader.load(k))); console.log(batches, r.join(','), 'unique keys: 2'); })();
```

**Answer:** 1 author1,author2,author1,author2,author1 unique keys: 2

A DataLoader batches keys within a tick and caches per key. Verified by running it with graphql-js.

**Q62.** What does DataLoader do?

**Answer:** Batches and caches loads within a request

One per request.

**Q63.** What must a batch function return?

**Answer:** Results in the same order and length as the keys

Key alignment.

**Q64.** Why not share a loader across requests?

**Answer:** Stale or leaked data between users

Per request.

### [07 · Pagination: Offsets, Cursors &amp; Connections](level-2/07-pagination.md)

**Q65.** What's cursor pagination (Relay connections)?

**Answer:** Edges with cursors and pageInfo for stable forward/backward paging

first/after.

**Q66.** Why prefer cursors over offsets?

**Answer:** Stable under inserts/deletes and efficient on large data

Keyset.

**Q67.** What does pageInfo contain?

**Answer:** hasNextPage, hasPreviousPage, startCursor, endCursor

Connection spec.

**Q68.** How limit page size?

**Answer:** Validate and cap the first/last arguments

Prevent abuse.

### [08 · Mutation Design: Payloads &amp; Errors as Data](level-2/08-mutation-design.md)

**Q69.** What makes a good mutation payload?

**Answer:** The changed object plus user-facing errors

Predictable.

**Q70.** How model validation failures?

**Answer:** Typed userErrors in the payload (vs. top-level errors for system faults)

Expected errors.

**Q71.** Why make mutations idempotent where possible?

**Answer:** Clients retry; repeating shouldn't double-apply

Idempotency keys.

**Q72.** Should mutations be fine-grained or CRUD-generic?

**Answer:** Task-based (e.g. publishPost) is clearer

Intent.

### [09 · Testing a GraphQL Server](level-2/09-testing.md)

**Q73.** How test a GraphQL API?

**Answer:** Execute operations against the schema with a test context and assert results

Unit + integration.

**Q74.** What's a snapshot test risk?

**Answer:** Brittle and hides intent; prefer explicit assertions

Be selective.

**Q75.** How mock data layers?

**Answer:** Inject fake services via context

Isolation.

**Q76.** What should tests cover?

**Answer:** Happy paths, errors, authorization and N+1 behavior

Quality.

### [10 · Project — A Bookstore API on SQLite](level-2/10-project-bookstore.md)

**Q77.** What does the bookstore project combine?

**Answer:** Interfaces, DataLoader, pagination, mutations and tests

Level 2 project.

**Q78.** How do you verify no N+1?

**Answer:** Assert DB call counts in tests

Measure.

**Q79.** What do you document?

**Answer:** Schema descriptions and deprecations

Self-documenting API.

**Q80.** How handle search?

**Answer:** Union/interface results with filters and pagination

Flexible.

## Level 3 · Advanced

### [01 · Authentication: Tokens and Context](level-3/01-authentication.md)

**Q81.** Where should authentication happen?

**Answer:** Outside resolvers (middleware) creating context.user

Identify first.

**Q82.** What does a JWT in the Authorization header provide?

**Answer:** A signed identity validated on each request

Bearer token.

**Q83.** What should unauthenticated requests get?

**Answer:** Clear authentication errors, not partial secrets

Fail safe.

**Q84.** Where should secrets and tokens not appear?

**Answer:** In logs, errors or schema responses

Hygiene.

### [02 · Authorization: Field, Object &amp; Role Rules](level-3/02-authorization.md)

**Q85.** Where enforce authorization?

**Answer:** In a shared service/data layer or directive, not only at the edge

Every path.

**Q86.** What is field-level authorization?

**Answer:** Restricting individual fields by role/ownership

Sensitive fields.

**Q87.** Why not rely on hiding fields in the UI?

**Answer:** Clients can request any field in the schema

Enforce server-side.

**Q88.** What's an IDOR in GraphQL?

**Answer:** Fetching another user's object by id without ownership checks

Check access.

### [03 · Custom Schema Directives](level-3/03-directives.md)

**Q89.** What are schema directives?

**Answer:** Annotations like @auth/@deprecated that change behavior or metadata

Custom ones are possible.

**Q90.** What does @skip(if: true) do?

**Answer:** Skips the field

Built-in.

**Q91.** When use a custom directive?

**Answer:** To apply cross-cutting logic (auth, formatting) declaratively

Keep readable.

**Q92.** What does @specifiedBy do?

**Answer:** Points to a custom scalar's specification URL

Documentation.

### [04 · Subscriptions over WebSockets](level-3/04-subscriptions.md)

**Q93.** What do subscriptions provide?

**Answer:** Server-pushed real-time updates over WebSockets/SSE

Live data.

**Q94.** What resolves a subscription field?

**Answer:** An async iterator that yields events

subscribe function.

**Q95.** What's a scaling concern?

**Answer:** Pub/sub backends (Redis/Kafka) are needed across server instances

Fan-out.

**Q96.** How secure subscriptions?

**Answer:** Authenticate on connection and authorize each subscription

Same as queries.

### [05 · Securing a Public Graph: Depth, Cost, Aliases &amp; Introspection](level-3/05-security-limits.md)

**Q97.** What attack do depth/complexity limits prevent?

**Answer:** Resource exhaustion via expensive nested queries

DoS.

**Q98.** What's query cost analysis?

**Answer:** Assigning costs to fields and rejecting queries over budget

Protects backend.

**Q99.** What are persisted queries/allowlists for?

**Answer:** Only running pre-approved operations in production

Security and caching.

**Q100.** Should you rate-limit GraphQL?

**Answer:** Yes, by cost/complexity as well as request count

One endpoint, varied cost.

### [06 · Caching &amp; Persisted Queries](level-3/06-caching-persisted-queries.md)

**Q101.** What are APQ (automatic persisted queries)?

**Answer:** Clients send a query hash to save bandwidth and enable GET caching

Optimization.

**Q102.** Why is HTTP caching harder in GraphQL?

**Answer:** POST requests to one URL with varying bodies

Use GET persisted queries/CDN.

**Q103.** What does @cacheControl express?

**Answer:** Max-age and scope hints per type/field

Response caching.

**Q104.** What's a safe cache key?

**Answer:** Operation + variables + relevant auth context

Avoid leaks.

### [07 · Clients: fetch, Apollo Client &amp; the Normalized Cache](level-3/07-apollo-client-cache.md)

**Q105.** What does Apollo Client's normalized cache store?

**Answer:** Entities by __typename and id, so updates propagate everywhere

Needs ids.

**Q106.** What does fetchPolicy cache-and-network do?

**Answer:** Returns cache immediately then refreshes from the network

Fast UI.

**Q107.** How update the cache after a mutation?

**Answer:** Return updated fields with ids or use update/refetch

Consistency.

**Q108.** What's optimistic UI?

**Answer:** Updating the UI immediately assuming success, then reconciling

Responsive.

### [08 · Type Safety with GraphQL Code Generator](level-3/08-codegen.md)

**Q109.** What does GraphQL Code Generator produce?

**Answer:** Typed operations, hooks and resolver types from the schema

Type safety.

**Q110.** Why generate types?

**Answer:** Catch schema/query mismatches at build time

Shift left.

**Q111.** When should codegen run?

**Answer:** In CI and on schema/query changes

Keep in sync.

**Q112.** What's resolver typing good for?

**Answer:** Ensuring resolvers return what the schema promises

Contract.

### [09 · GraphQL over HTTP in Depth: Status Codes, CSRF &amp; Uploads](level-3/09-graphql-over-http.md)

**Q113.** What does the GraphQL-over-HTTP spec define?

**Answer:** Request/response formats, status codes and content types (application/graphql-response+json)

Interop.

**Q114.** When return non-200?

**Answer:** Request-level failures like invalid JSON or validation errors

Spec-dependent.

**Q115.** What does GET support enable?

**Answer:** CDN/browser caching for queries

Queries only.

**Q116.** Why accept content negotiation?

**Answer:** Clients may request the newer response media type

Compatibility.

### [10 · Project — A Real-Time Task Board with Auth](level-3/10-project-task-board.md)

**Q117.** What does the task board project combine?

**Answer:** Auth, authorization, subscriptions, limits and caching

Level 3 project.

**Q118.** How test authorization?

**Answer:** Execute operations as different users and assert allowed/denied

Negative tests.

**Q119.** What makes real-time safe?

**Answer:** Authenticated connections and scoped events

Least privilege.

**Q120.** How prevent abuse?

**Answer:** Depth/cost limits, rate limits and timeouts

Defense.

## Level 4 · Master

### [01 · Inside graphql-js: Parse, Validate, Execute](level-4/01-execution-internals.md)

**Q121.** What are the execution phases?

**Answer:** Parse, validate, execute (and serialize)

Spec.

**Q122.** What happens at validation?

**Answer:** The operation is checked against the schema before running

Rules.

**Q123.** How does execution resolve fields?

**Answer:** Depth-first, parallel for queries, serial for top-level mutations

Spec behavior.

**Q124.** What does `info` contain?

**Answer:** Field name, return type, path and the operation AST

Advanced use.

### [02 · Server Plugins, Tracing &amp; Performance](level-4/02-plugins-performance.md)

**Q125.** What do server plugins/hooks enable?

**Answer:** Tracing, caching, auth and error handling around execution

Apollo/Yoga plugins.

**Q126.** What's tracing for?

**Answer:** Per-resolver timing to find slow fields

Performance.

**Q127.** What's the first performance step?

**Answer:** Measure resolver timings and DB calls

Evidence.

**Q128.** What reduces latency often?

**Answer:** Batching, caching and avoiding over-fetching in resolvers

Efficiency.

### [03 · Federation Concepts: Subgraphs, Entities &amp; @key](level-4/03-federation-concepts.md)

**Q129.** What problem does federation solve?

**Answer:** Composing multiple services' schemas into one supergraph

Distributed ownership.

**Q130.** What's an entity?

**Answer:** A type shared across subgraphs identified by @key

Reference resolution.

**Q131.** What does the router/gateway do?

**Answer:** Plans queries across subgraphs and merges results

Query planner.

**Q132.** What's a downside of federation?

**Answer:** Operational complexity and cross-service latency

Adopt when needed.

### [04 · Running a Federated Graph Locally](level-4/04-federation-gateway.md)

**Q133.** What does @key do?

**Answer:** Declares how an entity is identified across subgraphs

Fields.

**Q134.** What does __resolveReference do?

**Answer:** Loads an entity from its key representation

Subgraph resolver.

**Q135.** How do subgraphs extend a type?

**Answer:** By adding fields to the entity with the same key

Ownership.

**Q136.** What validates composition?

**Answer:** Schema composition checks in CI (rover)

Fail before deploy.

### [05 · Schema Evolution &amp; Breaking-Change Detection](level-4/05-schema-evolution.md)

**Q137.** What's a breaking schema change?

**Answer:** Removing/renaming fields, changing types or making args required

Avoid.

**Q138.** How evolve safely?

**Answer:** Add, deprecate, monitor usage, then remove

Usage-based.

**Q139.** What does schema checks in CI do?

**Answer:** Compare against production usage to flag breaking changes

Apollo Studio/Hive.

**Q140.** What's versioning in GraphQL?

**Answer:** Usually avoided via additive evolution

Evolve continuously.

### [06 · Errors, Masking &amp; Logging in Production](level-4/06-production-errors.md)

**Q141.** How should production errors be exposed?

**Answer:** Sanitized messages with codes; details go to logs

Security.

**Q142.** What's an error code convention?

**Answer:** extensions.code like UNAUTHENTICATED

Machine-readable.

**Q143.** What should be logged?

**Answer:** Operation name, variables (redacted), errors and trace ids

Debuggable.

**Q144.** How classify errors?

**Answer:** Expected (user errors) vs unexpected (alert)

Signal vs noise.

### [07 · Code-First Schemas: graphql-js Types &amp; Strawberry](level-4/07-code-first.md)

**Q145.** What's code-first?

**Answer:** Defining the schema in code (Nexus/Pothos/TypeGraphQL) and generating SDL

Type-safe.

**Q146.** What's the benefit?

**Answer:** Schema and resolver types stay in sync

Refactoring.

**Q147.** What's SDL-first's benefit?

**Answer:** Language-agnostic, readable schema as the contract

Collaboration.

**Q148.** How choose?

**Answer:** Team skills and tooling; both can work well

Pragmatic.

### [08 · Beyond Apollo Server: GraphQL Yoga &amp; graphql-js 17](level-4/08-yoga-graphql17.md)

**Q149.** What is GraphQL Yoga?

**Answer:** A fully featured, spec-compliant GraphQL server built on web standards (Fetch API)

Runs everywhere.

**Q150.** What does GraphQL v17 (graphql-js) bring?

**Answer:** Incremental delivery (@defer/@stream) and modernized APIs

Progressive.

**Q151.** What does @defer do?

**Answer:** Sends part of the response later so critical fields arrive first

Incremental delivery.

**Q152.** What do web-standard servers give?

**Answer:** Portability across Node, Deno, Cloudflare Workers etc.

Fetch API.

### [09 · Architecture Review: When GraphQL Fits and When It Doesn't](level-4/09-architecture-review.md)

**Q153.** What's in an architecture review of a GraphQL API?

**Answer:** Schema design, performance, security, observability and ownership

Holistic.

**Q154.** What risks to check?

**Answer:** Unbounded queries, N+1, over-exposed fields and weak auth

Checklist.

**Q155.** What metrics matter?

**Answer:** Latency per operation, error rates and resolver timings

SLOs.

**Q156.** When is REST better?

**Answer:** Simple resource APIs, caching-heavy public APIs or file transfer

Right tool.

### [10 · Capstone — A Federated Storefront Graph](level-4/10-capstone.md)

**Q157.** What does the capstone combine?

**Answer:** Schema design, performance, security, federation and operations

Production API.

**Q158.** How prove readiness?

**Answer:** Tests, load tests, schema checks and monitoring

Operate.

**Q159.** What documentation helps?

**Answer:** Schema descriptions, runbooks and a deprecation policy

Maintainable.

**Q160.** What sustains the API?

**Answer:** Governance, ownership and continuous evolution

Lifecycle.

## Mixed questions across all levels

**Q161.** A client needs only a book's title and its author's name. Why is GraphQL a good fit?

**Answer:** The query selects exactly those fields in one request (no over- or under-fetching)

Client-driven shape.

**Q162.** What does this print?

```javascript
const schema = buildSchema(`
  type Author { id: ID! name: String! books: [Book!]! }
  type Book { id: ID! title: String! year: Int author: Author! }
  type Query { book(id: ID!): Book books: [Book!]! }
`);
const A = { id: '1', name: 'Ada', books: () => B.filter(b => b.author.id === '1') };
const B = [ { id: '1', title: 'Engines', year: 1843, author: A }, { id: '2', title: 'Notes', year: null, author: A } ];
const root = { book: ({ id }) => B.find(b => b.id === id), books: () => B };
show(graphqlSync({ schema, source: '{ book(id: "2") { title year author { name } } }', rootValue: root }));
```

**Answer:** {"book":{"title":"Notes","year":null,"author":{"name":"Ada"}}}

The response mirrors the query; nullable year is null. Verified by running it with graphql-js.

**Q163.** What does this print?

```javascript
const schema = buildSchema(`
  type Author { id: ID! name: String! books: [Book!]! }
  type Book { id: ID! title: String! year: Int author: Author! }
  type Query { book(id: ID!): Book books: [Book!]! }
`);
const A = { id: '1', name: 'Ada', books: () => B.filter(b => b.author.id === '1') };
const B = [ { id: '1', title: 'Engines', year: 1843, author: A }, { id: '2', title: 'Notes', year: null, author: A } ];
const root = { book: ({ id }) => B.find(b => b.id === id), books: () => B };
show(graphqlSync({ schema, source: '{ books { id } }', rootValue: { books: () => [{ id: '1' }, { id: null }] } }));
```

**Answer:** {"errors":["Cannot return null for non-nullable field Book.id."],"data":null}

A null in a non-null field propagates to the nearest nullable parent (here the whole data because books is non-null). Verified by running it with graphql-js.

**Q164.** What does this print?

```javascript
const s = buildSchema(`type Query { a: String b: String }`);
show(graphqlSync({ schema: s, source: '{ a b }', rootValue: { a: () => { throw new Error('boom'); }, b: () => 'ok' } }));
```

**Answer:** {"errors":["boom"],"data":{"a":null,"b":"ok"}}

Nullable fields isolate failures: partial data plus errors. Verified by running it with graphql-js.

**Q165.** What does validation say about an unknown field?

```javascript
const schema = buildSchema(`
  type Author { id: ID! name: String! books: [Book!]! }
  type Book { id: ID! title: String! year: Int author: Author! }
  type Query { book(id: ID!): Book books: [Book!]! }
`);
const A = { id: '1', name: 'Ada', books: () => B.filter(b => b.author.id === '1') };
const B = [ { id: '1', title: 'Engines', year: 1843, author: A }, { id: '2', title: 'Notes', year: null, author: A } ];
const root = { book: ({ id }) => B.find(b => b.id === id), books: () => B };
const r = graphqlSync({ schema, source: '{ books { titel } }', rootValue: root });
console.log(r.errors[0].message);
```

**Answer:** Cannot query field "titel" on type "Book". Did you mean "title"?

Validation rejects the operation before execution. Verified by running it with graphql-js.

**Q166.** What does this print?

```javascript
const s = buildSchema(`input NewBook { title: String! } type Query { x: Int } type Mutation { addBook(input: NewBook!): String! }`);
show(graphqlSync({ schema: s, source: 'mutation { addBook(input: { title: 5 }) }', rootValue: {} }));
```

**Answer:** {"errors":["String cannot represent a non string value: 5"]}

Input values are validated against the schema. Verified by running it with graphql-js.

**Q167.** What does this print?

```javascript
const s = buildSchema(`type Query { greet(name: String = "world"): String }`);
show(graphqlSync({ schema: s, source: '{ a: greet b: greet(name: "Ada") }', rootValue: { greet: ({ name }) => `hi ${name}` } }));
```

**Answer:** {"a":"hi world","b":"hi Ada"}

Aliases allow the same field twice; defaults apply. Verified by running it with graphql-js.

**Q168.** What does __typename help clients do?

**Answer:** Identify the concrete type at runtime, e.g. for fragments and cache normalization

Included automatically by many clients.

**Q169.** A list of 50 books each resolves its author with one DB query. What is this and the fix?

**Answer:** The N+1 problem; batch with a per-request DataLoader

Batching collapses 50 lookups into one.

**Q170.** Why create DataLoaders per request?

**Answer:** Cached/batched data must not leak between users or go stale across requests

Put them in context.

**Q171.** Mutations return a Boolean and clients can't update their cache. What's the better design?

**Answer:** Return the changed object (with id) or a payload containing it and user errors

Normalized caches update from returned entities.

**Q172.** Authorization is checked only in the UI and a user queries another user's profile. What's the problem?

**Answer:** Missing server-side object-level authorization (IDOR); enforce it in resolvers/services

Never rely on the client.

**Q173.** A client sends a 20-level-deep nested query. What protects the server?

**Answer:** Query depth/complexity limits (and timeouts and rate limiting)

Cost control.

**Q174.** Which pagination approach is stable when items are inserted during paging?

**Answer:** Cursor-based (Relay connections)

Keyset cursors.

**Q175.** A field must be removed. What's the safe process?

**Answer:** Deprecate it, watch usage, notify clients, then remove once unused

Additive evolution.

**Q176.** Federation needs one shared `User` across two subgraphs. How is the entity identified?

**Answer:** With @key and a reference resolver in each subgraph

Entity references.

**Q177.** Why return errors with codes in extensions?

**Answer:** Clients can branch on machine-readable codes instead of parsing messages

extensions.code.

**Q178.** Subscriptions work on one server but not behind a load balancer. What's likely missing?

**Answer:** A shared pub/sub backend (Redis/Kafka) and sticky WebSocket handling

Fan-out across instances.

## Practical coding questions

### Batch lookups with a DataLoader

Implement createLoader(batchFn): keys requested in the same tick are sent to batchFn once (deduplicated, results aligned with the keys) and each key is cached. Compare the number of database calls for a list of five books with and without the loader.

Starter code:

```
// npm i graphql   (run with: node batch.js)
const { graphql, buildSchema } = require('graphql');

const AUTHORS = { 1: 'Ada', 2: 'Bo', 3: 'Cy' };
let dbCalls = 0;
const fetchAuthors = async (ids) => { dbCalls++; return ids.map((id) => ({ id, name: AUTHORS[id] })); };

// A minimal DataLoader: collect keys requested in the same tick, call the batch function once,
// cache per key, and return results aligned with the keys.
function createLoader(batchFn) {
  // load(key) returns a promise. Keys requested in the same tick must be sent to batchFn ONCE
  // (deduplicated, results aligned with the key array), and repeated keys must be cached per loader.
  return { load(key) { return Promise.resolve(undefined); } };
}

const schema = buildSchema(`
  type Author { name: String! }
  type Book { title: String! author: Author! }
  type Query { books: [Book!]! }
`);
const BOOKS = [{ title: 'a', authorId: 1 }, { title: 'b', authorId: 2 }, { title: 'c', authorId: 1 }, { title: 'd', authorId: 3 }, { title: 'e', authorId: 2 }];

async function run(useLoader) {
  dbCalls = 0;
  const loader = createLoader(fetchAuthors);                       // one loader per request
  const rootValue = {
    books: () => BOOKS.map((b) => ({
      title: b.title,
      author: () => (useLoader ? loader.load(b.authorId) : fetchAuthors([b.authorId]).then((r) => r[0])),
    })),
  };
  const res = await graphql({ schema, source: '{ books { title author { name } } }', rootValue });
  console.log(useLoader ? 'with loader:   ' : 'without loader:', 'db calls =', dbCalls, '|', res.data.books.map((b) => b.author.name).join(','));
}

(async () => { await run(false); await run(true); })();
```

Expected output:

```text
without loader: db calls = 5 | Ada,Bo,Ada,Cy,Bo
with loader:    db calls = 1 | Ada,Bo,Ada,Cy,Bo
```

??? success "Show a solution"

    ```
    // npm i graphql   (run with: node batch.js)
    const { graphql, buildSchema } = require('graphql');

    const AUTHORS = { 1: 'Ada', 2: 'Bo', 3: 'Cy' };
    let dbCalls = 0;
    const fetchAuthors = async (ids) => { dbCalls++; return ids.map((id) => ({ id, name: AUTHORS[id] })); };

    // A minimal DataLoader: collect keys requested in the same tick, call the batch function once,
    // cache per key, and return results aligned with the keys.
    function createLoader(batchFn) {
      const cache = new Map();
      let queue = [];
      return {
        load(key) {
          if (cache.has(key)) return cache.get(key);
          const p = new Promise((resolve, reject) => {
            queue.push({ key, resolve, reject });
            if (queue.length === 1) {
              Promise.resolve().then(async () => {
                const batch = queue; queue = [];
                const keys = batch.map((b) => b.key);
                try {
                  const results = await batchFn([...new Set(keys)]);
                  const byKey = new Map(results.map((r, i) => [[...new Set(keys)][i], r]));
                  batch.forEach((b) => b.resolve(byKey.get(b.key)));
                } catch (e) { batch.forEach((b) => b.reject(e)); }
              });
            }
          });
          cache.set(key, p);
          return p;
        },
      };
    }

    const schema = buildSchema(`
      type Author { name: String! }
      type Book { title: String! author: Author! }
      type Query { books: [Book!]! }
    `);
    const BOOKS = [{ title: 'a', authorId: 1 }, { title: 'b', authorId: 2 }, { title: 'c', authorId: 1 }, { title: 'd', authorId: 3 }, { title: 'e', authorId: 2 }];

    async function run(useLoader) {
      dbCalls = 0;
      const loader = createLoader(fetchAuthors);                       // one loader per request
      const rootValue = {
        books: () => BOOKS.map((b) => ({
          title: b.title,
          author: () => (useLoader ? loader.load(b.authorId) : fetchAuthors([b.authorId]).then((r) => r[0])),
        })),
      };
      const res = await graphql({ schema, source: '{ books { title author { name } } }', rootValue });
      console.log(useLoader ? 'with loader:   ' : 'without loader:', 'db calls =', dbCalls, '|', res.data.books.map((b) => b.author.name).join(','));
    }

    (async () => { await run(false); await run(true); })();
    ```

    Without batching, five books trigger five author lookups (N+1); with a loader they collapse into one batch call. A real DataLoader also handles per-key errors and max batch sizes. Verified by running the solution with graphql-js.

### A query depth limit

Write depthLimit(max) as a GraphQL validation rule that reports an error when an operation's field nesting exceeds max (ignoring introspection fields and following fragments). Check five operations.

Starter code:

```
// npm i graphql   (run with: node limit.js)
const { buildSchema, parse, validate, specifiedRules, GraphQLError, Kind } = require('graphql');

const schema = buildSchema(`
  type User { name: String! friends: [User!]! }
  type Query { me: User! }
`);

// A validation rule that rejects operations nested deeper than `max` selection levels.
function depthLimit(max) {
  // return a validation rule: for each OperationDefinition compute the field nesting depth
  // (ignore __ introspection fields; follow fragments) and context.reportError(new GraphQLError(
  // `query depth ${depth} exceeds limit ${max}`, { nodes: [node] })) when it is greater than max
  return () => ({});
}

function measure(selectionSet, context, seenFragments) {
  let best = 0;
  for (const sel of selectionSet.selections) {
    if (sel.kind === Kind.FIELD) {
      if (sel.name.value.startsWith('__')) continue;
      best = Math.max(best, 1 + (sel.selectionSet ? measure(sel.selectionSet, context, seenFragments) : 0));
    } else if (sel.kind === Kind.INLINE_FRAGMENT) {
      best = Math.max(best, measure(sel.selectionSet, context, seenFragments));
    } else if (sel.kind === Kind.FRAGMENT_SPREAD && !seenFragments.has(sel.name.value)) {
      seenFragments.add(sel.name.value);
      const frag = context.getDocument().definitions.find((d) => d.kind === Kind.FRAGMENT_DEFINITION && d.name.value === sel.name.value);
      if (frag) best = Math.max(best, measure(frag.selectionSet, context, seenFragments));
    }
  }
  return best;
}

function check(source, max) {
  const errors = validate(schema, parse(source), [...specifiedRules, depthLimit(max)]);
  return errors.length ? errors.map((e) => e.message).join('; ') : 'ok';
}

console.log(check('{ me { name } }', 3));
console.log(check('{ me { friends { friends { name } } } }', 3));
console.log(check('{ me { friends { friends { friends { name } } } } }', 3));
console.log(check('query Q { me { ...F } } fragment F on User { friends { friends { friends { name } } } }', 3));
console.log(check('{ me { nope } }', 3));
```

Expected output:

```text
ok
query depth 4 exceeds limit 3
query depth 5 exceeds limit 3
query depth 5 exceeds limit 3
Cannot query field "nope" on type "User". Did you mean "name"?
```

??? success "Show a solution"

    ```
    // npm i graphql   (run with: node limit.js)
    const { buildSchema, parse, validate, specifiedRules, GraphQLError, Kind } = require('graphql');

    const schema = buildSchema(`
      type User { name: String! friends: [User!]! }
      type Query { me: User! }
    `);

    // A validation rule that rejects operations nested deeper than `max` selection levels.
    function depthLimit(max) {
      return (context) => ({
        OperationDefinition(node) {
          const depth = measure(node.selectionSet, context, new Set());
          if (depth > max) {
            context.reportError(new GraphQLError(`query depth ${depth} exceeds limit ${max}`, { nodes: [node] }));
          }
        },
      });
    }

    function measure(selectionSet, context, seenFragments) {
      let best = 0;
      for (const sel of selectionSet.selections) {
        if (sel.kind === Kind.FIELD) {
          if (sel.name.value.startsWith('__')) continue;
          best = Math.max(best, 1 + (sel.selectionSet ? measure(sel.selectionSet, context, seenFragments) : 0));
        } else if (sel.kind === Kind.INLINE_FRAGMENT) {
          best = Math.max(best, measure(sel.selectionSet, context, seenFragments));
        } else if (sel.kind === Kind.FRAGMENT_SPREAD && !seenFragments.has(sel.name.value)) {
          seenFragments.add(sel.name.value);
          const frag = context.getDocument().definitions.find((d) => d.kind === Kind.FRAGMENT_DEFINITION && d.name.value === sel.name.value);
          if (frag) best = Math.max(best, measure(frag.selectionSet, context, seenFragments));
        }
      }
      return best;
    }

    function check(source, max) {
      const errors = validate(schema, parse(source), [...specifiedRules, depthLimit(max)]);
      return errors.length ? errors.map((e) => e.message).join('; ') : 'ok';
    }

    console.log(check('{ me { name } }', 3));
    console.log(check('{ me { friends { friends { name } } } }', 3));
    console.log(check('{ me { friends { friends { friends { name } } } } }', 3));
    console.log(check('query Q { me { ...F } } fragment F on User { friends { friends { friends { name } } } }', 3));
    console.log(check('{ me { nope } }', 3));
    ```

    Depth limits stop pathological nested queries before execution. Combine them with cost analysis, timeouts and rate limiting. Verified by running the solution with graphql-js.

---

More free practice: the [GraphQL Mastery Path](index.md) curriculum, the [Interview Prep Pack](https://learnmasterypath.com/interview-pack/) and the [free resume ATS check](https://learnmasterypath.com/resume/).
