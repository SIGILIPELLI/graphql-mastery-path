---
description: "Stop GraphQL errors leaking internals: Apollo Server defaults, masking with formatError and error ids, logging, and Yoga's defaults."
---

# 06 · Errors, Masking & Logging in Production

Earlier lessons kept noticing the same risk: a database driver's message — table names, hosts,
usernames — travelling straight into a client response. This lesson fixes it properly. It
compares what Apollo Server 5.5.1 sends by default with a masking `formatError` and with GraphQL
Yoga 5.24.4's defaults, using one query that triggers the three kinds of error every API has:
an expected error, an infrastructure failure, and a plain bug.

## The test

```js title="errors06.mjs"
import { randomUUID } from "node:crypto";
import { ApolloServer } from "@apollo/server";
import { unwrapResolverError } from "@apollo/server/errors";
import { GraphQLError } from "graphql";
import { createYoga, createSchema } from "graphql-yoga";

const typeDefs = `type Query { ok: String, notFound: String, dbDown: String, bug: String }`;
const resolvers = {
  Query: {
    ok: () => "fine",
    notFound: () => { throw new GraphQLError("Order 42 not found", { extensions: { code: "NOT_FOUND" } }); },
    dbDown: () => { throw new Error('connect ECONNREFUSED 10.0.3.17:5432 (user "orders_rw")'); },
    bug: () => { const order = undefined; return order.total; },
  },
};
const QUERY = "{ ok notFound dbDown bug }";

// 1. Apollo Server defaults
const plain = new ApolloServer({ typeDefs, resolvers, includeStacktraceInErrorResponses: false });
const r1 = await plain.executeOperation({ query: QUERY });
console.log("== Apollo, default formatting");
for (const e of r1.body.singleResult.errors) console.log(" ", JSON.stringify(e));

// 2. Apollo Server with masking: expected errors pass through, everything else is replaced
const SAFE_CODES = new Set(["NOT_FOUND", "BAD_USER_INPUT", "UNAUTHENTICATED", "FORBIDDEN",
  "GRAPHQL_VALIDATION_FAILED", "GRAPHQL_PARSE_FAILED", "BAD_REQUEST", "PERSISTED_QUERY_NOT_FOUND"]);
const logged = [];
const masked = new ApolloServer({
  typeDefs, resolvers, includeStacktraceInErrorResponses: false,
  formatError(formatted, error) {
    if (SAFE_CODES.has(formatted.extensions?.code)) return formatted;
    const original = unwrapResolverError(error);
    const errorId = randomUUID().slice(0, 8);
    logged.push({ errorId, path: formatted.path?.join("."), message: original.message, stack: original.stack?.split("\n")[1]?.trim() });
    return { message: "Internal server error", path: formatted.path, extensions: { code: "INTERNAL_SERVER_ERROR", errorId } };
  },
});
const r2 = await masked.executeOperation({ query: QUERY });
console.log("\n== Apollo with formatError masking — client sees");
for (const e of r2.body.singleResult.errors) console.log(" ", JSON.stringify(e));
console.log("data still returned:", JSON.stringify(r2.body.singleResult.data));
console.log("server log:");
for (const l of logged) console.log(" ", JSON.stringify(l));

// 3. GraphQL Yoga masks unexpected errors by default
const yoga = createYoga({ schema: createSchema({ typeDefs, resolvers }), logging: false });
const res = await yoga.fetch("http://yoga/graphql", { method: "POST", headers: { "content-type": "application/json" }, body: JSON.stringify({ query: QUERY }) });
console.log("\n== Yoga default");
for (const e of (await res.json()).errors) console.log(" ", JSON.stringify(e));
```

- `notFound` throws a `GraphQLError` with a code — an **expected** error the client should see.
- `dbDown` throws the kind of message a PostgreSQL client produces when it can't connect — an
  **infrastructure** failure.
- `bug` dereferences `undefined` — a **programming error**.

## Apollo Server's default

```text
== Apollo, default formatting
  {"message":"Order 42 not found",...,"extensions":{"code":"NOT_FOUND"}}
  {"message":"connect ECONNREFUSED 10.0.3.17:5432 (user \"orders_rw\")",...,"extensions":{"code":"INTERNAL_SERVER_ERROR"}}
  {"message":"Cannot read properties of undefined (reading 'total')",...,"extensions":{"code":"INTERNAL_SERVER_ERROR"}}
```

(`locations` and `path` elided.) Stack traces were already off
(`includeStacktraceInErrorResponses: false`), and Apollo labelled the unexpected errors
`INTERNAL_SERVER_ERROR`. But the **messages** went out verbatim: an internal IP address, a
database port and a database username, plus a hint at the server's code structure. Turning off
stack traces is not enough.

## Masking with `formatError`

`formatError(formattedError, error)` runs for every error just before it's sent. It receives the
already-formatted JSON shape and the original error, and returns what the client should see. The
policy in the script:

1. **Allow-list** expected errors by `extensions.code` and pass them through unchanged.
2. For everything else, **unwrap** the original error (`unwrapResolverError` removes the
   `GraphQLError` wrapper the executor added), **log it** with a short random id, and send the
   client only a generic message plus that id.

```text
== Apollo with formatError masking — client sees
  {"message":"Order 42 not found","locations":[{"line":1,"column":6}],"path":["notFound"],"extensions":{"code":"NOT_FOUND"}}
  {"message":"Internal server error","path":["dbDown"],"extensions":{"code":"INTERNAL_SERVER_ERROR","errorId":"4c243f3f"}}
  {"message":"Internal server error","path":["bug"],"extensions":{"code":"INTERNAL_SERVER_ERROR","errorId":"498a82d6"}}
data still returned: {"ok":"fine","notFound":null,"dbDown":null,"bug":null}
server log:
  {"errorId":"4c243f3f","path":"dbDown","message":"connect ECONNREFUSED 10.0.3.17:5432 (user \"orders_rw\")","stack":"at Object.dbDown (file:///…/errors06.mjs:12:27)"}
  {"errorId":"498a82d6","path":"bug","message":"Cannot read properties of undefined (reading 'total')","stack":"at Object.bug (file:///…/errors06.mjs:13:56)"}
```

(File paths in the log shortened; the ids are random per run.)

- The client keeps everything useful: the partial data, the `path` of each failure, the
  `NOT_FOUND` message, and an `errorId` support staff can search for.
- The server log keeps everything sensitive: the real message and where it was thrown.
- **Allow-listing** is the important design choice. A deny-list ("hide messages that look like
  SQL") fails open the first time a new library throws something unexpected; an allow-list fails
  closed.

Validation and parse errors are in the allow-list because clients need those messages to fix
their queries; they describe the client's document, not your internals.

## Yoga's default

```text
== Yoga default
  {"message":"Order 42 not found",...,"extensions":{"code":"NOT_FOUND"}}
  {"message":"Unexpected error.",...,"extensions":{"code":"INTERNAL_SERVER_ERROR"}}
  {"message":"Unexpected error.",...,"extensions":{"code":"INTERNAL_SERVER_ERROR"}}
```

GraphQL Yoga **masks by default**: any error that isn't a `GraphQLError` you threw deliberately
becomes "Unexpected error." That's the safer default — you opt *out* of masking rather than
remembering to opt in. Its `maskedErrors` option accepts a custom `maskError` function for the
same allow-list-plus-id policy.

One caveat from reading Yoga's `maskError` source: when `NODE_ENV` is exactly `development`, the
masked error gets an `extensions.originalError` containing the real message **and stack**. That's
convenient locally and a leak if a production deployment ever runs with that setting — check
your environment variables, or supply your own `maskError`.

## Logging that helps at 3 a.m.

A useful error log line contains: the error id sent to the client, the operation name, the field
`path`, the client name/version, the request id (also returned in a response header), the original
message and stack — and **not** the full variables, which may contain passwords or personal data.
In Apollo Server, the natural place is a plugin's `didEncounterErrors` hook, which has the whole
request context (operation name, client headers) alongside the errors; `formatError` then only
needs to do the masking. Keep error ids short enough to read over the phone.

Also decide what counts as an *alert*. A burst of `NOT_FOUND` is usually normal; any
`INTERNAL_SERVER_ERROR` usually isn't. Codes make that split easy.

## How It Actually Works

When a resolver throws, the executor wraps the thrown value with `locatedError`, producing a
`GraphQLError` whose `originalError` is your error and which carries `path` and `locations`. Apollo
Server then normalises every error with `ensureGraphQLError`, assigns `INTERNAL_SERVER_ERROR` to
errors without a code, and builds the JSON shape (`message`, `locations`, `path`, `extensions`)
via `toJSON`. Only after that does it call your `formatError(formattedError, error)` — with the
already-serialisable object first and the original `GraphQLError` second. Whatever you return is
sent as-is. `unwrapResolverError(error)` returns `error.originalError` when the error came from a
resolver, so you see the `Error` that was actually thrown.

Yoga applies masking in its result-processing stage. Its default `maskError` keeps an error
unchanged if it is an "original" `GraphQLError` — one you constructed and threw yourself, rather
than a wrapper around some other thrown value. Anything else is replaced by a new `GraphQLError`
with the generic message, `code: "INTERNAL_SERVER_ERROR"`, the original `path` and locations, and
(only in development mode) the serialised original error.

## Common mistakes

- **Assuming `includeStacktraceInErrorResponses: false` is enough.** Messages leak too.
- **Deny-lists** for masking instead of allow-lists.
- **Masking validation errors**, leaving client developers unable to fix queries.
- **Losing the original error** — masking without logging makes failures undebuggable.
- **Logging full variables** and creating a second leak in your log storage.
- **Changing error codes casually** — clients switch on them; treat them as API.

## Exercise

1. Move the logging into a `didEncounterErrors` plugin that includes the operation name and the
   `apollographql-client-name` header, and keep `formatError` for masking only.
2. Return the request id in an `x-request-id` header and include it in every log line.
3. Configure Yoga's `maskedErrors: { maskError }` to produce the same `errorId` responses as the
   Apollo version.
4. Write a test that fails if any response ever contains the substring `ECONNREFUSED` or `SELECT`.
