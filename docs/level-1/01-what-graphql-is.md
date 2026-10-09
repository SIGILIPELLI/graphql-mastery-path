---
description: "What GraphQL is and isn't: one endpoint, a typed schema, and responses shaped by the client — compared side by side with REST on the same data."
---

# 01 · What GraphQL Is: One Endpoint, a Typed Schema, Client-Shaped Responses

GraphQL is a **query language for APIs** plus a **specification for how a server executes
those queries against a typed schema**. That's the whole idea. It is not a database, not a
storage engine, not a transport protocol and not a replacement for SQL. A GraphQL server
sits where a REST API would sit: in front of your databases, services and business logic.
What changes is the contract between client and server.

With a typical REST API the *server* decides what each URL returns. With GraphQL the
server publishes a **schema** — a typed description of everything that can be asked for —
and each *client request* says exactly which fields it wants. The response mirrors the
shape of the request.

Facebook built GraphQL internally in 2012 for its mobile apps and published the
specification in 2015. Since 2018 the spec and the reference implementation (`graphql-js`)
have been stewarded by the GraphQL Foundation under the Linux Foundation. The spec is
published at `spec.graphql.org`; it is deliberately small and says nothing about HTTP,
databases or authentication. Those are left to servers and to companion specs such as
GraphQL-over-HTTP (covered in [Level 3 · 09](../level-3/09-graphql-over-http.md)).

## The same data, two ways

To make the difference concrete, here is one small Node.js server that exposes the same
in-memory data both as REST endpoints and as a GraphQL endpoint. The screen we're building
needs a user's name and the title and like-count of each of their posts.

```js title="compare.mjs"
import http from "node:http";
import { buildSchema, graphql } from "graphql";

const users = [{ id: "1", name: "Ada Lovelace", email: "ada@example.com",
  bio: "Wrote the first published algorithm.", joined: "2024-03-01" }];
const posts = [
  { id: "10", authorId: "1", title: "Notes on the Engine", body: "x".repeat(400), likes: 42 },
  { id: "11", authorId: "1", title: "On Bernoulli Numbers", body: "y".repeat(400), likes: 17 },
];

const schema = buildSchema(`
  type Query { user(id: ID!): User }
  type User { id: ID! name: String! email: String! bio: String joined: String! posts: [Post!]! }
  type Post { id: ID! title: String! body: String! likes: Int! }
`);
const rootValue = {
  user: ({ id }) => {
    const u = users.find((u) => u.id === id);
    return u && { ...u, posts: () => posts.filter((p) => p.authorId === id) };
  },
};

http.createServer(async (req, res) => {
  const send = (obj) => {
    res.setHeader("content-type", "application/json");
    res.end(JSON.stringify(obj));
  };
  if (req.url === "/users/1") return send(users[0]);
  if (req.url === "/users/1/posts") return send(posts.filter((p) => p.authorId === "1"));
  if (req.url === "/graphql" && req.method === "POST") {
    let body = "";
    for await (const chunk of req) body += chunk;
    const { query, variables } = JSON.parse(body);
    return send(await graphql({ schema, source: query, rootValue, variableValues: variables }));
  }
  res.statusCode = 404;
  res.end();
}).listen(4001, () => console.log("listening on 4001"));
```

Install the one dependency and start it:

```bash
npm init -y && npm pkg set type=module
npm install graphql@16
node compare.mjs
```

The REST client needs two requests, and gets every field of every object whether the
screen needs it or not:

```console
$ curl -s -o /dev/null -w "GET /users/1 -> %{http_code}, %{size_download} bytes\n" localhost:4001/users/1
GET /users/1 -> 200, 125 bytes
$ curl -s -o /dev/null -w "GET /users/1/posts -> %{http_code}, %{size_download} bytes\n" localhost:4001/users/1/posts
GET /users/1/posts -> 200, 958 bytes
```

The GraphQL client sends one request naming the fields it needs:

```console
$ curl -s localhost:4001/graphql -H 'content-type: application/json' \
    -d '{"query":"{ user(id: \"1\") { name posts { title likes } } }"}'
{"data":{"user":{"name":"Ada Lovelace","posts":[{"title":"Notes on the Engine","likes":42},{"title":"On Bernoulli Numbers","likes":17}]}}}
```

That response was 138 bytes against 1,083 for the two REST calls. Don't read too much
into the number — it's a toy, the post bodies were padded on purpose, and a REST API could
add `?fields=` filtering or a purpose-built `/users/1/summary` endpoint. The point is
*who decides*: in REST the server designs each response; in GraphQL the client composes
one from a published catalogue of fields.

## The three pieces you'll meet in every lesson

**1. The schema.** Written in the Schema Definition Language (SDL), it lists types and
their fields:

```graphql
type Query {
  user(id: ID!): User
}

type User {
  id: ID!
  name: String!
  posts: [Post!]!
}
```

`Query` is the entry point for reads. `ID!` means "an ID, never null". `[Post!]!` means "a
list that is never null, whose items are never null". The schema is the contract —
clients can only ask for what is in it, and the server promises the types.

**2. Operations.** A client sends a *document* containing a `query` (read), a `mutation`
(write) or a `subscription` (a stream of events):

```graphql
query UserCard {
  user(id: "1") {
    name
    posts { title likes }
  }
}
```

**3. Resolvers.** For every field the server needs a function that produces its value.
In the example above, `rootValue.user` resolves `Query.user`, and `posts` is a function on
the returned object that resolves `User.posts`. Fields such as `name` don't need a
function: the default behaviour is to read the property of the same name.
[Lesson 05](05-resolvers.md) is entirely about resolvers.

## How It Actually Works

When the server receives the GraphQL request it does three separate things, in order, and
you will see each of them by name throughout this course:

1. **Parse.** The query string is turned into an abstract syntax tree (AST). A typo like
   `{ user(id: "1" { name } }` fails here with a syntax error, before anything else runs.
2. **Validate.** The AST is checked against the schema using a fixed set of rules from the
   specification: every field must exist on its type, required arguments must be present,
   variables must be used with compatible types, fragments must be applicable, and so on.
   An invalid document is rejected *as a whole* — no resolver runs. This is why a client can
   ask for `author` on `Book` and get back `Cannot query field "author" on type "Book"`
   with no data at all.
3. **Execute.** The executor walks the selection set from the root type. For each field it
   calls the field's resolver with the parent object, collects the result, checks it
   against the field's declared type (coercing it, or raising an error if a non-null field
   came back null), and recurses into sub-selections. The response's `data` is built in
   the same shape as the query.

Because validation happens against a typed schema *before* execution, the server always
knows up front what a request will touch. That property drives most of GraphQL's
strengths (tooling, generated types, documentation that can't drift from the
implementation) and most of its operational problems (one request can ask for a lot of
work — [Level 3 · 05](../level-3/05-security-limits.md) deals with that).

Notice also what the spec does *not* fix: the URL (conventionally `/graphql`), the HTTP
method, how authentication works, how data is fetched. In our toy server, `/graphql`
accepted a JSON body with `query` and `variables` because that's the common convention,
not because GraphQL requires HTTP at all. In [lesson 02](02-first-schema.md) we'll execute
queries with no network involved.

## What GraphQL is good at — and what it costs

Good fits:

- **Many clients with different needs** (web, iOS, Android, partner integrations) reading
  overlapping data. Each asks for its own shape without a new endpoint per screen.
- **Deeply related data** — a product with its reviews, their authors and the author's
  other reviews — fetched in one round trip instead of a waterfall of requests.
- **A self-describing contract.** The schema can be introspected, which powers
  autocompletion, documentation and generated client types.
- **Evolving without versions.** New fields are additive; old ones are marked
  `@deprecated` and removed once no client uses them ([Level 4 · 05](../level-4/05-schema-evolution.md)).

Costs you take on:

- **HTTP caching is harder.** Most GraphQL traffic is `POST` to one URL, so CDNs and
  browser caches can't help by default ([Level 3 · 06](../level-3/06-caching-persisted-queries.md)).
- **Clients can ask for expensive things.** A nested query can fan out into thousands of
  database calls. You need batching (DataLoader), limits and cost analysis.
- **Errors don't map to status codes.** A response can be `200 OK` and still contain
  errors for some fields ([lesson 08](08-errors.md)).
- **More server machinery** than a handful of REST routes for simple CRUD.

The [REST API Mastery Path](https://sigilipelli.github.io/rest-api-mastery-path/) covers
REST design in depth, including its own GraphQL-vs-REST comparison. This course assumes you
have chosen GraphQL, or want to understand it well enough to choose; [Level 4 · 09](../level-4/09-architecture-review.md)
returns to the decision with much more experience behind it.

## Common misconceptions

- **"GraphQL queries the database directly."** It doesn't. Every field value comes from a
  resolver you wrote. A schema field called `posts` has nothing to do with a table called
  `posts` unless your resolver makes it so.
- **"GraphQL is faster than REST."** It can remove round trips and over-fetching, but each
  request does more work on the server, and naive resolvers are often *slower* than a
  hand-written REST endpoint with one SQL join. Speed comes from implementation, not from
  the query language.
- **"One endpoint means no structure."** The structure moves into the schema. A badly
  designed schema is as painful as a badly designed set of URLs.
- **"GraphQL means no versioning problems."** It means you evolve one schema continuously
  instead of shipping `/v2`. You still have to manage deprecation and know which clients use
  which fields.
- **"GraphQL is a Facebook/React thing."** Servers exist for every mainstream language
  (graphql-java, Strawberry and Graphene for Python, gqlgen for Go, Hot Chocolate for .NET),
  and any HTTP client can call a GraphQL API — you saw `curl` do it above.

## What this course uses

The lessons use JavaScript on Node.js because the reference implementation, `graphql-js`,
is a JavaScript library and the most widely used servers build on it. The versions every
example was run with are listed on each level's overview page. You don't need TypeScript
until [Level 3 · 08](../level-3/08-codegen.md); if JavaScript itself is new, start with
[JavaScript Mastery Path](https://sigilipelli.github.io/javascript-mastery-path/) and
[Node.js Mastery Path](https://sigilipelli.github.io/nodejs-mastery-path/).

## Exercise

1. Run `compare.mjs`. Using only the GraphQL endpoint, fetch the user's `email` and `joined`
   date and the `id` of each post, in one request.
2. Ask for a field that doesn't exist (for example `user { age }`). Read the error and
   confirm there is no `data` key in the response. Which of the three phases — parse,
   validate, execute — rejected it?
3. Send a request with a syntax error (drop a closing brace). Compare the error message to
   the one from step 2.
4. Add a `/users/1/summary` REST route that returns exactly what the GraphQL query returned.
   Then write down what you'd have to do when a second screen needs `name` and `bio` but
   not posts. This is the trade-off GraphQL is designed around.
