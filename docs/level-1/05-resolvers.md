---
description: "What each of the four resolver arguments really carries, how the default resolver fills the gaps, and how resolvers chain from parent to child."
---

# 05 · Resolvers: parent, args, context & info

A schema describes shapes; **resolvers** produce the values. Every field in a GraphQL schema
has a resolver, whether you write one or not. Getting comfortable with how they're called —
what each argument contains, in what order they run, and what happens when one throws — is
the single most useful skill for building GraphQL servers. Everything later in the course
(DataLoader, auth, directives, federation) is built on top of this one idea.

## The signature

In a resolver map, every resolver has the same shape:

```js
fieldName(parent, args, context, info) { /* return a value, a promise, or throw */ }
```

| Argument | What it is | Typical use |
|---|---|---|
| `parent` | The value the *parent* field's resolver returned. For root fields, the server's `rootValue` (usually `undefined`). | Read the current object: `book.title`, `book.author_id` |
| `args` | The field's arguments, already coerced and with defaults applied | Filters, ids, page sizes |
| `context` | One object shared by **every** resolver in a single request | Database handles, the current user, DataLoaders |
| `info` | Metadata about this field in this query: name, return type, path, the AST of the selection, the schema | Debugging, logging, look-ahead optimisations |

You'll see the parent called `obj`, `source`, `root` or `_` in different codebases. They're
all the same thing.

## A schema where every argument matters

```js title="resolvers05.mjs"
import { makeExecutableSchema } from "@graphql-tools/schema";
import { graphql } from "graphql";

const db = {
  books: [
    { id: "1", title: "Dune", author_id: "a1", priceCents: 1299 },
    { id: "2", title: "Emma", author_id: "a2", priceCents: 899 },
  ],
  authors: [
    { id: "a1", name: "Frank Herbert" },
    { id: "a2", name: "Jane Austen" },
  ],
};

const log = [];
const sleep = (ms) => new Promise((r) => setTimeout(r, ms));

const typeDefs = /* GraphQL */ `
  type Query {
    books: [Book!]!
    me: String
  }
  type Book {
    id: ID!
    title: String!
    price(currency: String = "USD"): String!
    author: Author!
  }
  type Author { id: ID! name: String! books: [Book!]! }
`;

const resolvers = {
  Query: {
    books: async (parent, args, ctx, info) => {
      log.push(`Query.books parent=${JSON.stringify(parent)} path=${pathStr(info.path)}`);
      await sleep(5);
      return ctx.db.books;
    },
    me: (_, __, ctx) => ctx.user?.name ?? null,
  },
  Book: {
    price: (book, { currency }, ctx, info) => {
      log.push(`Book.price parent.id=${book.id} args=${JSON.stringify({ currency })} returnType=${info.returnType}`);
      const rate = { USD: 1, EUR: 0.9 }[currency];
      if (rate === undefined) throw new Error(`Unsupported currency: ${currency}`);
      return `${((book.priceCents * rate) / 100).toFixed(2)} ${currency}`;
    },
    author: async (book, _, ctx, info) => {
      log.push(`Book.author parent.id=${book.id} path=${pathStr(info.path)}`);
      await sleep(5);
      return ctx.db.authors.find((a) => a.id === book.author_id);
    },
  },
  Author: {
    books: (author, _, ctx) => ctx.db.books.filter((b) => b.author_id === author.id),
  },
};

function pathStr(p) {
  const parts = [];
  for (; p; p = p.prev) parts.unshift(p.key);
  return parts.join(".");
}

const schema = makeExecutableSchema({ typeDefs, resolvers });

const r1 = await graphql({
  schema,
  source: `{ me books { title price eur: price(currency: "EUR") author { name } } }`,
  contextValue: { db, user: { name: "ada" } },
});
console.log(JSON.stringify(r1));
console.log(log.join("\n"));
```

Run with `node resolvers05.mjs` (graphql 16.14.2, @graphql-tools/schema 10.1.3):

```console
{"data":{"me":"ada","books":[{"title":"Dune","price":"12.99 USD","eur":"11.69 EUR","author":{"name":"Frank Herbert"}},{"title":"Emma","price":"8.99 USD","eur":"8.09 EUR","author":{"name":"Jane Austen"}}]}}
Query.books parent=undefined path=books
Book.price parent.id=1 args={"currency":"USD"} returnType=String!
Book.price parent.id=1 args={"currency":"EUR"} returnType=String!
Book.author parent.id=1 path=books.0.author
Book.price parent.id=2 args={"currency":"USD"} returnType=String!
Book.price parent.id=2 args={"currency":"EUR"} returnType=String!
Book.author parent.id=2 path=books.1.author
```

Reading the log line by line tells you most of what you need to know.

### `parent` flows downward

`Query.books` got `parent=undefined` because we passed no `rootValue`. It returned an array
of raw database rows. The executor then walked that array and, for each row, ran the
`Book` field resolvers with **that row** as `parent`. That's why `Book.price` sees
`book.priceCents` and `Book.author` sees `book.author_id` — fields that don't exist in the
schema at all. The parent is your internal representation; the schema is the public one.

`Author.books` closes the loop: it receives the author object returned by `Book.author` and
filters by its id. Resolvers never call each other; the executor chains them through
`parent`.

### `args` are already coerced

`price` with no argument arrived as `{"currency":"USD"}` — the schema default was filled in
before the resolver ran. The aliased `eur: price(currency: "EUR")` ran the same resolver a
second time with different args. Aliases don't change the resolver; they only change the
response key.

### `context` is per request

`me` read `ctx.user`, which we passed as `contextValue`. The same object was visible to every
resolver in that request. In an HTTP server you build a **fresh** context for every request
(lesson [07](07-apollo-server-http.md)); that's where the logged-in user, a database
connection and per-request caches live.

### `info` knows where you are

`info.path` is a linked list from the current field back to the root — `books.0.author` is
"the `author` field of item 0 of `books`". `info.returnType` printed as `String!`. Other
useful properties: `info.fieldName`, `info.parentType`, `info.fieldNodes` (the AST of this
field in the query), `info.operation`, `info.variableValues` and `info.schema`.

## The default resolver: why `title` needs no code

We wrote no resolver for `Book.title`, `Book.id` or `Author.name`, yet they resolved. Any
field without a resolver uses the **default field resolver**, which is essentially:

```js
function defaultFieldResolver(parent, args, context, info) {
  const value = parent?.[info.fieldName];
  return typeof value === "function" ? value.call(parent, args, context, info) : value;
}
```

Two consequences:

- If your stored property names already match the schema, you write nothing.
- If they don't match — `author_id` versus a schema field `authorId` — you get `null`
  silently, or an error if the field is non-null. Either rename in the resolver
  (`authorId: (b) => b.author_id`) or reshape rows when you load them.

## Async resolvers and ordering

`Query.books` and `Book.author` return promises. The executor doesn't wait for one
`Book.author` before starting the next: for the list it starts both, then awaits them
together. Sibling fields in a query execute "in parallel" in that sense (still one JS
thread, but the I/O overlaps). Top-level fields of a **mutation** are the exception — they
run strictly one after another, which you'll see in [lesson 06](06-mutations-inputs.md).

The log is in a stable order here because the synchronous resolvers run as the executor
reaches them, and the two `author` calls were *started* in list order. Don't rely on the
completion order of async resolvers; with real I/O it varies run to run.

## When a resolver throws

Same schema, but asking for a currency the resolver doesn't support:

```js
const r2 = await graphql({
  schema,
  source: `{ books { title price(currency: "GBP") } }`,
  contextValue: { db },
});
console.log(JSON.stringify(r2));
```

```console
{"errors":[{"message":"Unsupported currency: GBP","locations":[{"line":1,"column":17}],"path":["books",0,"price"]}],"data":null}
```

The whole `data` became `null`. The error happened at `books.0.price`, but `price` is
`String!`, so it can't be `null`; the null moves up to the `Book!` list item, which also
can't be `null`; that moves up to `books: [Book!]!`, which can't be `null` either, so it
lands on `data`. Only **one** error is reported even though book 2 would have failed too:
once the list was doomed, `graphql-js` stopped completing its remaining items.

That's non-null propagation, and the lesson is that `!` on output types is a promise you
must keep. [Lesson 08](08-errors.md) covers it fully; for now, note that making `price`
nullable would have given you both books' titles plus an error on each `price`.

## Writing resolvers well

A few habits that pay off from day one:

- **Keep resolvers thin.** Put business rules in plain functions or a model layer, and
  call them from resolvers. Resolvers are glue between the schema and your code, and thin
  glue is easy to test.
- **Return the parent's raw data and let child resolvers compute.** `Query.books` returns
  rows with `author_id`; it doesn't fetch authors up front, because the client may not ask
  for them. Each field does its own work only when selected.
- **Get everything shared from `context`.** No module-level `currentUser` variables —
  Node serves many requests concurrently, and a module-level variable would leak one
  user's identity into another's request.
- **Return `null` deliberately.** For a nullable field, `null` means "no value". Don't
  throw just because something is absent.

## How It Actually Works

When `graphql-js` executes an operation, it first *collects fields*: it flattens the
selection set (expanding fragments and evaluating `@skip`/`@include`) into an ordered map
of response key → field nodes. Then, for each entry, `executeField` looks up the field
definition on the parent type, builds `args` with `getArgumentValues` (applying defaults and
coercing variables), creates the `info` object, and calls `field.resolve ?? defaultFieldResolver`.

The return value goes into `completeValue`, which is driven by the field's **return type**:

- non-null wrapper → complete the inner type, then raise an error if the result is `null`;
- list → check it's iterable, then complete each item with the item type (each item gets a
  path segment with its index);
- leaf (scalar/enum) → call the type's `serialize`;
- object → collect the sub-selection's fields and recurse with the returned value as the
  new `parent`.

If any of those return promises, the executor gathers them with `Promise.all`-style
helpers and returns a promise; otherwise everything stays synchronous. An exception from a
resolver is caught, wrapped in a `GraphQLError` with `path` and `locations`, pushed to the
error list, and the field becomes `null` — then non-null rules decide how far up that
`null` travels. That one recursive function is the whole runtime.

## Common mistakes

- **Expecting `parent` to match the schema type.** It's whatever the parent resolver
  returned. If you return a Mongo document or an ORM model, that's what child resolvers see.
- **Fetching child data in the parent "to be efficient".** It wastes work when the client
  didn't ask for the field. Fix repeated child fetches with batching
  ([Level 2 · 06](../level-2/06-dataloader.md)) instead.
- **Mutating `context` inside resolvers to pass data between fields.** Sibling resolvers
  run in an order you shouldn't rely on. Pass data downward through `parent`.
- **Forgetting `await` inside an `async` resolver**, then returning a pending inner
  promise inside an object. The executor resolves the returned promise, not promises
  nested inside a plain object's properties.
- **Throwing for not-found on a nullable field.** Return `null`; reserve errors for things
  that actually went wrong.

## Exercise

1. Add `authorId: ID!` to `Book` in the schema without writing a resolver. Query it. What
   do you get, and why? Fix it with a one-line resolver.
2. Add `Book.priceHistory: [Int!]!` whose resolver returns `undefined`. Predict the response
   before running it.
3. Make `price` nullable (`String`) and re-run the GBP query. How many errors are there now,
   and what does `data` contain?
4. Log `info.fieldNodes[0].selectionSet?.selections.map(s => s.name.value)` inside
   `Query.books`. What could you use that for — and why is it a risky optimisation?
