---
description: "Capstone: a three-subgraph federated storefront with @requires, batched entities, router identity forwarding, cost limits and tests."
---

# 10 · Capstone — A Federated Storefront Graph

The capstone brings the course together in one system: three independently runnable subgraphs
owned by imaginary teams — **catalog**, **inventory** and **reviews** — composed behind a router
that authenticates clients, forwards identity, enforces a cost limit and plans queries across
services. Each subgraph uses patterns from earlier levels (DataLoader, errors with codes,
authorization), and an end-to-end test suite proves the properties that matter: correctness
across subgraphs, batching, identity handling, limits and error propagation.

## Architecture

```text
                    ┌──────────────────────────────┐
 client ──HTTP──►   │ router (Apollo Gateway)      │  verifies token → viewer
                    │ cost limit · query planning  │  forwards x-viewer header
                    └───────┬─────────┬────────┬───┘
                            │         │        │
                  ┌─────────▼──┐ ┌────▼─────┐ ┌▼─────────┐
                  │ catalog    │ │inventory │ │ reviews  │
                  │ Product:   │ │ Product: │ │ Product: │
                  │ name,price,│ │ inStock, │ │ reviews, │
                  │ weightGrams│ │ shipping │ │ average  │
                  └────────────┘ └──────────┘ └──────────┘
```

| Subgraph | Owns | Contributes to `Product` |
|---|---|---|
| catalog | product data | `name`, `priceCents`, `weightGrams` |
| inventory | stock and shipping | `inStock`, `shippingEstimateCents` (needs `weightGrams`) |
| reviews | reviews | `reviews`, `averageStars`; plus `myReviews`, `addReview` |

## Shared subgraph helper

```js title="federation.js"
// Shared helpers for all subgraphs.
import { ApolloServer } from "@apollo/server";
import { startStandaloneServer } from "@apollo/server/standalone";
import { buildSubgraphSchema } from "@apollo/subgraph";
import { parse } from "graphql";

export const LINK = `extend schema @link(url: "https://specs.apollo.dev/federation/v2.3",
  import: ["@key", "@shareable", "@external", "@requires"])`;

// Every subgraph counts the requests and entity lookups it serves, for the tests.
export async function startSubgraph({ name, typeDefs, resolvers, port = 0, context }) {
  const stats = { requests: 0, lookups: 0, batches: 0 };
  const schema = buildSubgraphSchema([{ typeDefs: parse(`${LINK}\n${typeDefs}`), resolvers }]);
  const server = new ApolloServer({
    schema,
    plugins: [{ async requestDidStart() { stats.requests++; } }],
  });
  const { url } = await startStandaloneServer(server, {
    listen: { port },
    context: async ({ req }) => ({ ...(await context?.({ req, stats })), stats }),
  });
  return { name, url, stats, stop: () => server.stop() };
}
```

Each subgraph is a real Apollo Server on its own port, built with the array form of
`buildSubgraphSchema` (the [lesson 03](03-federation-concepts.md) gotcha). The `stats` object exists
only so the tests can observe traffic.

## The catalog subgraph

```js title="catalog.js"
import DataLoader from "dataloader";
import { startSubgraph } from "./federation.js";

const PRODUCTS = [
  { upc: "lamp-1", name: "Desk lamp", priceCents: 2499, weightGrams: 900 },
  { upc: "shelf-1", name: "Bookshelf", priceCents: 8900, weightGrams: 15000 },
  { upc: "chair-1", name: "Reading chair", priceCents: 15900, weightGrams: 11000 },
];

export const startCatalog = (port) => startSubgraph({
  name: "catalog",
  port,
  typeDefs: `
    type Query {
      products(first: Int = 10): [Product!]!
      product(upc: ID!): Product
    }
    type Product @key(fields: "upc") {
      upc: ID!
      name: String!
      priceCents: Int!
      weightGrams: Int!
    }
  `,
  context: async ({ stats }) => ({
    productByUpc: new DataLoader(async (upcs) => {
      stats.batches++; // one "database query" per batch
      return upcs.map((u) => PRODUCTS.find((p) => p.upc === u) ?? null);
    }),
  }),
  resolvers: {
    Query: {
      products: (_, { first }) => PRODUCTS.slice(0, Math.min(first, 50)),
      product: (_, { upc }, { productByUpc }) => productByUpc.load(upc),
    },
    Product: {
      // Batched: one _entities request with N representations -> one loader batch.
      __resolveReference: (ref, { productByUpc, stats }) => {
        stats.lookups++;
        return productByUpc.load(ref.upc);
      },
    },
  },
});
```

`__resolveReference` goes through a per-request DataLoader, so one `_entities` request with N
representations becomes one batch — the fix [lesson 04](04-federation-gateway.md) asked for. List
size is capped at 50.

## The inventory subgraph and `@requires`

```js title="inventory.js"
import { startSubgraph } from "./federation.js";

const STOCK = { "lamp-1": 12, "shelf-1": 0, "chair-1": 3 };

export const startInventory = (port) => startSubgraph({
  name: "inventory",
  port,
  typeDefs: `
    type Product @key(fields: "upc") {
      upc: ID!
      weightGrams: Int! @external
      inStock: Boolean!
      "Needs weightGrams from the catalog subgraph."
      shippingEstimateCents: Int! @requires(fields: "weightGrams")
    }
  `,
  resolvers: {
    Product: {
      __resolveReference: (ref) => ref, // ref carries upc (and weightGrams when @requires applies)
      inStock: (p) => (STOCK[p.upc] ?? 0) > 0,
      shippingEstimateCents: (p) => 499 + Math.ceil(p.weightGrams / 1000) * 150,
    },
  },
});
```

Shipping cost depends on weight, which the catalog owns. Instead of duplicating weights,
inventory declares `weightGrams: Int! @external` (owned elsewhere) and
`shippingEstimateCents … @requires(fields: "weightGrams")`. The router then fetches `weightGrams`
from the catalog first and includes it in the representations it sends to inventory — so
`__resolveReference` can simply return the representation, which already carries the weight.

## The reviews subgraph

```js title="reviews.js"
import { GraphQLError } from "graphql";
import { startSubgraph } from "./federation.js";

const REVIEWS = [
  { id: "r1", upc: "lamp-1", author: "ada", stars: 5, body: "Bright and sturdy." },
  { id: "r2", upc: "lamp-1", author: "bob", stars: 3, body: "Cable is short." },
  { id: "r3", upc: "chair-1", author: "ada", stars: 4, body: "Comfortable." },
];
let nextId = 4;

export const startReviews = (port) => startSubgraph({
  name: "reviews",
  port,
  typeDefs: `
    type Query { myReviews: [Review!]! }
    type Mutation { addReview(upc: ID!, stars: Int!, body: String!): Review }
    type Review { id: ID! stars: Int! body: String! product: Product! }
    type Product @key(fields: "upc") {
      upc: ID!
      reviews: [Review!]!
      averageStars: Float
    }
  `,
  // The router forwards the viewer as a header; this subgraph trusts only the router.
  context: async ({ req }) => ({ viewer: req.headers["x-viewer"] || null }),
  resolvers: {
    Query: {
      myReviews: (_, __, { viewer }) => {
        if (!viewer) throw new GraphQLError("Log in to see your reviews", { extensions: { code: "UNAUTHENTICATED" } });
        return REVIEWS.filter((r) => r.author === viewer);
      },
    },
    Mutation: {
      addReview: (_, { upc, stars, body }, { viewer }) => {
        if (!viewer) throw new GraphQLError("Log in to review", { extensions: { code: "UNAUTHENTICATED" } });
        if (stars < 1 || stars > 5) throw new GraphQLError("stars must be 1-5", { extensions: { code: "BAD_USER_INPUT" } });
        const review = { id: `r${nextId++}`, upc, author: viewer, stars, body };
        REVIEWS.push(review);
        return review;
      },
    },
    Review: { product: (r) => ({ __typename: "Product", upc: r.upc }) },
    Product: {
      __resolveReference: (ref) => ref,
      reviews: (p) => REVIEWS.filter((r) => r.upc === p.upc),
      averageStars: (p) => {
        const rs = REVIEWS.filter((r) => r.upc === p.upc);
        return rs.length ? Math.round((rs.reduce((s, r) => s + r.stars, 0) / rs.length) * 10) / 10 : null;
      },
    },
  },
});
```

The reviews subgraph never sees a token. It reads `x-viewer`, a header **only the router sets**,
after the router has verified the client's credentials. That's a common federation pattern, and it
depends on one deployment rule: subgraphs must not be reachable by clients directly (network policy,
mutual TLS or a shared secret between router and subgraphs), or anyone could send `x-viewer: ada`.

## The router

```js title="router.js"
import { ApolloServer } from "@apollo/server";
import { startStandaloneServer } from "@apollo/server/standalone";
import { ApolloGateway, IntrospectAndCompose, RemoteGraphQLDataSource } from "@apollo/gateway";
import { GraphQLError } from "graphql";
import { getComplexity, simpleEstimator } from "graphql-query-complexity";

const TOKENS = { "token-ada": "ada", "token-bob": "bob" }; // stand-in for JWT verification (Level 3 · 01)
const MAX_COST = 2000;

class AuthForwardingDataSource extends RemoteGraphQLDataSource {
  willSendRequest({ request, context }) {
    // Never forward the client's credentials; send the verified identity instead.
    if (context.viewer) request.http.headers.set("x-viewer", context.viewer);
  }
}

const listCost = ({ args, childComplexity, field }) =>
  field.type.toString().startsWith("[") ? (args.first ?? 10) * (childComplexity + 1) : undefined;

export async function startRouter({ subgraphs, port = 0 }) {
  const gateway = new ApolloGateway({
    supergraphSdl: new IntrospectAndCompose({ subgraphs: subgraphs.map(({ name, url }) => ({ name, url })) }),
    buildService: ({ url }) => new AuthForwardingDataSource({ url }),
  });
  const server = new ApolloServer({
    gateway,
    includeStacktraceInErrorResponses: false,
    plugins: [{
      async requestDidStart() {
        return {
          async didResolveOperation({ request, document, schema }) {
            const cost = getComplexity({ schema, query: document, variables: request.variables ?? {},
              operationName: request.operationName, estimators: [listCost, simpleEstimator({ defaultComplexity: 1 })] });
            if (cost > MAX_COST)
              throw new GraphQLError(`Query cost ${cost} exceeds ${MAX_COST}`, { extensions: { code: "QUERY_TOO_COSTLY", http: { status: 400 } } });
          },
        };
      },
    }],
  });
  const { url } = await startStandaloneServer(server, {
    listen: { port },
    context: async ({ req }) => {
      const token = (req.headers.authorization ?? "").replace(/^Bearer /, "");
      if (token && !TOKENS[token])
        throw new GraphQLError("Invalid token", { extensions: { code: "UNAUTHENTICATED", http: { status: 401 } } });
      return { viewer: TOKENS[token] ?? null };
    },
  });
  return { url, stop: () => server.stop() };
}
```

- **Authentication** happens once, in the router's context function. A bad token is a 401 before any
  subgraph is contacted. (`TOKENS` stands in for the JWT verification from
  [Level 3 · 01](../level-3/01-authentication.md) to keep the capstone focused.)
- **Identity forwarding**: a custom `RemoteGraphQLDataSource` adds `x-viewer` in `willSendRequest`.
  The client's own `Authorization` header is deliberately *not* forwarded — subgraphs get a verified
  identity, not a credential they'd each have to verify.
- **Cost limit**: the `didResolveOperation` plugin from [Level 3 · 05](../level-3/05-security-limits.md),
  now computed against the **supergraph** schema, so it covers fields from every subgraph.

## Running it

```js title="main.js"
import { startCatalog } from "./catalog.js";
import { startInventory } from "./inventory.js";
import { startReviews } from "./reviews.js";
import { startRouter } from "./router.js";

const subgraphs = [await startCatalog(4501), await startInventory(4502), await startReviews(4503)];
const router = await startRouter({ subgraphs, port: Number(process.env.PORT ?? 4500) });
console.log(`storefront supergraph at ${router.url}`);
for (const s of subgraphs) console.log(`  subgraph ${s.name.padEnd(9)} ${s.url}`);
```

```console
$ node main.js
Enabling inline tracing for this subgraph. To disable, use ApolloServerPluginInlineTraceDisabled.
Enabling inline tracing for this subgraph. To disable, use ApolloServerPluginInlineTraceDisabled.
Enabling inline tracing for this subgraph. To disable, use ApolloServerPluginInlineTraceDisabled.
storefront supergraph at http://localhost:4500/
  subgraph catalog   http://localhost:4501/
  subgraph inventory http://localhost:4502/
  subgraph reviews   http://localhost:4503/
$ curl -s localhost:4500/ -H 'content-type: application/json' -H 'authorization: Bearer token-bob' \
    -d '{"query":"{ product(upc: \"lamp-1\") { name priceCents inStock shippingEstimateCents reviews { stars body } averageStars } }"}'
{"data":{"product":{"name":"Desk lamp","priceCents":2499,"inStock":true,"shippingEstimateCents":649,"reviews":[{"stars":5,"body":"Bright and sturdy."},{"stars":3,"body":"Cable is short."}],"averageStars":4}}}
```

## The query plan

For `{ products(first: 3) { name inStock shippingEstimateCents averageStars } }`, printed with the
`experimental_didResolveQueryPlan` hook from lesson 04 (a separate script, `plan.mjs`):

```text
QueryPlan {
  Sequence {
    Fetch(service: "catalog") {
      { products(first: 3) { __typename upc name weightGrams } }
    },
    Parallel {
      Flatten(path: "products.@") {
        Fetch(service: "reviews") {
          { ... on Product { __typename upc } } =>
          { ... on Product { averageStars } }
        },
      },
      Flatten(path: "products.@") {
        Fetch(service: "inventory") {
          { ... on Product { __typename upc weightGrams } } =>
          { ... on Product { inStock shippingEstimateCents } }
        },
      },
    },
  },
}
```

(Reformatted compactly.) Three things to read off it:

1. The catalog fetch includes `weightGrams` although the client didn't ask for it — the planner added
   it because of `@requires`.
2. The inventory representations carry `weightGrams`; the reviews representations don't. Each subgraph
   receives exactly what it declared it needs.
3. Reviews and inventory are fetched in **`Parallel`** — both depend only on the catalog result — so the
   query costs two sequential hops, not three.

## Tests

```js title="storefront.test.js"
import { test, before, after } from "node:test";
import assert from "node:assert/strict";
import { startCatalog } from "./catalog.js";
import { startInventory } from "./inventory.js";
import { startReviews } from "./reviews.js";
import { startRouter } from "./router.js";

let subgraphs, router, catalog;
before(async () => {
  subgraphs = [await startCatalog(), await startInventory(), await startReviews()];
  [catalog] = subgraphs;
  router = await startRouter({ subgraphs });
});
after(async () => { await router.stop(); for (const s of subgraphs) await s.stop(); });

async function gql(query, { token, variables } = {}) {
  const res = await fetch(router.url, {
    method: "POST",
    headers: { "content-type": "application/json", ...(token ? { authorization: `Bearer ${token}` } : {}) },
    body: JSON.stringify({ query, variables }),
  });
  return { status: res.status, ...(await res.json()) };
}
const resetStats = () => subgraphs.forEach((s) => Object.assign(s.stats, { requests: 0, lookups: 0, batches: 0 }));

test("one query spans all three subgraphs, including a @requires field", async () => {
  const r = await gql(`{ products(first: 3) { name priceCents inStock shippingEstimateCents averageStars } }`);
  assert.equal(r.errors, undefined);
  assert.deepEqual(r.data.products, [
    { name: "Desk lamp", priceCents: 2499, inStock: true, shippingEstimateCents: 649, averageStars: 4 },
    { name: "Bookshelf", priceCents: 8900, inStock: false, shippingEstimateCents: 2749, averageStars: null },
    { name: "Reading chair", priceCents: 15900, inStock: true, shippingEstimateCents: 2149, averageStars: 4 },
  ]);
});

test("each subgraph is called once per query level, not once per product", async () => {
  resetStats();
  await gql(`{ products(first: 3) { name inStock averageStars } }`);
  assert.deepEqual(subgraphs.map((s) => s.stats.requests), [1, 1, 1]);
});

test("entity lookups in the catalog are batched by its DataLoader", async () => {
  resetStats();
  const r = await gql(`{ myReviews { stars product { name } } }`, { token: "token-ada" });
  assert.deepEqual(r.data.myReviews, [{ stars: 5, product: { name: "Desk lamp" } }, { stars: 4, product: { name: "Reading chair" } }]);
  assert.equal(catalog.stats.requests, 1, "one _entities request to catalog");
  assert.equal(catalog.stats.lookups, 2, "two references resolved in that request");
  assert.equal(catalog.stats.batches, 1, "...with a single DataLoader batch");
});

test("identity reaches subgraphs only via the router", async () => {
  const anon = await gql(`{ myReviews { stars } }`);
  assert.equal(anon.errors[0].extensions.code, "UNAUTHENTICATED");
  const bad = await gql(`{ products { name } }`, { token: "forged" });
  assert.equal(bad.status, 401);
  const added = await gql(`mutation { addReview(upc: "shelf-1", stars: 4, body: "Solid.") { stars product { name averageStars } } }`, { token: "token-bob" });
  assert.deepEqual(added.data.addReview, { stars: 4, product: { name: "Bookshelf", averageStars: 4 } });
});

test("the router rejects queries over the cost limit", async () => {
  const r = await gql(`query($n: Int) { products(first: $n) { name reviews { body product { name reviews { body } } } } }`, { variables: { n: 50 } });
  assert.equal(r.status, 400);
  assert.equal(r.errors[0].extensions.code, "QUERY_TOO_COSTLY");
});

test("subgraph errors reach the client with their codes", async () => {
  const r = await gql(`mutation { addReview(upc: "lamp-1", stars: 9, body: "!!") { id } }`, { token: "token-ada" });
  assert.equal(r.data.addReview, null);
  assert.equal(r.errors[0].extensions.code, "BAD_USER_INPUT");
});
```

```console
$ node --test
✔ one query spans all three subgraphs, including a @requires field
✔ each subgraph is called once per query level, not once per product
✔ entity lookups in the catalog are batched by its DataLoader
✔ identity reaches subgraphs only via the router
✔ the router rejects queries over the cost limit
✔ subgraph errors reach the client with their codes
ℹ tests 6
ℹ pass 6
ℹ fail 0
```

(Timings trimmed; the first test includes composing the supergraph.) What each one guards:

- **Cross-subgraph correctness**, including the `@requires` arithmetic: 900 g → 499 + 1×150 = 649;
  15,000 g → 499 + 15×150 = 2,749; 11,000 g → 2,149.
- **Fan-out stays constant**: `[1, 1, 1]` requests to the three subgraphs for three products. A
  regression to per-entity fetching would show up here immediately.
- **Batching inside a subgraph**: Ada's two reviews reference two products; the catalog received one
  `_entities` request, resolved two references, and ran **one** loader batch.
- **Identity**: anonymous calls are rejected by the reviews subgraph with `UNAUTHENTICATED`; a forged
  token is a 401 at the router; Bob's mutation is attributed to Bob.
- **Limits**: a nested query with `first: 50` exceeds the 2,000-point budget and never reaches a subgraph.
- **Error codes survive the hop**: a subgraph's `BAD_USER_INPUT` arrives at the client unchanged, with
  `addReview` nulled. Combine this with router-side masking ([lesson 06](06-production-errors.md)) so
  that only allow-listed codes and messages pass.

## Going to production

What this capstone deliberately left out, and what you'd add:

- **Static composition in CI** (`rover supergraph compose` or a schema registry) instead of
  `IntrospectAndCompose`, with composition and breaking-change checks per subgraph pull request
  ([lesson 05](05-schema-evolution.md)).
- **A production router** (the Rust Apollo Router, or another federation-compatible gateway) with its
  own limits, timeouts and telemetry.
- **Real token verification** and a trust boundary between router and subgraphs (mTLS or signed
  internal headers).
- **Tracing across services** — the inline traces the subgraphs already offer, or OpenTelemetry spans
  propagated through the router ([lesson 02](02-plugins-performance.md)).
- **Databases** behind each subgraph, each team owning its own.

None of these was run here; each needs infrastructure beyond a single laptop process.

## How It Actually Works

A request to the router passes through Apollo Server's pipeline: CSRF check, the context function
(token → viewer), parse, validate against the supergraph's API schema, the cost plugin in
`didResolveOperation`, and then — instead of `graphql-js`'s `execute` — the gateway's executor. It asks
the query planner for a plan (cached per operation), then walks it: each `Fetch` becomes an HTTP request
built by `AuthForwardingDataSource` (where `x-viewer` is added), each `Flatten` gathers representations
from the results so far, `Parallel` children are awaited together, and results are merged into one
response tree that's finally shaped to the client's exact selection. Errors returned by subgraphs are
re-pathed to the client's response paths and appended to `errors`. Inside each subgraph, `_entities`
calls `__resolveReference` for each representation in the same tick, which is why the catalog's
DataLoader could batch them.

## Final checklist

- [ ] Every entity has a stable key and a batched `__resolveReference`.
- [ ] Fields that depend on other subgraphs use `@requires`/`@external`, not copied data.
- [ ] The router authenticates; subgraphs trust only the router.
- [ ] Limits are enforced at the router against the whole supergraph.
- [ ] Tests assert fan-out and batching, not just results.
- [ ] Composition and breaking-change checks run in CI.

## Exercise

1. Add a **users** subgraph that owns `User { id name }` and make `Review.author` an entity reference to
   it. Update the tests so `myReviews { author { name } }` costs one request per subgraph.
2. Move `averageStars` to a new **ratings** subgraph using `@override(from: "reviews")` and confirm the
   query plan changes while clients see no difference.
3. Add router-side error masking with an allow-list, and a test proving a subgraph's raw exception text
   never reaches the client.
4. Stop the inventory subgraph mid-test and decide what clients should receive. Make `inStock` nullable
   if needed so the rest of the product still renders, and test it.
