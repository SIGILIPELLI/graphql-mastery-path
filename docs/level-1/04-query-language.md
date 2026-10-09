---
description: "The GraphQL query language hands-on: arguments, aliases, fragments, variables with defaults, @include/@skip, __typename and named operations."
---

# 04 · Writing Queries: Arguments, Aliases, Fragments, Variables & Directives

The schema says what *can* be asked. The query language is how a client asks. This lesson
runs every major feature of the language against one small schema, including the ways each
one fails, so you recognise the error messages when you meet them in real clients.

## The schema we'll query

```js title="schema04.mjs"
import { makeExecutableSchema } from "@graphql-tools/schema";

export const books = [
  { id: "1", title: "Dune", year: 1965, authorId: "a1", tags: ["sci-fi", "classic"] },
  { id: "2", title: "Emma", year: 1815, authorId: "a2", tags: ["classic"] },
  { id: "3", title: "Neuromancer", year: 1984, authorId: "a3", tags: ["sci-fi", "cyberpunk"] },
];
const authors = {
  a1: { id: "a1", name: "Frank Herbert" },
  a2: { id: "a2", name: "Jane Austen" },
  a3: { id: "a3", name: "William Gibson" },
};

export const schema = makeExecutableSchema({
  typeDefs: `
    type Query {
      book(id: ID!): Book
      books(tag: String, limit: Int = 10): [Book!]!
    }
    type Book {
      id: ID!
      title: String!
      year: Int!
      tags: [String!]!
      author: Author!
      coverUrl(width: Int = 300): String!
    }
    type Author { id: ID! name: String! }
  `,
  resolvers: {
    Query: {
      book: (_, { id }) => books.find((b) => b.id === id) ?? null,
      books: (_, { tag, limit }) =>
        books.filter((b) => !tag || b.tags.includes(tag)).slice(0, limit),
    },
    Book: {
      author: (b) => authors[b.authorId],
      coverUrl: (b, { width }) => `https://covers.example/${b.id}?w=${width}`,
    },
  },
});
```

And a small runner that prints each result on one line:

```js title="queries.mjs"
import { graphql } from "graphql";
import { schema } from "./schema04.mjs";

const run = async (label, source, variableValues, operationName) => {
  const r = await graphql({ schema, source, variableValues, operationName });
  console.log(`--- ${label}\n${JSON.stringify(r, null, 1).replace(/\n\s*/g, " ")}`);
};
// ...each run(...) call below is appended here
```

All outputs below are exactly what this printed with `graphql` 16.14.2.

## Arguments and aliases

Response keys default to field names, so you can't ask for the same field twice with
different arguments — the keys would collide. An **alias** renames the key:

```js
await run("aliases", `{
  dune: book(id: "1") { title }
  emma: book(id: "2") { title small: coverUrl(width: 80) large: coverUrl }
}`);
```

```text
--- aliases
{ "data": { "dune": { "title": "Dune" }, "emma": { "title": "Emma", "small": "https://covers.example/2?w=80", "large": "https://covers.example/2?w=300" } } }
```

`large: coverUrl` omitted `width`, so the schema default `300` applied. Without aliases:

```js
await run("same field, different args, no alias", `{ book(id: "1") { title } book(id: "2") { title } }`);
```

```text
{ "errors": [ { "message": "Fields \"book\" conflict because they have differing arguments. Use different aliases on the fields to fetch both if this was intentional.", "locations": [ { "line": 1, "column": 3 }, { "line": 1, "column": 27 } ] } ] }
```

Selecting the *same* field with the *same* arguments twice is fine — the selections are
merged. The conflict only arises when they'd produce different values under one key.

## Fragments

A **fragment** is a named, reusable selection on a type. Client codebases use them heavily:
each UI component declares the fields it needs as a fragment, and the page query spreads them.

```js
await run("fragment", `
  query {
    scifi: books(tag: "sci-fi") { ...BookCard }
    all: books(limit: 1) { ...BookCard }
  }
  fragment BookCard on Book { id title author { name } }`);
```

```text
--- fragment
{ "data": { "scifi": [ { "id": "1", "title": "Dune", "author": { "name": "Frank Herbert" } }, { "id": "3", "title": "Neuromancer", "author": { "name": "William Gibson" } } ], "all": [ { "id": "1", "title": "Dune", "author": { "name": "Frank Herbert" } } ] } }
```

`on Book` is the fragment's **type condition**. Spreading it where the type isn't `Book` is
a validation error. An *inline* fragment (`... on Book { title }`) has no name; it matters
for interfaces and unions, covered in [Level 2 · 02](../level-2/02-interfaces-unions.md).
Every fragment you define must be used, or validation fails.

## Variables

Hard-coding values into the query string means building strings at runtime — awkward,
injection-prone, and it defeats caching of the query text. **Variables** separate the
query from its inputs:

```js
const shelf = `query Shelf($tag: String = "classic", $limit: Int) {
  books(tag: $tag, limit: $limit) { title }
}`;
await run("variables + defaults", shelf, {});
await run("variables provided", shelf, { tag: "sci-fi", limit: 1 });
```

```text
--- variables + defaults
{ "data": { "books": [ { "title": "Dune" }, { "title": "Emma" } ] } }
--- variables provided
{ "data": { "books": [ { "title": "Dune" } ] } }
```

With no variables sent, `$tag` took its declared default `"classic"`. `$limit` had no
default and wasn't provided, so the `limit` argument counted as *not provided* and the
**schema's** default (`10`) applied.

Now send an explicit `null`:

```js
await run("explicit null limit", `query Shelf($limit: Int) { books(limit: $limit) { title } }`, { limit: null });
```

```text
--- explicit null limit
{ "data": { "books": [] } }
```

An empty list, and no error. Explicit `null` is *not* the same as "not provided": the
resolver received `limit: null`, and `Array.prototype.slice(0, null)` treats `null` as
`0`. This is a real bug in our resolver, and a common one. Either make the argument
`limit: Int! = 10` (so `null` is rejected) or handle `null` in the resolver.

Variables are validated too:

```js
await run("unused variable", `query($x: Int) { book(id: "1") { title } }`);
await run("undefined variable", `query { books(limit: $n) { title } }`);
```

```text
{ "errors": [ { "message": "Variable \"$x\" is never used.", "locations": [ { "line": 1, "column": 7 } ] } ] }
{ "errors": [ { "message": "Variable \"$n\" is not defined.", "locations": [ { "line": 1, "column": 22 }, { "line": 1, "column": 1 } ] } ] }
```

Variables can only be scalars, enums, input objects or lists of them — never object types.

## Directives: @include and @skip

The two built-in executable directives drop a field (or fragment) based on a Boolean:

```js
await run("directives", `query Card($withAuthor: Boolean!, $compact: Boolean!) {
  book(id: "3") {
    title
    year @skip(if: $compact)
    author @include(if: $withAuthor) { name }
  }
}`, { withAuthor: false, compact: true });
```

```text
--- directives
{ "data": { "book": { "title": "Neuromancer" } } }
```

The skipped fields aren't returned as `null` — their keys are absent, and their resolvers
never ran. That's the difference from asking for a field and getting `null`.

## __typename

Every object type has an implicit `__typename` field returning the type's name:

```text
--- __typename
{ "data": { "book": { "__typename": "Book", "title": "Dune", "author": { "__typename": "Author", "name": "Frank Herbert" } } } }
```

Client caches (Apollo Client adds it to every selection automatically) use `__typename` plus
`id` as the cache key, and it's how a client tells union members apart.

## Operation names and multiple operations

A document may contain several operations. The server then needs to know which to run:

```js
const two = `query A { book(id: "1") { title } } query B { book(id: "2") { title } }`;
await run("two operations, no name", two);
await run("two operations, operationName B", two, undefined, "B");
```

```text
--- two operations, no name
{ "errors": [ { "message": "Must provide operation name if query contains multiple operations." } ] }
--- two operations, operationName B
{ "data": { "book": { "title": "Emma" } } }
```

Even with one operation, **name it**. Names show up in server logs, tracing and metrics
([Level 4 · 02](../level-4/02-plugins-performance.md)); `query { ... }` everywhere makes a
production incident much harder to trace back to a screen.

## How It Actually Works

Before execution, `graphql-js` runs **field collection**: for each selection set it walks
the selections, evaluates `@skip`/`@include` against the coerced variables, expands
fragment spreads and inline fragments whose type condition matches the runtime type, and
groups the resulting fields by **response key** (the alias, or the field name). Fields that
share a key are merged into one execution of the resolver with their sub-selections combined.
That's why duplicate selections are harmless and why validation (`OverlappingFieldsCanBeMergedRule`)
must reject same-key fields whose arguments differ — there'd be no single answer.

Variable handling happens in `getVariableValues` before any resolver runs: each declared
variable is looked up in the JSON map, its default applied if absent, and its value run
through the type's `parseValue`. Argument values are then built by `getArgumentValues`,
which distinguishes three cases — absent (use the argument's default or leave the key out),
explicit `null` (pass `null`), and a value (coerce it). Your resolver's `args` object
reflects exactly that, which is how `limit: null` reached `slice`.

## Common mistakes

- **String-building queries** (`` `book(id: "${id}")` ``) instead of using variables.
  It breaks on quotes, invites injection of extra fields, and prevents persisted queries.
- **Assuming `null` means "use the default".** Only an omitted argument gets the default.
- **Forgetting aliases** when fetching the same field twice; the error tells you, but only
  at runtime if the query is built dynamically.
- **Anonymous operations.** They work, but you lose observability.
- **Over-sharing fragments.** A fragment used by a dozen screens means every screen fetches
  every field any of them needs. Prefer one fragment per component.

## Exercise

1. Write one query that returns Dune's title, its cover at widths 100 and 600, and the names
   of the authors of all classic books, using aliases where needed.
2. Turn it into a named operation with variables `$id: ID!` and `$tag: String`, plus a
   `$withCovers: Boolean! = true` variable controlling the cover fields.
3. Fix the `limit: null` bug two ways — in the schema and in the resolver — and decide which
   you prefer.
4. Define a fragment `on Author` and try to spread it inside `book { ... }`. Read the
   validation error.
