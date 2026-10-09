---
description: "Write Apollo Server plugins: lifecycle hook order, per-field timing with willResolveField, Server-Timing, and reading traces correctly."
---

# 02 · Server Plugins, Tracing & Performance

You can't improve what you can't see, and a GraphQL request is harder to see into than a REST
call: one HTTP request fans out into dozens of resolvers, some batched, some concurrent. Apollo
Server's **plugin API** is the hook for observability — and for the cost limits, allowlists and
logging you've already written. This lesson maps the lifecycle precisely, builds a per-field
timing plugin, and uses it on a request with a hidden slow field — including a trap in reading
the numbers.

## The plugin shape

A plugin is an object of async hooks. Server-level hooks (`serverWillStart`) run once.
`requestDidStart` runs per request and returns an object of request-level hooks;
`executionDidStart` can in turn return `willResolveField`, which runs for **every field**, and
can return a function called when that field finishes.

```js title="plugins02.mjs"
import { ApolloServer } from "@apollo/server";
import DataLoader from "dataloader";

const sleep = (ms) => new Promise((r) => setTimeout(r, ms));
const products = Array.from({ length: 20 }, (_, i) => ({ id: String(i + 1), name: `Product ${i + 1}` }));

const typeDefs = /* GraphQL */ `
  type Query { products: [Product!]! }
  type Product { id: ID! name: String! price: Int! stock: Int! reviews: Int! }
`;
const resolvers = {
  Query: { products: async () => { await sleep(5); return products; } },
  Product: {
    price: async (p, _, { loaders }) => loaders.price.load(p.id),
    stock: async (p) => { await sleep(30); return 7; },          // one slow call per product
    reviews: async (p, _, { loaders }) => loaders.reviews.load(p.id),
  },
};

// --- Plugin 1: show the lifecycle hook order
const hookLog = [];
const lifecycle = {
  async serverWillStart() { hookLog.push("serverWillStart"); },
  async requestDidStart() {
    hookLog.push("requestDidStart");
    return {
      async didResolveSource() { hookLog.push("didResolveSource"); },
      async parsingDidStart() { hookLog.push("parsingDidStart"); },
      async validationDidStart() { hookLog.push("validationDidStart"); },
      async didResolveOperation() { hookLog.push("didResolveOperation"); },
      async responseForOperation() { hookLog.push("responseForOperation"); return null; },
      async executionDidStart() { hookLog.push("executionDidStart"); },
      async didEncounterErrors() { hookLog.push("didEncounterErrors"); },
      async willSendResponse() { hookLog.push("willSendResponse"); },
    };
  },
};

// --- Plugin 2: per-field timing, aggregated by Type.field
const fieldTiming = {
  async requestDidStart() {
    const start = performance.now();
    const stats = new Map(); // "Type.field" -> { count, totalMs, maxMs }
    return {
      async executionDidStart() {
        return {
          willResolveField({ info }) {
            const t0 = performance.now();
            return () => {
              const ms = performance.now() - t0;
              const key = `${info.parentType.name}.${info.fieldName}`;
              const s = stats.get(key) ?? { count: 0, totalMs: 0, maxMs: 0 };
              s.count++; s.totalMs += ms; s.maxMs = Math.max(s.maxMs, ms);
              stats.set(key, s);
            };
          },
        };
      },
      async willSendResponse({ response }) {
        const totalMs = performance.now() - start;
        response.http.headers.set("server-timing", `total;dur=${totalMs.toFixed(1)}`);
        const rows = [...stats].sort((a, b) => b[1].totalMs - a[1].totalMs).slice(0, 4);
        console.log(`request total ${totalMs.toFixed(0)} ms; slowest fields by summed time:`);
        for (const [k, s] of rows)
          console.log(`  ${k.padEnd(16)} calls=${String(s.count).padStart(3)} sum=${s.totalMs.toFixed(0).padStart(5)} ms  max=${s.maxMs.toFixed(0).padStart(3)} ms`);
      },
    };
  },
};

const server = new ApolloServer({ typeDefs, resolvers, plugins: [lifecycle, fieldTiming] });
await server.start();

const loaders = () => ({
  price: new DataLoader(async (ids) => { await sleep(10); return ids.map((id) => Number(id) * 100); }),
  reviews: new DataLoader(async (ids) => { await sleep(10); return ids.map(() => 3); }),
});

hookLog.length = 0;
await server.executeOperation({ query: "{ products { id } }" }, { contextValue: { loaders: loaders() } });
console.log("hooks:", hookLog.join(" → "), "\n");

const r = await server.executeOperation({ query: "{ products { name price stock reviews } }" }, { contextValue: { loaders: loaders() } });
console.log("server-timing header:", r.http.headers.get("server-timing"));

hookLog.length = 0;
await server.executeOperation({ query: "{ products { nope } }" }, { contextValue: { loaders: loaders() } });
console.log("\nhooks for an invalid query:", hookLog.join(" → "));
await server.stop();
```

## The lifecycle, observed

```text
hooks: requestDidStart → didResolveSource → parsingDidStart → validationDidStart → didResolveOperation → responseForOperation → executionDidStart → willSendResponse
```

And for a query that fails validation:

```text
hooks for an invalid query: requestDidStart → didResolveSource → parsingDidStart → validationDidStart → didEncounterErrors → willSendResponse
```

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

Two observations from the invalid case: `didResolveOperation` and execution never ran, and
`didEncounterErrors` did. Code that must run for every request — closing a timer, releasing a
resource — belongs in `willSendResponse`. And as found in [Level 3 · 06](../level-3/06-caching-persisted-queries.md),
throwing from `didResolveOperation` produces a clean error response, while throwing from
`didResolveSource` does not.

## Per-field timing

The `fieldTiming` plugin wraps every field: `willResolveField` records a start time and returns
a callback that records the duration, aggregated by `Type.field`. The second query has a slow
field hidden among fast ones:

```text
request total 40 ms; slowest fields by summed time:
  Product.stock    calls= 20 sum=  637 ms  max= 32 ms
  Product.reviews  calls= 20 sum=  253 ms  max= 13 ms
  Product.price    calls= 20 sum=  251 ms  max= 13 ms
  Query.products   calls=  1 sum=    6 ms  max=  6 ms
server-timing header: total;dur=39.6
```

### The trap: summed time isn't elapsed time

The request took **40 ms**, yet `Product.stock` alone "took" 637 ms. The 20 `stock` calls ran
**concurrently**: each waited about 30 ms, all at the same time. Summing them measures how much
waiting happened, not how long the user waited. Likewise `price` and `reviews` show ~13 ms per
call, but each was one batched DataLoader call that every field waited on together.

What actually matters for latency is the **critical path**: the longest chain of sequential
waits. Here that's `Query.products` (≈6 ms) followed by the slowest product field (≈32 ms) —
about 38 ms, close to the 40 ms total. `stock` is still the culprit, but its *max* (32 ms), not
its sum, is what tells you so. Summed time is useful for a different question — total load your
resolvers put on backends — and per-call count is the N+1 detector (20 calls of anything per
request is worth a look).

### `Server-Timing`

The plugin also sets a `Server-Timing` header, which browser developer tools display in the
network panel. It's a cheap way to give front-end developers server-side numbers per request.
(Don't expose detailed timings on public APIs; they can leak information about your backend.)

## Production tracing

Field-level timing on every request is expensive — `willResolveField` runs for each of possibly
thousands of fields. In production you'd typically:

- **Sample**: trace 1% of requests, or only slow ones.
- **Use OpenTelemetry**: instrumentation packages for `graphql` create spans for parse, validate,
  execute and (optionally) resolvers, exported to Jaeger, Tempo, Honeycomb, Datadog and similar.
  That wasn't run here; it needs a collector to send spans to.
- **Use a vendor's usage reporting**: Apollo GraphOS's usage reporting plugin sends per-field
  statistics and traces to Apollo's hosted service (requires an account; not run here). It also
  powers field-usage data for safe schema changes ([lesson 05](05-schema-evolution.md)).

Whatever the tool, keep the same mental model: spans for each phase, child spans for resolvers,
and a waterfall view that shows concurrency — the view that would have shown `stock` running in
parallel.

## How It Actually Works

Apollo Server wraps every field resolver in the schema once, at startup
(`schemaInstrumentation.js` — you saw it in stack traces in Level 1). The wrapper checks whether
the current request has any `willResolveField` listeners; if not, it calls the original resolver
with almost no overhead. If it does, it calls the listeners with `{ source, args, contextValue,
info }`, calls the resolver, and when the result (or its promise) settles, calls each listener's
returned callback with `(error, result)`. That's why the timing measures until the resolver's
promise settles — including time spent waiting for a DataLoader batch.

Request-level hooks run in plugin order; all plugins' `requestDidStart` are awaited together
(`Promise.all`), and a hook that throws in `didResolveOperation` short-circuits to an error
response. The document cache means `parsingDidStart` and `validationDidStart` only fire the first
time a given query string is seen.

## Common mistakes

- **Summing resolver times** and treating the total as latency.
- **Field tracing on every production request** without sampling.
- **Doing slow work in hooks** (synchronous logging to disk in `willResolveField`) — hooks run in
  the request path.
- **Cleanup in `didEncounterErrors` or execution hooks only**, which don't run in every path.
- **Exposing internal timings** to untrusted clients.

## Exercise

1. Change the timing plugin to record, per field, the *offset from request start* at which it
   began and ended, and print a text waterfall. Find the critical path from it.
2. Add sampling: trace a request if a random number is below 0.1 or if the `x-trace` header is
   present.
3. Count resolver calls per `Type.field` and log a warning when any field resolves more than 100
   times in one request.
4. Make `stock` use a DataLoader with a single 30 ms batch and compare the summed and max times.
