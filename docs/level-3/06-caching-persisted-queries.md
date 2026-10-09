---
description: "Make GraphQL cacheable: @cacheControl hints, Cache-Control headers, GET requests, automatic persisted queries and trusted documents."
---

# 06 · Caching & Persisted Queries

"GraphQL can't be cached" is a common complaint, and it's half true. Every request is a
`POST` to the same URL with a different body, which defeats HTTP caches and CDNs by default.
But nothing forces that: Apollo Server can compute a `Cache-Control` header from per-field
hints, queries can be sent as `GET`, and **persisted queries** shrink those GET URLs to a
hash. The same mechanism, run in reverse, lets you refuse every operation your own apps didn't
ship — the strongest security control from [lesson 05](05-security-limits.md). Everything here
was run against Apollo Server 5.5.1.

## Cache hints with `@cacheControl`

Apollo Server's built-in cache-control plugin reads a `@cacheControl` directive. You declare it
in your SDL (it isn't built in to GraphQL):

```js title="cache06.mjs"
import { createHash } from "node:crypto";
import { ApolloServer } from "@apollo/server";
import { startStandaloneServer } from "@apollo/server/standalone";

const typeDefs = /* GraphQL */ `
  enum CacheControlScope { PUBLIC PRIVATE }
  directive @cacheControl(maxAge: Int, scope: CacheControlScope, inheritMaxAge: Boolean)
    on FIELD_DEFINITION | OBJECT | INTERFACE | UNION

  type Query {
    catalog: [Book!]! @cacheControl(maxAge: 300)
    bestseller: Book @cacheControl(maxAge: 60)
    viewer: User @cacheControl(maxAge: 0, scope: PRIVATE)
  }
  type Book @cacheControl(maxAge: 600) {
    title: String!
    stock: Int! @cacheControl(maxAge: 10)
  }
  type User { name: String! }
`;
const resolvers = {
  Query: {
    catalog: () => [{ title: "Dune", stock: 3 }, { title: "Emma", stock: 0 }],
    bestseller: () => ({ title: "Dune", stock: 3 }),
    viewer: () => ({ name: "Ada" }),
  },
};

const server = new ApolloServer({ typeDefs, resolvers, includeStacktraceInErrorResponses: false });
const { url } = await startStandaloneServer(server, { listen: { port: 4312 } });

const sha256 = (s) => createHash("sha256").update(s).digest("hex");
const show = async (label, res) =>
  console.log(`--- ${label}\n${res.status} cache-control: ${res.headers.get("cache-control") ?? "(none)"}\n${(await res.text()).trim()}`);
const post = (body) => fetch(url, { method: "POST", headers: { "content-type": "application/json" }, body: JSON.stringify(body) });
const get = (params) => fetch(`${url}?${new URLSearchParams(Object.entries(params).map(([k, v]) => [k, typeof v === "string" ? v : JSON.stringify(v)]))}`,
  { headers: { "apollo-require-preflight": "true" } });

console.log("== cache hints");
await show("catalog titles (POST)", await post({ query: "{ catalog { title } }" }));
await show("catalog titles (GET)", await get({ query: "{ catalog { title } }" }));
await show("catalog with stock (GET)", await get({ query: "{ catalog { title stock } }" }));
await show("bestseller title (GET)", await get({ query: "{ bestseller { title } }" }));
await show("viewer + catalog (GET)", await get({ query: "{ viewer { name } catalog { title } }" }));

console.log("\n== automatic persisted queries");
const query = "query Bestseller { bestseller { title } }";
const ext = { persistedQuery: { version: 1, sha256Hash: sha256(query) } };
console.log("hash:", ext.persistedQuery.sha256Hash);
await show("1. hash only (unknown)", await get({ extensions: ext }));
await show("2. hash + query (registers it)", await post({ query, extensions: ext }));
await show("3. hash only, now known", await get({ extensions: ext }));
await show("4. hash that doesn't match the query", await post({ query: "{ viewer { name } }", extensions: ext }));
await server.stop();
```

The policy for a whole response is the **most restrictive** hint of every field in it:

```text
== cache hints
--- catalog titles (POST)
200 cache-control: max-age=300, public
--- catalog titles (GET)
200 cache-control: max-age=300, public
--- catalog with stock (GET)
200 cache-control: max-age=10, public
--- bestseller title (GET)
200 cache-control: max-age=60, public
--- viewer + catalog (GET)
200 cache-control: no-store
```

(Bodies omitted here; see the full output when you run it.)

- `catalog { title }`: the field says 300 s, the `Book` type says 600 s, the minimum wins — 300.
- Selecting `stock`, a fast-changing field with 10 s, drops the whole response to 10 s. Clients
  that don't need stock shouldn't ask for it, and now they're rewarded for that.
- `viewer` is `PRIVATE` with `maxAge: 0`, so any response containing it gets `no-store`, even
  though `catalog` alone would be cacheable for 5 minutes. That's the safety property you
  need: personal data can't end up in a shared cache by being bundled with public data.

Some rules from Apollo's documented behaviour worth knowing: root fields, and fields returning
object/interface/union types, default to `maxAge: 0` unless a hint applies; scalar fields
inherit their parent's policy. A single unhinted object field therefore makes a response
uncacheable — the safe default.

The header was sent on the `POST` too, but HTTP caches and CDNs don't generally cache POST
responses. To use a CDN, clients send queries as `GET`.

## Automatic persisted queries (APQ)

A GET URL containing a whole query is long and can exceed URL limits. **APQ** replaces the text
with its SHA-256 hash. Apollo Server supports it out of the box with an in-memory store:

```text
== automatic persisted queries
hash: c50e18d85c234f3d3c770131b349a903226cfa598a05fd675efad3d3a7e3df1d
--- 1. hash only (unknown)
200 cache-control: private, no-cache, must-revalidate
{"errors":[{"message":"PersistedQueryNotFound","extensions":{"code":"PERSISTED_QUERY_NOT_FOUND"}}]}
--- 2. hash + query (registers it)
200 cache-control: max-age=60, public
{"data":{"bestseller":{"title":"Dune"}}}
--- 3. hash only, now known
200 cache-control: max-age=60, public
{"data":{"bestseller":{"title":"Dune"}}}
--- 4. hash that doesn't match the query
400 cache-control: no-store
{"errors":[{"message":"provided sha does not match query","extensions":{"code":"INTERNAL_SERVER_ERROR"}}]}
```

The protocol:

1. The client optimistically sends only `extensions.persistedQuery.sha256Hash`, as a GET.
2. If the server doesn't know the hash, it answers `PERSISTED_QUERY_NOT_FOUND` — explicitly
   marked uncacheable so a CDN doesn't remember the miss.
3. The client retries with the hash **and** the full query. The server verifies the hash and
   stores the text.
4. From then on, the short hash-only GET works — for every client — and is cacheable by URL.

Step 4 in the output shows the server refuses a hash that doesn't match the text, so nobody can
poison the store with a different query under a known hash. (The `INTERNAL_SERVER_ERROR` code
on that 400 is what Apollo Server returned; it's a client error in practice.)

APQ's store is in memory per server instance by default; with several instances, configure a
shared cache (Redis, Memcached) via the `persistedQueries.cache` option, or each instance learns
hashes separately.

## Trusted documents: only run what you shipped

APQ registers *any* query a client sends. For an API used only by your own apps, flip it around:
the app's build step extracts every operation, computes the hashes, and produces a manifest;
the server runs only operations in that manifest. This is variously called **trusted documents**,
**persisted operations** or an operation **safelist**.

```js title="trusted06.mjs"
import { createHash } from "node:crypto";
import { ApolloServer } from "@apollo/server";
import { GraphQLError } from "graphql";

const sha256 = (s) => createHash("sha256").update(s).digest("hex");

// Built by the client's build step from every operation in the app's source.
const appOperations = [
  "query Catalog { catalog { title } }",
  "query Bestseller { bestseller { title } }",
];
const manifest = new Map(appOperations.map((q) => [sha256(q), q]));

// A read-only "cache" pre-filled from the manifest; Apollo prefixes APQ keys with "apq:".
const trustedStore = {
  async get(key) { return manifest.get(key.replace(/^apq:/, "")); },
  async set() { /* never register new operations at runtime */ },
  async delete() {},
};

const server = new ApolloServer({
  typeDefs: `type Query { catalog: [Book!]! bestseller: Book } type Book { title: String! }`,
  resolvers: { Query: { catalog: () => [{ title: "Dune" }], bestseller: () => ({ title: "Dune" }) } },
  persistedQueries: { cache: trustedStore },
  includeStacktraceInErrorResponses: false,
  plugins: [{
    async requestDidStart() {
      return {
        async didResolveOperation({ source }) {
          if (!manifest.has(sha256(source)))
            throw new GraphQLError("Only registered operations are accepted", { extensions: { code: "OPERATION_NOT_ALLOWED", http: { status: 403 } } });
        },
      };
    },
  }],
});
await server.start();

const run = async (label, request) => {
  const res = await server.executeOperation(request);
  console.log(`${label.padEnd(38)} ${JSON.stringify(res.body.singleResult)}`);
};
const id = (q) => ({ extensions: { persistedQuery: { version: 1, sha256Hash: sha256(q) } } });

await run("registered op by hash only", id(appOperations[0]));
await run("registered op sent as full text", { query: appOperations[1] });
await run("attacker's own query", { query: "{ catalog { title } bestseller { title } }" });
await run("attacker's query + its own hash", { query: "{ bestseller { title } }", ...id("{ bestseller { title } }") });
await server.stop();
```

```console
$ node trusted06.mjs
registered op by hash only             {"data":{"catalog":[{"title":"Dune"}]}}
registered op sent as full text        {"data":{"bestseller":{"title":"Dune"}}}
attacker's own query                   {"errors":[{"message":"Only registered operations are accepted","extensions":{"code":"OPERATION_NOT_ALLOWED"}}]}
attacker's query + its own hash        {"errors":[{"message":"Only registered operations are accepted","extensions":{"code":"OPERATION_NOT_ALLOWED"}}]}
```

The read-only store makes APQ-by-hash work for manifest entries and never learns new ones; the
plugin rejects any operation text whose hash isn't in the manifest, however it arrived. Depth
attacks, alias attacks and schema probing all stop here, because the attacker can only run
queries you wrote.

A detail found while building this: the first version threw the error from the
`didResolveSource` hook, and Apollo Server treated it as an *unexpected* failure —
`executeOperation` rejected with `Internal server error` instead of returning a GraphQL error.
Throwing from `didResolveOperation` (as the cost plugin did in lesson 05) produces a proper
error response. The trade-off is that `didResolveOperation` runs after validation, so an
unregistered query that is also invalid gets validation errors rather than the allowlist error;
if that leak of schema information matters, reject unknown operations earlier, in your HTTP
layer, before calling Apollo.

Trusted documents fit first-party clients. Public APIs, where third parties write their own
queries, still need cost limits.

## Other caching layers

- **Client caches** — Apollo Client's normalized cache ([lesson 07](07-apollo-client-cache.md))
  avoids refetching data the client already has. That's where most "GraphQL caching" happens.
- **Whole-response server caches** — Apollo's `@apollo/server-plugin-response-cache` stores
  full responses keyed by query, variables and (for private data) session, honouring the same
  hints. Not run here.
- **Data-layer caches** — per-request DataLoaders ([Level 2 · 06](../level-2/06-dataloader.md)),
  and shared caches (Redis) in front of slow backends, independent of GraphQL.

## How It Actually Works

The cache-control plugin installs itself when the server starts. In `executionDidStart` it adds
a `willResolveField` hook: for each field it computes a hint from the field's directive, the
return type's directive, and the inheritance rules, and merges it into a request-wide policy
with `restrict()` — taking the minimum `maxAge` and `PRIVATE` over `PUBLIC`. Resolvers can also
call `info.cacheControl.setCacheHint(...)` for dynamic hints. In `willSendResponse`, if the
response has no errors and the policy's `maxAge > 0`, it writes `Cache-Control: max-age=N,
public|private`; otherwise it writes `no-store`.

APQ processing happens at the start of the request pipeline: if `extensions.persistedQuery` is
present, the server looks up `apq:<hash>` in its key-value cache. With no query text and a miss,
it returns `PERSISTED_QUERY_NOT_FOUND`; with query text, it computes the SHA-256, compares, and
stores the text after the operation validates successfully. The rest of the pipeline then sees
an ordinary query string.

## Common mistakes

- **Expecting CDNs to cache POST requests.**
- **Forgetting `scope: PRIVATE` on viewer-specific data** — the one mistake here that leaks
  data between users through a shared cache.
- **Long `maxAge` on fields whose source changes often**, with no way to purge.
- **Per-instance APQ stores** behind a load balancer, causing repeated misses.
- **Using APQ as a security measure.** It registers whatever clients send; only a fixed manifest
  restricts operations.

## Exercise

1. Add a resolver-level dynamic hint: `bestseller` should be cacheable for 300 s on weekends and
   60 s on weekdays, using `info.cacheControl.setCacheHint`.
2. Send the APQ flow with `curl` only (GET with an `extensions` URL parameter), and confirm the
   miss is not cacheable.
3. Write a script that scans `.graphql` files in a folder and writes the trusted-documents
   manifest as JSON (`{ hash: query }`). Load it in `trusted06.mjs`.
4. Move the allowlist check into the HTTP layer (before Apollo) so that unregistered invalid
   queries get the allowlist error rather than validation errors.
