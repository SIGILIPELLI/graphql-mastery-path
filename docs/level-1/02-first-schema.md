---
description: "Build a schema with graphql-js, execute queries with no HTTP server, and watch the parse, validate and execute phases succeed and fail separately."
---

# 02 · Your First Schema and Query with graphql-js

Before adding a web server, it's worth running GraphQL in its purest form: a schema, some
data, and a function call. `graphql-js` is the reference implementation of the
specification, and nearly every JavaScript server — Apollo Server, GraphQL Yoga, Mercurius,
Envelop-based servers — calls into it to do the actual work. If you understand what
`graphql-js` does on its own, the servers become thin wrappers you can reason about.

## Setting up

```bash
mkdir gql-basics && cd gql-basics
npm init -y
npm pkg set type=module
npm install graphql@16
```

`type=module` lets us use `import` and top-level `await` in `.mjs`/`.js` files. This
course pins `graphql@16` (16.14.2 when these lessons were run) because Apollo Server 5 and
Apollo Federation declare a peer dependency on `graphql ^16`. Version 17 of `graphql-js` was
released in June 2026; [Level 4 · 08](../level-4/08-yoga-graphql17.md) looks at what it
changes. Everything in this lesson behaves the same in both.

## A schema and a query, no network

```js title="first.mjs"
import { buildSchema, graphql } from "graphql";

const schema = buildSchema(`
  type Query {
    greeting(name: String): String
    book(id: ID!): Book
  }
  type Book {
    id: ID!
    title: String!
    pages: Int
  }
`);

const books = { "1": { id: "1", title: "Dune", pages: 412 } };

const rootValue = {
  greeting: ({ name }) => `Hello, ${name ?? "world"}!`,
  book: ({ id }) => books[id] ?? null,
};

const run = async (source) =>
  console.log(JSON.stringify(await graphql({ schema, source, rootValue }), null, 2));

await run(`{ greeting }`);
await run(`{ greeting(name: "Ada") book(id: "1") { title } }`);
await run(`{ book(id: "9") { title } }`);
await run(`{ book(id: "1") { author } }`);
await run(`{ book { title } }`);
```

What's going on:

- `buildSchema` takes SDL and returns a `GraphQLSchema` object.
- `rootValue` is the object that the top-level `Query` fields are resolved against. With
  `buildSchema`, a root field that is a function is called with the field's **arguments**
  as its first parameter, which is why `greeting` destructures `{ name }`.
- `graphql({ schema, source, rootValue })` runs the full pipeline and returns a promise of
  `{ data, errors }`.

Run it with `node first.mjs`. The five results, as printed:

```json
{
  "data": {
    "greeting": "Hello, world!"
  }
}
{
  "data": {
    "greeting": "Hello, Ada!",
    "book": {
      "title": "Dune"
    }
  }
}
{
  "data": {
    "book": null
  }
}
{
  "errors": [
    {
      "message": "Cannot query field \"author\" on type \"Book\".",
      "locations": [
        {
          "line": 1,
          "column": 19
        }
      ]
    }
  ]
}
{
  "errors": [
    {
      "message": "Field \"book\" argument \"id\" of type \"ID!\" is required, but it was not provided.",
      "locations": [
        {
          "line": 1,
          "column": 3
        }
      ]
    }
  ]
}
```

Three things to notice:

1. `{ greeting }` with no argument works because `name: String` is nullable, so the
   argument is optional. `book(id: ID!)` makes `id` required — leaving it out is a
   validation error.
2. Looking up a book that doesn't exist returns `"book": null` with **no error**. The field
   type is `Book` (nullable), so `null` is a perfectly valid answer meaning "no such book".
3. The two invalid queries return `errors` and **no `data` key at all**. Nothing executed.
   `locations` points at the line and column in the query text where the problem is.

## Running the phases yourself

`graphql()` is a convenience wrapper. You can call the three phases separately, which makes
it obvious where each kind of failure happens:

```js title="phases.mjs"
import { buildSchema, parse, validate, execute } from "graphql";

const schema = buildSchema(`
  type Query { book(id: ID!): Book  books: [Book!]! }
  type Book { id: ID! title: String! pages: Int }
`);
const books = [
  { id: "1", title: "Dune", pages: 412 },
  { id: "2", title: "Emma", pages: null },
];
const rootValue = {
  book: ({ id }) => books.find((b) => b.id === id),
  books: () => books,
};

function run(source) {
  let document;
  try {
    document = parse(source);
  } catch (e) {
    console.log("PARSE ERROR:", e.message);
    return;
  }
  const errors = validate(schema, document);
  if (errors.length) {
    console.log("VALIDATION ERRORS:", errors.map((e) => e.message));
    return;
  }
  const result = execute({ schema, document, rootValue });
  console.log("RESULT:", JSON.stringify(result));
}

run(`{ books { title pages } }`);
run(`{ books { title pages }`);
run(`{ books { title isbn } book { id } }`);
run(`query { book(id: 2) { title } }`);
```

Output:

```console
$ node phases.mjs
RESULT: {"data":{"books":[{"title":"Dune","pages":412},{"title":"Emma","pages":null}]}}
PARSE ERROR: Syntax Error: Expected Name, found <EOF>.
VALIDATION ERRORS: [
  'Cannot query field "isbn" on type "Book".',
  'Field "book" argument "id" of type "ID!" is required, but it was not provided.'
]
RESULT: {"data":{"book":{"title":"Emma"}}}
```

- `parse` **throws** on a syntax error. The parser stops at the first problem.
- `validate` **returns an array** and reports *every* problem it finds — two here.
- `execute` returned a plain object, not a promise, because none of our resolvers were
  async. If any resolver returns a promise, `execute` returns a promise. `graphql()` always
  returns a promise, which is why it's the safer default.
- `book(id: 2)` worked even though `2` is an integer literal. The built-in `ID` scalar
  accepts both string and integer input and always serializes as a string — you'll see the
  coercion rules in [lesson 03](03-type-system.md).

## Looking at the AST

The parsed document is a plain tree of objects. You rarely build one by hand, but tools
(linters, code generators, the validation rules themselves) all work on it:

```js title="ast.mjs"
import { parse, print } from "graphql";

const doc = parse(`query Shelf { books { title } }`);
const op = doc.definitions[0];
console.log(op.kind, op.operation, op.name.value);
console.log(op.selectionSet.selections[0].name.value, "->",
  op.selectionSet.selections[0].selectionSet.selections.map((s) => s.name.value));
console.log(print(parse(`{books{title    pages}}`)));
```

```console
$ node ast.mjs
OperationDefinition query Shelf
books -> [ 'title' ]
{
  books {
    title
    pages
  }
}
```

`print` turns an AST back into canonical query text; that's how formatters and some
caching schemes normalize queries.

## The resolver-map style

`buildSchema` + `rootValue` is fine for top-level fields, but it gives you no clean place to
put a resolver for `Book.someField`. Most real servers use a **resolver map**: an object
keyed by type name, then by field name. The `@graphql-tools/schema` package (which Apollo
Server uses internally) builds an executable schema from SDL plus such a map:

```bash
npm install @graphql-tools/schema
```

```js title="resolvermap.mjs"
import { makeExecutableSchema } from "@graphql-tools/schema";
import { graphql } from "graphql";

const typeDefs = /* GraphQL */ `
  type Query { books: [Book!]! }
  type Book { id: ID! title: String! shout: String! }
`;
const resolvers = {
  Query: { books: () => [{ id: "1", title: "Dune" }] },
  Book: { shout: (book) => book.title.toUpperCase() + "!" },
};
const schema = makeExecutableSchema({ typeDefs, resolvers });
console.log(JSON.stringify(await graphql({ schema, source: "{ books { title shout } }" })));

try {
  makeExecutableSchema({
    typeDefs,
    resolvers: { ...resolvers, Book: { ...resolvers.Book, author: () => "x" } },
  });
} catch (e) {
  console.log("ERROR:", e.message);
}
```

```console
$ node resolvermap.mjs
{"data":{"books":[{"title":"Dune","shout":"DUNE!"}]}}
ERROR: Book.author defined in resolvers, but not in schema
```

Note the signature change: in a resolver map, the **first** argument is the parent object
(`book`), not the arguments. That's the standard four-argument resolver signature covered in
[lesson 05](05-resolvers.md). From lesson 05 on, every example uses resolver maps.

The second half shows a useful safety net: a resolver for a field that isn't in the schema
fails at startup instead of silently never running — typically a typo or a field you
removed from the SDL but forgot in code.

## How It Actually Works

`buildSchema` parses the SDL into an AST and constructs real `GraphQLObjectType`,
`GraphQLField` and `GraphQLScalarType` instances from it. The schema object you get back is
the same kind you could build by hand with `new GraphQLObjectType({ ... })` (the
"code-first" style in [Level 4 · 07](../level-4/07-code-first.md)). SDL is just a
convenient notation for it.

`buildSchema` gives every field the **default field resolver**. In `graphql-js` that
resolver looks up `source[fieldName]` on the parent value; if the property is a function,
it calls it with `(args, contextValue, info)` and uses the return value. At the root, the
"parent" is `rootValue`. That's the whole mechanism behind the `rootValue` style: the
functions on it are just properties the default resolver happens to call.

`makeExecutableSchema` does something different: it builds the schema from SDL and then
*attaches* your functions as each field's `resolve` property. When a field has its own
`resolve`, the executor calls it with `(parent, args, context, info)`. When it doesn't —
`Book.title` above — the default resolver still applies, which is why `title` works without
any code.

Validation runs a list of rule functions (`specifiedRules` in `graphql-js`, one per rule
in the spec, such as `FieldsOnCorrectTypeRule` and `ProvidedRequiredArgumentsRule`). Each
rule is a visitor over the AST that reports errors instead of throwing, which is why you
get all problems at once.

## Common mistakes

- **Expecting the arguments in the first parameter of a resolver-map function.** With
  `rootValue`, root functions receive `args` first. With resolver maps, they receive
  `parent` first and `args` second. Mixing the two styles produces `undefined` arguments.
- **Treating `execute` as always-async.** It returns a promise only when something async
  happened. Use `graphql()` or always `await` the result.
- **Catching only thrown errors.** `graphql()` almost never throws for a bad query; it
  *returns* `{ errors }`. Code that wraps it in `try/catch` and assumes success otherwise
  will happily render `undefined`.
- **Returning `undefined` from a resolver for "not found".** It's treated the same as
  `null`, but returning `null` explicitly makes the intent obvious to readers.

## Exercise

1. Add `author: String` to `Book` in `first.mjs`, give Dune an author, and query it. Then
   query the author of a book that has no `author` property. What do you get, and why is it
   not an error?
2. Change `title: String!` to `title: String` and back. Query a book object that has no
   `title`. Compare the two responses — you're previewing null propagation from
   [lesson 08](08-errors.md).
3. Using `phases.mjs`, write a query that triggers three validation errors at once.
4. Rewrite `first.mjs` using `makeExecutableSchema` and a resolver map. Make sure
   `greeting(name:)` still works — where do the arguments arrive now?
