---
description: "Test a GraphQL server at four levels — units, executeOperation, HTTP and schema snapshots — plus a guard that catches N+1 regressions."
---

# 09 · Testing a GraphQL Server

A GraphQL API has more ways to break than its resolvers: a nullability change that crashes
old mobile clients, a refactor that quietly brings back N+1 queries, an HTTP setting that
starts rejecting a real client. A good test suite covers each of those at the cheapest level
that can catch it. This lesson builds a four-layer suite for a small bookstore API, with every
test run, and then deliberately breaks the code twice to show the guards firing.

## The application under test

It's the bookstore from lessons 04–06, packaged so tests can build it without listening on a
port. `db.js` and `loaders.js` are unchanged from [lesson 06](06-dataloader.md).

```js title="app.js"
import { ApolloServer } from "@apollo/server";
import { GraphQLError } from "graphql";
import { withQueryLog } from "./db.js";
import { createLoaders } from "./loaders.js";

export const typeDefs = /* GraphQL */ `
  type Query {
    books: [Book!]!
    book(id: ID!): Book
  }
  type Mutation {
    addReview(bookId: ID!, stars: Int!, body: String!): Review!
  }
  type Book { id: ID! title: String! author: Author! reviews: [Review!]! averageStars: Float }
  type Author { id: ID! name: String! }
  type Review { id: ID! stars: Int! body: String! }
`;

export function average(reviews) {
  if (reviews.length === 0) return null;
  return Math.round((reviews.reduce((s, r) => s + r.stars, 0) / reviews.length) * 10) / 10;
}

export const resolvers = {
  Query: {
    books: (_, __, { sql }) => sql.all("SELECT * FROM books ORDER BY id"),
    book: (_, { id }, { sql }) => sql.get("SELECT * FROM books WHERE id = ?", id) ?? null,
  },
  Mutation: {
    addReview: (_, { bookId, stars, body }, { sql, loaders }) => {
      if (stars < 1 || stars > 5)
        throw new GraphQLError("stars must be between 1 and 5", { extensions: { code: "BAD_USER_INPUT" } });
      if (!sql.get("SELECT 1 FROM books WHERE id = ?", bookId))
        throw new GraphQLError(`No book ${bookId}`, { extensions: { code: "NOT_FOUND" } });
      const { lastInsertRowid } = sql.run("INSERT INTO reviews (book_id, stars, body) VALUES (?, ?, ?)", bookId, stars, body);
      loaders.reviewsByBook.clear(Number(bookId));
      return sql.get("SELECT * FROM reviews WHERE id = ?", lastInsertRowid);
    },
  },
  Book: {
    author: (b, _, { loaders }) => loaders.author.load(b.author_id),
    reviews: (b, _, { loaders }) => loaders.reviewsByBook.load(b.id),
    averageStars: async (b, _, { loaders }) => average(await loaders.reviewsByBook.load(b.id)),
  },
};

export function createServer() {
  return new ApolloServer({ typeDefs, resolvers, includeStacktraceInErrorResponses: false });
}

export function createContext(db) {
  const sql = withQueryLog(db);
  return { sql, loaders: createLoaders(sql) };
}
```

Three seams make it testable:

- `average` is a plain exported function — business logic with no GraphQL in it.
- `createServer()` builds the server without starting HTTP.
- `createContext(db)` builds a request context from any database, so each test can use a
  fresh in-memory one.

Also note `loaders.reviewsByBook.clear(Number(bookId))` in the mutation: `bookId` arrives as a
string `ID`, while the loader is keyed by the integer column ([lesson 06](06-dataloader.md),
pitfall 3).

## Layer 1: unit tests

```js title="unit.test.js"
import { test } from "node:test";
import assert from "node:assert/strict";
import { average } from "./app.js";
import { createLoaders } from "./loaders.js";
import { openDb, seed, withQueryLog } from "./db.js";

test("average rounds to one decimal and is null for no reviews", () => {
  assert.equal(average([]), null);
  assert.equal(average([{ stars: 5 }, { stars: 4 }, { stars: 4 }]), 4.3);
});

test("author loader returns rows in key order with null for missing ids", async () => {
  const sql = withQueryLog(seed(openDb()));
  const { author } = createLoaders(sql);
  const rows = await author.loadMany([3, 99, 1]);
  assert.deepEqual(rows.map((r) => r?.name ?? null), ["William Gibson", null, "Frank Herbert"]);
  assert.equal(sql.log.length, 1, "one batched query");
});
```

Fast and precise. Use them for logic that's awkward to reach through queries: rounding rules,
loader ordering. The loader test also asserts **one** query, which pins down the batching
contract directly.

## Layer 2: operations through the server

```js title="api.test.js"
import { test, beforeEach } from "node:test";
import assert from "node:assert/strict";
import { createServer, createContext } from "./app.js";
import { openDb, seed } from "./db.js";

const server = createServer();
let db, ctx;
beforeEach(() => { db = seed(openDb()); });

async function gql(query, variables) {
  ctx = createContext(db);
  const res = await server.executeOperation({ query, variables }, { contextValue: ctx });
  return JSON.parse(JSON.stringify(res.body.singleResult));
}

test("book with author, reviews and average", async () => {
  const { data, errors } = await gql(`{ book(id: 1) { title author { name } averageStars reviews { stars } } }`);
  assert.equal(errors, undefined);
  assert.deepEqual(data.book, { title: "Dune", author: { name: "Frank Herbert" }, averageStars: 4.5, reviews: [{ stars: 5 }, { stars: 4 }] });
});

test("listing books with authors and reviews uses a constant number of statements", async () => {
  const { errors } = await gql(`{ books { title author { name } reviews { stars } averageStars } }`);
  assert.equal(errors, undefined);
  assert.equal(ctx.sql.log.length, 3, ctx.sql.log.join("\n"));
});

test("addReview validates stars", async () => {
  const { data, errors } = await gql(`mutation { addReview(bookId: 1, stars: 0, body: "x") { id } }`);
  assert.equal(data, null);
  assert.equal(errors[0].extensions.code, "BAD_USER_INPUT");
});

test("addReview on a missing book is NOT_FOUND", async () => {
  const { errors } = await gql(`mutation { addReview(bookId: 404, stars: 3, body: "x") { id } }`);
  assert.equal(errors[0].extensions.code, "NOT_FOUND");
});

test("reviews added by mutations show up in averageStars", async () => {
  const { data, errors } = await gql(`mutation {
    before: addReview(bookId: 2, stars: 3, body: "meh") { stars }
    after: addReview(bookId: 2, stars: 5, body: "better on reread") { stars }
  }`);
  assert.equal(errors, undefined);
  const q = await gql(`{ book(id: 2) { averageStars reviews { stars } } }`);
  assert.deepEqual(q.data.book, { averageStars: 4, reviews: [{ stars: 3 }, { stars: 5 }] });
});
```

`executeOperation` runs the full request pipeline — parse, validate, context, execute, error
formatting — without HTTP. This is where most of your tests should live, because it checks
schema and resolvers together, exactly as clients use them. Each test gets a fresh database in
`beforeEach`, so tests don't depend on each other's writes. The JSON round trip turns
`graphql-js`'s null-prototype objects into plain ones for `deepEqual`, as explained in
[Level 1 · 10](../level-1/10-project-library-api.md).

The second test is the important one: **it asserts the number of SQL statements**. The
context exposes the query log, so the test can say "this list query must take exactly 3
statements, regardless of how many books exist."

## Layer 3: real HTTP

```js title="http.test.js"
import { test, before, after } from "node:test";
import assert from "node:assert/strict";
import { startStandaloneServer } from "@apollo/server/standalone";
import { createServer, createContext } from "./app.js";
import { openDb, seed } from "./db.js";

let server, url;
before(async () => {
  const db = seed(openDb());
  server = createServer();
  ({ url } = await startStandaloneServer(server, { listen: { port: 0 }, context: async () => createContext(db) }));
});
after(() => server.stop());

const post = (body, headers = {}) =>
  fetch(url, { method: "POST", headers: { "content-type": "application/json", ...headers }, body: JSON.stringify(body) });

test("POST returns 200 and JSON", async () => {
  const res = await post({ query: "{ books { title } }" });
  assert.equal(res.status, 200);
  assert.match(res.headers.get("content-type"), /application\/json/);
  assert.equal((await res.json()).data.books.length, 8);
});

test("validation errors are 400", async () => {
  const res = await post({ query: "{ books { isbn } }" });
  assert.equal(res.status, 400);
  assert.equal((await res.json()).errors[0].extensions.code, "GRAPHQL_VALIDATION_FAILED");
});

test("simple cross-site requests are blocked by CSRF prevention", async () => {
  const res = await fetch(url, { method: "POST", headers: { "content-type": "text/plain" }, body: '{"query":"{ books { id } }"}' });
  assert.equal(res.status, 400);
});
```

`port: 0` asks the OS for any free port, so tests can run in parallel and in CI without
collisions. These tests cover what `executeOperation` skips: status codes, content types and
CSRF prevention ([Level 1 · 07](../level-1/07-apollo-server-http.md)). Keep this layer thin —
a handful of tests for transport behaviour, not one per resolver.

## Layer 4: the schema contract

```js title="schema.test.js"
import { test } from "node:test";
import assert from "node:assert/strict";
import { readFileSync, writeFileSync, existsSync } from "node:fs";
import { buildSchema, printSchema, findBreakingChanges } from "graphql";
import { typeDefs } from "./app.js";

const SNAPSHOT = new URL("./schema.snapshot.graphql", import.meta.url);

test("schema has no breaking changes against the committed snapshot", () => {
  const current = buildSchema(typeDefs);
  if (!existsSync(SNAPSHOT) || process.env.UPDATE_SCHEMA) {
    writeFileSync(SNAPSHOT, printSchema(current) + "\n");
    return;
  }
  const previous = buildSchema(readFileSync(SNAPSHOT, "utf8"));
  const breaking = findBreakingChanges(previous, current);
  assert.deepEqual(breaking.map((c) => c.description), []);
  assert.equal(printSchema(current) + "\n", readFileSync(SNAPSHOT, "utf8"),
    "schema changed compatibly - run with UPDATE_SCHEMA=1 and commit the snapshot");
});
```

The first run writes `schema.snapshot.graphql`, which you commit. After that, the test fails
for two different reasons with two different messages:

- a **breaking** change, listed by `findBreakingChanges` — stop and think about clients;
- any **other** change — fine, but regenerate the snapshot (`UPDATE_SCHEMA=1 node --test`) so
  the diff shows up in code review.

## The whole suite

```console
$ node --test
✔ book with author, reviews and average
✔ listing books with authors and reviews uses a constant number of statements
✔ addReview validates stars
✔ addReview on a missing book is NOT_FOUND
✔ reviews added by mutations show up in averageStars
✔ POST returns 200 and JSON
✔ validation errors are 400
✔ simple cross-site requests are blocked by CSRF prevention
✔ schema has no breaking changes against the committed snapshot
✔ average rounds to one decimal and is null for no reviews
✔ author loader returns rows in key order with null for missing ids
ℹ tests 11
ℹ pass 11
ℹ fail 0
```

(Timings trimmed.) The whole suite took well under a second on this machine.

## Breaking it on purpose

**Regression 1 — someone "simplifies" `Book.author`** back to a direct query:

```js
author: (b, _, { sql }) => sql.get("SELECT * FROM authors WHERE id = ?", b.author_id),
```

```text
✖ listing books with authors and reviews uses a constant number of statements
  AssertionError [ERR_ASSERTION]: SELECT * FROM books ORDER BY id
  SELECT * FROM authors WHERE id = ?
  SELECT * FROM authors WHERE id = ?
  SELECT * FROM authors WHERE id = ?
  SELECT * FROM authors WHERE id = ?
  SELECT * FROM authors WHERE id = ?
  SELECT * FROM authors WHERE id = ?
  SELECT * FROM authors WHERE id = ?
  SELECT * FROM authors WHERE id = ?
  SELECT * FROM reviews WHERE book_id IN (?,?,?,?,?,?,?,?) ORDER BY id

  10 !== 3
```

All functional tests still passed — the data was correct — so only the statement-count guard
noticed. Passing the log as the assertion message makes the cause obvious.

**Regression 2 — `Book.title` loosened to `String`:**

```text
✖ schema has no breaking changes against the committed snapshot
  AssertionError [ERR_ASSERTION]: Expected values to be strictly deep-equal:
  + actual - expected

  + [
  +   'Book.title changed type from String! to String.'
  + ]
```

And adding a harmless nullable `Author.bio` produced the second message:

```text
✖ schema has no breaking changes against the committed snapshot
  AssertionError [ERR_ASSERTION]: schema changed compatibly - run with UPDATE_SCHEMA=1 and commit the snapshot
```

## What to test where

| Concern | Best layer |
|---|---|
| Business rules, formatting, calculations | unit |
| Loader ordering/batching contracts | unit |
| Field results, nullability, error codes, auth rules per field | `executeOperation` |
| Query-count budgets for important screens | `executeOperation` with a counting context |
| Status codes, headers, CSRF, CORS, body limits | HTTP |
| Breaking API changes | schema snapshot |

Testing real client operations is a strong habit: if your web app's queries live in `.graphql`
files, load *those* in the `executeOperation` tests, so a schema change that breaks a real
screen fails a test by name.

## How It Actually Works

`node --test` with no arguments finds files matching patterns like `*.test.js` and runs each
file in its own child process, in parallel. That isolation is why `http.test.js` can start a
server and `api.test.js` can keep module-level state without interfering. Inside a file, tests
run sequentially by default, and `beforeEach` runs before each one.

`executeOperation(request, { contextValue })` calls Apollo Server's internal
`internalExecuteOperation` — the same function the HTTP path calls after it has parsed the
HTTP request and run your `context` function. That's why it's a faithful test of everything
GraphQL-specific and nothing HTTP-specific. Passing `contextValue` yourself replaces the
context function entirely, which is how each test injects its own database.

## Common mistakes

- **Only unit-testing resolver functions** with hand-built `parent`/`args`. You end up testing
  your assumptions about the schema rather than the schema.
- **Sharing a database between tests** so order matters and failures cascade.
- **Asserting on full error messages** instead of `extensions.code`.
- **Hundreds of HTTP tests** that are slow and add nothing over `executeOperation`.
- **No guard on query counts**, so N+1 regressions only show up in production latency graphs.
- **Snapshotting the schema without the breaking-change check**, so reviewers wave through a
  diff they don't recognise as breaking.

## Exercise

1. Add a test proving that `{ books { reviews { stars } } }` runs exactly 2 statements, then
   remove the `clear()` call in `addReview` and write a test that fails because of it.
   (Hint: you need a mutation that reads `averageStars` back in the same request — add
   `Review.book` so the mutation can select it.)
2. Add a `fixtures/` directory with `.graphql` files containing "client" operations, and a
   test that validates each against the schema with `validate()`.
3. Add an HTTP test checking that a GET with `apollo-require-preflight` returns data and that a
   mutation over GET returns 405.
4. Make the schema snapshot test print *dangerous* changes as warnings (`findDangerousChanges`)
   without failing.
