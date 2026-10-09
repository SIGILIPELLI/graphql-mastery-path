---
description: "Model 'one of several types' with GraphQL interfaces and unions: inline fragments, __resolveType, the Node pattern and common errors."
---

# 02 · Interfaces and Unions

Real data isn't always one type. A search box returns books, authors and series. A feed mixes
posts, comments and reactions. A `node(id:)` lookup can return anything. GraphQL has two
**abstract types** for this: **interfaces**, which promise a common set of fields, and
**unions**, which promise nothing except "one of these". This lesson builds a schema with
both, queries them, and then breaks them in every way the executor and validator will tell
you about.

## The schema

```js title="abstract02.mjs"
import { makeExecutableSchema } from "@graphql-tools/schema";
import { graphql } from "graphql";

const rows = [
  { kind: "book", id: "b1", title: "Dune", pages: 412, authorId: "a1" },
  { kind: "author", id: "a1", name: "Frank Herbert" },
  { kind: "audiobook", id: "ab1", title: "Dune (Unabridged)", minutes: 1258, narrator: "Scott Brick", authorId: "a1" },
  { kind: "series", id: "s1", name: "Dune Chronicles", size: 6 },
];

const typeDefs = /* GraphQL */ `
  interface Node { id: ID! }

  interface Publication implements Node {
    id: ID!
    title: String!
    author: Author!
  }

  type Book implements Publication & Node { id: ID! title: String! author: Author! pages: Int! }
  type Audiobook implements Publication & Node {
    id: ID! title: String! author: Author! minutes: Int! narrator: String!
  }
  type Author implements Node { id: ID! name: String! }
  type Series { name: String! size: Int! }

  union SearchResult = Book | Audiobook | Author | Series

  type Query {
    node(id: ID!): Node
    publications: [Publication!]!
    search(text: String!): [SearchResult!]!
  }
`;

const typeByKind = { book: "Book", audiobook: "Audiobook", author: "Author", series: "Series" };

const resolvers = {
  Query: {
    node: (_, { id }) => rows.find((r) => r.id === id) ?? null,
    publications: () => rows.filter((r) => r.kind === "book" || r.kind === "audiobook"),
    search: (_, { text }) =>
      rows.filter((r) => (r.title ?? r.name).toLowerCase().includes(text.toLowerCase())),
  },
  Node: { __resolveType: (r) => typeByKind[r.kind] },
  Publication: { __resolveType: (r) => typeByKind[r.kind] },
  SearchResult: { __resolveType: (r) => typeByKind[r.kind] },
  Book: { author: (b) => rows.find((r) => r.id === b.authorId) },
  Audiobook: { author: (b) => rows.find((r) => r.id === b.authorId) },
};

const schema = makeExecutableSchema({ typeDefs, resolvers });
```

Points about the SDL:

- `type Book implements Publication & Node` — a type can implement several interfaces, and
  must declare every field of each, with compatible types.
- `interface Publication implements Node` — interfaces can implement interfaces (added to the
  spec in 2021). An implementing type must then list *both* interfaces, which is why `Book`
  says `Publication & Node`.
- `Series` implements nothing and has no `id`; it can still be a union member. Unions only
  contain object types, never interfaces, scalars or other unions.

## `__resolveType`: telling the executor what you returned

A resolver returning a `Node` returns some JavaScript object. The executor has to decide
whether that object is a `Book` or an `Author` before it can resolve sub-fields, and it asks
the abstract type's `__resolveType` function, which returns a type **name**. Here every row
carries a `kind` discriminator, so one lookup table serves all three abstract types.

Store a discriminator in your data. Guessing from the shape (`"pages" in obj ? "Book" : …`)
works until two types share a field.

## Querying an interface

```js
await run("interface fields + inline fragments", `{
  publications {
    __typename title author { name }
    ... on Book { pages }
    ... on Audiobook { minutes narrator }
  }
}`);
```

```text
--- interface fields + inline fragments
{"data":{"publications":[{"__typename":"Book","title":"Dune","author":{"name":"Frank Herbert"},"pages":412},{"__typename":"Audiobook","title":"Dune (Unabridged)","author":{"name":"Frank Herbert"},"minutes":1258,"narrator":"Scott Brick"}]}}
```

Fields declared on the interface (`title`, `author`) can be selected directly. Fields that
only some implementations have need an **inline fragment** — `... on Book { pages }` — which
applies only when the runtime type matches. `__typename` tells the client which branch it got,
and a typed client (or a TypeScript discriminated union generated from this query) switches
on it.

## Querying a union

```js
await run("union needs fragments for every field", `{
  search(text: "dune") {
    __typename
    ... on Publication { title }
    ... on Series { name size }
  }
}`);
```

```text
--- union needs fragments for every field
{"data":{"search":[{"__typename":"Book","title":"Dune"},{"__typename":"Audiobook","title":"Dune (Unabridged)"},{"__typename":"Series","name":"Dune Chronicles","size":6}]}}
```

A union has no fields of its own (except `__typename`), so *everything* goes in fragments.
Note `... on Publication { title }`: a fragment's type condition can be an **interface**, and
it matches any member that implements it — so one fragment covered both `Book` and
`Audiobook`. No `Author` appeared because "Frank Herbert" doesn't contain "dune". If one
had matched, it would have come back as `{"__typename":"Author"}` with no other keys, because
no fragment selects fields for `Author`. That's legal, and clients must expect it.

## The `Node` pattern

```js
await run("node lookup", `{ a: node(id: "a1") { id __typename ... on Author { name } } b: node(id: "b1") { id ... on Book { title } } }`);
```

```text
--- node lookup
{"data":{"a":{"id":"a1","__typename":"Author","name":"Frank Herbert"},"b":{"id":"b1","title":"Dune"}}}
```

An interface `Node { id: ID! }` plus a root `node(id:)` field lets a client refetch *any*
object by id — the backbone of normalized client caches ([Level 3 · 07](../level-3/07-apollo-client-cache.md))
and part of the Relay conventions. For it to work, ids must be **globally unique** across
types. Our rows have prefixes (`a1`, `b1`); production systems often base64-encode
`"Type:dbId"` so the server can route the lookup without searching every table.

## The validation errors you'll meet

```js
await run("selecting a field directly on a union", `{ search(text: "dune") { title } }`);
await run("selecting a member-only field on an interface", `{ publications { pages } }`);
await run("impossible fragment", `{ publications { ... on Series { name } } }`);
```

```text
--- selecting a field directly on a union
{"errors":[{"message":"Cannot query field \"title\" on type \"SearchResult\". Did you mean to use an inline fragment on \"Publication\", \"Audiobook\", or \"Book\"?","locations":[{"line":1,"column":26}]}]}
--- selecting a member-only field on an interface
{"errors":[{"message":"Cannot query field \"pages\" on type \"Publication\". Did you mean to use an inline fragment on \"Book\"?","locations":[{"line":1,"column":18}]}]}
--- impossible fragment
{"errors":[{"message":"Fragment cannot be spread here as objects of type \"Publication\" can never be of type \"Series\".","locations":[{"line":1,"column":18}]}]}
```

The validator is helpful here: it suggests the fragment you probably meant. The third error
shows it also knows the *possible types* of each abstract type, so fragments that could
never match are rejected outright.

## When `__resolveType` goes wrong

A separate experiment with a `Pet` interface and two implementations:

```js title="noresolve.mjs"
import { makeExecutableSchema } from "@graphql-tools/schema";
import { graphql } from "graphql";
const typeDefs = `interface Pet { name: String! } type Cat implements Pet { name: String! lives: Int! } type Dog implements Pet { name: String! } type Query { pets: [Pet!]! }`;
const pets = [{ name: "Tom", lives: 9 }, { name: "Rex" }];
const run = async (label, resolvers) => {
  const schema = makeExecutableSchema({ typeDefs, resolvers: { Query: { pets: () => pets }, ...resolvers } });
  console.log(label, JSON.stringify(await graphql({ schema, source: "{ pets { __typename name } }" })));
};
await run("no __resolveType:", {});
pets[0].__typename = "Cat"; pets[1].__typename = "Dog";
await run("with __typename field:", {});
await run("resolveType returns bad name:", { Pet: { __resolveType: () => "Hamster" } });
```

```text
no __resolveType: {"errors":[{"message":"Abstract type \"Pet\" must resolve to an Object type at runtime for field \"Query.pets\". Either the \"Pet\" type should provide a \"resolveType\" function or each possible type should provide an \"isTypeOf\" function.","locations":[{"line":1,"column":3}],"path":["pets",0]}],"data":null}
with __typename field: {"data":{"pets":[{"__typename":"Cat","name":"Tom"},{"__typename":"Dog","name":"Rex"}]}}
resolveType returns bad name: {"errors":[{"message":"Abstract type \"Pet\" was resolved to a type \"Hamster\" that does not exist inside the schema.","locations":[{"line":1,"column":3}],"path":["pets",0]}],"data":null}
```

Three lessons:

1. Forgetting `__resolveType` isn't caught when the schema is built — `makeExecutableSchema`
   printed nothing — it fails at runtime on the first abstract value.
2. Without a `__resolveType`, the default implementation looks for a `__typename` property on
   the value. Setting it in your data layer is a legitimate alternative.
3. A typo in the returned name is a runtime error too. Tests that query each abstract field
   catch all three.

## Interface or union?

| Choose an interface when… | Choose a union when… |
|---|---|
| the types share meaningful fields clients query the same way (`title`, `author`) | the types have little in common (a search over books, people and places) |
| you want clients to handle unknown future implementations generically | the set is closed and each member is rendered differently |
| you need `node(id:)`-style polymorphic lookup | you're modelling a result-or-error outcome ([lesson 08](08-mutation-design.md)) |

Adding a new implementation or union member is a *dangerous* (not breaking) change: old
clients receive a `__typename` they've never seen. Make "unknown type" a handled case in UI
code from day one.

## How It Actually Works

When `completeValue` meets an abstract return type, it calls `completeAbstractValue`. That
function calls `returnType.resolveType(value, context, info, returnType)` — the function
`makeExecutableSchema` attached from your `__resolveType` — or, if there isn't one,
`defaultTypeResolver`. The default first checks `value.__typename` (if it's a string, that's
the answer), and otherwise calls `isTypeOf(value)` on each possible type in turn, returning
the first that says yes. The returned name is looked up in the schema and checked to be one
of `schema.getPossibleTypes(returnType)`; then execution continues as if the field had
returned that concrete object type.

On the client side, inline fragments are evaluated during field collection: a fragment whose
type condition is an interface or union applies when the runtime type is one of its possible
types (`doesFragmentConditionMatch`). That's how `... on Publication` matched a `Book`.

## Common mistakes

- **No discriminator in the data**, leading to shape-sniffing `__resolveType` functions that
  break when types converge.
- **Assuming you'll get fields for every union member.** A member with no matching fragment
  arrives as just `{ __typename }` — and only if you asked for `__typename`.
- **Non-global ids with a `Node` interface.** `node(id: "1")` can't know which table to search.
- **Interfaces as code reuse.** An interface is a promise to clients, not an inheritance
  mechanism for the server. Don't add one just because two types share fields internally.
- **Forgetting both interfaces** when a type implements an interface that itself implements
  another — schema validation will reject it.

## Exercise

1. Add `type Podcast implements Publication & Node` with `episodes: Int!`. Which queries in this
   lesson need changing to show its extra field, and which keep working untouched?
2. Make `search` return an `Author` and observe the shape when no fragment covers it.
3. Replace the `__resolveType` functions with `isTypeOf` on each object type and confirm the
   results are identical.
4. Change ids to base64 `"Type:id"` global ids and make `node(id:)` decode the type instead
   of scanning every row.
