---
description: "Consume GraphQL from clients: plain fetch, then Apollo Client 4's normalized cache — cache keys, mutation updates and cache redirects."
---

# 07 · Clients: fetch, Apollo Client & the Normalized Cache

Everything so far has been server-side. This lesson switches seats. A GraphQL client can be as
simple as one `fetch` call; the reason teams adopt a client library is the **normalized
cache**, which stores each object once by identity and keeps every query that shows it
consistent. Understanding how that cache keys objects explains several server-side
conventions — always exposing `id`, returning changed objects from mutations — that otherwise
look like style preferences. All examples run in Node against a small Apollo Server, using
**Apollo Client 4.3.3** without any UI framework.

## The server

```js title="api07.mjs"
import { ApolloServer } from "@apollo/server";
import { startStandaloneServer } from "@apollo/server/standalone";

const books = [
  { id: "1", title: "Dune", stock: 3, authorId: "a1" },
  { id: "2", title: "Dune Messiah", stock: 0, authorId: "a1" },
];
const authors = [{ id: "a1", name: "Frank Herbert" }];
export const stats = { requests: 0 };

const server = new ApolloServer({
  typeDefs: /* GraphQL */ `
    type Query { books: [Book!]! book(id: ID!): Book }
    type Mutation { restock(id: ID!, amount: Int!): Book }
    type Book { id: ID! title: String! stock: Int! author: Author! }
    type Author { id: ID! name: String! }
  `,
  resolvers: {
    Query: { books: () => books, book: (_, { id }) => books.find((b) => b.id === id) },
    Mutation: { restock: (_, { id, amount }) => { const b = books.find((x) => x.id === id); b.stock += amount; return b; } },
    Book: { author: (b) => authors.find((a) => a.id === b.authorId) },
  },
  plugins: [{ async requestDidStart() { stats.requests++; } }],
});
export const { url } = await startStandaloneServer(server, { listen: { port: 4313 } });
export const stop = () => server.stop();
```

The plugin counts HTTP requests that reach the server, so we can see when the client used its
cache instead.

## A plain `fetch` is a complete client

```js title="client07.mjs"
import { ApolloClient, InMemoryCache, HttpLink, gql } from "@apollo/client";
import { url, stats, stop } from "./api07.mjs";

// --- 1. Plain fetch: no library needed
const res = await fetch(url, {
  method: "POST",
  headers: { "content-type": "application/json" },
  body: JSON.stringify({ query: "query One($id: ID!) { book(id: $id) { title } }", variables: { id: "1" } }),
});
console.log("fetch:", JSON.stringify(await res.json()));

// --- 2. Apollo Client with a normalized cache
const client = new ApolloClient({ link: new HttpLink({ uri: url }), cache: new InMemoryCache() });
stats.requests = 0;

const LIST = gql`query List { books { id title stock author { id name } } }`;
const ONE = gql`query One($id: ID!) { book(id: $id) { id title stock } }`;

await client.query({ query: LIST });
console.log("\nafter List, requests:", stats.requests);
console.log("cache keys:", Object.keys(client.cache.extract()).join(", "));
console.log("Book:1 entry:", JSON.stringify(client.cache.extract()["Book:1"]));

// Same query again: served from cache
await client.query({ query: LIST });
console.log("List again, requests:", stats.requests);

// A *different* query for data already cached: still a network request by default...
const one = await client.query({ query: ONE, variables: { id: "1" } });
console.log("One(1) (default cache-first):", JSON.stringify(one.data.book), "requests:", stats.requests);

// --- 3. Mutation results update every query that shows the same object
await client.mutate({ mutation: gql`mutation { restock(id: "2", amount: 5) { id stock } }` });
const fromCache = client.readQuery({ query: LIST });
console.log("\nafter restock (no refetch), List from cache shows Messiah stock:",
  fromCache.books.find((b) => b.id === "2").stock, "| requests:", stats.requests);

// --- 4. Without an id, the mutation result can't be matched to the cached object
await client.mutate({ mutation: gql`mutation { restock(id: "1", amount: 1) { stock } }` });
console.log("restock without id -> cached Dune stock:", client.readQuery({ query: LIST }).books[0].stock,
  "| server says:", (await client.query({ query: ONE, variables: { id: "1" }, fetchPolicy: "network-only" })).data.book.stock);

// --- 5. Reading a fragment straight from the cache
const author = client.readFragment({ id: "Author:a1", fragment: gql`fragment A on Author { name }` });
console.log("\nreadFragment Author:a1:", JSON.stringify(author));

// --- 6. A cache redirect: teach the cache that book(id:) means Book:<id>
const client2 = new ApolloClient({
  link: new HttpLink({ uri: url }),
  cache: new InMemoryCache({
    typePolicies: {
      Query: {
        fields: {
          book: { read: (_, { args, toReference }) => toReference({ __typename: "Book", id: args.id }) },
        },
      },
    },
  }),
});
await client2.query({ query: LIST });
const before = stats.requests;
const one2 = await client2.query({ query: ONE, variables: { id: "2" } });
console.log("\nwith cache redirect, One(2):", JSON.stringify(one2.data.book), "| new requests:", stats.requests - before);

await stop();
```

The first block needs nothing but the platform:

```text
fetch: {"data":{"book":{"title":"Dune"}}}
```

For scripts, server-to-server calls and simple pages, this is often all you need. Add a typed
wrapper with code generation ([lesson 08](08-codegen.md)) and it's production-grade. What it
doesn't give you is memory: every call goes to the network, and two parts of a UI showing the
same book can disagree after one of them updates it.

## Apollo Client's normalized cache

The output from the remaining sections, in order:

```text
after List, requests: 1
cache keys: Author:a1, Book:1, Book:2, ROOT_QUERY
Book:1 entry: {"__typename":"Book","id":"1","title":"Dune","stock":3,"author":{"__ref":"Author:a1"}}
List again, requests: 1
One(1) (default cache-first): {"__typename":"Book","id":"1","title":"Dune","stock":3} requests: 2

after restock (no refetch), List from cache shows Messiah stock: 5 | requests: 3
restock without id -> cached Dune stock: 3 | server says: 4

readFragment Author:a1: {"__typename":"Author","name":"Frank Herbert"}

with cache redirect, One(2): {"__typename":"Book","id":"2","title":"Dune Messiah","stock":5} | new requests: 0
```

### Normalization

After one `List` query, the cache doesn't hold "the result of List". It holds four entries:
`Book:1`, `Book:2`, `Author:a1`, and `ROOT_QUERY`, which records that the root field `books`
points at `[Book:1, Book:2]`. Each object is stored **once**, keyed by `__typename:id`, and
references to other objects are stored as `{"__ref": "Author:a1"}`. Apollo Client adds
`__typename` to every selection set automatically to build these keys — another reason
`__typename` matters ([Level 2 · 02](../level-2/02-interfaces-unions.md)).

### Cache hits are per field, not per object

Running `List` again cost **no** request: every field it needs is in the cache.

Running `One(id: "1")` **did** cost a request, even though `Book:1` with `id`, `title` and `stock`
was already cached. The cache had no record of what the root field `book(id: "1")` returns —
only `books` had been queried. Section 6 fixes that with a **cache redirect**: a `read`
function on `Query.book` that tells the cache "`book(id: X)` is `Book:X`". With it, `One(2)` was
answered entirely from cache — zero new requests.

### Mutations update every view

`restock(id: "2")` returned `{ id stock }`. Because the result included `__typename` and `id`,
the cache merged the new `stock` into the existing `Book:2` entry — and the cached `List` result,
which references `Book:2`, now shows 5 without being refetched. This is why mutation payloads
should return the changed objects with their ids ([Level 2 · 08](../level-2/08-mutation-design.md)).

The next mutation omitted `id`. The server updated Dune's stock to 4, but the cache couldn't tell
which object the result belonged to, so the cached `List` still said 3 — a stale UI. Lists that
gain or lose items are similar: a cache can't know that a new review belongs in some list, so
mutations that add or remove items need `refetchQueries` or a manual `cache.modify`.

### Reading fragments

`readFragment({ id: "Author:a1", … })` reads one normalized object directly. UI components that
declare their data needs as fragments use this to render from cache.

## Fetch policies

`client.query` defaults to `cache-first`: use the cache when it can satisfy every field, otherwise
go to the network. Others you'll use:

| Policy | Behaviour |
|---|---|
| `cache-first` | cache if complete, else network (default for `query`) |
| `network-only` | always network, then write to cache (used above to check the server's value) |
| `cache-and-network` | return cache immediately, then update from network (watched queries) |
| `no-cache` | network, don't write to cache |
| `cache-only` | never network |

## Choosing a client

- **`fetch` / `graphql-request`-style thin clients** — scripts, servers, simple pages.
- **Apollo Client** — normalized cache, React integration, widely used. Version 4 reorganised the
  package (React hooks live in `@apollo/client/react`; `rxjs` is a peer dependency).
- **urql** — smaller, document cache by default with an optional normalized cache ("Graphcache").
- **Relay** — compiler-driven, fragment-colocation-first, and strict about server conventions
  (global `Node` ids, connections). Very efficient for large apps that follow its rules.

All of them reward the same server design: stable ids, `__typename`, connections for lists,
and mutations that return what they change.

## How It Actually Works

When a result arrives, `InMemoryCache` walks it alongside the query document. For each object
it computes an id with `dataIdFromObject` — by default `${__typename}:${id}` (or `_id`), or a
`keyFields` policy you configure per type. Objects with an id are written to a flat map under
that key and replaced in their parent by a reference; objects without one are stored inline in
their parent. Writes **merge** fields into existing entries rather than replacing them, which is
how a mutation returning only `{ id stock }` updated one field of `Book:2`.

Reads walk the query document again, following references. If any requested field is missing,
the read is incomplete and a `cache-first` query goes to the network. Root fields with arguments
are stored under keys that include the arguments (`book({"id":"1"})`), which is why the cache
couldn't connect `book(id: "1")` to `Book:1` until the `read` function supplied that mapping.
Watched queries (used by UI hooks) subscribe to the entries they read and re-render when a write
touches any of them.

## Common mistakes

- **Omitting `id` from selections** — objects can't be normalized and mutations can't update them.
- **Ids that aren't unique per type** (or are reused across types without `__typename`), so
  different objects overwrite each other.
- **Expecting list membership to update automatically** after create/delete mutations.
- **Leaving the default `cache-first` on data that must be fresh** (balances, stock).
- **Disabling `__typename`** to "save bytes", which breaks normalization and union handling.

## Exercise

1. Add `restock`'s result to a list query in another client instance and observe that separate
   `ApolloClient` instances don't share a cache.
2. Add `addBook(title: String!): Book!` and update `List` in the cache after the mutation with
   `cache.modify`, without refetching.
3. Give `Author` a `keyFields: ["name"]` policy and explain what changes in `cache.extract()`.
4. Use `client.watchQuery` on `List`, subscribe to it, run the restock mutation, and log each
   emission.
