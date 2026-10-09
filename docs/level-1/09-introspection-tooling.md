---
description: "Query a GraphQL schema about itself with __schema and __type, dump SDL from a live server, and diff two schemas for breaking changes."
---

# 09 · Introspection, SDL & Tooling

A GraphQL server can describe itself. The same endpoint that answers `{ books { title } }`
also answers "what types do you have, what fields do they have, which are deprecated?" This
feature — **introspection** — is why GraphQL IDEs autocomplete, why code generators can
produce typed clients, and why schema diffs can be automated. It's also something you may
want to switch off in production. This lesson queries it by hand, then builds the two tools
every GraphQL team ends up with: a schema dumper and a breaking-change check.

## A schema with documentation

```js title="intro09.mjs"
import { makeExecutableSchema } from "@graphql-tools/schema";
import {
  graphql, getIntrospectionQuery, buildClientSchema, printSchema,
  findBreakingChanges, buildSchema,
} from "graphql";

const typeDefs = /* GraphQL */ `
  """A book in the catalog."""
  type Book {
    id: ID!
    title: String!
    "Use \`isbn13\` instead."
    isbn: String @deprecated(reason: "Use isbn13.")
    isbn13: String
    format: Format!
  }
  enum Format { HARDCOVER PAPERBACK EBOOK }
  type Query {
    "Look a book up by id."
    book(id: ID!): Book
  }
`;
const schema = makeExecutableSchema({ typeDefs });
```

Strings placed before a definition are **descriptions** — documentation that becomes part of
the schema, not comments. `"…"` and `"""…"""` (block strings, which can span lines) both
work. `# comments` are discarded by the parser and never reach clients.

`@deprecated(reason:)` is a built-in directive. It doesn't stop anyone using the field; it
marks it so tools can warn.

## The meta-fields

Every schema implicitly has two extra root fields, `__schema` and `__type(name:)`, plus a
`__typename` field on every object type. Ask about `Book`:

```js
const r1 = await graphql({ schema, source: `{
  __type(name: "Book") {
    kind name description
    fields { name type { kind name ofType { kind name } } isDeprecated deprecationReason }
  }
}` });
console.log(JSON.stringify(r1.data, null, 1).replace(/\n\s*/g, " "));
```

```text
{ "__type": { "kind": "OBJECT", "name": "Book", "description": "A book in the catalog.", "fields": [ { "name": "id", "type": { "kind": "NON_NULL", "name": null, "ofType": { "kind": "SCALAR", "name": "ID" } }, "isDeprecated": false, "deprecationReason": null }, { "name": "title", "type": { "kind": "NON_NULL", "name": null, "ofType": { "kind": "SCALAR", "name": "String" } }, "isDeprecated": false, "deprecationReason": null }, { "name": "isbn13", "type": { "kind": "SCALAR", "name": "String", "ofType": null }, "isDeprecated": false, "deprecationReason": null }, { "name": "format", "type": { "kind": "NON_NULL", "name": null, "ofType": { "kind": "ENUM", "name": "Format" } }, "isDeprecated": false, "deprecationReason": null } ] } }
```

Two details worth noticing:

- **Wrapper types have no name.** `ID!` is represented as `{ kind: NON_NULL, ofType: { kind:
  SCALAR, name: "ID" } }`. A type like `[Book!]!` nests three levels deep, which is why the
  standard introspection query has a deeply nested `ofType` fragment.
- **`isbn` is missing.** `fields` takes an `includeDeprecated` argument that defaults to
  `false`. To see deprecated fields you must ask for them with `fields(includeDeprecated: true)`.

`__typename` tells a client the concrete type of any object — essential once you have
interfaces and unions ([Level 2 · 02](../level-2/02-interfaces-unions.md)):

```js
const r3 = await graphql({ schema, source: `{ book(id: "1") { __typename } __typename }` });
```

```text
{"data":{"book":null,"__typename":"Query"}}
```

(`book` is `null` because this schema has no resolvers; the root `__typename` still works.)

## The full introspection query and back to SDL

`graphql-js` exports the canonical query that tools send:

```js
const q = getIntrospectionQuery();
console.log("introspection query length:", q.length, "chars");
const full = await graphql({ schema, source: q });
console.log("types in result:", full.data.__schema.types.length,
  full.data.__schema.types.map((t) => t.name).filter((n) => !n.startsWith("__")).join(","));
const clientSchema = buildClientSchema(full.data);
console.log(printSchema(clientSchema));
```

```text
introspection query length: 1927 chars
types in result: 14 Book,ID,String,Format,Query,Boolean
"""A book in the catalog."""
type Book {
  id: ID!
  title: String!

  """Use `isbn13` instead."""
  isbn: String @deprecated(reason: "Use isbn13.")
  isbn13: String
  format: Format!
}

enum Format {
  HARDCOVER
  PAPERBACK
  EBOOK
}

type Query {
  """Look a book up by id."""
  book(id: ID!): Book
}
```

The 14 types are our 6 plus the 8 introspection types (`__Schema`, `__Type`, `__Field`,
`__InputValue`, `__EnumValue`, `__Directive`, `__TypeKind`, `__DirectiveLocation`).
`buildClientSchema` turns the JSON back into a `GraphQLSchema` — one with no resolvers, so
it can validate queries but not execute them. `printSchema` prints it as SDL. That round
trip is the basis of most GraphQL tooling.

## Tool 1: dump a live server's schema

```js title="dump-schema.mjs"
import { getIntrospectionQuery, buildClientSchema, printSchema } from "graphql";
import { writeFile } from "node:fs/promises";

const endpoint = process.argv[2] ?? "http://localhost:4000/";
const res = await fetch(endpoint, {
  method: "POST",
  headers: { "content-type": "application/json" },
  body: JSON.stringify({ query: getIntrospectionQuery() }),
});
const { data, errors } = await res.json();
if (errors) {
  console.error("Introspection failed:", errors.map((e) => e.message).join("; "));
  process.exit(1);
}
const sdl = printSchema(buildClientSchema(data));
await writeFile("schema.graphql", sdl + "\n");
console.log(`Wrote schema.graphql (${sdl.split("\n").length} lines) from ${endpoint}`);
```

Against the server from [lesson 07](07-apollo-server-http.md), started normally and then
with `NODE_ENV=production`:

```console
$ node dump-schema.mjs
Wrote schema.graphql (10 lines) from http://localhost:4000/
$ cat schema.graphql
type Query {
  books: [Book!]!
  myBooks: [Book!]!
  whoami: String
}

type Book {
  id: ID!
  title: String!
}

$ node dump-schema.mjs        # server now in production mode
Introspection failed: GraphQL introspection is not allowed by Apollo Server, but the query contained __schema or __type. To enable introspection, pass introspection: true to ApolloServer in production
```

Committing a `schema.graphql` file to your repository gives code review a readable diff of
every API change, and lets client tooling work offline. Better still is generating it from
the server's code in CI rather than from a running server — `printSchema(schema)` on the
schema object directly — so it can't drift.

## Tool 2: catch breaking changes

`graphql-js` ships `findBreakingChanges(oldSchema, newSchema)`. Suppose someone proposes
removing the deprecated field, making `title` nullable and dropping a format:

```js
const next = buildSchema(`
  type Book { id: ID! title: String isbn13: String format: Format! }
  enum Format { HARDCOVER EBOOK }
  type Query { book(id: ID!): Book }
`);
for (const c of findBreakingChanges(clientSchema, next)) console.log(c.type, "-", c.description);
```

```text
FIELD_REMOVED - Book.isbn was removed.
FIELD_CHANGED_KIND - Book.title changed type from String! to String.
VALUE_REMOVED_FROM_ENUM - PAPERBACK was removed from enum type Format.
```

Each of these can break a deployed client: a query selecting `isbn` now fails validation;
generated TypeScript that declared `title: string` now receives `null`; code that switched on
`PAPERBACK` never sees it again — or, worse, an old value stored somewhere fails to
serialise. A check like this in CI is cheap insurance. [Level 4 · 05](../level-4/05-schema-evolution.md)
goes further, including which changes are only "dangerous" (`findDangerousChanges`) and how
to phase removals.

## Other tools that run on introspection

- **GraphiQL / Apollo Sandbox** — in-browser IDEs; they send the introspection query on load
  and use the result for autocomplete, docs and inline validation.
- **GraphQL Code Generator** — turns schema + operations into TypeScript types
  ([Level 3 · 08](../level-3/08-codegen.md)).
- **Editor plugins** (the GraphQL language server used by VS Code extensions) — read a
  `schema.graphql` or introspect an endpoint, configured by a `graphql.config.*` file.

## Should introspection be on in production?

Introspection reveals nothing that a determined client couldn't partly discover by probing,
but it hands over the complete map — every field, including ones your own clients never use
and that might be less carefully protected. Common practice:

- **Private APIs** (your own web and mobile apps): turn it off in production and ship the
  schema to client developers through the repo or a schema registry.
- **Public APIs** meant for third parties (GitHub's GraphQL API is the classic example): leave
  it on; discoverability is the product.

Either way, disabling introspection is not access control. Each field still needs its own
authorization, covered in [Level 3 · 02](../level-3/02-authorization.md).

## How It Actually Works

Introspection is not a special mode — it's ordinary execution against an ordinary schema.
`graphql-js` defines the introspection types (`__Schema`, `__Type`, …) as real
`GraphQLObjectType`s in `type/introspection.js`, with resolvers that read from the
`GraphQLSchema` object passed in `info.schema`. When the executor looks up a field on the
root type, it checks for the names `__schema`, `__type` and `__typename` first and uses those
meta-field definitions; on any other object type, only `__typename` is special, and it
resolves to `info.parentType.name`.

Validation also knows about them, which is how Apollo's production mode works: it adds a
validation rule that reports an error whenever a `__schema` or `__type` field appears in the
document. `__typename` is untouched by that rule, so clients that rely on it keep working.

## Common mistakes

- **Writing `# comments` expecting them to show in docs.** Use description strings.
- **Removing a field without deprecating it first.** Deprecate, watch usage drop (field
  usage metrics need tracing — [Level 4 · 02](../level-4/02-plugins-performance.md)), then remove.
- **Treating disabled introspection as security.** Field-level auth is still required.
- **Forgetting `includeDeprecated: true`** when writing your own introspection queries, then
  wondering why a field "disappeared".
- **Generating `schema.graphql` from a running staging server** — it reflects whatever was
  deployed, not the code under review.

## Exercise

1. Write a query that lists every enum in the schema with its values and descriptions.
2. Using `__type`, write a recursive helper that prints a field's full type signature
   (e.g. `[Book!]!`) from the nested `ofType` structure.
3. Add a field argument with a default value and find where that default appears in the
   introspection result.
4. Turn `findBreakingChanges` into a CLI: `node check.mjs old.graphql new.graphql` that exits
   non-zero when breaking changes exist. Test it with the example above.
