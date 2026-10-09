---
description: "Measure the GraphQL N+1 problem on a 500-book SQLite database: statement counts by query shape, why resolvers cause it, and why naive fixes fall short."
---

# 05 · The N+1 Problem: Seeing and Counting It

[Lesson 04](04-database-context.md) ended with nine SQL statements for eight books. This
lesson takes that observation seriously: it builds a bigger database, counts statements for
different query shapes, times them, and explains why GraphQL's execution model produces this
pattern so naturally. The fix — batching — is the next lesson. Understanding the problem
precisely first is what makes that fix obvious instead of magical.

## What "N+1" means

You run **1** query to get a list of N things, then **N** more queries — one per item — to get
something related to each. In REST code it's a known anti-pattern that a careful developer
avoids by writing a join. In GraphQL it's the *default* outcome, because each field's resolver
runs independently for each parent object and knows nothing about its siblings.

## A measuring harness

50 authors × 10 books = 500 books, each with 2 reviews, in in-memory SQLite. Every resolver
does the obvious thing:

```js title="nplus1.mjs"
import { makeExecutableSchema } from "@graphql-tools/schema";
import { graphql } from "graphql";
import { openDb, withQueryLog } from "./db.js";

function bigDb(nAuthors, booksPerAuthor) {
  const db = openDb();
  const insA = db.prepare("INSERT INTO authors (name) VALUES (?)");
  const insB = db.prepare("INSERT INTO books (title, year, author_id) VALUES (?, ?, ?)");
  const insR = db.prepare("INSERT INTO reviews (book_id, stars, body) VALUES (?, ?, ?)");
  db.exec("BEGIN");
  let bookId = 0;
  for (let a = 1; a <= nAuthors; a++) {
    insA.run(`Author ${a}`);
    for (let b = 0; b < booksPerAuthor; b++) {
      insB.run(`Book ${a}.${b}`, 1950 + b, a);
      bookId++;
      insR.run(bookId, 1 + (bookId % 5), "ok");
      insR.run(bookId, 1 + ((bookId + 2) % 5), "fine");
    }
  }
  db.exec("COMMIT");
  return db;
}

const typeDefs = /* GraphQL */ `
  type Query { books(limit: Int = 1000): [Book!]! }
  type Book { id: ID! title: String! author: Author! reviews: [Review!]! }
  type Author { id: ID! name: String! books: [Book!]! }
  type Review { stars: Int! }
`;
const resolvers = {
  Query: { books: (_, { limit }, { sql }) => sql.all("SELECT * FROM books ORDER BY id LIMIT ?", limit) },
  Book: {
    author: (b, _, { sql }) => sql.get("SELECT * FROM authors WHERE id = ?", b.author_id),
    reviews: (b, _, { sql }) => sql.all("SELECT * FROM reviews WHERE book_id = ?", b.id),
  },
  Author: { books: (a, _, { sql }) => sql.all("SELECT * FROM books WHERE author_id = ?", a.id) },
};
const schema = makeExecutableSchema({ typeDefs, resolvers });

export async function measure(db, source, variableValues) {
  const sql = withQueryLog(db);
  const t = performance.now();
  const r = await graphql({ schema, source, contextValue: { sql }, variableValues });
  const ms = performance.now() - t;
  if (r.errors) throw new Error(r.errors[0].message);
  return { statements: sql.log.length, distinct: new Set(sql.log).size, ms, log: sql.log };
}

if (import.meta.url === `file://${process.argv[1]}`) {
  const db = bigDb(50, 10); // 50 authors x 10 books = 500 books, 1000 reviews
  const queries = {
    "books { title }": `query($n:Int){ books(limit:$n) { title } }`,
    "books { author }": `query($n:Int){ books(limit:$n) { title author { name } } }`,
    "books { author reviews }": `query($n:Int){ books(limit:$n) { title author { name } reviews { stars } } }`,
    "books { author { books } }": `query($n:Int){ books(limit:$n) { author { books { title } } } }`,
  };
  console.log("query".padEnd(28), "N=10".padStart(8), "N=100".padStart(8), "N=500".padStart(8));
  for (const [label, q] of Object.entries(queries)) {
    const cells = [];
    for (const n of [10, 100, 500]) cells.push(String((await measure(db, q, { n })).statements).padStart(8));
    console.log(label.padEnd(28), ...cells);
  }
  const m = await measure(db, queries["books { author }"], { n: 10 });
  console.log("\nfirst 4 statements for books { author } N=10:\n " + m.log.slice(0, 4).join("\n "));
  const authorIds = new Set();
  for (const r of db.prepare("SELECT author_id FROM books ORDER BY id LIMIT 10").all()) authorIds.add(r.author_id);
  console.log(`distinct authors among those 10 books: ${authorIds.size}`);

  // Timing: warm up then median of 7 runs
  for (const [label, q] of [["books { author reviews }", queries["books { author reviews }"]]]) {
    const times = [];
    for (let i = 0; i < 8; i++) times.push((await measure(db, q, { n: 500 })).ms);
    times.shift(); times.sort((a, b) => a - b);
    console.log(`\n${label} N=500: median ${times[3].toFixed(1)} ms over 7 runs (in-memory SQLite)`);
  }
  // Same data with two hand-written queries
  const times = [];
  for (let i = 0; i < 8; i++) {
    const t = performance.now();
    const books = db.prepare("SELECT * FROM books ORDER BY id LIMIT 500").all();
    const authors = db.prepare("SELECT * FROM authors").all();
    const reviews = db.prepare("SELECT * FROM reviews WHERE book_id <= 500").all();
    times.push(performance.now() - t);
  }
  times.shift(); times.sort((a, b) => a - b);
  console.log(`3 hand-written queries for the same rows: median ${times[3].toFixed(1)} ms`);
}
```

`measure` reuses `withQueryLog` from lesson 04 to count statements. (`db.js` is the same
module as in lesson 04.)

## The numbers

```console
$ node nplus1.mjs
query                            N=10    N=100    N=500
books { title }                     1        1        1
books { author }                   11      101      501
books { author reviews }           21      201     1001
books { author { books } }         21      201     1001

first 4 statements for books { author } N=10:
 SELECT * FROM books ORDER BY id LIMIT ?
 SELECT * FROM authors WHERE id = ?
 SELECT * FROM authors WHERE id = ?
 SELECT * FROM authors WHERE id = ?
distinct authors among those 10 books: 1

books { author reviews } N=500: median 23.7 ms over 7 runs (in-memory SQLite)
3 hand-written queries for the same rows: median 0.6 ms
```

Read the table as formulas:

| Query shape | Statements |
|---|---|
| list only | 1 |
| list → one relation | 1 + N |
| list → two relations | 1 + 2N |
| list → relation → relation | 1 + N + N |

Two things make it worse than the formula suggests:

1. **Duplicates.** The first ten books were all by "Author 1", so ten statements fetched the
   *same* author row ten times. With `{ author { books } }` that author's ten books were also
   fetched ten times over.
2. **It's driven by the client.** The server code didn't change between rows of the table; only
   the query did. Any client can add one more nested field and multiply your database load.
   That's also a security concern ([Level 3 · 05](../level-3/05-security-limits.md)).

## Is it actually slow?

On this laptop, against an in-memory database, the 1,001-statement query took a median of
about 24 ms. To separate GraphQL's own execution cost from the database round trips, the same
GraphQL query was run with resolvers reading from in-memory maps built by three queries up
front:

```js title="nplus1-fair.mjs"
import { makeExecutableSchema } from "@graphql-tools/schema";
import { graphql } from "graphql";
import { openDb } from "./db.js";
// same 500-book dataset, but resolvers read from maps built by 3 queries up front
const db = openDb();
const insA = db.prepare("INSERT INTO authors (name) VALUES (?)"), insB = db.prepare("INSERT INTO books (title, year, author_id) VALUES (?, ?, ?)"), insR = db.prepare("INSERT INTO reviews (book_id, stars, body) VALUES (?, ?, ?)");
db.exec("BEGIN"); let id = 0;
for (let a = 1; a <= 50; a++) { insA.run(`Author ${a}`); for (let b = 0; b < 10; b++) { insB.run(`Book ${a}.${b}`, 1950 + b, a); id++; insR.run(id, 1 + (id % 5), "ok"); insR.run(id, 1 + ((id + 2) % 5), "fine"); } }
db.exec("COMMIT");
const schema = makeExecutableSchema({
  typeDefs: `type Query { books: [Book!]! } type Book { title: String! author: Author! reviews: [Review!]! } type Author { name: String! } type Review { stars: Int! }`,
  resolvers: {
    Query: { books: (_, __, c) => c.books },
    Book: { author: (b, _, c) => c.authors.get(b.author_id), reviews: (b, _, c) => c.reviews.get(b.id) ?? [] },
  },
});
const times = [];
for (let i = 0; i < 8; i++) {
  const t = performance.now();
  const books = db.prepare("SELECT * FROM books ORDER BY id LIMIT 500").all();
  const authors = new Map(db.prepare("SELECT * FROM authors").all().map((a) => [a.id, a]));
  const reviews = new Map();
  for (const r of db.prepare("SELECT * FROM reviews WHERE book_id <= 500").all()) (reviews.get(r.book_id) ?? reviews.set(r.book_id, []).get(r.book_id)).push(r);
  const r = await graphql({ schema, source: "{ books { title author { name } reviews { stars } } }", contextValue: { books, authors, reviews } });
  if (r.errors) throw r.errors[0];
  times.push(performance.now() - t);
}
times.shift(); times.sort((a, b) => a - b);
console.log(`same GraphQL query, 3 queries + in-memory lookups: median ${times[3].toFixed(1)} ms`);
```

```console
$ node nplus1-fair.mjs
same GraphQL query, 3 queries + in-memory lookups: median 3.7 ms
```

So roughly 20 of the 24 ms were the 998 extra statements. These are local measurements on one
machine with a database in the same process — treat them as a ratio, not a benchmark. With a
networked database the gap grows sharply, because each statement also pays a network round
trip. As arithmetic only (not measured here): at a 1 ms round trip, 1,000 sequential-ish
statements add on the order of a second, and they also hold database connections and CPU
that other requests need.

## Why the resolvers can't see it

Look at what each resolver knows when it runs:

```js
author: (b, _, { sql }) => sql.get("SELECT * FROM authors WHERE id = ?", b.author_id),
```

One parent, one id. It doesn't know it's one of 500 siblings, or that 49 of them want the same
author. The executor calls it 500 times, and each call does what it can with what it has.
That isolation is a feature — it's why resolvers are simple and composable — but it rules out
a resolver "just" writing a join.

## Fixes that don't quite work

**Join in the parent.** `Query.books` could `SELECT books.*, authors.name … JOIN authors`.
That fixes `books { author }` but makes every `books` query pay for the join even when the
client didn't ask for authors, and does nothing for `reviews`, or for `author` reached through
a different path (`review { book { author } }`).

**Look ahead with `info`.** A resolver can inspect `info.fieldNodes` to see which sub-fields
were requested and choose a join accordingly. Libraries exist that compile whole GraphQL
queries into SQL this way. It can be very efficient, but it couples resolvers tightly to the
database layout and gets complex with fragments, aliases, interfaces and arguments on nested
fields. It's an advanced trade-off, not a default.

**Memoise by id per request.** Cache `authors` lookups in the context so each id is fetched
once. That removes duplicates (ten "Author 1" lookups become one) but not the pattern:
500 books by 50 distinct authors would still be 1 + 50 author statements, one per distinct
id.

What's needed is a way for those 500 independent `author` calls to be *collected* and turned
into one `WHERE id IN (…)` query, without the resolvers knowing about each other. That's
exactly what DataLoader does.

## How It Actually Works

The executor resolves a list field by calling `completeListValue`, which loops over the items
and calls `completeValue` for each. For an object item that means `executeFields` on its
selection set, which calls each field's resolver for that one item. With synchronous
resolvers (as with `node:sqlite`), the order is strictly depth-first: book 1's `author`, book
1's `reviews`, then book 2's, and so on — each statement runs and finishes before the next
starts. With asynchronous resolvers, all 500 `author` calls are *started* in the same tick of
the event loop before any of them resolve. That detail is what makes batching possible:
something can wait until the end of the current tick, see all 500 requested ids, and run one
query. The next lesson builds on exactly that.

## Common mistakes

- **Not measuring.** N+1 is invisible in development with ten rows and a local database.
  Count statements per request in tests and logs.
- **Fixing one path with a join** and assuming the problem is solved.
- **Blaming GraphQL for slowness** that's really unbatched data access — the same code shape
  in REST would be just as slow.
- **Adding a cache with a long lifetime** to hide N+1. It hides the load until the cache is
  cold, and it serves stale data across users.

## Exercise

1. Add `Review.book` and measure `{ books { reviews { book { title } } } }` at N=10/100/500.
   Write the formula.
2. Implement the "memoise by id per request" fix for `Book.author` using a `Map` in context.
   Re-run the table. Which cells changed, and which didn't?
3. Change the dataset to 500 authors with one book each and re-run with memoisation. Why does
   memoisation stop helping?
4. Write a test that fails if `{ books { author { name } } }` ever runs more than 3 statements.
   Keep it for the next lesson.
