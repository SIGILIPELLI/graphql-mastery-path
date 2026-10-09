---
description: "Build one schema code-first twice — graphql-js type objects and Python's Strawberry — and compare with SDL-first, nullability included."
---

# 07 · Code-First Schemas: graphql-js Types & Strawberry

Every schema in this course so far was **SDL-first**: write the schema in GraphQL's own
language, then attach resolvers. The alternative is **code-first**: write types in your
programming language and *generate* the SDL from them. Many popular libraries are code-first —
Strawberry and Graphene in Python, Pothos and Nexus in TypeScript, Hot Chocolate in .NET (which
supports both styles). This lesson builds one small schema code-first in
JavaScript (with `graphql-js`'s own classes, which every SDL-first schema becomes anyway) and in
Python with **Strawberry** 0.332.0, then compares the printed results line by line.

## Code-first with graphql-js

`buildSchema` and `makeExecutableSchema` produce `GraphQLObjectType`, `GraphQLField` and friends
from SDL. You can construct those objects directly:

```js title="codefirst07.mjs"
import {
  GraphQLSchema, GraphQLObjectType, GraphQLString, GraphQLInt, GraphQLID, GraphQLNonNull, GraphQLList,
  GraphQLEnumType, graphql, printSchema,
} from "graphql";

const authors = [{ id: "a1", name: "Ursula K. Le Guin" }];
const books = [
  { id: "b1", title: "The Dispossessed", year: 1974, authorId: "a1", format: "pb" },
  { id: "b2", title: "The Lathe of Heaven", year: 1971, authorId: "a1", format: "eb" },
];

const Format = new GraphQLEnumType({
  name: "Format",
  values: { PAPERBACK: { value: "pb" }, EBOOK: { value: "eb", description: "DRM-free EPUB" } },
});

// Types reference each other, so `fields` is a thunk (a function) evaluated lazily.
const Author = new GraphQLObjectType({
  name: "Author",
  fields: () => ({
    id: { type: new GraphQLNonNull(GraphQLID) },
    name: { type: new GraphQLNonNull(GraphQLString) },
    books: {
      type: new GraphQLNonNull(new GraphQLList(new GraphQLNonNull(Book))),
      resolve: (author) => books.filter((b) => b.authorId === author.id),
    },
  }),
});

const Book = new GraphQLObjectType({
  name: "Book",
  description: "A published work.",
  fields: () => ({
    id: { type: new GraphQLNonNull(GraphQLID) },
    title: { type: new GraphQLNonNull(GraphQLString) },
    year: { type: GraphQLInt },
    format: { type: new GraphQLNonNull(Format) },
    author: { type: new GraphQLNonNull(Author), resolve: (b) => authors.find((a) => a.id === b.authorId) },
    // extensions: arbitrary metadata tools can read (e.g. cost estimators)
    similar: {
      type: new GraphQLList(new GraphQLNonNull(Book)),
      args: { first: { type: GraphQLInt, defaultValue: 3 } },
      extensions: { complexity: 10 },
      resolve: (b, { first }) => books.filter((x) => x.id !== b.id).slice(0, first),
    },
  }),
});

const schema = new GraphQLSchema({
  query: new GraphQLObjectType({
    name: "Query",
    fields: { books: { type: new GraphQLNonNull(new GraphQLList(new GraphQLNonNull(Book))), resolve: () => books } },
  }),
});

console.log(printSchema(schema));
console.log(JSON.stringify(await graphql({ schema, source: "{ books { title format author { name books { title } } similar(first: 1) { title } } }" })));
console.log("Book.similar extensions:", JSON.stringify(schema.getType("Book").getFields().similar.extensions));
```

Things only visible in this style:

- **Thunks.** `Author.books` returns `Book` and `Book.author` returns `Author`. Since one must be
  defined first, `fields` is a *function*, called only when the schema is built — the standard way
  to express cycles in code-first libraries.
- **Resolvers live next to fields.** There's no separate resolver map to keep in sync; a typo in a
  field name can't leave a resolver unattached.
- **Internal enum values.** `PAPERBACK: { value: "pb" }` means resolvers return `"pb"` and clients
  see `PAPERBACK` — the mapping you'd otherwise write by hand.
- **`extensions`.** Arbitrary metadata on any type or field (`{ complexity: 10 }`), readable by tools
  such as `fieldExtensionsEstimator` from [Level 3 · 05](../level-3/05-security-limits.md). SDL has
  no syntax for this; directives are the nearest equivalent.

The printed SDL and a query result:

```graphql
type Query {
  books: [Book!]!
}

"""A published work."""
type Book {
  id: ID!
  title: String!
  year: Int
  format: Format!
  author: Author!
  similar(first: Int = 3): [Book!]
}

enum Format {
  PAPERBACK

  """DRM-free EPUB"""
  EBOOK
}

type Author {
  id: ID!
  name: String!
  books: [Book!]!
}
```

```text
{"data":{"books":[{"title":"The Dispossessed","format":"PAPERBACK","author":{"name":"Ursula K. Le Guin","books":[{"title":"The Dispossessed"},{"title":"The Lathe of Heaven"}]},"similar":[{"title":"The Lathe of Heaven"}]},…]}}
Book.similar extensions: {"complexity":10}
```

Writing `new GraphQLNonNull(new GraphQLList(new GraphQLNonNull(Book)))` by hand is verbose, which
is why TypeScript projects use builders like Pothos that infer these from types. But it's worth
seeing once: it's what every GraphQL JavaScript server runs on.

## Code-first in Python with Strawberry

```bash
python3 -m venv .venv && .venv/bin/pip install strawberry-graphql
```

```python title="schema.py"
from __future__ import annotations

import enum
from typing import Optional

import strawberry


@strawberry.enum
class Format(enum.Enum):
    PAPERBACK = "pb"
    EBOOK = "eb"


AUTHORS = {"a1": {"id": "a1", "name": "Ursula K. Le Guin"}}
BOOKS = [
    {"id": "b1", "title": "The Dispossessed", "year": 1974, "author_id": "a1", "format": "pb"},
    {"id": "b2", "title": "The Lathe of Heaven", "year": 1971, "author_id": "a1", "format": "eb"},
]


@strawberry.type
class Author:
    id: strawberry.ID
    name: str

    @strawberry.field
    def books(self) -> list[Book]:
        return [Book.from_row(b) for b in BOOKS if b["author_id"] == self.id]


@strawberry.type(description="A published work.")
class Book:
    id: strawberry.ID
    title: str
    year: Optional[int]
    format: Format
    author_id: strawberry.Private[str]  # internal: not exposed in the schema

    @strawberry.field
    def author(self) -> Author:
        return Author(**AUTHORS[self.author_id])

    @strawberry.field
    def similar(self, first: int = 3) -> list[Book]:
        return [Book.from_row(b) for b in BOOKS if b["id"] != self.id][:first]

    @classmethod
    def from_row(cls, row: dict) -> Book:
        return cls(id=row["id"], title=row["title"], year=row["year"],
                   format=Format(row["format"]), author_id=row["author_id"])


@strawberry.type
class Query:
    @strawberry.field
    def books(self) -> list[Book]:
        return [Book.from_row(b) for b in BOOKS]


schema = strawberry.Schema(query=Query)

if __name__ == "__main__":
    import json
    print(schema.as_str())
    result = schema.execute_sync("{ books { title format author { name books { title } } similar(first: 1) { title } } }")
    print(json.dumps({"data": result.data, "errors": [str(e) for e in result.errors or []] or None}))
```

Strawberry reads **type annotations**: a `str` field becomes `String!`, `Optional[int]` becomes
`Int`, `list[Book]` becomes `[Book!]!`, and a method decorated with `@strawberry.field` becomes a
field with a resolver whose parameters become arguments. `strawberry.Private[str]` keeps
`author_id` on the Python object for resolvers to use without exposing it in the schema — the
code-first counterpart of the "rows as parents" pattern from Level 2.

```console
$ .venv/bin/python schema.py
type Author {
  id: ID!
  name: String!
  books: [Book!]!
}

"""A published work."""
type Book {
  id: ID!
  title: String!
  year: Int
  format: Format!
  author: Author!
  similar(first: Int! = 3): [Book!]!
}

enum Format {
  PAPERBACK
  EBOOK
}

type Query {
  books: [Book!]!
}
{"data": {"books": [{"title": "The Dispossessed", "format": "PAPERBACK", …}]}, "errors": null}
```

(Result abbreviated; it matched the JavaScript version.) Strawberry runs on `graphql-core` (3.3.0
here), the Python port of `graphql-js`, so execution semantics are the same.

## Spot the difference

The two printed schemas differ on one line:

```graphql
similar(first: Int = 3): [Book!]          # graphql-js version
similar(first: Int! = 3): [Book!]!        # Strawberry version
```

In Python, `first: int = 3` is a non-optional `int`, so Strawberry made the argument `Int!` (with a
default, so clients can still omit it — but can't pass `null`). The return annotation `list[Book]`
is non-optional, so the result became `[Book!]!`. Neither is wrong, but they're different API
contracts, and by the rules in [Level 2 · 01](../level-2/01-schema-design.md) the non-null output is
the one you can't loosen later. In code-first schemas, **nullability comes from your language's type
annotations** — `Optional[...]` in Python, `nullable: true` options in most TypeScript builders — so
review the generated SDL, not just the code.

The Strawberry enum also lacks the `EBOOK` description; it would need `strawberry.enum_value(...,
description=...)`. Metadata that SDL makes obvious is easy to forget in code.

## SDL-first or code-first?

| | SDL-first | Code-first |
|---|---|---|
| Source of truth | the `.graphql` file | the code; SDL is generated |
| Schema review | read the SDL diff directly | review the *generated* SDL (commit it) |
| Resolver/schema drift | possible (caught at startup by `graphql-tools`) | impossible by construction |
| Type safety | via codegen ([Level 3 · 08](../level-3/08-codegen.md)) | native to the language's types |
| Design-first workflow with clients | natural | possible, but the schema emerges from code |
| Cross-language teams | SDL is a neutral contract | each language has its own library |

Both produce identical runtime schemas. Many teams that choose code-first still commit the
generated SDL and run the breaking-change checks from [lesson 05](05-schema-evolution.md) on it, which
gives reviewers the best of both.

## How It Actually Works

A `GraphQLObjectType` stores its config and resolves `fields` lazily: the first call to
`getFields()` invokes the thunk, wraps each entry in a `GraphQLField` (normalising `args` into
argument definitions and keeping `resolve`, `description`, `deprecationReason` and `extensions`),
and caches the map. `new GraphQLSchema({ query })` walks every type reachable from the root types
(`collectReferencedTypes`) to build the type map, which is when the thunks run — that's how two types
can refer to each other. SDL-first builds go through `buildASTSchema`, which ends up constructing
exactly these objects, with resolvers attached afterwards.

Strawberry's decorators turn each class into a dataclass and record a `StrawberryField` for each
annotated attribute or decorated method. When `strawberry.Schema` is created, a schema converter maps
those to `graphql-core` types, translating annotations into wrappers (`Optional[T]` → nullable, otherwise
`GraphQLNonNull`), method signatures into arguments, and methods into resolvers that receive `self` as the
parent object.

## Common mistakes

- **Not reviewing generated SDL**, so annotation choices silently become non-null API contracts.
- **Exposing internal fields** in code-first types by accident (use `Private`/omit them explicitly).
- **Forgetting thunks** for mutually recursive types in hand-written `graphql-js` types — you get a
  `ReferenceError` at startup.
- **Mixing SDL and code-first** carelessly in one schema, leaving no single source of truth.
- **Assuming descriptions carry over** from code comments — they usually need explicit description
  parameters.

## Exercise

1. Change the Strawberry `similar` field so its SDL matches the graphql-js version exactly
   (`first: Int = 3` and a nullable list).
2. Add `@strawberry.enum_value` descriptions and a deprecated field (`deprecation_reason=`) and check the
   printed SDL.
3. Add a mutation `addBook(title: String!): Book!` in both styles.
4. Add a breaking-change check that compares the committed SDL of the Strawberry schema with
   `schema.as_str()` in CI. (Hint: `graphql-core` has `find_breaking_changes`.)
