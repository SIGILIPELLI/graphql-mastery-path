---
description: "Apollo Federation from the inside: subgraphs, entities, @key, __resolveReference, and the _service and _entities fields queried directly."
---

# 03 · Federation Concepts: Subgraphs, Entities & @key

One GraphQL schema for a whole company is great for clients and hard for the teams behind it:
one repository, one deploy, one team bottlenecking every change. **Federation** splits the
graph into **subgraphs**, each owned and deployed by a team, and composes them into one
**supergraph** that clients query through a **router**. This lesson builds two subgraphs with
`@apollo/subgraph` 2.15.1 and — before introducing any router — queries them directly, so the
two mechanisms everything rests on, `_service` and `_entities`, are visible rather than magic.
[Lesson 04](04-federation-gateway.md) adds the router.

## The idea: entities

An **entity** is a type that more than one subgraph contributes fields to, identified by a
**key**:

```text
products subgraph                         reviews subgraph
type Product @key(fields: "upc") {        type Product @key(fields: "upc") {
  upc: ID!                                  upc: ID!
  name: String!                             reviews: [Review!]!
  priceCents: Int!                          averageStars: Float
  weightGrams: Int!                       }
}
```

Clients see one `Product` with all six fields. Each subgraph only knows its own fields plus the
key. When a query needs fields from both, the router fetches from one subgraph, extracts the
keys, and asks the other subgraph "give me these fields for the products with these keys".

## Two subgraphs

```js title="subgraphs.js"
import { buildSubgraphSchema } from "@apollo/subgraph";
import { parse } from "graphql";

const FED = `extend schema @link(url: "https://specs.apollo.dev/federation/v2.3", import: ["@key", "@shareable", "@external", "@requires"])`;

// ---------- products subgraph: owns Product
const products = [
  { upc: "p1", name: "Desk lamp", priceCents: 2499, weightGrams: 900 },
  { upc: "p2", name: "Bookshelf", priceCents: 8900, weightGrams: 15000 },
];
export const productsSchema = buildSubgraphSchema([{
  typeDefs: parse(`${FED}
    type Query { products: [Product!]! product(upc: ID!): Product }
    type Product @key(fields: "upc") {
      upc: ID!
      name: String!
      priceCents: Int!
      weightGrams: Int!
    }
  `),
  resolvers: {
    Query: {
      products: () => products,
      product: (_, { upc }) => products.find((p) => p.upc === upc) ?? null,
    },
    Product: {
      // Called when another subgraph's result needs Product fields owned here.
      __resolveReference: (ref) => products.find((p) => p.upc === ref.upc) ?? null,
    },
  },
}]);

// ---------- reviews subgraph: owns Review, adds Product.reviews
const reviews = [
  { id: "r1", productUpc: "p1", stars: 5, body: "Bright and sturdy." },
  { id: "r2", productUpc: "p1", stars: 3, body: "Cable is short." },
  { id: "r3", productUpc: "p2", stars: 4, body: "Took an hour to build." },
];
export const reviewsSchema = buildSubgraphSchema([{
  typeDefs: parse(`${FED}
    type Query { latestReviews: [Review!]! }
    type Review { id: ID! stars: Int! body: String! product: Product! }
    type Product @key(fields: "upc") {
      upc: ID!
      reviews: [Review!]!
      averageStars: Float
    }
  `),
  resolvers: {
    Query: { latestReviews: () => reviews.slice().reverse() },
    Review: { product: (r) => ({ __typename: "Product", upc: r.productUpc }) }, // a reference
    Product: {
      __resolveReference: (ref) => ref, // nothing stored per product here; the key is enough
      reviews: (p) => reviews.filter((r) => r.productUpc === p.upc),
      averageStars: (p) => {
        const rs = reviews.filter((r) => r.productUpc === p.upc);
        return rs.length ? rs.reduce((s, r) => s + r.stars, 0) / rs.length : null;
      },
    },
  },
}]);
```

Points to note:

- `extend schema @link(url: ".../federation/v2.3", import: [...])` opts the SDL into **Federation 2**
  and imports the directives used.
- `Review.product` returns `{ __typename: "Product", upc }` — a **reference**. The reviews
  subgraph doesn't know a product's name or price, and doesn't pretend to; the router will fill
  those in from the products subgraph.
- `Product.__resolveReference` in each subgraph is called with a representation
  (`{ __typename, upc }`) and returns the subgraph's own object for that key.

### A version gotcha

Many examples call `buildSubgraphSchema({ typeDefs, resolvers })` with a single object. With
2.15.1 that failed here:

```text
TypeError: doc.definitions is not iterable (cannot read property undefined)
    at concatAST (.../graphql/utilities/concatAST.js:32:17)
    at buildSubgraphSchema (.../@apollo/subgraph/dist/buildSubgraphSchema.js:19:50)
```

This version treats a non-array argument as a bare SDL document; the `{ typeDefs, resolvers }`
module form is only recognised **inside an array**, so the code above passes
`[{ typeDefs, resolvers }]`. The array form works across versions, so prefer it.

## What `buildSubgraphSchema` adds

```js title="concepts03.mjs"
import { graphql, printSchema } from "graphql";
import { productsSchema, reviewsSchema } from "./subgraphs.js";

const run = async (label, schema, source, variableValues) =>
  console.log(`--- ${label}\n${JSON.stringify(await graphql({ schema, source, variableValues }))}`);

// 1. What the subgraph adds to its own schema
console.log("root Query fields in reviews subgraph:", Object.keys(reviewsSchema.getQueryType().getFields()).join(", "));
console.log("_Entity union members:", reviewsSchema.getType("_Entity").getTypes().map((t) => t.name).join(", "));

// 2. _service { sdl }: the subgraph's SDL *with* federation directives
const sdl = (await graphql({ schema: reviewsSchema, source: "{ _service { sdl } }" })).data._service.sdl;
console.log("\n--- reviews _service.sdl (Product part)\n" + sdl.split("\n").filter((l, i, a) => {
  const start = a.findIndex((x) => x.startsWith("type Product"));
  return i >= start && i <= start + 4;
}).join("\n"));

// 3. _entities: how a router asks a subgraph about objects it doesn't own
const ENTITIES = `query($reps: [_Any!]!) { _entities(representations: $reps) { ... on Product { upc reviews { stars } averageStars } } }`;
await run("reviews._entities for two products", reviewsSchema, ENTITIES,
  { reps: [{ __typename: "Product", upc: "p2" }, { __typename: "Product", upc: "p1" }] });

await run("products._entities resolving references", productsSchema,
  `query($reps: [_Any!]!) { _entities(representations: $reps) { ... on Product { upc name priceCents } } }`,
  { reps: [{ __typename: "Product", upc: "p1" }, { __typename: "Product", upc: "zzz" }] });

// 4. A subgraph alone can only answer its own part
await run("reviews subgraph alone", reviewsSchema, `{ latestReviews { stars product { upc } } }`);
```

```console
$ node concepts03.mjs
root Query fields in reviews subgraph: latestReviews, _entities, _service
_Entity union members: Product
```

Two root fields appeared that we never wrote:

- **`_service: _Service!`** returns `{ sdl }`, the subgraph's SDL **including** federation
  directives. Standard introspection can't express applied directives like `@key`
  ([Level 3 · 03](../level-3/03-directives.md)), so federation needs this side channel.
- **`_entities(representations: [_Any!]!): [_Entity]!`** — the entity-fetching entry point.
  `_Entity` is a union of every type with a `@key`.

### `_service`

```text
--- reviews _service.sdl (Product part)
type Product @key(fields: "upc") {
  upc: ID!
  reviews: [Review!]!
  averageStars: Float
}
```

This is what composition tools read to build the supergraph.

### `_entities`

```text
--- reviews._entities for two products
{"data":{"_entities":[{"upc":"p2","reviews":[{"stars":4}],"averageStars":4},{"upc":"p1","reviews":[{"stars":5},{"stars":3}],"averageStars":4}]}}
--- products._entities resolving references
{"data":{"_entities":[{"upc":"p1","name":"Desk lamp","priceCents":2499},null]}}
```

This is exactly the request a router sends. It passes a list of representations; the subgraph
calls `__resolveReference` for each, in order, and the router's selection (`... on Product {
… }`) picks the fields. Results line up with the input positions — `p2` then `p1` — and an
unknown key (`zzz`) comes back as `null` in its slot. That ordered, batched shape is a
DataLoader in disguise: implement `__resolveReference` with a loader and one `_entities` call
becomes one database query.

### A subgraph alone

```text
--- reviews subgraph alone
{"data":{"latestReviews":[{"stars":4,"product":{"upc":"p2"}},{"stars":3,"product":{"upc":"p1"}},{"stars":5,"product":{"upc":"p1"}}]}}
```

Queried directly, the reviews subgraph can only return product *keys*. Asking it for
`product { name }` would be a validation error — `name` isn't in its schema. The router is what
turns references into full objects.

## Federation vocabulary

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

## When federation is worth it

Federation solves an *organisational* problem — many teams, one graph — with real costs: an extra
network hop per subgraph involved, query planning, composition in CI, and distributed debugging.
For one team with one service, a single schema (perhaps modular in code) is simpler.
[Lesson 09](09-architecture-review.md) weighs this in more detail.

## How It Actually Works

`buildSubgraphSchema` parses your SDL, processes the `@link` to learn which federation
definitions to add, and appends them: the `_Any` scalar, `_Service` type, `_Entity` union (of all
`@key` types) and the two root fields. It builds the schema with `buildASTSchema`, attaches your
resolvers, and stores each type's `__resolveReference` in `type.extensions.apollo.subgraph`. The
`_entities` resolver maps over `representations`: for each, it looks up the type by `__typename`,
calls that type's stored `__resolveReference(representation, context, info)` (or, if there is
none, returns the representation itself), and the result's type is resolved through the `_Entity`
union using the representation's `__typename`. `_service.sdl` is printed from the *original*
document, which is how applied directives survive.

## Common mistakes

- **Returning full objects you don't own** instead of references — the subgraph then has to know
  fields it doesn't own, and they drift.
- **Keys that aren't stable or unique** across subgraphs (one uses a database id, another a SKU).
- **Slow `__resolveReference`** without batching — an N+1 across the network.
- **Exposing subgraphs directly to clients.** `_entities` lets callers fetch any entity by key,
  bypassing whatever checks the router path applies. Subgraphs should only accept traffic from the
  router.
- **Splitting by technical layer** (a "database subgraph") instead of by domain ownership.

## Exercise

1. Add an `inventory` subgraph that contributes `Product.inStock: Boolean!`, and query its
   `_entities` directly.
2. Implement `Product.__resolveReference` in the products subgraph with a DataLoader and log how many
   batch calls one `_entities` request with 50 representations makes.
3. Add `shippingEstimate: Int @requires(fields: "weightGrams")` to the inventory subgraph (with
   `weightGrams: Int! @external`). What does its `_entities` representation need to contain?
4. Print the full `_service.sdl` of both subgraphs and identify everything `buildSubgraphSchema` added.
