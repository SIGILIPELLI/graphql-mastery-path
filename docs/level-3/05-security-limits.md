---
description: "Measure how nested and aliased GraphQL queries amplify server work, then defend with depth limits, cost analysis, page caps, batching limits and timeouts."
---

# 05 · Securing a Public Graph: Depth, Cost, Aliases & Introspection

GraphQL lets the client decide how much work the server does. For your own front end that's
the point; for an attacker it's a lever. A 127-character query can make a server run hundreds
of thousands of resolvers. This lesson measures that amplification on a small schema, then
builds and tests the standard defences: depth limits, cost analysis, caps on page sizes and
batching — and explains why none of them is sufficient alone.

## The amplification, measured

A schema with a cycle — authors have books, books have an author — and 20 authors with 20 books
each:

```js title="limits05.mjs"
import { makeExecutableSchema } from "@graphql-tools/schema";
import { graphql, parse, validate, specifiedRules, GraphQLError, Kind } from "graphql";
import { getComplexity, simpleEstimator, fieldExtensionsEstimator } from "graphql-query-complexity";

// 20 authors, each with 20 books; every book points back at its author.
const authors = Array.from({ length: 20 }, (_, i) => ({ id: `a${i}`, name: `Author ${i}` }));
const books = authors.flatMap((a) => Array.from({ length: 20 }, (_, j) => ({ id: `${a.id}b${j}`, title: `Book ${j}`, authorId: a.id })));
let resolverCalls = 0;
const count = (fn) => (...args) => { resolverCalls++; return fn(...args); };

const schema = makeExecutableSchema({
  typeDefs: /* GraphQL */ `
    type Query { authors(first: Int = 10): [Author!]! }
    type Author { id: ID! name: String! books(first: Int = 10): [Book!]! }
    type Book { id: ID! title: String! author: Author! }
  `,
  resolvers: {
    Query: { authors: count((_, { first }) => authors.slice(0, first)) },
    Author: { books: count((a, { first }) => books.filter((b) => b.authorId === a.id).slice(0, first)) },
    Book: { author: count((b) => authors.find((a) => a.id === b.authorId)) },
  },
});

const nest = (depth) => {
  let sel = "name";
  for (let i = 0; i < depth; i++) sel = `books(first: 20) { author { ${sel} } }`;
  return `{ authors(first: 20) { ${sel} } }`;
};

async function cost(label, source) {
  resolverCalls = 0;
  const t = performance.now();
  const r = await graphql({ schema, source });
  const ms = performance.now() - t;
  const bytes = JSON.stringify(r).length;
  console.log(`${label.padEnd(30)} resolver calls=${String(resolverCalls).padStart(9)}  response=${String(bytes).padStart(10)} bytes  ${ms.toFixed(0).padStart(6)} ms`);
}

console.log("== nesting without limits");
for (const d of [1, 2, 3]) await cost(`depth ${d} (query ${nest(d).length} chars)`, nest(d));

console.log("\n== aliases multiply work in a single short query");
const aliases = (n) => `{ ${Array.from({ length: n }, (_, i) => `a${i}: authors(first: 20) { books(first: 20) { title } }`).join(" ")} }`;
await cost(`100 aliases (${aliases(100).length} chars)`, aliases(100));

// ---- Defence 1: a depth limit as a custom validation rule
function depthLimit(max) {
  return (context) => {
    const fragments = Object.fromEntries(
      context.getDocument().definitions.filter((d) => d.kind === Kind.FRAGMENT_DEFINITION).map((d) => [d.name.value, d]));
    function depthOf(selectionSet, seen = new Set()) {
      if (!selectionSet) return 0;
      let max = 0;
      for (const sel of selectionSet.selections) {
        if (sel.kind === Kind.FIELD) {
          if (sel.name.value.startsWith("__")) continue;
          max = Math.max(max, sel.selectionSet ? 1 + depthOf(sel.selectionSet, seen) : 1);
        } else if (sel.kind === Kind.INLINE_FRAGMENT) {
          max = Math.max(max, depthOf(sel.selectionSet, seen));
        } else if (sel.kind === Kind.FRAGMENT_SPREAD && !seen.has(sel.name.value)) {
          max = Math.max(max, depthOf(fragments[sel.name.value]?.selectionSet, new Set(seen).add(sel.name.value)));
        }
      }
      return max;
    }
    return {
      OperationDefinition(node) {
        const depth = depthOf(node.selectionSet);
        if (depth > max) context.reportError(new GraphQLError(`Query depth ${depth} exceeds maximum of ${max}`, { nodes: [node] }));
      },
    };
  };
}

// ---- Defence 2: cost analysis
function costOf(source, variables = {}) {
  return getComplexity({
    schema,
    query: parse(source),
    variables,
    estimators: [
      // list fields cost (child cost) x (requested page size)
      ({ args, childComplexity, field }) =>
        field.type.toString().startsWith("[") ? (args.first ?? 10) * (childComplexity + 1) : undefined,
      simpleEstimator({ defaultComplexity: 1 }),
    ],
  });
}

console.log("\n== validation with a depth limit of 5");
for (const [label, q] of [["depth 1", nest(1)], ["depth 2", nest(2)], ["100 aliases", aliases(100)]]) {
  const errs = validate(schema, parse(q), [...specifiedRules, depthLimit(5)]);
  console.log(label.padEnd(12), errs.length ? `REJECTED: ${errs[0].message}` : "allowed");
}

console.log("\n== estimated cost (limit 5,000)");
for (const [label, q] of [
  ["10 authors' names", `{ authors { name } }`],
  ["depth 1", nest(1)], ["depth 2", nest(2)], ["depth 3", nest(3)], ["100 aliases", aliases(100)],
]) {
  const c = costOf(q);
  console.log(label.padEnd(18), String(c).padStart(10), c > 5000 ? "REJECTED" : "allowed");
}
```

```console
$ node limits05.mjs
== nesting without limits
depth 1 (query 63 chars)       resolver calls=      421  response=     12862 bytes       5 ms
depth 2 (query 95 chars)       resolver calls=     8821  response=    261462 bytes      20 ms
depth 3 (query 127 chars)      resolver calls=   176821  response=   5233462 bytes     254 ms

== aliases multiply work in a single short query
100 aliases (5493 chars)       resolver calls=     2100  response=    804800 bytes      40 ms
```

(Timings are from one laptop with in-memory data and only indicate growth.) Each extra level of
`books(first: 20) { author { … } }` multiplies the work by about 20. Depth 3 — 127 characters —
produced 176,821 resolver calls and a 5 MB response. Depth 4 would be roughly 3.5 million calls;
it wasn't run. With a real database behind each resolver, even depth 2 would hurt, and
DataLoader doesn't save you: batching reduces round trips, but the server still materialises
every object and serialises every byte.

Aliases are the second lever. The 100-alias query is *shallow* — depth 3 — but asks for the same
expensive list 100 times under different names. GraphQL doesn't de-duplicate fields with
different aliases, so the work is repeated.

## Defence 1: a depth limit

A depth limit is a **validation rule**: a function that visits the parsed document and reports
errors before anything executes. The `depthLimit` in the script counts nested selection sets,
following fragment spreads (with a guard against fragment cycles) and ignoring introspection
fields.

```text
== validation with a depth limit of 5
depth 1      allowed
depth 2      REJECTED: Query depth 6 exceeds maximum of 5
100 aliases  allowed
```

It stopped deep nesting cheaply, but the 100-alias attack sailed through: it's wide, not deep. A
depth limit is a useful backstop and nothing more. (Packages such as `graphql-depth-limit`
implement the same rule; writing it shows there's no magic.)

## Defence 2: cost analysis

Cost analysis estimates the work *before* execution by walking the query with the schema and the
variables. The script uses `graphql-query-complexity` (2.0.0) with a custom estimator: a list
field costs its page size × (the cost of its children + 1); everything else costs 1.

```text
== estimated cost (limit 5,000)
10 authors' names          20 allowed
depth 1                  1220 allowed
depth 2                 24820 REJECTED
depth 3                496820 REJECTED
100 aliases             82000 REJECTED
```

Cost tracks the real resolver counts closely (depth 1: estimated 1,220 for 421 actual calls;
it's an upper bound because it assumes every list is full), and it catches the alias attack,
because aliases add up. Cost analysis is the main defence for public APIs; GitHub's and
Shopify's public GraphQL APIs both document point-based limits computed from requested page
sizes.

### Wiring it into Apollo Server

Validation rules can't see variable values, but page sizes often come from variables. Apollo
Server's `didResolveOperation` hook runs after parsing and validation, with variables available,
and before execution:

```js title="apollo-limits.mjs"
import { ApolloServer } from "@apollo/server";
import { startStandaloneServer } from "@apollo/server/standalone";
import { GraphQLError } from "graphql";
import { getComplexity, simpleEstimator } from "graphql-query-complexity";

const typeDefs = `type Query { authors(first: Int = 10): [Author!]! } type Author { name: String! books(first: Int = 10): [Book!]! } type Book { title: String! author: Author! }`;
const resolvers = { Query: { authors: (_, { first }) => Array.from({ length: first }, (_, i) => ({ name: `A${i}` })) },
  Author: { books: (_, { first }) => Array.from({ length: first }, (_, i) => ({ title: `B${i}` })) }, Book: { author: () => ({ name: "A" }) } };

const MAX_COST = 5000;
const listCost = ({ args, childComplexity, field }) =>
  field.type.toString().startsWith("[") ? (args.first ?? 10) * (childComplexity + 1) : undefined;

const server = new ApolloServer({
  typeDefs, resolvers, includeStacktraceInErrorResponses: false,
  plugins: [{
    async requestDidStart() {
      return {
        // runs after parsing and validation, before execution, with variables available
        async didResolveOperation({ request, document, schema }) {
          const cost = getComplexity({ schema, query: document, variables: request.variables ?? {},
            operationName: request.operationName, estimators: [listCost, simpleEstimator({ defaultComplexity: 1 })] });
          if (cost > MAX_COST)
            throw new GraphQLError(`Query cost ${cost} exceeds the limit of ${MAX_COST}`,
              { extensions: { code: "QUERY_TOO_COSTLY", cost, limit: MAX_COST, http: { status: 400 } } });
        },
      };
    },
  }],
});
const { url } = await startStandaloneServer(server, { listen: { port: 4311 } });
const post = async (body) => { const r = await fetch(url, { method: "POST", headers: { "content-type": "application/json" }, body: JSON.stringify(body) }); return `${r.status} ${await r.text()}`; };
console.log(await post({ query: "{ authors { books { title } } }" }));
console.log(await post({ query: "query($n: Int) { authors(first: $n) { books(first: $n) { title } } }", variables: { n: 100 } }));
console.log((await post([{ query: "{ authors { name } }" }, { query: "{ authors { name } }" }])).slice(0, 200));
await server.stop();
```

```console
$ node apollo-limits.mjs
200 {"data":{"authors":[{"books":[{"title":"B0"},…]}]}}
400 {"errors":[{"message":"Query cost 20100 exceeds the limit of 5000","extensions":{"code":"QUERY_TOO_COSTLY","cost":20100,"limit":5000}}]}
400 {"errors":[{"message":"Operation batching disabled.","extensions":{"code":"BAD_REQUEST"}}]}
```

(First response abbreviated.) The second request looked innocent — its text is short — but
`$n: 100` made it cost 100 × (100 × 2 + 1) = 20,100. Returning the cost and limit in
`extensions` lets well-behaved clients understand the rejection.

The third request was a JSON **array** of operations — HTTP batching. Apollo Server rejects it
unless `allowBatchedHttpRequests: true` is set. If you enable it, cap the batch size, or one HTTP
request becomes an unbounded number of operations that per-request limits never see.

## Defence 3: limits that need no analysis

- **Maximum page size** on every list argument — the `first > 50` check from
  [Level 2 · 07](../level-2/07-pagination.md). It bounds what cost analysis assumes.
- **Maximum query size** (bytes of query text) and **maximum number of aliases** or selections:
  cheap checks that stop absurd documents before validation even runs.
- **Execution timeouts** and an **overall response size cap**: something must stop a request the
  estimators misjudged.
- **Rate limiting per client**, ideally by accumulated *cost* rather than by request count.

## Defence 4: persisted operations for first-party apps

If only your own apps call the API, the strongest defence is to not accept arbitrary queries at
all: register the operations your apps use at build time, and have the server execute only those,
by id. Any query an attacker writes is simply unknown. That's covered in
[lesson 06](06-caching-persisted-queries.md).

## What about introspection?

Disabling introspection in production ([Level 1 · 09](../level-1/09-introspection-tooling.md))
hides the map but doesn't stop any attack above — `books { author { books … } }` can be guessed
from the app's own network traffic. Treat it as reducing exposure, not as a defence.

## How It Actually Works

`validate(schema, document, rules)` runs all rules in **parallel** over a single traversal of the
AST: `graphql-js` merges the rules' visitors (`visitInParallel`) and walks the tree once, calling
each rule's enter/leave functions for each node. A custom rule is therefore cheap — it shares the
walk with the built-in rules. `context.reportError` collects errors without stopping the walk,
which is why validation reports everything at once.

`getComplexity` does its own walk using `TypeInfo` so that at each field it knows the parent type,
the field definition and the coerced arguments (with variables substituted). For each field, it
first computes the children's complexity recursively, then asks each estimator in order until one
returns a number. That bottom-up order is what lets a list estimator multiply `childComplexity` by
the page size. Fields under `@skip(if: true)` / `@include(if: false)` are excluded, because they
won't execute.

## Common mistakes

- **Only a depth limit.** Wide queries and aliases bypass it.
- **Cost limits computed without variables**, so `first: $n` is assumed to be the default.
- **Unbounded list arguments** (or none at all), which make every estimate meaningless.
- **Enabling HTTP batching without a cap.**
- **Relying on disabled introspection** as a security control.
- **One global limit for everyone.** Internal services, first-party apps and anonymous third
  parties usually deserve different budgets.

## Exercise

1. Add an alias-count limit as a validation rule (count `Field` nodes that have an `alias`) and
   show it rejecting the 100-alias query while allowing normal ones.
2. Give `Book.author` a lower cost than `Author.books` using `fieldExtensionsEstimator` and field
   `extensions` in a code-first schema, or the `@complexity` directive from
   `createComplexityDirective` in SDL.
3. Add a 2-second execution timeout: race the execution against a timer in a plugin, and decide
   what the client should receive.
4. Make the cost limit depend on the viewer: 5,000 for anonymous, 50,000 for logged-in users.
