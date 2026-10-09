---
description: "How GraphQL reports failures: request errors vs field errors, partial data, the path and extensions keys, and how non-null fields propagate nulls upward."
---

# 08 · Errors, Partial Data & Null Propagation

A REST endpoint usually succeeds or fails as a whole. A GraphQL response can do both at
once: some fields resolve, some fail, and the client gets **partial data** plus a list of
what went wrong. That's powerful — one slow microservice doesn't blank out a whole page —
but only if you design nullability on purpose. This lesson shows exactly what the executor
does with an error, using one schema and four failure scenarios.

## Two kinds of error

1. **Request errors** happen before execution: syntax errors, validation errors, invalid
   variables. The response has `errors` and **no `data` key**. You saw these in lessons
   [02](02-first-schema.md) and [06](06-mutations-inputs.md).
2. **Field errors** (also called execution errors) happen while resolving: a resolver
   throws, or returns `null` for a non-null field. The response has `data` — possibly `null`
   — and `errors`, each with a `path` saying which field failed.

The presence of the `data` key is how a client tells them apart.

## The test schema

```js title="errors08.mjs"
import { makeExecutableSchema } from "@graphql-tools/schema";
import { graphql, GraphQLError } from "graphql";

const books = [
  { id: "1", title: "Dune", reviewsDown: false },
  { id: "2", title: "Emma", reviewsDown: true },
  { id: "3", title: null, reviewsDown: false }, // corrupt row: title missing
];

const typeDefs = /* GraphQL */ `
  type Query {
    books: [Book!]!
    booksLoose: [Book]
    book(id: ID!): Book
    stats: Stats!
  }
  type Book {
    id: ID!
    title: String!
    rating: Float
    ratingStrict: Float!
  }
  type Stats { count: Int! }
`;

const resolvers = {
  Query: {
    books: () => books,
    booksLoose: () => books,
    book: (_, { id }) => {
      const b = books.find((x) => x.id === id);
      if (!b) throw new GraphQLError(`No book with id ${id}`, { extensions: { code: "NOT_FOUND", id } });
      return b;
    },
    stats: () => ({ count: books.length }),
  },
  Book: {
    rating: (b) => {
      if (b.reviewsDown) throw new Error("reviews service timed out");
      return 4.5;
    },
    ratingStrict: (b) => {
      if (b.reviewsDown) throw new Error("reviews service timed out");
      return 4.5;
    },
  },
};
const schema = makeExecutableSchema({ typeDefs, resolvers });
const run = async (label, source) => {
  const r = await graphql({ schema, source });
  console.log(`--- ${label}\n${JSON.stringify(r)}`);
};
```

`rating` and `ratingStrict` are identical except for one `!`. Book 2's reviews are "down";
book 3 has a corrupt `title`.

## Scenario 1: a nullable field fails

```js
await run("nullable field fails", `{ books { id rating } }`);
```

```text
--- nullable field fails
{"errors":[{"message":"reviews service timed out","locations":[{"line":1,"column":14}],"path":["books",1,"rating"]}],"data":{"books":[{"id":"1","rating":4.5},{"id":"2","rating":null},{"id":"3","rating":4.5}]}}
```

This is the ideal outcome. The failure is contained to exactly one field —
`path: ["books", 1, "rating"]` points at the second book's rating — and everything else
rendered. A UI can show "rating unavailable" for one card.

## Scenario 2: the same failure on a non-null field

```js
await run("non-null field fails", `{ stats { count } books { id ratingStrict } }`);
```

```text
--- non-null field fails
{"errors":[{"message":"reviews service timed out","locations":[{"line":1,"column":30}],"path":["books",1,"ratingStrict"]}],"data":null}
```

Same error, but now `data` is `null` — including `stats`, which had nothing to do with
reviews. Here's the chain:

```text
books.1.ratingStrict  Float!     → can't be null → null the parent
books.1               Book!      → can't be null → null the parent
books                 [Book!]!   → can't be null → null the parent
data                             → null
```

The rule, from the spec: when a field error occurs, the field becomes `null`; if the field's
type is non-null, the `null` **propagates to the nearest nullable ancestor**. If there is
none, `data` itself is `null`. One `!` on a field backed by an unreliable service turned a
minor failure into a blank page.

## Scenario 3: a resolver returns null for a non-null field

Errors don't have to be thrown. Returning `null` where `!` promises a value is a field error
too:

```js
await run("bad data in non-null field, loose list", `{ booksLoose { id title } }`);
```

```text
--- bad data in non-null field, loose list
{"errors":[{"message":"Cannot return null for non-nullable field Book.title.","locations":[{"line":1,"column":19}],"path":["booksLoose",2,"title"]}],"data":{"booksLoose":[{"id":"1","title":"Dune"},{"id":"2","title":"Emma"},null]}}
```

`booksLoose` is `[Book]` — list items are nullable. So the null from `title` propagated to
the `Book` item and stopped there: the list has `null` in position 2 and the other books are
intact. With `[Book!]!` the whole list (and then `data`) would have gone. Where you put
nullable "breakpoints" decides the blast radius.

## Scenario 4: GraphQLError and extensions

```js
await run("thrown GraphQLError with extensions", `{ book(id: "9") { title } stats { count } }`);
```

```text
--- thrown GraphQLError with extensions
{"errors":[{"message":"No book with id 9","locations":[{"line":1,"column":3}],"path":["book"],"extensions":{"code":"NOT_FOUND","id":"9"}}],"data":{"book":null,"stats":{"count":3}}}
```

Throwing a `GraphQLError` lets you attach **`extensions`** — the only spec-sanctioned place
for extra, machine-readable error information. A `code` in `extensions` is the convention
clients switch on (Apollo Server adds codes like `GRAPHQL_VALIDATION_FAILED` itself); the
`message` is for humans and may change.

Whether "not found" should be an error at all is a design question. Returning `null` from a
nullable `book` already means "no such book" without an error. Use an error when the client
needs to distinguish *why* — and see [Level 2 · 08](../level-2/08-mutation-design.md) for
modelling expected failures as data instead.

Plain `Error`s get wrapped:

```js
const r = await graphql({ schema, source: `{ books { id rating } }` });
const err = r.errors[0];
console.log("instanceof GraphQLError:", err instanceof GraphQLError);
console.log("originalError:", err.originalError?.constructor.name, "-", err.originalError?.message);
```

```text
instanceof GraphQLError: true
originalError: Error - reviews service timed out
```

Every entry in `errors` is a `GraphQLError` on the server, and the thing you threw is kept as
`originalError` — handy for logging, and important for security: the default serialisation
exposes the original **message**. If a database driver throws
`relation "users" does not exist`, that text reaches the client unless you mask it
([Level 4 · 06](../level-4/06-production-errors.md)).

## Designing nullability

Practical guidance that follows from the scenarios:

- **Fields that call other services, can time out, or depend on permissions** should
  usually be nullable. A `null` plus an error is better than a blank page.
- **Fields that are part of the object's identity and come from the same row** — `id`,
  `title` stored in the same table — are reasonable as non-null. If they fail, the object
  is meaningless anyway.
- **List items**: `[Book!]!` is fine when items come from one query that either works or
  doesn't; `[Book]!` is safer when items are assembled from several sources.
- **Arguments** are a separate matter — non-null *arguments* are about required input and
  have no propagation effect.

The [schema design lesson](../level-2/01-schema-design.md) revisits these with more context.

## How It Actually Works

Inside `graphql-js`, `executeField` wraps the resolver call and `completeValue` in a
`try`/`catch` (and a `.then(…, onError)` for promises). On failure it calls
`locatedError(rawError, fieldNodes, pathToArray(path))`, which returns the error unchanged
if it's already a located `GraphQLError`, or wraps it — setting `originalError`,
`locations` and `path`. Then `handleFieldError` runs:

```js
// paraphrased from graphql-js 16
function handleFieldError(error, returnType, path, exeContext) {
  if (isNonNullType(returnType)) throw error; // let the parent deal with it
  exeContext.collectedErrors.add(error, path); // record it once
  return null;                                // this field becomes null
}
```

That's the whole propagation mechanism: a non-null field *re-throws*, so the error climbs
the recursive `completeValue` calls until it reaches a field whose type is nullable, which
records the error and returns `null`. At the root, `executeOperation` catches anything that
escaped and sets `data: null`. Because the error is recorded once — at the nullable field
that absorbed it — you see one entry even though several levels were nulled.

## Common mistakes

- **Marking everything non-null** because it "looks stricter". It makes every failure as
  big as possible.
- **Checking only `response.errors`.** Partial data is valid data; show what you have.
  Equally, don't assume a missing field means "empty" — check whether an error has that path.
- **Putting error codes in the message** (`"NOT_FOUND: No book"`). Use `extensions.code`.
- **Leaking internals** through raw error messages from drivers and libraries.
- **Throwing for expected outcomes** like "email already taken" — those are better as
  typed results the client can render (Level 2 · 08).

## Exercise

1. Change `books` to `[Book]!` and re-run scenario 2. What survives now?
2. Make `stats` nullable and keep `books: [Book!]!`. Run scenario 2 again. Explain why
   `stats` still doesn't appear.
3. Add a resolver that returns `"abc"` for an `Int` field. What error do you get, and which
   phase produces it?
4. Write a tiny client helper `fieldError(response, path)` that returns the error for a given
   path, if any. Use it to show "rating unavailable" only for book 2.
