---
description: "Generate TypeScript from a GraphQL schema with GraphQL Code Generator: typed documents for clients, typed resolvers with mappers."
---

# 08 · Type Safety with GraphQL Code Generator

A GraphQL schema is already a precise type system. **GraphQL Code Generator** turns it — plus
the operations your client actually sends — into TypeScript types, so the compiler catches a
misspelled field, an unhandled `null` or a wrongly typed resolver before anything runs. This
lesson sets up both halves (client operations and server resolvers) in one small project,
compiles and runs it, and then breaks it five ways to show exactly what gets caught and where.

Versions used: `@graphql-codegen/cli` 7.4.5, `client-preset` 6.2.2, `typescript` and
`typescript-resolvers` plugins 6.1.1, TypeScript 7.0.2, Node 26.3.

```bash
npm install -D @graphql-codegen/cli @graphql-codegen/client-preset \
  @graphql-codegen/typescript @graphql-codegen/typescript-resolvers typescript
npm install graphql @apollo/server
```

## The schema and the internal models

```graphql title="schema.graphql"
type Query {
  books(first: Int = 10): [Book!]!
  book(id: ID!): Book
}
type Mutation {
  addReview(bookId: ID!, stars: Int!, body: String!): Review
}
type Book {
  id: ID!
  title: String!
  year: Int
  author: Author!
  format: Format!
}
type Author { id: ID! name: String! }
type Review { id: ID! stars: Int! body: String! }
enum Format { HARDCOVER PAPERBACK EBOOK }
```

```ts title="models.ts"
// Internal representations — what resolvers actually pass around.
export interface BookRow { id: number; title: string; year: number | null; author_id: number; format: "HARDCOVER" | "PAPERBACK" | "EBOOK" }
export interface AuthorRow { id: number; name: string }
export interface Context {
  books: BookRow[];
  authors: AuthorRow[];
}
```

Note the deliberate mismatch, as in [Level 2](../level-2/04-database-context.md): the database
row has `author_id` and a numeric `id`; the schema has `author: Author!` and `id: ID!`. Typed
resolvers must understand both sides.

## Configuration

```ts title="codegen.ts"
import type { CodegenConfig } from "@graphql-codegen/cli";

const config: CodegenConfig = {
  schema: "schema.graphql",
  documents: ["src/**/*.ts", "!src/gql/**/*"],
  // emit ".js" extensions in generated imports, required by "module": "nodenext"
  emitLegacyCommonJSImports: false,
  generates: {
    // Client side: typed documents for every operation found in src/
    "src/gql/": {
      preset: "client",
      config: { enumsAsTypes: true },
    },
    // Server side: resolver signatures
    "src/resolvers-types.ts": {
      plugins: ["typescript", "typescript-resolvers"],
      config: {
        enumsAsTypes: true,
        mappers: { Book: "./models.js#BookRow", Author: "./models.js#AuthorRow" },
        contextType: "./models.js#Context",
      },
    },
  },
};
export default config;
```

Two outputs from one schema:

- **`src/gql/`** with the `client` preset scans `documents` for ``graphql(`…`)`` calls and
  generates a typed `graphql()` function: pass it a query string and you get back a
  `TypedDocumentNode` carrying the exact result and variables types for *that* operation.
- **`src/resolvers-types.ts`** generates a `Resolvers` type. `mappers` tell it that when a
  resolver returns a `Book`, it's really returning a `BookRow` — so `Book.author`'s `parent`
  argument is typed as `BookRow`, with `author_id` available. `contextType` types the third
  argument.

### The ESM gotcha

The first run, without `emitLegacyCommonJSImports: false`, generated files with extensionless
imports, which a `"module": "nodenext"` project rejects:

```text
src/gql/index.ts(2,15): error TS2835: Relative import paths need explicit file extensions in ECMAScript imports when '--moduleResolution' is 'node16' or 'nodenext'. Did you mean './fragment-masking.js'?
src/client.ts(1,10): error TS2305: Module '"./gql/index.js"' has no exported member 'graphql'.
```

(Two of the six errors shown.) Setting the option makes the generator emit `.js` extensions,
and the errors disappear.

## Client: typed operations

```ts title="client.ts"
import { graphql } from "./gql/index.js";
import type { TypedDocumentNode } from "@graphql-typed-document-node/core";
import { print } from "graphql";

export const BookListQuery = graphql(`
  query BookList($first: Int) {
    books(first: $first) { id title year format author { name } }
  }
`);

// A tiny typed client: result and variables types come from the document.
export async function request<TResult, TVariables>(
  url: string, document: TypedDocumentNode<TResult, TVariables>, variables: TVariables,
): Promise<TResult> {
  const res = await fetch(url, {
    method: "POST",
    headers: { "content-type": "application/json" },
    body: JSON.stringify({ query: print(document), variables }),
  });
  const json = await res.json();
  if (json.errors) throw new Error(json.errors[0].message);
  return json.data as TResult;
}

export async function showList(url: string) {
  const data = await request(url, BookListQuery, { first: 5 });
  for (const b of data.books) {
    console.log(`${b.title} (${b.year ?? "n.d."}) by ${b.author.name} [${b.format.toLowerCase()}]`);
  }
}
```

`BookListQuery` is a normal GraphQL document at runtime, but its TypeScript type is
`TypedDocumentNode<BookListQuery, BookListQueryVariables>`. The generated result type for this
operation is exactly what was selected:

```ts
export type BookListQuery = { books: Array<{ id: string, title: string, year: number | null, format: Format, author: { name: string } }> };
export type BookListQueryVariables = Exact<{
  first?: number | null | undefined;
}>;
```

`year` is `number | null` because the schema field is nullable; `author` has only `name` because
that's all the query asked for. The tiny `request` helper infers both types from the document,
so `data.books` is fully typed with no manual annotations. Apollo Client, urql and other clients
accept `TypedDocumentNode` the same way.

## Server: typed resolvers

```ts title="resolvers.ts"
import type { Resolvers } from "./resolvers-types.js";

export const resolvers: Resolvers = {
  Query: {
    books: (_, { first }, { books }) => books.slice(0, first ?? 10),
    book: (_, { id }, { books }) => books.find((b) => b.id === Number(id)) ?? null,
  },
  Book: {
    author: (book, _, { authors }) => authors.find((a) => a.id === book.author_id)!,
  },
};
```

```ts title="main.ts"
import { readFileSync } from "node:fs";
import { ApolloServer } from "@apollo/server";
import { startStandaloneServer } from "@apollo/server/standalone";
import { resolvers } from "./resolvers.js";
import { showList } from "./client.js";
import type { Context } from "./models.js";

const data: Context = {
  authors: [{ id: 1, name: "Frank Herbert" }, { id: 2, name: "Jane Austen" }],
  books: [
    { id: 1, title: "Dune", year: 1965, author_id: 1, format: "PAPERBACK" },
    { id: 2, title: "Emma", year: null, author_id: 2, format: "EBOOK" },
  ],
};
const server = new ApolloServer<Context>({ typeDefs: readFileSync("schema.graphql", "utf8"), resolvers });
const { url } = await startStandaloneServer(server, { listen: { port: 4314 }, context: async () => data });
await showList(url);
await server.stop();
```

```json title="tsconfig.json"
{
  "compilerOptions": {
    "target": "es2023",
    "module": "nodenext",
    "moduleResolution": "nodenext",
    "strict": true,
    "rootDir": "src",
    "outDir": "dist",
    "skipLibCheck": true
  },
  "include": ["src"]
}
```

```console
$ npx graphql-codegen
✔ Parse Configuration
…
✔ Generate to src/gql/
…
✔ Generate to src/resolvers-types.ts
✔ Generate outputs
$ npx tsc -p .
$ node dist/main.js
Dune (1965) by Frank Herbert [paperback]
Emma (n.d.) by Jane Austen [ebook]
```

(Codegen prints a line per step; the `…` marks lines trimmed here.) TypeScript 7 also insisted on an explicit `rootDir` when
`outDir` is set — the first build failed with error TS5011 until it was added.

## Five mistakes, caught

Each change below was made, checked, and reverted.

**1. Reading a field the query didn't select** (`b.isbn`):

```text
src/client.ts(28,34): error TS2339: Property 'isbn' does not exist on type '{ id: string; title: string; year: number | null; format: Format; author: { name: string; }; }'.
```

**2. Ignoring a nullable field** (`b.year.toFixed(0)`):

```text
src/client.ts(28,32): error TS18047: 'b.year' is possibly 'null'.
```

**3. Wrong variable type** (`{ first: "5" }`):

```text
src/client.ts(26,52): error TS2322: Type 'string' is not assignable to type 'number'.
```

**4. A resolver returning the wrong shape** (`Book.author` returning the author's name string):

```text
src/resolvers.ts(9,5): error TS2322: Type '(book: BookRow, _: Record<PropertyKey, never>, { authors }: Context) => string' is not assignable to type 'Resolver<ResolverTypeWrapper<AuthorRow>, BookRow, Context, Record<PropertyKey, never>> | undefined'.
```

Note `book: BookRow` in the message — the mapper at work.

**5. A schema change that breaks a client operation** (renaming `Book.title` to `name`) fails at
generation time, before TypeScript is even involved:

```text
✖ Generate [FAILED: GraphQL Document Validation failed with 1 errors;
  Error 0: Cannot query field "title" on type "Book".
  ✖ One or more errors occurred, no files were generated. To allow output on errors, set config.allowPartialOutputs=true
```

Run codegen in CI against the server's current schema and this becomes a check that a server
change won't break the app — a client-side complement to the breaking-change test in
[Level 2 · 09](../level-2/09-testing.md).

## What codegen doesn't do

- **Runtime validation.** Types describe what the server *promises*; a misbehaving server can
  still send something else. The GraphQL server's own output validation (non-null checks,
  scalar serialisation) is what enforces the promise.
- **Custom scalars default to `any`.** Map them with the `scalars` config option
  (`DateTime: "string"`) or you lose type safety exactly where formats matter.
- **Keep generated code in sync.** Either commit it and check it in CI, or generate it in the
  build. Stale generated types are worse than none.

## How It Actually Works

Codegen loads the schema (SDL files, an introspection result or a live endpoint) into a
`GraphQLSchema`, then loads documents by scanning source files for `graphql(` calls or
`gql` tags and parsing the template strings. It **validates** every document against the schema
with `graphql-js`'s `validate()` — that's where mistake 5 was caught. Then each plugin walks
the schema and documents with `TypeInfo`, emitting TypeScript: for an operation it follows each
selection to its field definition and writes the field's type, wrapping nullable fields as
`T | null` and lists as `Array<T>`. The client preset additionally writes a `graphql()` function
whose overloads map each exact document string to its typed document — that's how passing a
string literal returns the right type. `typescript-resolvers` writes, for every object type, a
map of field resolvers typed as `(parent: Mapped<Parent>, args, context, info) => Result |
Promise<Result>`, substituting mapper types wherever an object type appears.

## Common mistakes

- **Hand-writing TypeScript interfaces for API responses** that drift from the schema.
- **Skipping `mappers`**, then fighting resolver types that assume parents already look like
  schema types.
- **Leaving custom scalars as `any`.**
- **Generating from a production endpoint** instead of the schema in the repo, so CI results
  depend on what's deployed.
- **Committing generated files without a CI check** that regenerating produces no diff.

## Exercise

1. Add `scalar DateTime` with `Review.createdAt: DateTime!` and map it to `string` with the
   `scalars` option. Check the generated client type.
2. Add a `fragment BookCard on Book { title author { name } }` in a second file and use it in
   `BookList`. Explore what fragment masking (`useFragment`/`getFragmentData` in the preset's
   output) does to the result type.
3. Add `Mutation.addReview` to `resolvers.ts` without returning `stars`, and read the error.
4. Point `schema:` at the running Apollo Server URL instead of the file and compare the result.
