---
description: "Fix N+1 with DataLoader: batch functions, ordering and missing keys, one-to-many loaders, per-request caching, and the pitfalls each one hides."
---

# 06 · DataLoader: Batching & Per-Request Caching

[Lesson 05](05-n-plus-one.md) measured 1,001 SQL statements for a query that needs three. The
missing piece was a way for hundreds of independent resolver calls to be gathered into one
database query without the resolvers knowing about each other. **DataLoader** — a tiny library
originally written at Facebook alongside GraphQL itself, and now maintained by the GraphQL
Foundation — does exactly that. Version 2.2.3 is used here.

```bash
npm install dataloader
```

## The idea in one paragraph

You give DataLoader a **batch function** that takes an array of keys and returns a promise of
an array of values, *in the same order*. Resolvers call `loader.load(key)`, which returns a
promise immediately but doesn't run anything yet. DataLoader waits until the current tick of
the event loop has finished — by which time the executor has called every sibling resolver —
then calls your batch function **once** with all the keys collected, de-duplicated. Each
`load` promise resolves with its value. A loader also remembers results, so a second
`load(5)` in the same request costs nothing.

## Loaders for the bookstore

```js title="loaders.js"
import DataLoader from "dataloader";

// Return values in the same order as keys; null (or an Error) for missing keys.
function byId(rows, keys, key = "id") {
  const map = new Map(rows.map((r) => [r[key], r]));
  return keys.map((k) => map.get(k) ?? null);
}

// One-to-many: group rows by a foreign key, [] for keys with no rows.
function groupBy(rows, keys, key) {
  const groups = new Map(keys.map((k) => [k, []]));
  for (const r of rows) groups.get(r[key])?.push(r);
  return keys.map((k) => groups.get(k));
}

const qmarks = (n) => Array(n).fill("?").join(",");

export function createLoaders(sql) {
  return {
    author: new DataLoader(async (ids) => {
      const rows = sql.all(`SELECT * FROM authors WHERE id IN (${qmarks(ids.length)})`, ...ids);
      return byId(rows, ids);
    }),
    reviewsByBook: new DataLoader(async (bookIds) => {
      const rows = sql.all(
        `SELECT * FROM reviews WHERE book_id IN (${qmarks(bookIds.length)}) ORDER BY id`, ...bookIds);
      return groupBy(rows, bookIds, "book_id");
    }),
    booksByAuthor: new DataLoader(async (authorIds) => {
      const rows = sql.all(
        `SELECT * FROM books WHERE author_id IN (${qmarks(authorIds.length)}) ORDER BY id`, ...authorIds);
      return groupBy(rows, authorIds, "author_id");
    }, { maxBatchSize: 500 }),
  };
}
```

Two shapes cover almost every case:

- **One-to-one** (`author`): fetch rows by primary key, then **reorder** to match the keys and
  fill gaps with `null`. SQL `IN (…)` returns rows in whatever order it likes, and simply
  omits ids that don't exist.
- **One-to-many** (`reviewsByBook`, `booksByAuthor`): fetch all child rows for all parent ids,
  then **group** them, returning `[]` for parents with no children.

`maxBatchSize` splits very large batches. Databases cap the number of bound parameters per
statement (SQLite's default limit is 32,766 in current versions; other databases differ), and
a gigantic `IN` list may also plan badly. Pick a size your database handles well.

## Wiring them in

```js title="batched.mjs"
import { makeExecutableSchema } from "@graphql-tools/schema";
import { graphql } from "graphql";
import { openDb, withQueryLog } from "./db.js";
import { createLoaders } from "./loaders.js";

const db = openDb();
{
  const insA = db.prepare("INSERT INTO authors (name) VALUES (?)"), insB = db.prepare("INSERT INTO books (title, year, author_id) VALUES (?, ?, ?)"), insR = db.prepare("INSERT INTO reviews (book_id, stars, body) VALUES (?, ?, ?)");
  db.exec("BEGIN"); let id = 0;
  for (let a = 1; a <= 50; a++) { insA.run(`Author ${a}`); for (let b = 0; b < 10; b++) { insB.run(`Book ${a}.${b}`, 1950 + b, a); id++; insR.run(id, 1 + (id % 5), "ok"); insR.run(id, 1 + ((id + 2) % 5), "fine"); } }
  db.exec("COMMIT");
}

const schema = makeExecutableSchema({
  typeDefs: /* GraphQL */ `
    type Query { books(limit: Int = 1000): [Book!]! }
    type Book { id: ID! title: String! author: Author! reviews: [Review!]! }
    type Author { id: ID! name: String! books: [Book!]! }
    type Review { stars: Int! }
  `,
  resolvers: {
    Query: { books: (_, { limit }, { sql }) => sql.all("SELECT * FROM books ORDER BY id LIMIT ?", limit) },
    Book: {
      author: (b, _, { loaders }) => loaders.author.load(b.author_id),
      reviews: (b, _, { loaders }) => loaders.reviewsByBook.load(b.id),
    },
    Author: { books: (a, _, { loaders }) => loaders.booksByAuthor.load(a.id) },
  },
});

async function measure(source, variableValues) {
  const sql = withQueryLog(db);
  const contextValue = { sql, loaders: createLoaders(sql) }; // fresh loaders per request
  const t = performance.now();
  const r = await graphql({ schema, source, contextValue, variableValues });
  if (r.errors) throw new Error(r.errors[0].message);
  return { statements: sql.log.length, ms: performance.now() - t, log: sql.log };
}

const queries = {
  "books { title }": `query($n:Int){ books(limit:$n) { title } }`,
  "books { author }": `query($n:Int){ books(limit:$n) { title author { name } } }`,
  "books { author reviews }": `query($n:Int){ books(limit:$n) { title author { name } reviews { stars } } }`,
  "books { author { books } }": `query($n:Int){ books(limit:$n) { author { books { title } } } }`,
};
console.log("query".padEnd(28), "N=10".padStart(8), "N=100".padStart(8), "N=500".padStart(8));
for (const [label, q] of Object.entries(queries)) {
  const cells = [];
  for (const n of [10, 100, 500]) cells.push(String((await measure(q, { n })).statements).padStart(8));
  console.log(label.padEnd(28), ...cells);
}
const m = await measure(queries["books { author reviews }"], { n: 10 });
console.log("\nstatements for books { author reviews } N=10:");
for (const s of m.log) console.log(" ", s.replace(/(\?,){3,}\?/, (x) => `?×${x.split(",").length}`));

const times = [];
for (let i = 0; i < 8; i++) times.push((await measure(queries["books { author reviews }"], { n: 500 })).ms);
times.shift(); times.sort((a, b) => a - b);
console.log(`\nbooks { author reviews } N=500 with DataLoader: median ${times[3].toFixed(1)} ms over 7 runs`);
```

The resolvers changed by one line each — `sql.get(…)` became `loaders.author.load(…)`. The
important line is in `measure`: **`createLoaders(sql)` is called for every request**. Why that
matters is in the pitfalls below.

## The numbers, again

```console
$ node batched.mjs
query                            N=10    N=100    N=500
books { title }                     1        1        1
books { author }                    2        2        2
books { author reviews }            3        3        3
books { author { books } }          3        3        3

statements for books { author reviews } N=10:
  SELECT * FROM books ORDER BY id LIMIT ?
  SELECT * FROM authors WHERE id IN (?)
  SELECT * FROM reviews WHERE book_id IN (?×10) ORDER BY id

books { author reviews } N=500 with DataLoader: median 4.7 ms over 7 runs
```

(The `?×10` is the script abbreviating ten `?` parameters.) Compare with lesson 05:

| Query | Before | After |
|---|---|---|
| `books { author }`, N=500 | 501 | 2 |
| `books { author reviews }`, N=500 | 1,001 | 3 |
| `books { author { books } }`, N=500 | 1,001 | 3 |

Statement count is now one per **level of the query**, not per row. Note `authors WHERE id IN
(?)` with a single `?`: the first ten books share one author, and DataLoader de-duplicated
ten `load(1)` calls into one key. The timing — about 4.7 ms versus about 24 ms before, on the
same machine — is close to the 3.7 ms of the hand-optimised version from lesson 05.

## Pitfalls, each one run

```js title="pitfalls06.mjs"
import DataLoader from "dataloader";
import { openDb, seed } from "./db.js";
const db = seed(openDb());
const q = (ids) => db.prepare(`SELECT * FROM authors WHERE id IN (${ids.map(() => "?").join(",")})`).all(...ids);

// 1. Forgetting to reorder / fill gaps
const naive = new DataLoader(async (ids) => q(ids));
try { await Promise.all([naive.load(3), naive.load(1), naive.load(99)]); }
catch (e) { console.log("1. naive:", e.message.split("\n")[0]); }

const unordered = new DataLoader(async (ids) => q(ids)); // same length when all exist, but wrong order
const [x, y] = await Promise.all([unordered.load(3), unordered.load(1)]);
console.log("1b. asked for 3 then 1, got:", x.id, y.id);

// 2. Missing keys: null vs Error
const careful = new DataLoader(async (ids) => {
  const m = new Map(q(ids).map((r) => [r.id, r]));
  return ids.map((id) => m.get(id) ?? new Error(`Author ${id} not found`));
});
const results = await Promise.allSettled([careful.load(2), careful.load(99)]);
console.log("2.", results.map((r) => r.status === "fulfilled" ? r.value.name : `rejected: ${r.reason.message}`));

// 3. Key types: "1" vs 1
let batches = [];
const typed = new DataLoader(async (ids) => { batches.push(ids); return ids.map((id) => ({ id })); });
await Promise.all([typed.load(1), typed.load("1")]);
console.log("3. keys sent to batch fn:", JSON.stringify(batches));

// 4. Long-lived loader serves stale data
const shared = new DataLoader(async (ids) => { const m = new Map(q(ids).map((r) => [r.id, r])); return ids.map((i) => m.get(i) ?? null); });
console.log("4. before:", (await shared.load(2)).name);
db.prepare("UPDATE authors SET name = ? WHERE id = ?").run("J. Austen", 2);
console.log("4. after UPDATE, same loader:", (await shared.load(2)).name);
shared.clear(2);
console.log("4. after clear(2):", (await shared.load(2)).name);

// 5. Batching only collects loads made in the same tick
batches = [];
const tick = new DataLoader(async (ids) => { batches.push([...ids]); return ids.map((id) => ({ id })); });
await tick.load(1); await tick.load(2);           // sequential awaits
await Promise.all([tick.load(3), tick.load(4)]);  // concurrent
console.log("5. batches:", JSON.stringify(batches));
```

```console
$ node pitfalls06.mjs
1. naive: Cannot convert object to primitive value
1b. asked for 3 then 1, got: 1 3
2. [ 'Jane Austen', 'rejected: Author 99 not found' ]
3. keys sent to batch fn: [[1,"1"]]
4. before: Jane Austen
4. after UPDATE, same loader: Jane Austen
4. after clear(2): J. Austen
5. batches: [[1],[2],[3,4]]
```

**1. Returning rows straight from the database.** With a missing key (99), the array is
shorter than the keys array. DataLoader detects that and tries to build a helpful message
listing the keys and values — but `node:sqlite` rows are null-prototype objects that can't be
converted to strings, so the message-building itself throws and you get the baffling
`Cannot convert object to primitive value`. With plain objects, the real message is:

```text
DataLoader must be constructed with a function which accepts Array<key> and returns Promise<Array<value>>, but the function did not return a Promise of an Array of the same length as the Array of keys.

Keys:
3,1,99

Values:
[object Object],[object Object]
```

**1b. Wrong order is worse, because it's silent.** When every key exists the lengths match, so
there's no error — but asking for author 3 and then author 1 returned author 1 and then
author 3. Every book would show *some* author, just not its own. Always reorder by key.

**2. Missing keys: `null` or `Error`.** Returning `null` for a missing key resolves that `load`
with `null`. Returning an `Error` instance rejects only that key's promise; the others still
succeed. Choose based on whether "missing" is normal (nullable field) or a data integrity
problem.

**3. Key identity.** `1` and `"1"` are different cache keys, so both went to the batch
function. GraphQL `ID` arguments arrive as strings while database foreign keys are often
integers — normalise keys (or provide `cacheKeyFn`) or you'll get duplicate fetches and
cache misses.

**4. Caching outlives the data.** A loader that lives longer than a request kept returning
"Jane Austen" after the row was updated. Within one request that's usually what you want
(consistent reads); across requests it's a stale-data and data-leak bug — a loader created at
module level would also serve user A's permission-filtered results to user B. Create loaders
per request, and after a mutation in the same request, call `loader.clear(key)` (or
`loader.prime(key, newValue)`) before reading the changed object back.

**5. Batching needs concurrency.** Two `load`s awaited one after the other produced two
batches. Batching only collects loads made before the scheduler runs, so code like
`for (const id of ids) await loader.load(id)` gets none of the benefit. Use
`loader.loadMany(ids)` or `Promise.all`.

## How It Actually Works

Each `load(key)` first checks the loader's cache (a `Map` keyed by `cacheKeyFn(key)`); a hit
returns the stored promise. On a miss it creates a promise, stores it, and pushes the key and
the promise's resolve/reject callbacks onto the current batch. When the first key of a new
batch is added, DataLoader schedules a dispatch. In Node its default scheduler is roughly
"after the current promise jobs have run": it waits for a resolved promise and then
`process.nextTick` — late enough that the executor has called every sibling resolver in the
same tick, early enough not to add a timer's delay.

Dispatch calls your batch function with the collected keys (split by `maxBatchSize`), checks
that the result is an array of the same length, and then resolves or rejects each stored
promise by index — which is why order matters so much. Because the cached value is the
*promise*, two loads for the same key made before the batch even ran share one result.

On the GraphQL side, this works because `completeListValue` calls each item's resolvers
synchronously in a loop: all 500 `author.load()` calls happen inside that loop, before the
scheduled dispatch can run.

## Common mistakes

- **A module-level loader.** Stale data and cross-user leaks. One set of loaders per request,
  built in the context function.
- **Not reordering results** — silent wrong answers.
- **Sequential awaits** of `load` in loops.
- **Mismatched key types** between string `ID`s and integer columns.
- **Loaders for paginated or filtered children** without including the arguments in the key.
  `reviewsByBook.load(id)` for "latest 3 reviews" and "all reviews" must be different loaders
  or use composite keys.
- **Forgetting to clear after writes** when a mutation reads back what it just changed.

## Exercise

1. Add `Review.book` with a `bookById` loader and confirm
   `{ books { reviews { book { title } } } }` runs a constant number of statements.
2. Write a `cacheKeyFn` that normalises `"1"` and `1` to the same key, and prove it with the
   pitfall 3 script.
3. Add `Book.reviews(minStars: Int)` and make the loader handle it correctly. (Hint: key by
   `` `${bookId}:${minStars}` `` or create a loader per argument value.)
4. Turn the "no more than 3 statements" test from lesson 05's exercise green.
