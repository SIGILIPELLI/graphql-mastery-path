---
description: "Put a schema behind HTTP with Apollo Server 5: request shapes, per-request context from headers, GET vs POST, CSRF protection and production-mode defaults."
---

# 07 · Serving GraphQL over HTTP with Apollo Server

So far every query has been a function call. Real clients talk to GraphQL over HTTP, and
the transport adds its own rules: how the query is encoded, which methods are allowed,
what status codes mean, and how per-request state like "who is logged in" gets into
resolvers. This lesson uses **Apollo Server 5** (5.5.1 when run here), the most widely
deployed JavaScript GraphQL server, and pokes it with `curl` so you can see the raw HTTP.

## Install and start

```bash
npm install @apollo/server graphql
```

```js title="server07.mjs"
import { ApolloServer } from "@apollo/server";
import { startStandaloneServer } from "@apollo/server/standalone";

const books = [
  { id: "1", title: "Dune", ownerId: "u1" },
  { id: "2", title: "Emma", ownerId: "u2" },
];
const users = { "token-ada": { id: "u1", name: "Ada" }, "token-bob": { id: "u2", name: "Bob" } };

const typeDefs = /* GraphQL */ `
  type Query {
    books: [Book!]!
    myBooks: [Book!]!
    whoami: String
  }
  type Book { id: ID! title: String! }
`;

const resolvers = {
  Query: {
    books: () => books,
    myBooks: (_, __, { user }) => (user ? books.filter((b) => b.ownerId === user.id) : []),
    whoami: (_, __, { user, requestId }) => (user ? `${user.name} (request ${requestId})` : null),
  },
};

const server = new ApolloServer({ typeDefs, resolvers });
let counter = 0;
const { url } = await startStandaloneServer(server, {
  listen: { port: 4000 },
  context: async ({ req }) => {
    const token = (req.headers.authorization ?? "").replace(/^Bearer /, "");
    return { user: users[token] ?? null, requestId: ++counter };
  },
});
console.log(`Server ready at ${url}`);
```

```console
$ node server07.mjs
Server ready at http://localhost:4000/
```

`startStandaloneServer` wraps Node's `http` module for you. In a real application you'd
usually mount Apollo Server into an existing framework (Express via
`@as-integrations/express5`, Fastify, a serverless handler…), but the GraphQL behaviour is
identical — the integration just translates the framework's request into Apollo's
`HTTPGraphQLRequest` and back.

The tokens here are hard-coded strings to keep the focus on the plumbing. Real token
verification is [Level 3 · 01](../level-3/01-authentication.md).

## The request shape

A GraphQL HTTP request is a JSON body with up to four keys: `query` (the document text),
`variables` (an object), `operationName` (which operation to run if the document has more
than one) and `extensions` (protocol add-ons such as persisted-query hashes).

```console
$ curl -s -X POST localhost:4000/ -H 'content-type: application/json' \
    -d '{"query":"{ books { id title } }"}'
{"data":{"books":[{"id":"1","title":"Dune"},{"id":"2","title":"Emma"}]}}
```

Every operation goes to the **same URL**. There is no `/books` route; the body decides what
happens.

## Context: turning headers into resolver state

The `context` function runs **once per request**, before any resolver, and whatever it
returns becomes the third resolver argument. Here it reads the `Authorization` header:

```console
$ curl -s -X POST localhost:4000/ -H 'content-type: application/json' \
    -H 'authorization: Bearer token-ada' \
    -d '{"query":"query Mine { whoami myBooks { title } }","operationName":"Mine"}'
{"data":{"whoami":"Ada (request 2)","myBooks":[{"title":"Dune"}]}}
```

`request 2` shows that each HTTP request got a fresh context — the counter went up once
for this request, not once per resolver. That's the property you rely on later for
per-request DataLoaders and caches.

Note the design choice in `myBooks`: an anonymous caller gets an empty list rather than an
error. Whether "not logged in" should be an error or an empty result is an API decision;
[Level 3 · 02](../level-3/02-authorization.md) discusses the trade-offs.

## GET requests

Queries (not mutations) can also be sent as GET, with the same keys as URL parameters —
useful for HTTP caches and CDNs:

```console
$ curl -s 'localhost:4000/?query=%7Bbooks%7Btitle%7D%7D' -H 'apollo-require-preflight: true'
{"data":{"books":[{"title":"Dune"},{"title":"Emma"}]}}
```

Without that extra header the same request is rejected:

```console
$ curl -s -i 'localhost:4000/?query=%7Bbooks%7Btitle%7D%7D'
HTTP/1.1 400 Bad Request
{"errors":[{"message":"This operation has been blocked as a potential Cross-Site Request Forgery (CSRF). Please either specify a 'content-type' header (with a type that is not one of application/x-www-form-urlencoded, multipart/form-data, text/plain) or provide a non-empty value for one of the following headers: x-apollo-operation-name, apollo-require-preflight\n", ...
```

(Output trimmed after the message; the full response also included a stack trace — more
on that below.)

This is Apollo Server's **CSRF prevention**, on by default. A malicious web page can make
your browser send a "simple" request — a GET, or a POST with `text/plain` or form content
types — to another origin *with your cookies attached* and without a CORS preflight. If
your API authenticates with cookies, that request would run as you. Apollo refuses any
request that a browser could send without a preflight: it must have a JSON content type or
one of those custom headers, both of which force the browser to ask the server's CORS policy
first. A `text/plain` POST fails the same way:

```console
$ curl -s -i -X POST localhost:4000/ -H 'content-type: text/plain' -d '{"query":"{ books { id } }"}'
HTTP/1.1 400 Bad Request
{"errors":[{"message":"This operation has been blocked as a potential Cross-Site Request Forgery (CSRF). ...
```

Mutations are never allowed over GET, because GETs must be safe to repeat and cache:

```console
$ curl -s -i 'localhost:4000/?query=mutation%7Bx%7D' -H 'apollo-require-preflight: true'
HTTP/1.1 405 Method Not Allowed
{"errors":[{"message":"GET requests only support query operations, not mutation operations", ...
```

## Status codes

GraphQL's in-band `errors` array doesn't replace HTTP status codes; they describe
different things. What Apollo Server returned for each failure:

| Request | Status | `extensions.code` |
|---|---|---|
| valid query (even if a resolver later errors) | 200 | — / your code |
| syntax error `{ books { ` | 400 | `GRAPHQL_PARSE_FAILED` |
| unknown field `{ books { isbn } }` | 400 | `GRAPHQL_VALIDATION_FAILED` |
| empty JSON body `{}` | 400 | `BAD_REQUEST` ("POST body missing, invalid Content-Type, or JSON object has no keys.") |
| CSRF-unsafe request | 400 | `BAD_REQUEST` |
| mutation over GET | 405 | `BAD_REQUEST` |

Roughly: if the request never reached execution, it's a 4xx. Once execution starts, the
response is 200 with whatever `data` and `errors` resulted. [Level 3 · 09](../level-3/09-graphql-over-http.md)
covers the official GraphQL-over-HTTP spec and the `application/graphql-response+json`
media type, which tightens these rules.

## Development versus production mode

Every error above came back with an `extensions.stacktrace` array full of file paths from
the server. That's the development default. Apollo Server switches several defaults when
`NODE_ENV=production`:

```console
$ NODE_ENV=production node server07.mjs
$ curl -s -i -X POST localhost:4000/ -H 'content-type: application/json' -d '{"query":"{ books { isbn } }"}'
HTTP/1.1 400 Bad Request
{"errors":[{"message":"Cannot query field \"isbn\" on type \"Book\".","locations":[{"line":1,"column":11}],"extensions":{"code":"GRAPHQL_VALIDATION_FAILED"}}]}

$ curl -s -X POST localhost:4000/ -H 'content-type: application/json' -d '{"query":"{ __schema { queryType { name } } }"}'
{"errors":[{"message":"GraphQL introspection is not allowed by Apollo Server, but the query contained __schema or __type. To enable introspection, pass introspection: true to ApolloServer in production","locations":[{"line":1,"column":3}],"extensions":{"validationErrorCode":"INTROSPECTION_DISABLED","code":"GRAPHQL_VALIDATION_FAILED"}}]}
```

Stack traces are gone and **introspection is disabled**. Both can be set explicitly with
the `includeStacktraceInErrorResponses` and `introspection` constructor options — and it's
better to set them explicitly than to depend on an environment variable someone might
forget. A browser GET with `Accept: text/html` returns a landing page (title
"Apollo Server") in both modes; in development it embeds Apollo Sandbox, a hosted query
IDE loaded from Apollo's CDN.

## How It Actually Works

Apollo Server is a pipeline wrapped around `graphql-js`. For each HTTP request:

1. The integration builds an `HTTPGraphQLRequest` (method, headers, search params, parsed
   body). For GET, `query`/`variables`/`operationName`/`extensions` come from the URL; for
   POST, from the JSON body. Missing or unparsable input is a `BadRequestError` (400).
2. `preventCsrf` checks the content type and headers as described above.
3. Your `context` function runs.
4. The request pipeline runs `parse` → `validate` (with the spec rules plus Apollo's, such
   as the introspection-disabling rule in production) → `execute`, firing plugin hooks
   at each stage (`didResolveSource`, `parsingDidStart`, `validationDidStart`,
   `executionDidStart`, `willSendResponse`). Parsed and validated documents are cached by
   query hash, so a repeated query string skips both steps.
5. Errors are passed through `formatError` (if you supplied one) and stack traces are
   attached or stripped, then the result is serialised to JSON and the status code chosen.

GET-mutation rejection happens after parsing, because Apollo has to parse the document to
know it's a mutation — that's why it's a 405 with a GraphQL-shaped error body rather than a
plain HTTP error.

## Common mistakes

- **Building context once at startup.** It must be a function that runs per request;
  otherwise every request shares one user.
- **Disabling CSRF prevention to "fix" a client** that sends `text/plain`. Fix the client's
  content type instead.
- **Relying on `NODE_ENV` for security settings.** Set `introspection` and
  `includeStacktraceInErrorResponses` explicitly per environment.
- **Treating any 200 as success.** A 200 response can still contain `errors`; clients must
  check the body.
- **Throwing in the context function for anonymous users.** It turns every
  unauthenticated request — including ones for public fields — into an error. Return
  `user: null` and let resolvers decide.

## Exercise

1. Add a `x-request-id` header to the curl request and include it in the context; return it
   from a `requestId` field.
2. Send a document containing two named operations (`query A {...} query B {...}`) without
   `operationName`. What does Apollo return? Then add `operationName` to fix it.
3. Start the server with `includeStacktraceInErrorResponses: false` and without
   `NODE_ENV`, and confirm stack traces are gone.
4. Add a mutation and try sending it via GET with and without the preflight header. Explain
   why the status codes differ.
