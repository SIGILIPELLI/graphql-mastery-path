---
description: "Write data with GraphQL mutations: input object types, partial updates, null vs absent, and why mutation fields run one at a time."
---

# 06 · Mutations and Input Types

Queries read. **Mutations** write. Mechanically, a mutation is just another root operation
type — `type Mutation` sits next to `type Query`, and its fields have resolvers like any
other. Two things make it different, though: a mutation's top-level fields execute
**serially**, and the arguments you send are usually **input object types**, which follow
their own rules. This lesson covers both, with every case run.

## Input types versus object types

You can't pass a `Book` object as an argument. Output types (`type`) can have fields with
arguments, can reference interfaces and unions, and can form cycles (`Book.author.books`).
None of that makes sense for data the client *sends*, so the spec has a separate kind:

```graphql
input AddBookInput {
  title: String!
  year: Int!
  tags: [String!] = []
}
```

An `input` type's fields can only be scalars, enums, lists, or other input types. Fields can
have defaults. It's common — and clearer — to keep input and output types separate even
when they look alike: `Book` has a server-generated `id`; `AddBookInput` doesn't.

What happens if you try to use an output type as an argument? Something surprising:

```js title="inputerr.mjs"
import { makeExecutableSchema } from "@graphql-tools/schema";
import { graphql, validateSchema } from "graphql";
const schema = makeExecutableSchema({ typeDefs: `type Query { x(b: Book): Int } type Book { id: ID }` });
console.log("built OK; validateSchema says:", validateSchema(schema).map((e) => e.message));
console.log(JSON.stringify(await graphql({ schema, source: "{ x }" })));
```

```console
$ node inputerr.mjs
built OK; validateSchema says: [ 'The type of Query.x(b:) must be Input Type but got: Book.' ]
{"errors":[{"message":"The type of Query.x(b:) must be Input Type but got: Book.","locations":[{"line":1,"column":19}]}]}
```

The schema **built** without complaint. `graphql-js` validates a schema lazily, the first
time it's used for a request, and then every request fails. Apollo Server validates at
startup, but if you use `graphql-js` directly, call `validateSchema` (or
`assertValidSchema`) in your startup code or a test so a broken schema fails the deploy,
not the first user.

## A small mutation API

```js title="mutations06.mjs"
import { makeExecutableSchema } from "@graphql-tools/schema";
import { graphql } from "graphql";

const books = [{ id: "1", title: "Dune", year: 1965, tags: ["sci-fi"] }];
let nextId = 2;

const schema = makeExecutableSchema({
  typeDefs: /* GraphQL */ `
    type Query { books: [Book!]! }
    type Mutation {
      addBook(input: AddBookInput!): Book!
      updateBook(id: ID!, input: UpdateBookInput!): Book
      deleteBook(id: ID!): Boolean!
    }
    input AddBookInput {
      title: String!
      year: Int!
      tags: [String!] = []
    }
    input UpdateBookInput {
      title: String
      year: Int
    }
    type Book { id: ID! title: String! year: Int! tags: [String!]! }
  `,
  resolvers: {
    Query: { books: () => books },
    Mutation: {
      addBook: (_, { input }) => {
        const book = { id: String(nextId++), ...input };
        books.push(book);
        return book;
      },
      updateBook: (_, { id, input }) => {
        const book = books.find((b) => b.id === id);
        if (!book) return null;
        for (const [k, v] of Object.entries(input)) {
          if (v === null) throw new Error(`${k} cannot be set to null`);
          book[k] = v;
        }
        return book;
      },
      deleteBook: (_, { id }) => {
        const i = books.findIndex((b) => b.id === id);
        if (i === -1) return false;
        books.splice(i, 1);
        return true;
      },
    },
  },
});

const run = async (label, source, variableValues) => {
  const r = await graphql({ schema, source, variableValues });
  console.log(`--- ${label}\n${JSON.stringify(r)}`);
};
```

Each case below was appended to that file and run in order with `node mutations06.mjs`
(state carries over between cases).

### Creating, with variables

```js
await run("add with variables", `
  mutation Add($input: AddBookInput!) { addBook(input: $input) { id title year tags } }`,
  { input: { title: "Neuromancer", year: 1984 } });
```

```text
--- add with variables
{"data":{"addBook":{"id":"2","title":"Neuromancer","year":1984,"tags":[]}}}
```

The client sent a JSON object as the variable; `graphql-js` coerced it into
`AddBookInput`, filling in `tags: []` from the input field default. The mutation's
**selection set** works exactly like a query's: the client chooses which fields of the new
`Book` to read back. That's the main reason mutations return the changed object — the
client can refresh its cache in the same round trip.

### Partial updates: absent versus null

`UpdateBookInput` makes every field nullable so a client can send only what it wants to
change:

```js
await run("update partial", `
  mutation { updateBook(id: "2", input: { year: 1985 }) { id title year } }`);
await run("update explicit null", `
  mutation { updateBook(id: "2", input: { title: null }) { id title } }`);
```

```text
--- update partial
{"data":{"updateBook":{"id":"2","title":"Neuromancer","year":1985}}}
--- update explicit null
{"errors":[{"message":"title cannot be set to null","locations":[{"line":2,"column":14}],"path":["updateBook"]}],"data":{"updateBook":null}}
```

In the first call the `input` object the resolver received was `{ year: 1985 }` — `title`
was **absent**, so it isn't a key at all and `Object.entries` skipped it. In the second,
`title` was explicitly `null`, so it's present with value `null`. GraphQL preserves that
distinction, and a well-behaved update resolver must too: "don't change it" and "clear it"
are different requests. Here `title` can't be cleared, so the resolver refuses.

Because `updateBook` returns a nullable `Book`, the error nulls only that field and `data`
survives. That's one reason mutation return types are usually nullable or wrapped in
payload types ([Level 2 · 08](../level-2/08-mutation-design.md)).

### Validation of inputs happens before any resolver

```js
await run("unknown input field", `
  mutation { addBook(input: { title: "X", year: 1, author: "Y" }) { id } }`);
await run("missing required via variables", `
  mutation Add($input: AddBookInput!) { addBook(input: $input) { id } }`,
  { input: { title: "No year" } });
```

```text
--- unknown input field
{"errors":[{"message":"Field \"author\" is not defined by type \"AddBookInput\".","locations":[{"line":2,"column":52}]}]}
--- missing required via variables
{"errors":[{"message":"Variable \"$input\" got invalid value { title: \"No year\" }; Field \"year\" of required type \"Int!\" was not provided.","locations":[{"line":2,"column":16}]}]}
```

No `data` key in either: nothing ran, nothing was written. The first is caught by
validation of the literal in the query text; the second by variable coercion, which also
happens before execution. You get this for free, so your resolvers can trust that `title`
is a string and `year` an integer.

### Several mutations in one request

```js
await run("several in one request", `
  mutation {
    a: addBook(input: { title: "Emma", year: 1815 }) { id }
    d: deleteBook(id: "1")
    gone: deleteBook(id: "1")
  }`);
await run("books now", `{ books { id title year } }`);
```

```text
--- several in one request
{"data":{"a":{"id":"3"},"d":true,"gone":false}}
--- books now
{"data":{"books":[{"id":"2","title":"Neuromancer","year":1985},{"id":"3","title":"Emma","year":1815}]}}
```

The second `deleteBook(id: "1")` returned `false` because the first had already removed
the book. That deterministic result depends on the next rule.

## Serial execution, demonstrated

The spec says top-level fields of a **mutation** execute serially: each finishes (including
its sub-selection) before the next starts. Query fields have no such guarantee. To see it,
add a slow field to both root types:

```js
// in typeDefs:  Query { slow(ms: Int!, label: String!): String! }
//               Mutation { slowWrite(ms: Int!, label: String!): String! }
const events = [];
const sleep = (ms) => new Promise((r) => setTimeout(r, ms));
const slow = async (_, { ms, label }) => {
  events.push(`start ${label}`); await sleep(ms); events.push(`end ${label}`); return label;
};
// Query.slow = slow; Mutation.slowWrite = slow;

await run("query fields", `{ a: slow(ms: 60, label: "A") b: slow(ms: 10, label: "B") }`);
await run("mutation fields", `mutation { a: slowWrite(ms: 60, label: "A") b: slowWrite(ms: 10, label: "B") }`);
```

The event logs (with wall-clock time rounded to 10 ms):

```text
query fields:    start A, start B, end B, end A   ~60ms
mutation fields: start A, end A, start B, end B   ~70ms
```

In the query both fields started immediately and B finished first; total time was about the
slowest field. In the mutation, B didn't start until A was completely done, so the times
add up. Serial execution makes `mutation { withdraw debit }` predictable — but it's
ordering *within one request* only. It isn't a transaction: if the second field fails, the
first one's write has already happened. Atomic multi-step writes still need a database
transaction inside a single resolver.

## How It Actually Works

`execute` chooses a strategy from the operation type. For `query` it calls
`executeFields`, which invokes every top-level resolver, collects the results (some of them
promises) into an object and waits for all of them together. For `mutation` it calls
`executeFieldsSerially`, which is a `reduce` over the fields: each field's result promise is
awaited before the next field's resolver is even called. Below the top level, a mutation's
sub-selections are executed with the normal parallel strategy.

Input coercion is a separate step that runs first. `getVariableValues` walks each variable's
declared type with `coerceInputValue`: for an input object it rejects unknown keys, applies
field defaults for missing keys, errors on missing non-null keys, and recursively coerces
each value through scalars' `parseValue`. Literal arguments in the query text go through
`valueFromAST` instead, and validation rules (`ValuesOfCorrectTypeRule`) catch bad literals
before that. The result is a plain JavaScript object where absent keys are truly absent —
which is how the resolver can tell `{}` from `{ title: null }`.

## Common mistakes

- **Reusing output types as inputs.** It isn't allowed, and even the workaround of
  mirroring them 1:1 tends to leak server-owned fields (`id`, `createdAt`) into inputs.
- **Treating `null` and absent the same** in update resolvers, so a client clearing a
  field and a client not mentioning it produce the same result.
- **Assuming serial execution means atomicity.** It doesn't roll anything back.
- **Returning only `Boolean` from create/update mutations.** The client then has to make a
  second request to read the result.
- **Not validating the schema at startup** when using `graphql-js` directly — a bad input
  type passes `makeExecutableSchema` and breaks the first request.

## Exercise

1. Add `input BookFilter { tag: String, publishedAfter: Int }` and a query field
   `books(filter: BookFilter)`. Implement it, treating a missing filter as "everything".
2. Add `tags: [String!]` to `UpdateBookInput`. What should `tags: null` mean? Implement
   your decision and test absent, `null` and `[]` separately.
3. Write a mutation request that adds a book and then updates it **using the new id** in the
   same request. Why can't you? (Hint: variables are fixed before execution starts.)
4. Move the `deleteBook` call before `addBook` in the "several in one request" example and
   predict the output, then run it.
