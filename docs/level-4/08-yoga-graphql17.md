---
description: "What changed in graphql-js 17, verified by running it: @defer/@stream, abort signals, findSchemaChanges — plus Yoga streaming over HTTP."
---

# 08 · Beyond Apollo Server: GraphQL Yoga & graphql-js 17

This course pinned `graphql@16` because Apollo Server 5 and Apollo Federation require it. But
`graphql-js` 17 has been released (17.0.0 on 15 June 2026; 17.0.2 is used here), and much of the
ecosystem — GraphQL Yoga, `graphql-ws`, `graphql-tools` — already supports it. Rather than
summarising release notes from memory, this lesson **diffs the two versions' exports**, runs the
headline features, and then uses GraphQL Yoga to stream deferred data over HTTP.

## Who supports 17?

The `graphql` peer-dependency ranges published on npm at the time of writing:

| Package | `graphql` peer range |
|---|---|
| `@apollo/server` | `^16.11.0` |
| `@apollo/subgraph` | `^16.11.0` |
| `graphql-yoga` | `^15.2.0 \|\| ^16.0.0 \|\| ^17.0.0` |
| `graphql-ws` | `^15.10.1 \|\| ^16 \|\| ^17` |
| `@graphql-tools/schema` | `^14.0.0 \|\| ^15.0.0 \|\| ^16.0.0 \|\| ^17.0.0` |

So an Apollo Server project stays on 16 for now; a Yoga project can move. These ranges change —
check `npm view <package> peerDependencies` before upgrading. `graphql` 17 also declares
`"engines": { "node": "^22.0.0 || ^24.0.0 || ^25.0.0 || >=26.0.0" }`.

## What the exports diff says

```js
const g16 = await import("graphql@16");   // two installs side by side
const g17 = await import("graphql@17");
// compare Object.keys(...)
```

```text
REMOVED in 17: __esModule, assertValidName, default, formatError, getOperationRootType, getVisitFn, isValidNameError, module.exports, printError
ADDED in 17: AbortedGraphQLExecutionError, DeferStreamDirectiveLabelRule, DeferStreamDirectiveOnRootFieldRule, DeferStreamDirectiveOnValidOperationsRule, GraphQLDeferDirective, GraphQLStreamDirective, KnownOperationTypesRule, SafeChangeType, StreamDirectiveOnListFieldRule, assertArgument, assertEnumValue, assertField, assertInputField, coerceInputLiteral, defaultHarness, enableDevMode, executeRootSelectionSet, executeSubscriptionEvent, experimentalExecuteIncrementally, experimentalExecuteRootSelectionSet, findSchemaChanges, isArgument, isDevModeEnabled, isEnumValue, isField, isInputField, isSubscriptionOperationDefinitionNode, legacyExecuteIncrementally, legacyExecuteRootSelectionSet, mapSourceToResponseEvent, printDirective, replaceVariables, validateExecutionArgs, validateInputLiteral, validateInputValue, validateSubscriptionArgs, valueToLiteral
```

(`__esModule`, `default` and `module.exports` are artefacts of how the CommonJS build of 16 was
imported, not real API changes.) The removals are functions deprecated during 16 —
`formatError`/`printError` (use `error.toJSON()` / `error.toString()`), `getOperationRootType`
(use `schema.getRootType(operation)`), `assertValidName`. The additions cluster into a few themes,
each run below.

## Running the changes

```js title="probe.mjs"
import * as g from "graphql";
const { buildSchema, parse, execute, experimentalExecuteIncrementally, GraphQLError, isDevModeEnabled, findSchemaChanges, graphql } = g;

const sleep = (ms) => new Promise((r) => setTimeout(r, ms));
const schema = buildSchema(`
  type Query { book: Book }
  type Book { title: String! reviews: [String!]! related: [String!]! }
`);
const rootValue = {
  book: () => ({
    title: "Dune",
    reviews: async () => { await sleep(50); return ["Great", "Long"]; },
    related: async function* () { for (const t of ["Messiah", "Children"]) { await sleep(20); yield t; } },
  }),
};

console.log("dev mode enabled by default:", isDevModeEnabled());

// 1. @defer and @stream
const doc = parse(`query { book { title ... @defer(label: "rev") { reviews } related @stream(initialCount: 0) } }`);
const result = await experimentalExecuteIncrementally({ schema, document: doc, rootValue });
console.log("has initialResult:", "initialResult" in result);
console.log("initial:", JSON.stringify(result.initialResult));
for await (const part of result.subsequentResults) console.log("subsequent:", JSON.stringify(part));

// 2. plain execute refuses defer
try { await execute({ schema, document: doc, rootValue }); }
catch (e) { console.log("execute() with @defer threw:", e.constructor.name, "-", e.message); }
console.log("validate() against buildSchema schema:", g.validate(schema, doc).map((e) => e.message));
const withDefer = new g.GraphQLSchema({ ...schema.toConfig(), directives: [...g.specifiedDirectives, g.GraphQLDeferDirective, g.GraphQLStreamDirective] });
console.log("validate() with defer/stream directives added:", g.validate(withDefer, doc).map((e) => e.message));
console.log("specifiedDirectives:", g.specifiedDirectives.map((d) => d.name).join(", "));

// 3. Aborting execution
const ac = new AbortController();
setTimeout(() => ac.abort(new Error("client went away")), 10);
try {
  const r = await execute({ schema, document: parse("{ book { reviews } }"), rootValue, abortSignal: ac.signal });
  console.log("abort result:", JSON.stringify(r));
} catch (e) { console.log("abort threw:", e.constructor.name, "-", e.message); }

// 4. GraphQLError positional constructor
try { const e = new GraphQLError("x", null, null, null, ["a"]); console.log("positional GraphQLError path:", JSON.stringify(e.path)); }
catch (e) { console.log("positional GraphQLError:", e.message); }

// 5. graphql() positional args
try { console.log("graphql(schema, source):", JSON.stringify(await graphql(schema, "{ __typename }"))); }
catch (e) { console.log("graphql(schema, source) threw:", e.message); }

// 6. findSchemaChanges
const a = buildSchema("type Query { a: String }"), b = buildSchema("type Query { a: String b: Int }");
console.log("findSchemaChanges:", JSON.stringify(findSchemaChanges(a, b)));
console.log("findBreakingChanges still exported:", typeof g.findBreakingChanges, typeof g.findDangerousChanges);
```

### 1. `@defer` and `@stream` in graphql-js itself

```text
initial: {"data":{"book":{"title":"Dune","related":[]}},"pending":[{"id":"0","path":["book"],"label":"rev"},{"id":"1","path":["book","related"]}],"hasNext":true}
subsequent: {"hasNext":true,"incremental":[{"id":"1","items":["Messiah"]}]}
subsequent: {"hasNext":true,"incremental":[{"id":"1","items":["Children"]}],"completed":[{"id":"1"}]}
subsequent: {"hasNext":false,"incremental":[{"id":"0","data":{"reviews":["Great","Long"]}}],"completed":[{"id":"0"}]}
```

**Incremental delivery**: `@defer` marks a fragment the client can receive later; `@stream` sends
list items as they're produced. `experimentalExecuteIncrementally` returns an `initialResult` plus an
async iterator of `subsequentResults`. In this format, the initial payload announces `pending`
parts with ids; later payloads deliver `incremental` data by id and report `completed` ids.

Three guard rails, all observed:

```text
execute() with @defer threw: AbortedGraphQLExecutionError - Executing this GraphQL operation would unexpectedly produce multiple payloads (due to @defer or @stream directive)
validate() against buildSchema schema: [ 'Unknown directive "@defer".', 'Unknown directive "@stream".' ]
validate() with defer/stream directives added: []
specifiedDirectives: include, skip, deprecated, specifiedBy, oneOf
```

- Plain `execute()` refuses operations that would need several payloads.
- The directives are **not** in `specifiedDirectives`; a schema must include `GraphQLDeferDirective`
  and `GraphQLStreamDirective` explicitly to accept them. (`execute` doesn't validate, which is why
  the incremental call above worked against a schema that would fail validation — always validate
  first in real code.)
- `@defer`/`@stream` are still a **proposal** in the GraphQL specification process, hence the
  `experimental` prefix.

### 2. Cancelling execution

```text
abort threw: AbortedGraphQLExecutionError - client went away
```

`execute` accepts an `abortSignal`. When the client disconnects or a timeout fires, the server can
stop executing further fields instead of finishing work nobody will read — useful with the
execution timeouts suggested in [Level 3 · 05](../level-3/05-security-limits.md).

### 3. Old error-construction style removed

```text
positional GraphQLError: Cannot destructure property 'nodes' of 'options' as it is null.
```

`new GraphQLError(message, nodes, source, positions, path)` worked (deprecated) in 16 and is gone in
17; use `new GraphQLError(message, { nodes, path, extensions, originalError })`, the form used
throughout this course. (`graphql(schema, source)` with positional arguments was already removed in
16 — the probe confirmed the same error on both versions.)

### 4. `findSchemaChanges`

```text
findSchemaChanges: [{"type":"TYPE_ADDED","description":"Int was added."},{"type":"FIELD_ADDED","description":"Field Query.b was added."}]
findBreakingChanges still exported: function function
```

Alongside breaking and dangerous changes, 17 can report **safe** changes (`SafeChangeType`), so a
schema-diff tool can list *everything* that changed from one function — handy for changelogs
([lesson 05](05-schema-evolution.md)).

### 5. Dev mode is explicit

```text
dev mode enabled by default: false
```

In 16, extra development checks — notably the "multiple copies of graphql" `instanceof` check that
produces the infamous *"Cannot use GraphQLSchema from another module or realm"* error — were tied to
`NODE_ENV`. In 17, they're switched on explicitly with `enableDevMode()`. Turn it on in development
and tests to get the clearer diagnostics.

## GraphQL Yoga: streaming over HTTP

GraphQL Yoga (5.24.4) is a Fetch-API-based server from The Guild, built on the Envelop plugin system.
It runs on Node, Deno, Bun, Cloudflare Workers and other runtimes, and has built-in GraphiQL,
subscriptions over Server-Sent Events, error masking ([lesson 06](06-production-errors.md)) and file
uploads ([Level 3 · 09](../level-3/09-graphql-over-http.md)). With the defer/stream plugin it delivers
incremental results over HTTP:

```js title="yoga08.mjs"
import { createServer } from "node:http";
import { createYoga, createSchema } from "graphql-yoga";
import { useDeferStream } from "@graphql-yoga/plugin-defer-stream";
import { version } from "graphql";

const sleep = (ms) => new Promise((r) => setTimeout(r, ms));
const t0 = Date.now();
const stamp = () => `${String(Date.now() - t0).padStart(4)}ms`;

const yoga = createYoga({
  logging: false,
  plugins: [useDeferStream()],
  schema: createSchema({
    typeDefs: /* GraphQL */ `
      type Query { product(id: ID!): Product }
      type Product {
        id: ID!
        name: String!
        "Slow: computed by a recommendation service."
        recommendations: [String!]!
        "Slow: aggregated from the reviews database."
        reviewSummary: String!
      }
    `,
    resolvers: {
      Query: { product: (_, { id }) => ({ id, name: "Desk lamp" }) },
      Product: {
        reviewSummary: async () => { await sleep(300); return "4.2 stars from 87 reviews"; },
        recommendations: async function* () {
          for (const r of ["Bulb", "Shade", "Dimmer"]) { await sleep(100); yield r; }
        },
      },
    },
  }),
});

const server = createServer(yoga);
await new Promise((r) => server.listen(4330, r));
console.log(`graphql ${version}`);

const query = `{ product(id: "1") { name ... @defer { reviewSummary } recommendations @stream(initialCount: 1) } }`;
const res = await fetch("http://localhost:4330/graphql", {
  method: "POST",
  headers: { "content-type": "application/json", accept: "multipart/mixed" },
  body: JSON.stringify({ query }),
});
console.log("status", res.status, "content-type:", res.headers.get("content-type"));
const decoder = new TextDecoder();
for await (const chunk of res.body) {
  for (const part of decoder.decode(chunk).split("\r\n")) {
    if (part.startsWith("{")) console.log(stamp(), part);
  }
}
server.close();
```

```console
$ node yoga08.mjs
graphql 17.0.2
status 200 content-type: multipart/mixed; boundary="-"
 151ms {"data":{"product":{"name":"Desk lamp","recommendations":["Bulb"]}},"hasNext":true}
 247ms {"incremental":[{"items":["Shade"],"path":["product","recommendations",1]}],"hasNext":true}
 344ms {"incremental":[{"data":{"reviewSummary":"4.2 stars from 87 reviews"},"path":["product"]}],"hasNext":true}
 347ms {"incremental":[{"items":["Dimmer"],"path":["product","recommendations",2]}],"hasNext":true}
 347ms {"hasNext":false}
```

(Timestamps are from script start, so the first includes server startup.) The client got the
product name and the first recommendation immediately; each further recommendation arrived as it was
produced; the slow review summary arrived when ready — one HTTP response, delivered as
`multipart/mixed` parts. A UI can render the page shell at once and fill in slow parts.

Notice the payload format differs from the graphql-js 17 output above: Yoga executes operations
with `normalizedExecutor` from `@graphql-tools/executor` rather than graphql-js's own `execute`, and
that executor emits an earlier version of the incremental-delivery format, with `path` on each part
instead of `pending`/`id`. The format changed while the proposal
evolved, so **clients and servers must agree on a version** — check what your client library expects
before enabling `@defer` in production.

## Choosing a server

| | Apollo Server 5 | GraphQL Yoga 5 |
|---|---|---|
| graphql-js versions (now) | 16 | 15, 16, 17 |
| Runtimes | Node (integrations for frameworks/serverless) | any Fetch-API runtime |
| Error masking | opt-in (`formatError`) | default |
| Uploads, SSE subscriptions, defer/stream | not built in | built in / plugins |
| Federation | first-class (`@apollo/subgraph`, gateway) | supported via plugins/tools |
| Plugin model | Apollo plugin hooks | Envelop plugins |

Both are solid; Apollo Server is the natural choice inside Apollo's federation/GraphOS tooling,
Yoga when you want runtime flexibility, newer protocol features or graphql-js 17 today.

## How It Actually Works

In graphql-js 17, execution was reorganised around an `Executor` class (you can see
`Executor.mjs` and `ExecutorThrowingOnIncremental.mjs` in the stack traces). `execute()` uses the
"throwing" variant, which aborts as soon as a deferred fragment or stream is encountered;
`experimentalExecuteIncrementally()` uses one that collects deferred fragments and streams as
**incremental work**, executes them after (or alongside) the initial payload, and publishes completed
parts through an async iterator. The abort signal is checked between field executions; when it fires,
pending work is abandoned and an `AbortedGraphQLExecutionError` carries the partial result.

Yoga always executes through `@graphql-tools/executor`'s `normalizedExecutor`, which supports
incremental delivery on any graphql-js version. The defer/stream plugin adds the `@defer`/`@stream`
directives to the schema and registers their validation rules; Yoga's response layer then
serialises each payload as a `multipart/mixed` part (or as SSE events if the client asks for
`text/event-stream`).

## Common mistakes

- **Upgrading `graphql` past your server's peer range**, ending up with two copies of graphql-js and
  "from another module or realm" errors.
- **Enabling `@defer` without checking the client's expected format.**
- **Deferring cheap fields** — every deferred part costs a payload and client re-render; defer only
  genuinely slow, non-critical data.
- **Relying on `NODE_ENV` for dev checks** after moving to 17; call `enableDevMode()`.

## Exercise

1. Add `abortSignal` with a 100 ms `AbortSignal.timeout()` to the graphql-js probe's slow query and
   observe what partial result the error carries.
2. In `yoga08.mjs`, request `Accept: text/event-stream` instead and compare the wire format.
3. Write a script that runs `findSchemaChanges` between two schema files and prints a Markdown
   changelog grouped into breaking, dangerous and safe changes.
4. Take the Level 2 bookstore and port its server to Yoga. What changes, and what stays identical?
