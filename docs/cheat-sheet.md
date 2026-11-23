---
title: "GraphQL Cheat Sheet: Quick Reference, Beginner to Advanced"
description: "A free GraphQL cheat sheet: 20 quick-reference tables from every level of the GraphQL Mastery Path, each linked to its full lesson."
---

# GraphQL Cheat Sheet

Quick-reference tables from **20 lessons** of the free GraphQL Mastery Path, in course order from beginner to advanced. Each section links to the full lesson with explanations and runnable examples.

## Level 1 · Entry

### [03 · The Type System: Scalars, Objects, Lists, Non-Null &amp; Enums](level-1/03-type-system.md)

*The kinds of type*

| Kind | SDL keyword | What it is |
|------|-------------|------------|
| Scalar | `scalar` (5 built in) | A leaf value: `Int`, `Float`, `String`, `Boolean`, `ID` |
| Object | `type` | A named set of fields, each with its own type |
| Enum | `enum` | A leaf with a fixed set of allowed names |
| Input object | `input` | A set of fields used only as argument values ([lesson 06](level-1/06-mutations-inputs.md)) |
| Interface | `interface` | A set of fields that several object types share ([Level 2 · 02](level-2/02-interfaces-unions.md)) |
| Union | `union` | "One of these object types" ([Level 2 · 02](level-2/02-interfaces-unions.md)) |

### [05 · Resolvers: parent, args, context &amp; info](level-1/05-resolvers.md)

*The signature*

| Argument | What it is | Typical use |
|---|---|---|
| `parent` | The value the *parent* field's resolver returned. For root fields, the server's `rootValue` (usually `undefined`). | Read the current object: `book.title`, `book.author_id` |
| `args` | The field's arguments, already coerced and with defaults applied | Filters, ids, page sizes |
| `context` | One object shared by **every** resolver in a single request | Database handles, the current user, DataLoaders |
| `info` | Metadata about this field in this query: name, return type, path, the AST of the selection, the schema | Debugging, logging, look-ahead optimisations |

### [07 · Serving GraphQL over HTTP with Apollo Server](level-1/07-apollo-server-http.md)

*Status codes*

| Request | Status | `extensions.code` |
|---|---|---|
| valid query (even if a resolver later errors) | 200 | — / your code |
| syntax error `{ books { ` | 400 | `GRAPHQL_PARSE_FAILED` |
| unknown field `{ books { isbn } }` | 400 | `GRAPHQL_VALIDATION_FAILED` |
| empty JSON body `{}` | 400 | `BAD_REQUEST` ("POST body missing, invalid Content-Type, or JSON object has no keys.") |
| CSRF-unsafe request | 400 | `BAD_REQUEST` |
| mutation over GET | 405 | `BAD_REQUEST` |

## Level 2 · Intermediate

### [02 · Interfaces and Unions](level-2/02-interfaces-unions.md)

*Interface or union?*

| Choose an interface when… | Choose a union when… |
|---|---|
| the types share meaningful fields clients query the same way (`title`, `author`) | the types have little in common (a search over books, people and places) |
| you want clients to handle unknown future implementations generically | the set is closed and each member is rendered differently |
| you need `node(id:)`-style polymorphic lookup | you're modelling a result-or-error outcome ([lesson 08](level-2/08-mutation-design.md)) |

### [03 · Custom Scalars](level-2/03-custom-scalars.md)

*Three functions, three directions*

| Function | Called when | Converts |
|---|---|---|
| `serialize(value)` | a resolver returned a value for a field of this type | internal value → JSON |
| `parseValue(value)` | a client sent this type as a **variable** | JSON → internal value |
| `parseLiteral(ast)` | a client wrote this type **inline** in the query text | AST node → internal value |

### [05 · The N+1 Problem: Seeing and Counting It](level-2/05-n-plus-one.md)

*The numbers*

| Query shape | Statements |
|---|---|
| list only | 1 |
| list → one relation | 1 + N |
| list → two relations | 1 + 2N |
| list → relation → relation | 1 + N + N |

### [06 · DataLoader: Batching &amp; Per-Request Caching](level-2/06-dataloader.md)

*The numbers, again*

| Query | Before | After |
|---|---|---|
| `books { author }`, N=500 | 501 | 2 |
| `books { author reviews }`, N=500 | 1,001 | 3 |
| `books { author { books } }`, N=500 | 1,001 | 3 |

### [08 · Mutation Design: Payloads &amp; Errors as Data](level-2/08-mutation-design.md)

*Which to choose*

| | `userErrors` payload | Result union |
|---|---|---|
| Possible outcomes visible in schema | no — by convention | yes |
| Multiple problems at once | natural | needs a list inside one outcome type |
| Null checks on success | `user` is nullable | `user` is non-null |
| Adding a new failure kind | new enum value (dangerous change) | new union member (dangerous change) |
| Client code | check list length | switch on `__typename` |

### [09 · Testing a GraphQL Server](level-2/09-testing.md)

*What to test where*

| Concern | Best layer |
|---|---|
| Business rules, formatting, calculations | unit |
| Loader ordering/batching contracts | unit |
| Field results, nullability, error codes, auth rules per field | `executeOperation` |
| Query-count budgets for important screens | `executeOperation` with a counting context |
| Status codes, headers, CSRF, CORS, body limits | HTTP |
| Breaking API changes | schema snapshot |

## Level 3 · Advanced

### [01 · Authentication: Tokens and Context](level-3/01-authentication.md)

*Tokens or cookies?*

| | `Authorization: Bearer` header | `HttpOnly` session cookie |
|---|---|---|
| Sent automatically by browsers | no — client code adds it | yes |
| Readable by page JavaScript (XSS exposure) | yes, if stored in JS-accessible storage | no |
| CSRF exposure | none (browsers don't add it to cross-site requests) | yes — needs CSRF protection |
| Works for mobile apps and server-to-server | naturally | awkwardly |

### [02 · Authorization: Field, Object &amp; Role Rules](level-3/02-authorization.md)

*Three levels*

| Level | Question | Example | Typical response when denied |
|---|---|---|---|
| Operation / field-on-root | may this viewer call this at all? | only admins list users | error (`UNAUTHENTICATED` / `FORBIDDEN`) |
| Object | may this viewer see *this* object? | private posts visible to owner and admins | filter it out, or `null` |
| Field | may this viewer see *this field* of an object they can see? | email visible to self and admins | `null` for that field |

### [03 · Custom Schema Directives](level-3/03-directives.md)

*Two families of directive*

| | Executable directives | Type system (schema) directives |
|---|---|---|
| Written in | operations (queries, fragments) | SDL |
| Locations | `QUERY`, `FIELD`, `FRAGMENT_SPREAD`, `INLINE_FRAGMENT`, … | `OBJECT`, `FIELD_DEFINITION`, `ARGUMENT_DEFINITION`, `ENUM_VALUE`, … |
| Examples | `@include(if:)`, `@skip(if:)`, `@defer` | `@deprecated`, `@specifiedBy`, `@oneOf`, federation's `@key` |
| Interpreted by | the executor, per request | whatever code reads the schema |

### [07 · Clients: fetch, Apollo Client &amp; the Normalized Cache](level-3/07-apollo-client-cache.md)

*Fetch policies*

| Policy | Behaviour |
|---|---|
| `cache-first` | cache if complete, else network (default for `query`) |
| `network-only` | always network, then write to cache (used above to check the server's value) |
| `cache-and-network` | return cache immediately, then update from network (watched queries) |
| `no-cache` | network, don't write to cache |
| `cache-only` | never network |

## Level 4 · Master

### [01 · Inside graphql-js: Parse, Validate, Execute](level-4/01-execution-internals.md)

*What each phase costs*

| Phase | Median | Scales with |
|---|---|---|
| parse | 0.04 ms | query text length |
| validate | 0.3 ms | query size × schema lookups |
| execute | 26 ms | **number of field values produced** |
| serialise | 4.6 ms | response size |

### [02 · Server Plugins, Tracing &amp; Performance](level-4/02-plugins-performance.md)

*The lifecycle, observed*

| Hook | When | Typical uses |
|---|---|---|
| `requestDidStart` | request received, context built | start timers, read headers |
| `didResolveSource` | query text known (after APQ lookup) | log query hash |
| `parsingDidStart` / `validationDidStart` | around each phase (skipped when the document cache hits) | phase timing |
| `didResolveOperation` | operation chosen, variables known | cost limits, allowlists, auth on operation name |
| `responseForOperation` | before execution; return a response to skip execution | response caches |
| `executionDidStart` | execution begins | return `willResolveField` for field tracing |
| `didEncounterErrors` | any errors occurred | error logging |
| `willSendResponse` | response ready | headers, final metrics, logging |

### [03 · Federation Concepts: Subgraphs, Entities &amp; @key](level-4/03-federation-concepts.md)

*Federation vocabulary*

| Term | Meaning |
|---|---|
| Subgraph | A GraphQL service with federation directives; owned by one team |
| Supergraph | The composed schema of all subgraphs, plus routing metadata |
| Composition | Checking subgraphs are compatible and producing the supergraph (build time) |
| Router / gateway | Receives client queries, plans them across subgraphs, merges results |
| Entity | A type with `@key`, resolvable by key from any subgraph that defines it |
| Reference | `{ __typename, ...keyFields }` pointing at an entity another subgraph will resolve |
| `@shareable` | Allows a field to be resolved by more than one subgraph |
| `@external` / `@requires` | Lets a subgraph compute a field from fields another subgraph owns |
| `@override` | Migrates a field from one subgraph to another |

### [07 · Code-First Schemas: graphql-js Types &amp; Strawberry](level-4/07-code-first.md)

*SDL-first or code-first?*

| | SDL-first | Code-first |
|---|---|---|
| Source of truth | the `.graphql` file | the code; SDL is generated |
| Schema review | read the SDL diff directly | review the *generated* SDL (commit it) |
| Resolver/schema drift | possible (caught at startup by `graphql-tools`) | impossible by construction |
| Type safety | via codegen ([Level 3 · 08](level-3/08-codegen.md)) | native to the language's types |
| Design-first workflow with clients | natural | possible, but the schema emerges from code |
| Cross-language teams | SDL is a neutral contract | each language has its own library |

### [08 · Beyond Apollo Server: GraphQL Yoga &amp; graphql-js 17](level-4/08-yoga-graphql17.md)

*Who supports 17?*

| Package | `graphql` peer range |
|---|---|
| `@apollo/server` | `^16.11.0` |
| `@apollo/subgraph` | `^16.11.0` |
| `graphql-yoga` | `^15.2.0 \|\| ^16.0.0 \|\| ^17.0.0` |
| `graphql-ws` | `^15.10.1 \|\| ^16 \|\| ^17` |
| `@graphql-tools/schema` | `^14.0.0 \|\| ^15.0.0 \|\| ^16.0.0 \|\| ^17.0.0` |

### [09 · Architecture Review: When GraphQL Fits and When It Doesn't](level-4/09-architecture-review.md)

*What it costs — with evidence from this course*

| Cost | Where you saw it |
|---|---|
| N+1 data access is the default, not the exception | 1,001 statements for one query ([Level 2 · 05](level-2/05-n-plus-one.md)) |
| Clients control server work; abuse needs cost analysis | 176,821 resolver calls from 127 characters ([Level 3 · 05](level-3/05-security-limits.md)) |
| HTTP caching needs deliberate work (GET, hints, persisted queries) | [Level 3 · 06](level-3/06-caching-persisted-queries.md) |
| Authorization must hold on every path to an object | the second-path bug ([Level 3 · 02](level-3/02-authorization.md)) |
| Status codes and errors are less uniform than REST | Apollo vs Yoga table ([Level 3 · 09](level-3/09-graphql-over-http.md)) |
| Observability needs field-level tooling; summed timings mislead | [lesson 02](level-4/02-plugins-performance.md) |
| Federation adds hops, planning and composition | query plans ([lesson 04](level-4/04-federation-gateway.md)) |

### [10 · Capstone — A Federated Storefront Graph](level-4/10-capstone.md)

*Architecture*

| Subgraph | Owns | Contributes to `Product` |
|---|---|---|
| catalog | product data | `name`, `priceCents`, `weightGrams` |
| inventory | stock and shipping | `inStock`, `shippingEstimateCents` (needs `weightGrams`) |
| reviews | reviews | `reviews`, `averageStars`; plus `myReviews`, `addReview` |

---

Go deeper with the full [GraphQL Mastery Path](index.md) curriculum or test yourself with the [interview questions](interview-questions.md).
