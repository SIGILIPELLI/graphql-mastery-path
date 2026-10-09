---
description: "Run a federated GraphQL graph locally with @apollo/gateway: compose two subgraphs, read query plans, watch the _entities traffic."
---

# 04 · Running a Federated Graph Locally

[Lesson 03](03-federation-concepts.md) built two subgraphs and queried them directly. Now they
get a **router** in front of them, so a client sees one schema and one endpoint. This lesson
runs the whole thing in one Node process using `@apollo/gateway` 2.14.4 — the JavaScript
router — prints the **query plan** for each client query, and logs exactly which requests the
router sends to which subgraph. Those plans are how you reason about federated performance.

## Production versus local routing

Apollo's recommended production router is the **Apollo Router** (Rust, now often called
GraphOS Router), fed a supergraph schema composed in CI with the `rover` CLI or by Apollo's
hosted registry. That setup wasn't run here — it needs either a separate binary or an Apollo
account. `@apollo/gateway` implements the same federation query planning in JavaScript and runs
inside Apollo Server, which makes it ideal for learning and local development. Its
`IntrospectAndCompose` mode composes the supergraph at startup by calling each subgraph's
`_service { sdl }`; it's documented as unsuitable for production because composition errors then
surface at runtime rather than in CI.

## The setup

```js title="gateway04.mjs"
import { ApolloServer } from "@apollo/server";
import { startStandaloneServer } from "@apollo/server/standalone";
import { ApolloGateway, IntrospectAndCompose } from "@apollo/gateway";
import { prettyFormatQueryPlan } from "@apollo/query-planner";
import { productsSchema, reviewsSchema } from "./subgraphs.js";

// Log every request each subgraph receives, so we can see the router's traffic.
const subgraphLog = [];
const logPlugin = (name) => ({
  async requestDidStart({ request }) {
    const q = request.query.replace(/\s+/g, " ").trim();
    subgraphLog.push(`${name}: ${q.length > 110 ? q.slice(0, 107) + "..." : q}` +
      (request.variables?.representations ? `  [${request.variables.representations.length} representations]` : ""));
  },
});

async function startSubgraph(name, schema, port) {
  const server = new ApolloServer({ schema, plugins: [logPlugin(name)] });
  const { url } = await startStandaloneServer(server, { listen: { port } });
  return { server, url };
}
const products = await startSubgraph("products", productsSchema, 4401);
const reviews = await startSubgraph("reviews", reviewsSchema, 4402);

let lastPlan = "";
const gateway = new ApolloGateway({
  // Dev-only: introspect running subgraphs and compose at startup.
  supergraphSdl: new IntrospectAndCompose({
    subgraphs: [
      { name: "products", url: products.url },
      { name: "reviews", url: reviews.url },
    ],
  }),
  experimental_didResolveQueryPlan: ({ queryPlan }) => { lastPlan = prettyFormatQueryPlan(queryPlan); },
});
const router = new ApolloServer({ gateway });
const { url } = await startStandaloneServer(router, { listen: { port: 4400 } });
console.log(`router at ${url}`);

async function query(label, q) {
  subgraphLog.length = 0;
  const res = await fetch(url, { method: "POST", headers: { "content-type": "application/json" }, body: JSON.stringify({ query: q }) });
  console.log(`\n=== ${label}\n${JSON.stringify(await res.json())}`);
  console.log("--- query plan\n" + lastPlan);
  console.log("--- subgraph requests\n" + subgraphLog.join("\n"));
}

await query("one subgraph", `{ products { name priceCents } }`);
await query("products with their reviews", `{ products { name reviews { stars } averageStars } }`);
await query("reviews with product names", `{ latestReviews { stars product { name } } }`);

await router.stop(); await products.server.stop(); await reviews.server.stop();
```

`subgraphs.js` is unchanged from lesson 03. Each subgraph runs on its own Apollo Server with a
logging plugin; the router is an Apollo Server constructed with `gateway` instead of a schema.
`experimental_didResolveQueryPlan` captures the plan for each operation, and the query planner's
`prettyFormatQueryPlan` prints it.

On start, Apollo Server printed this once per subgraph:

```text
Enabling inline tracing for this subgraph. To disable, use ApolloServerPluginInlineTraceDisabled.
```

Apollo Server detects a federated schema and enables **inline tracing**: when the router asks, the
subgraph returns per-field timing in the response extensions so the router can assemble a
cross-subgraph trace. It's harmless locally.

## Query 1: one subgraph

```text
=== one subgraph
{"data":{"products":[{"name":"Desk lamp","priceCents":2499},{"name":"Bookshelf","priceCents":8900}]}}
--- query plan
QueryPlan {
  Fetch(service: "products") {
    {
      products {
        name
        priceCents
      }
    }
  },
}
--- subgraph requests
products: {products{name priceCents}}
```

Every field lives in `products`, so the plan is a single `Fetch` that forwards the query almost
verbatim. Federation adds one network hop and nothing else.

## Query 2: products with their reviews

```text
=== products with their reviews
{"data":{"products":[{"name":"Desk lamp","reviews":[{"stars":5},{"stars":3}],"averageStars":4},{"name":"Bookshelf","reviews":[{"stars":4}],"averageStars":4}]}}
--- query plan
QueryPlan {
  Sequence {
    Fetch(service: "products") {
      { products { __typename upc name } }
    },
    Flatten(path: "products.@") {
      Fetch(service: "reviews") {
        { ... on Product { __typename upc } } =>
        { ... on Product { reviews { stars } averageStars } }
      },
    },
  },
}
--- subgraph requests
products: {products{__typename upc name}}
reviews: query($representations:[_Any!]!){_entities(representations:$representations){...on Product{reviews{stars}av...  [2 representations]
```

(Plan reformatted more compactly; the logged subgraph query is truncated by the script.)

Read it as a program:

1. **`Sequence`** — steps that depend on each other run in order.
2. **`Fetch(products)`** — the planner *added* `__typename` and `upc` to the selection: the key
   fields it needs for the next step, though the client never asked for them.
3. **`Flatten(path: "products.@")`** — for each item in the `products` list (`@` means "every
   element"), take its `{ __typename, upc }` as a representation…
4. **`Fetch(reviews)`** — …and send all of them in **one** `_entities` request (`[2
   representations]`), selecting the fields the reviews subgraph owns. The results are merged back
   into the products at the same path.

Two subgraph requests regardless of how many products there are: the router batches across the
list, the way DataLoader batches across resolvers. The cost is that the steps are sequential —
`reviews` can't start until `products` returns.

## Query 3: the other direction

```text
=== reviews with product names
{"data":{"latestReviews":[{"stars":4,"product":{"name":"Bookshelf"}},{"stars":3,"product":{"name":"Desk lamp"}},{"stars":5,"product":{"name":"Desk lamp"}}]}}
--- subgraph requests
reviews: {latestReviews{stars product{__typename upc}}}
products: query($representations:[_Any!]!){_entities(representations:$representations){...on Product{name}}}  [3 representations]
```

The plan is the mirror image: `reviews` returns references, `products` resolves them. Note **3
representations** for 2 distinct products — the router sent `p1` twice, once per review. The
subgraph's `__resolveReference` ran three times. With a database behind it, that's exactly where a
DataLoader in the subgraph earns its keep, de-duplicating and batching the lookups.

## Reading plans for performance

- **Count the `Fetch` nodes** — each is a network round trip.
- **Look for `Sequence` depth** — sequential fetches add their latencies; `Parallel` nodes (when
  independent subgraphs can be queried at once) don't.
- **Watch for chains through entities**: a query going products → reviews → users → products is
  four sequential hops. Sometimes moving or duplicating (`@shareable`) a field removes a hop.
- **Check what the planner adds** (key fields, `__typename`) — cheap, but they must be resolvable.

## How It Actually Works

At startup, `IntrospectAndCompose` fetches each subgraph's `_service.sdl` and runs **composition**
(`@apollo/composition`): it merges the subgraph schemas, checks that shared types agree and that
every field can be reached, and produces a **supergraph** SDL annotated with `@join__*` directives
recording which subgraph owns each field and how to fetch entities. From that, the query planner
builds a graph of "how can I reach field F of type T from subgraph S" (the *query graph*).

For each client operation, the planner walks the selection set and finds the cheapest way to
satisfy every field: stay in the current subgraph when it can resolve a field; otherwise jump via
an entity key to a subgraph that can, inserting the key fields into the previous fetch. The result
is a tree of `Fetch`, `Flatten`, `Sequence` and `Parallel` nodes, cached per operation. Execution
walks the tree: it sends each `Fetch` to its subgraph (via a `RemoteGraphQLDataSource`), collects
representations at `Flatten` paths from the data so far, and merges each response into the
combined result before shaping it to the client's exact selection.

## Common mistakes

- **Using `IntrospectAndCompose` in production.** Compose in CI and deploy a static supergraph.
- **Ignoring plans until production.** Print them in development for important queries.
- **No batching in `__resolveReference`**, turning one `_entities` call into N database queries.
- **Letting clients reach subgraphs directly** (bypassing the router's limits and auth).
- **Long entity chains** for hot queries, each hop adding latency.

## Exercise

1. Add a third subgraph (`inventory`, contributing `Product.inStock`) and query
   `{ products { name inStock reviews { stars } } }`. Does the plan use `Parallel`? Why?
2. Implement `Product.__resolveReference` in the products subgraph with a DataLoader and confirm
   query 3 now performs one lookup for the 3 representations.
3. Stop the reviews subgraph after the router starts and run query 2. What does the client receive —
   partial data, an error, or both?
4. Add `@shareable` to `Product.name` and also resolve it in the reviews subgraph. How does query 3's
   plan change?
