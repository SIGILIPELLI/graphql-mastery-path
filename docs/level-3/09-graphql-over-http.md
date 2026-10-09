---
description: "GraphQL over HTTP in practice: Accept headers, graphql-response+json, status codes compared in Apollo and Yoga, uploads and CSRF."
---

# 09 · GraphQL over HTTP in Depth: Status Codes, CSRF & Uploads

The GraphQL specification deliberately says nothing about transport. For years every server
picked its own HTTP conventions, and clients coped. The GraphQL Foundation's
**GraphQL-over-HTTP** specification (a working draft at the time of writing, already
implemented by major servers) pins those conventions down. This lesson tests two popular
servers against it — Apollo Server 5.5.1 and GraphQL Yoga 5.24.4 — and finds they really do
behave differently, then looks at file uploads, the one HTTP feature that reopens a security
hole the previous lessons closed.

## Media types and the `Accept` header

The spec defines two response media types:

- **`application/json`** — the legacy type every client understands.
- **`application/graphql-response+json`** — a newer type whose status codes carry meaning:
  a response that never reached execution must use a 4xx/5xx status, so proxies, CDNs and
  monitoring can tell a broken request from a successful one without parsing the body.

Clients state which they understand in `Accept`; servers answer with one of them.

## Status codes, measured

```js title="http09.mjs"
import { createServer } from "node:http";
import { createYoga, createSchema } from "graphql-yoga";
import { ApolloServer } from "@apollo/server";
import { startStandaloneServer } from "@apollo/server/standalone";
import { GraphQLError } from "graphql";

const typeDefs = `type Query { ok: String! boom: String } type Mutation { touch: Boolean! }`;
const resolvers = {
  Query: { ok: () => "fine", boom: () => { throw new GraphQLError("resolver failed"); } },
  Mutation: { touch: () => true },
};

const yoga = createYoga({ schema: createSchema({ typeDefs, resolvers }), maskedErrors: false, logging: false });
const yogaServer = createServer(yoga);
await new Promise((r) => yogaServer.listen(4315, r));
const apollo = new ApolloServer({ typeDefs, resolvers, includeStacktraceInErrorResponses: false });
const { url: apolloUrl } = await startStandaloneServer(apollo, { listen: { port: 4316 } });
const servers = { "Apollo 5.5.1": apolloUrl, "Yoga 5.24.4": "http://localhost:4315/graphql" };

const JSON_ = "application/json";
const GQLRESP = "application/graphql-response+json";
const cases = [
  ["valid query", "POST", { query: "{ ok }" }],
  ["field error", "POST", { query: "{ ok boom }" }],
  ["parse error", "POST", { query: "{ ok " }],
  ["validation error", "POST", { query: "{ nope }" }],
  ["bad variables", "POST", { query: "query($n: Int!) { ok }", variables: { n: "x" } }],
  ["mutation via GET", "GET", { query: "mutation { touch }" }],
];

for (const [name, base] of Object.entries(servers)) {
  console.log(`\n== ${name}`);
  console.log("case".padEnd(18), "Accept: json".padEnd(40), "Accept: graphql-response+json");
  for (const [label, method, body] of cases) {
    const cells = [];
    for (const accept of [JSON_, GQLRESP]) {
      const headers = { accept, "apollo-require-preflight": "1" };
      let res;
      if (method === "POST") res = await fetch(base, { method, headers: { ...headers, "content-type": JSON_ }, body: JSON.stringify(body) });
      else res = await fetch(`${base}?${new URLSearchParams({ query: body.query })}`, { headers });
      const ct = (res.headers.get("content-type") ?? "").split(";")[0].replace("application/", "");
      const j = await res.json().catch(() => ({}));
      cells.push(`${res.status} ${ct} ${"data" in j ? "data" : "no-data"}`.padEnd(40));
    }
    console.log(label.padEnd(18), ...cells);
  }
}
yogaServer.close(); await apollo.stop();
```

```console
$ node http09.mjs

== Apollo 5.5.1
case               Accept: json                             Accept: graphql-response+json
valid query        200 json data                            200 graphql-response+json data
field error        200 json data                            200 graphql-response+json data
parse error        400 json no-data                         400 graphql-response+json no-data
validation error   400 json no-data                         400 graphql-response+json no-data
bad variables      400 json no-data                         400 graphql-response+json no-data
mutation via GET   405 json no-data                         405 graphql-response+json no-data

== Yoga 5.24.4
case               Accept: json                             Accept: graphql-response+json
valid query        200 json data                            200 graphql-response+json data
field error        200 json data                            200 graphql-response+json data
parse error        200 json no-data                         400 graphql-response+json no-data
validation error   200 json no-data                         400 graphql-response+json no-data
bad variables      200 json no-data                         400 graphql-response+json no-data
mutation via GET   405 json no-data                         405 graphql-response+json no-data
```

What the table shows:

- Both servers negotiate the media type and echo it back.
- **Field errors are always 200**, with `data`. Execution happened, partial data exists, and the
  spec treats that as a successful HTTP exchange — the details are in the body.
- **Request errors** (parse, validation, variables) never produce `data`. With
  `graphql-response+json`, both servers return 400.
- With plain **`application/json`**, they disagree. Yoga returns **200** for request errors —
  the spec's recommended behaviour for the legacy type, because older clients treated any
  non-200 as a network failure and never read the error body. Apollo returns **400** either way.
- A mutation over GET is 405 in both, regardless of `Accept`.

The practical upshot for clients: **send `Accept: application/graphql-response+json,
application/json`**, and decide success from the body (`data` present?) rather than the status
alone. For monitoring, the new media type makes status codes trustworthy.

## Request shape rules worth knowing

From the spec, all observed in earlier lessons:

- `POST` with `Content-Type: application/json` and a body of `{ query, operationName?,
  variables?, extensions? }` must be supported.
- `GET` with the same keys as URL parameters (`variables`/`extensions` JSON-encoded) may be
  supported, for queries only.
- Unknown keys are ignored; an invalid body is a 400.

## File uploads

GraphQL has no file type. The popular workaround is the **GraphQL multipart request** convention
(a community spec, not part of GraphQL-over-HTTP): a `multipart/form-data` POST with an
`operations` part (the usual JSON body with `null` where each file goes), a `map` part saying which file
goes into which variable, and the files themselves.

```js title="upload09.mjs"
import { createServer } from "node:http";
import { createYoga, createSchema } from "graphql-yoga";
import { ApolloServer } from "@apollo/server";
import { startStandaloneServer } from "@apollo/server/standalone";

const typeDefs = `scalar File type Query { ok: Boolean } type Mutation { upload(file: File!): String! }`;
const resolvers = { Mutation: { upload: async (_, { file }) => `${file.name} ${file.type} ${(await file.text()).length} bytes` } };
const yoga = createYoga({ schema: createSchema({ typeDefs, resolvers }), logging: false });
const ys = createServer(yoga); await new Promise((r) => ys.listen(4317, r));

// GraphQL multipart request: "operations" + "map" + the file parts
function multipart() {
  const form = new FormData();
  form.append("operations", JSON.stringify({ query: "mutation($f: File!) { upload(file: $f) }", variables: { f: null } }));
  form.append("map", JSON.stringify({ 0: ["variables.f"] }));
  form.append("0", new Blob(["hello, uploads"], { type: "text/plain" }), "note.txt");
  return form;
}
let r = await fetch("http://localhost:4317/graphql", { method: "POST", body: multipart() });
console.log("Yoga, multipart:", r.status, await r.text());

const apollo = new ApolloServer({ typeDefs: `type Query { ok: Boolean }`, resolvers: { Query: { ok: () => true } } });
const { url } = await startStandaloneServer(apollo, { listen: { port: 4318 } });
r = await fetch(url, { method: "POST", body: multipart() });
console.log("Apollo, multipart:", r.status, (await r.text()).slice(0, 160));
ys.close(); await apollo.stop();
```

```console
$ node upload09.mjs
Yoga, multipart: 200 {"data":{"upload":"note.txt text/plain 14 bytes"}}
Apollo, multipart: 400 {"errors":[{"message":"This operation has been blocked as a potential C
```

(Apollo's message is truncated by the script.)

- **Yoga** supports uploads natively: declare `scalar File` and the resolver receives a standard
  `File` object.
- **Apollo Server 5** has no built-in upload support, and its CSRF prevention rejects the request
  outright: `multipart/form-data` is one of the content types a browser can send cross-site
  without a preflight ([Level 1 · 07](../level-1/07-apollo-server-http.md)).

That last point is the real lesson. An endpoint that accepts multipart requests can be driven by
a plain HTML form on a malicious site, with the victim's cookies attached. Yoga accepted the
upload above with no special header at all; its CSRF prevention is an opt-in plugin, and if you
authenticate with cookies you should enable it, so that uploads require a custom header that
cross-site forms can't send. Checked separately with `@graphql-yoga/plugin-csrf-prevention`
3.24.4:

```js
import { useCSRFPrevention } from "@graphql-yoga/plugin-csrf-prevention";
const yoga = createYoga({ schema, plugins: [useCSRFPrevention({ requestHeaders: ["x-graphql-csrf"] })] });
```

```text
{} 403 {"errors":[{"message":"Required CSRF header(s) not present"}]}
{"x-graphql-csrf":"1"} 200 {"data":{"upload":"a.txt"}}
```

The same multipart upload is refused without the header and accepted with it.

### The alternative most teams choose

Many production APIs avoid multipart GraphQL entirely:

1. A mutation `createUploadUrl(filename, contentType)` returns a short-lived **signed URL** for
   object storage (S3, GCS, Azure Blob…).
2. The client uploads the bytes directly to storage with a plain `PUT`.
3. A second mutation `attachFile(uploadId)` records the finished upload.

The GraphQL server never streams file bytes, large files don't tie up API workers, and storage
handles resumable uploads. Signed URLs need a cloud storage account, so this wasn't run here.

## How It Actually Works

Both servers parse the HTTP request into the same abstract `{ query, variables, operationName,
extensions }` and then call `graphql-js`. The differences are entirely in the HTTP layer around
it. Yoga runs on the Fetch API's `Request`/`Response` model (via `@whatwg-node/server`), and
its response stage inspects `Accept`: for `graphql-response+json` it maps request-level
`GraphQLError`s to a 400; for `application/json` it keeps 200 to stay compatible with legacy
clients. Apollo Server sets status codes from `extensions.http` on errors (validation and parse
errors carry `http: { status: 400 }`) independently of the negotiated media type.

For multipart, Yoga's request parser recognises `multipart/form-data`, reads `operations` and
`map`, and substitutes each `File` from the form into the variables at the mapped paths before
execution — so by the time the resolver runs, `file` is an ordinary argument value. Apollo's CSRF
check runs before body parsing and sees a "simple" content type, so it never gets that far.

## Common mistakes

- **Treating HTTP 200 as success** without checking for `errors`, or treating any non-200 as a
  network failure without reading the body.
- **Not sending an `Accept` header**, then being surprised by different status codes from
  different servers.
- **Enabling multipart uploads with cookie authentication and no CSRF protection.**
- **Streaming large files through the GraphQL server** when signed URLs would do.
- **Assuming all servers follow one convention.** Test the server you actually run — this lesson's
  table is the kind of check worth keeping in your HTTP test layer.

## Exercise

1. Add an HTTP test that fails if your server ever returns 200 for a validation error when the
   client sends `Accept: application/graphql-response+json`.
2. Send `Accept: text/html` to both servers with a POST and see what happens. Then send
   `Accept: application/xml`.
3. Add the CSRF plugin to `upload09.mjs` and make a browser-style `<form method="post"
   enctype="multipart/form-data">` request (no custom headers) fail while your app's `fetch` works.
4. Design the schema for the signed-URL upload flow: the mutations, their payloads and errors.
