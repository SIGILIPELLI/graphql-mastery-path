---
description: "Level 2 project: a SQLite bookstore GraphQL API with keyset pagination, DataLoader, a DateTime scalar, userErrors and query-count tests."
---

# 10 · Project — A Bookstore API on SQLite

This project assembles everything from Level 2 into one service: a client-shaped schema, a
custom scalar, a real database, batched data access, cursor pagination, mutation payloads
with errors as data, and a test suite that guards both behaviour and query counts. Along the
way, the tests caught a cache-invalidation bug worth seeing in detail.

## Requirements

- Browse books sorted by title, 10 per page by default, with optional title search.
- Book details: author, reviews, average stars. Author pages list their books by year.
- Add a review; invalid input and unknown books come back as `userErrors`, not GraphQL errors.
- Review timestamps exposed as ISO-8601 UTC.
- Any catalog query, however deeply nested, runs a bounded number of SQL statements.
- A persistent SQLite file in development; in-memory databases in tests.

## Layout

```text
bookstore/
├── schema.graphql      the contract
├── db.js               schema DDL, seed data, statement logger
├── loaders.js          DataLoaders, created per request
├── scalars.js          DateTime
├── pagination.js       keyset pagination over books
├── resolvers.js
├── server.js           createServer / createContext (no HTTP)
├── main.js             HTTP entry point
└── bookstore.test.js
```

## The schema

```graphql title="schema.graphql"
scalar DateTime

type Query {
  books(first: Int = 10, after: String, search: String): BookConnection!
  book(id: ID!): Book
  author(id: ID!): Author
}

type Mutation {
  addReview(input: AddReviewInput!): AddReviewPayload!
}

type Book {
  id: ID!
  title: String!
  year: Int!
  author: Author!
  reviews: [Review!]!
  averageStars: Float
}

type Author {
  id: ID!
  name: String!
  books: [Book!]!
}

type Review {
  id: ID!
  stars: Int!
  body: String!
  createdAt: DateTime!
  book: Book!
}

type BookConnection {
  edges: [BookEdge!]!
  pageInfo: PageInfo!
}
type BookEdge { cursor: String! node: Book! }
type PageInfo { hasNextPage: Boolean! endCursor: String }

input AddReviewInput {
  bookId: ID!
  stars: Int!
  body: String!
}

type AddReviewPayload {
  review: Review
  userErrors: [UserError!]!
}

type UserError {
  field: [String!]
  message: String!
  code: UserErrorCode!
}

enum UserErrorCode { INVALID NOT_FOUND }
```

Design notes, referring back to the lessons:

- `books` returns a connection with the Relay shape from [lesson 07](07-pagination.md). The sort
  is by title, so the cursor encodes `(title, id)`.
- `addReview` takes a single input and returns a payload with `userErrors`, style A from
  [lesson 08](08-mutation-design.md), with paths like `["input", "stars"]`.
- `averageStars` is nullable: a book with no reviews has no average, and `null` says so
  honestly where `0` would lie.
- `DateTime` is used for output only, so its input functions refuse to run (see `scalars.js`).

## Database

```js title="db.js"
import { DatabaseSync } from "node:sqlite";

export function openDb(file = ":memory:") {
  const db = new DatabaseSync(file);
  db.exec(`
    PRAGMA foreign_keys = ON;
    CREATE TABLE IF NOT EXISTS authors (id INTEGER PRIMARY KEY, name TEXT NOT NULL);
    CREATE TABLE IF NOT EXISTS books (
      id INTEGER PRIMARY KEY,
      title TEXT NOT NULL,
      year INTEGER NOT NULL,
      author_id INTEGER NOT NULL REFERENCES authors(id)
    );
    CREATE INDEX IF NOT EXISTS books_author ON books (author_id);
    CREATE TABLE IF NOT EXISTS reviews (
      id INTEGER PRIMARY KEY,
      book_id INTEGER NOT NULL REFERENCES books(id),
      stars INTEGER NOT NULL CHECK (stars BETWEEN 1 AND 5),
      body TEXT NOT NULL,
      created_at INTEGER NOT NULL
    );
    CREATE INDEX IF NOT EXISTS reviews_book ON reviews (book_id);
  `);
  return db;
}

export function seed(db) {
  const count = db.prepare("SELECT count(*) AS n FROM authors").get().n;
  if (count > 0) return db; // already seeded (file database)
  const authors = ["Frank Herbert", "Jane Austen", "William Gibson", "Ursula K. Le Guin"];
  const books = [
    ["Dune", 1965, 1], ["Dune Messiah", 1969, 1], ["Emma", 1815, 2], ["Persuasion", 1817, 2],
    ["Neuromancer", 1984, 3], ["Count Zero", 1986, 3], ["The Dispossessed", 1974, 4],
    ["The Left Hand of Darkness", 1969, 4],
  ];
  db.exec("BEGIN");
  for (const a of authors) db.prepare("INSERT INTO authors (name) VALUES (?)").run(a);
  for (const b of books) db.prepare("INSERT INTO books (title, year, author_id) VALUES (?, ?, ?)").run(...b);
  const t0 = Date.UTC(2026, 0, 1);
  [[1, 5, "Sand everywhere, worth it."], [1, 4, "Slow start."], [3, 5, "Witty."], [5, 4, "Dense but rewarding."]]
    .forEach(([book, stars, body], i) =>
      db.prepare("INSERT INTO reviews (book_id, stars, body, created_at) VALUES (?, ?, ?, ?)").run(book, stars, body, t0 + i * 3600_000));
  db.exec("COMMIT");
  return db;
}

export function withQueryLog(db) {
  const log = [];
  const prep = (sql) => { log.push(sql); return db.prepare(sql); };
  return {
    log,
    all: (sql, ...p) => prep(sql).all(...p),
    get: (sql, ...p) => prep(sql).get(...p),
    run: (sql, ...p) => prep(sql).run(...p),
  };
}
```

Indexes on `books(author_id)` and `reviews(book_id)` support the loaders' `IN (…)` queries.
Timestamps are stored as epoch milliseconds — an integer is unambiguous, sorts correctly and
leaves formatting to the scalar. `seed` is idempotent so `main.js` can run against an existing
file.

## Loaders

```js title="loaders.js"
import DataLoader from "dataloader";

const ph = (n) => Array(n).fill("?").join(",");
const toKey = (k) => Number(k); // ID args are strings, columns are integers

function byId(rows, keys) {
  const m = new Map(rows.map((r) => [r.id, r]));
  return keys.map((k) => m.get(k) ?? null);
}
function groupBy(rows, keys, col) {
  const g = new Map(keys.map((k) => [k, []]));
  for (const r of rows) g.get(r[col])?.push(r);
  return keys.map((k) => g.get(k));
}

export function createLoaders(sql) {
  const opts = { cacheKeyFn: toKey };
  return {
    book: new DataLoader(async (ids) => {
      ids = ids.map(toKey);
      return byId(sql.all(`SELECT * FROM books WHERE id IN (${ph(ids.length)})`, ...ids), ids);
    }, opts),
    author: new DataLoader(async (ids) => {
      ids = ids.map(toKey);
      return byId(sql.all(`SELECT * FROM authors WHERE id IN (${ph(ids.length)})`, ...ids), ids);
    }, opts),
    booksByAuthor: new DataLoader(async (ids) => {
      ids = ids.map(toKey);
      return groupBy(sql.all(`SELECT * FROM books WHERE author_id IN (${ph(ids.length)}) ORDER BY year, id`, ...ids), ids, "author_id");
    }, opts),
    reviewsByBook: new DataLoader(async (ids) => {
      ids = ids.map(toKey);
      return groupBy(sql.all(`SELECT * FROM reviews WHERE book_id IN (${ph(ids.length)}) ORDER BY created_at, id`, ...ids), ids, "book_id");
    }, opts),
  };
}
```

Compared with [lesson 06](06-dataloader.md), every loader normalises keys with `cacheKeyFn`:
`Query.book` passes the string `ID` `"1"`, while `Review.book` passes the integer column
`1`. Without normalisation those would be two cache entries and two fetches for the same row.

## Scalar and pagination

```js title="scalars.js"
import { GraphQLScalarType, GraphQLError } from "graphql";

// Output-only in this API: stored as epoch milliseconds, sent as ISO-8601 UTC.
export const DateTime = new GraphQLScalarType({
  name: "DateTime",
  description: "ISO-8601 timestamp in UTC.",
  serialize(value) {
    const d = typeof value === "number" ? new Date(value) : value;
    if (!(d instanceof Date) || Number.isNaN(d.getTime()))
      throw new GraphQLError(`DateTime cannot represent ${JSON.stringify(value)}`);
    return d.toISOString();
  },
  parseValue() {
    throw new TypeError("DateTime is output-only in this API");
  },
  parseLiteral() {
    throw new TypeError("DateTime is output-only in this API");
  },
});
```

```js title="pagination.js"
import { GraphQLError } from "graphql";

export const MAX_PAGE = 50;
const bad = (message) => new GraphQLError(message, { extensions: { code: "BAD_USER_INPUT" } });

export const encodeCursor = (row) => Buffer.from(JSON.stringify([row.title, row.id])).toString("base64url");

export function decodeCursor(cursor) {
  try {
    const [title, id] = JSON.parse(Buffer.from(cursor, "base64url").toString());
    if (typeof title === "string" && Number.isInteger(id)) return { title, id };
  } catch {}
  throw bad("Invalid cursor");
}

export const escapeLike = (s) => s.replace(/[\\%_]/g, (c) => "\\" + c);

// Books sorted by title (id as tie-breaker), keyset-paginated.
export function pageOfBooks(sql, { first, after, search }) {
  if (!Number.isInteger(first) || first < 0 || first > MAX_PAGE) throw bad(`first must be between 0 and ${MAX_PAGE}`);
  const where = [], params = [];
  if (search) { where.push("title LIKE ? ESCAPE '\\'"); params.push(`%${escapeLike(search)}%`); }
  if (after) { const c = decodeCursor(after); where.push("(title, id) > (?, ?)"); params.push(c.title, c.id); }
  const rows = sql.all(
    `SELECT * FROM books ${where.length ? "WHERE " + where.join(" AND ") : ""} ORDER BY title, id LIMIT ?`,
    ...params, first + 1);
  const page = rows.slice(0, first);
  return {
    edges: page.map((node) => ({ cursor: encodeCursor(node), node })),
    pageInfo: { hasNextPage: rows.length > first, endCursor: page.length ? encodeCursor(page.at(-1)) : null },
  };
}
```

`pageOfBooks` combines search and cursor conditions with `AND`, fetches `first + 1` rows to
compute `hasNextPage`, escapes `LIKE` wildcards ([lesson 04](04-database-context.md)) and
validates both the page size and the cursor.

## Resolvers and server

```js title="resolvers.js"
import { DateTime } from "./scalars.js";
import { pageOfBooks } from "./pagination.js";

export function averageStars(reviews) {
  if (!reviews.length) return null;
  return Math.round((reviews.reduce((s, r) => s + r.stars, 0) / reviews.length) * 10) / 10;
}

function validateReview({ stars, body }) {
  const errors = [];
  if (!Number.isInteger(stars) || stars < 1 || stars > 5)
    errors.push({ field: ["input", "stars"], message: "Stars must be a whole number from 1 to 5.", code: "INVALID" });
  if (body.trim().length < 3 || body.length > 2000)
    errors.push({ field: ["input", "body"], message: "Reviews must be 3–2000 characters.", code: "INVALID" });
  return errors;
}

export const resolvers = {
  DateTime,
  Query: {
    books: (_, args, { sql }) => pageOfBooks(sql, args),
    book: (_, { id }, { loaders }) => loaders.book.load(id),
    author: (_, { id }, { loaders }) => loaders.author.load(id),
  },
  Mutation: {
    addReview: async (_, { input }, { sql, loaders, now }) => {
      const userErrors = validateReview(input);
      if (userErrors.length) return { review: null, userErrors };
      if (!(await loaders.book.load(input.bookId)))
        return { review: null, userErrors: [{ field: ["input", "bookId"], message: "No such book.", code: "NOT_FOUND" }] };
      const { lastInsertRowid } = sql.run(
        "INSERT INTO reviews (book_id, stars, body, created_at) VALUES (?, ?, ?, ?)",
        Number(input.bookId), input.stars, input.body.trim(), now());
      loaders.reviewsByBook.clear(input.bookId); // reads later in this request must see it
      return { review: sql.get("SELECT * FROM reviews WHERE id = ?", lastInsertRowid), userErrors: [] };
    },
  },
  Book: {
    author: (b, _, { loaders }) => loaders.author.load(b.author_id),
    reviews: (b, _, { loaders }) => loaders.reviewsByBook.load(b.id),
    averageStars: async (b, _, { loaders }) => averageStars(await loaders.reviewsByBook.load(b.id)),
  },
  Author: {
    books: (a, _, { loaders }) => loaders.booksByAuthor.load(a.id),
  },
  Review: {
    createdAt: (r) => r.created_at,
    book: (r, _, { loaders }) => loaders.book.load(r.book_id),
  },
};
```

```js title="server.js"
import { readFileSync } from "node:fs";
import { ApolloServer } from "@apollo/server";
import { resolvers } from "./resolvers.js";
import { withQueryLog } from "./db.js";
import { createLoaders } from "./loaders.js";

export const typeDefs = readFileSync(new URL("./schema.graphql", import.meta.url), "utf8");

export const createServer = () =>
  new ApolloServer({ typeDefs, resolvers, includeStacktraceInErrorResponses: false });

export function createContext(db, now = () => Date.now()) {
  const sql = withQueryLog(db);
  return { sql, loaders: createLoaders(sql), now };
}
```

```js title="main.js"
import { startStandaloneServer } from "@apollo/server/standalone";
import { createServer, createContext } from "./server.js";
import { openDb, seed } from "./db.js";

const db = seed(openDb(process.env.DB_FILE ?? "bookstore.db"));
const { url } = await startStandaloneServer(createServer(), {
  listen: { port: Number(process.env.PORT ?? 4000) },
  context: async () => createContext(db),
});
console.log(`Bookstore API at ${url}`);
```

Running it against a file database:

```console
$ node main.js
Bookstore API at http://localhost:4000/
$ curl -s localhost:4000/ -H 'content-type: application/json' \
    -d '{"query":"{ books(first: 2) { edges { node { title author { name } averageStars } } pageInfo { hasNextPage endCursor } } }"}'
{"data":{"books":{"edges":[{"node":{"title":"Count Zero","author":{"name":"William Gibson"},"averageStars":null}},{"node":{"title":"Dune","author":{"name":"Frank Herbert"},"averageStars":4.5}}],"pageInfo":{"hasNextPage":true,"endCursor":"WyJEdW5lIiwxXQ"}}}}
```

## Tests

```js title="bookstore.test.js"
import { test, beforeEach } from "node:test";
import assert from "node:assert/strict";
import { createServer, createContext } from "./server.js";
import { openDb, seed } from "./db.js";

const server = createServer();
const NOW = Date.UTC(2026, 4, 1, 12);
let db, ctx;
beforeEach(() => { db = seed(openDb()); });

async function gql(query, variables) {
  ctx = createContext(db, () => NOW);
  const res = await server.executeOperation({ query, variables }, { contextValue: ctx });
  return JSON.parse(JSON.stringify(res.body.singleResult));
}

const PAGE = `query($first: Int, $after: String, $search: String) {
  books(first: $first, after: $after, search: $search) {
    edges { node { title } } pageInfo { hasNextPage endCursor } } }`;
const titles = (r) => r.data.books.edges.map((e) => e.node.title);

test("pages through all books by title without gaps or repeats", async () => {
  const seen = [];
  let after = null, pages = 0;
  do {
    const r = await gql(PAGE, { first: 3, after });
    seen.push(...titles(r));
    after = r.data.books.pageInfo.hasNextPage ? r.data.books.pageInfo.endCursor : null;
    pages++;
  } while (after);
  assert.equal(pages, 3);
  assert.deepEqual(seen, [
    "Count Zero", "Dune", "Dune Messiah", "Emma", "Neuromancer", "Persuasion",
    "The Dispossessed", "The Left Hand of Darkness",
  ]);
});

test("search treats LIKE wildcards literally", async () => {
  assert.deepEqual(titles(await gql(PAGE, { search: "dune" })), ["Dune", "Dune Messiah"]);
  assert.deepEqual(titles(await gql(PAGE, { search: "%" })), []);
});

test("rejects oversized pages and forged cursors", async () => {
  assert.equal((await gql(PAGE, { first: 500 })).errors[0].extensions.code, "BAD_USER_INPUT");
  assert.equal((await gql(PAGE, { after: "bm9wZQ" })).errors[0].message, "Invalid cursor");
});

test("a deep catalog query runs a fixed number of statements", async () => {
  const r = await gql(`{ books(first: 50) { edges { node {
    title averageStars author { name books { title } } reviews { stars book { title } } } } } }`);
  assert.equal(r.errors, undefined);
  // books page, authors, books-by-author, reviews-by-book, books for Review.book
  assert.equal(ctx.sql.log.length, 5, ctx.sql.log.join("\n"));
});

test("author with books in publication order", async () => {
  const r = await gql(`{ author(id: 2) { name books { title year } } }`);
  assert.deepEqual(r.data.author, { name: "Jane Austen", books: [{ title: "Emma", year: 1815 }, { title: "Persuasion", year: 1817 }] });
});

const ADD = `mutation($input: AddReviewInput!) { addReview(input: $input) {
  review { stars createdAt book { title averageStars } } userErrors { field message code } } }`;

test("addReview returns the review and an updated average in the same response", async () => {
  const r = await gql(ADD, { input: { bookId: "1", stars: 3, body: "  Good, long.  " } });
  assert.deepEqual(r.data.addReview, {
    review: { stars: 3, createdAt: "2026-05-01T12:00:00.000Z", book: { title: "Dune", averageStars: 4 } },
    userErrors: [],
  });
});

test("addReview reports every invalid field as data", async () => {
  const r = await gql(ADD, { input: { bookId: "1", stars: 6, body: "x" } });
  assert.equal(r.errors, undefined);
  assert.equal(r.data.addReview.review, null);
  assert.deepEqual(r.data.addReview.userErrors.map((e) => e.field.join(".")), ["input.stars", "input.body"]);
});

test("addReview on a missing book", async () => {
  const r = await gql(ADD, { input: { bookId: "999", stars: 4, body: "Where is it?" } });
  assert.deepEqual(r.data.addReview.userErrors, [{ field: ["input", "bookId"], message: "No such book.", code: "NOT_FOUND" }]);
});

test("two reviews in one request: the second sees the first (loader cache cleared)", async () => {
  const r = await gql(`mutation {
    first: addReview(input: { bookId: "1", stars: 1, body: "Changed my mind." }) { review { book { averageStars } } }
    second: addReview(input: { bookId: "1", stars: 1, body: "Still no." }) { review { book { averageStars } } }
  }`);
  assert.equal(r.data.first.review.book.averageStars, 3.3);  // (5 + 4 + 1) / 3
  assert.equal(r.data.second.review.book.averageStars, 2.8); // (5 + 4 + 1 + 1) / 4 = 2.75
});
```

```console
$ node --test
✔ pages through all books by title without gaps or repeats
✔ search treats LIKE wildcards literally
✔ rejects oversized pages and forged cursors
✔ a deep catalog query runs a fixed number of statements
✔ author with books in publication order
✔ addReview returns the review and an updated average in the same response
✔ addReview reports every invalid field as data
✔ addReview on a missing book
✔ two reviews in one request: the second sees the first (loader cache cleared)
ℹ tests 9
ℹ pass 9
ℹ fail 0
```

The deep-catalog test selects authors, each author's books, reviews and each review's book,
for up to 50 books — and asserts **5** statements: one per distinct loader plus the page query.
Adding more books doesn't change that number.

## The bug the last test pins down

The first eight tests all passed with or without this line in `addReview`:

```js
loaders.reviewsByBook.clear(input.bookId); // reads later in this request must see it
```

In those tests nothing had loaded the book's reviews *before* the insert, so there was no stale
cache entry to clear. The problem only appears when one request adds two reviews to the same
book. The first `addReview` field's selection reads `averageStars`, which caches the review
list; the second `addReview` field runs afterwards (mutation fields run serially) and, without
the `clear`, reads that cached list — missing both new reviews in its average. Without the line,
the request returned:

```text
"first":  {"averageStars":3.3,"reviews":[{"stars":5},{"stars":4},{"stars":1}]}
"second": {"averageStars":3.3,"reviews":[{"stars":5},{"stars":4},{"stars":1}]}
```

The second response is wrong: it should include both one-star reviews and average 2.75
(rounded to 2.8). With the `clear`, it does. The ninth test now fails if anyone removes that
line:

```text
✖ two reviews in one request: the second sees the first (loader cache cleared)
  AssertionError [ERR_ASSERTION]: Expected values to be strictly equal:
```

The general rule: **after a write, clear (or prime) every loader whose cached value the write
changed.** Here only `reviewsByBook` was affected; the book and author rows didn't change.

## How It Actually Works

A request for `{ books { edges { node { author { books { title } } reviews { book { title } } } } } }`
executes level by level. `Query.books` runs one SQL statement and returns rows. Completing the
`edges` list calls `Book.author` and `Book.reviews` for every node in the same synchronous loop,
so each loader collects all its keys before DataLoader's scheduled dispatch runs — one
statement each. When those promises resolve, the executor continues into `Author.books` for all
authors and `Review.book` for all reviews, again collecting keys per loader. `Review.book`
uses the `book` loader, which nothing has touched yet in this request — the page rows came
from `pageOfBooks`, not from the loader — so it runs one batched statement: that's the fifth.
The statement count is therefore proportional to
the number of *distinct loaders reached*, not the number of rows.

## Review checklist

- [ ] Pagination is keyset-based, bounded, and its cursor is validated.
- [ ] Every relation goes through a per-request loader with normalised keys.
- [ ] Writes clear the loaders they affect.
- [ ] Expected failures are `userErrors`; unexpected ones are GraphQL errors without stack traces.
- [ ] A test fails if a catalog query's statement count grows.

## Exercise

1. Prime the `book` loader with the page rows in `pageOfBooks` (`loaders.book.prime(row.id, row)`)
   and update the statement-count test. Which statement disappears?
2. Add `Author.reviewCount: Int!` without adding an N+1 (a new loader running a grouped
   `COUNT`).
3. Add `deleteReview(id: ID!)` with a payload returning `deletedReviewId` and the updated book.
   What does it need to clear?
4. Add a `sort: BookSort = TITLE` argument with `TITLE` and `NEWEST` options. How must the cursor
   change so a cursor from one sort order can't be used with the other?
