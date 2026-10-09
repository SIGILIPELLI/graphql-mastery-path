---
description: "Back a GraphQL API with SQLite: per-request context, parameterised queries, counting SQL per request, keeping DB errors out of responses."
---

# 04 · Connecting a Database: SQLite & Per-Request Context

Until now the data lived in arrays. This lesson connects a real database and introduces the
pattern every production GraphQL server uses: a **per-request context** that carries the
database handle, the current user and request-scoped helpers into every resolver. It also
installs an instrument you'll rely on for the next three lessons — a counter of SQL statements
per request — and uses it to spot a problem we'll fix properly in [lesson 05](05-n-plus-one.md).

SQLite is used because it needs no server: Node's built-in `node:sqlite` module (Node 26.3
here, bundling SQLite 3.53.4) gives a synchronous API with no native packages to install. The
patterns transfer directly to PostgreSQL or MySQL clients; the main difference is that their
calls return promises.

## The database module

```js title="db.js"
import { DatabaseSync } from "node:sqlite";

export function openDb(file = ":memory:") {
  const db = new DatabaseSync(file);
  db.exec(`
    PRAGMA foreign_keys = ON;
    CREATE TABLE IF NOT EXISTS authors (
      id   INTEGER PRIMARY KEY,
      name TEXT NOT NULL
    );
    CREATE TABLE IF NOT EXISTS books (
      id        INTEGER PRIMARY KEY,
      title     TEXT NOT NULL,
      year      INTEGER NOT NULL,
      author_id INTEGER NOT NULL REFERENCES authors(id)
    );
    CREATE TABLE IF NOT EXISTS reviews (
      id      INTEGER PRIMARY KEY,
      book_id INTEGER NOT NULL REFERENCES books(id),
      stars   INTEGER NOT NULL CHECK (stars BETWEEN 1 AND 5),
      body    TEXT NOT NULL
    );
  `);
  return db;
}

export function seed(db) {
  const authors = ["Frank Herbert", "Jane Austen", "William Gibson", "Ursula K. Le Guin"];
  const books = [
    ["Dune", 1965, 1], ["Dune Messiah", 1969, 1], ["Emma", 1815, 2], ["Persuasion", 1817, 2],
    ["Neuromancer", 1984, 3], ["Count Zero", 1986, 3], ["The Dispossessed", 1974, 4],
    ["The Left Hand of Darkness", 1969, 4],
  ];
  const insA = db.prepare("INSERT INTO authors (name) VALUES (?)");
  const insB = db.prepare("INSERT INTO books (title, year, author_id) VALUES (?, ?, ?)");
  const insR = db.prepare("INSERT INTO reviews (book_id, stars, body) VALUES (?, ?, ?)");
  db.exec("BEGIN");
  for (const a of authors) insA.run(a);
  for (const b of books) insB.run(...b);
  [[1, 5, "Sand everywhere, worth it."], [1, 4, "Slow start."], [3, 5, "Witty."], [5, 4, "Dense but rewarding."], [7, 5, "Quietly radical."]]
    .forEach((r) => insR.run(...r));
  db.exec("COMMIT");
  return db;
}

// Wrap a DatabaseSync so every statement run is recorded — handy for counting queries.
export function withQueryLog(db) {
  const log = [];
  return {
    log,
    all(sql, ...params) { log.push(sql); return db.prepare(sql).all(...params); },
    get(sql, ...params) { log.push(sql); return db.prepare(sql).get(...params); },
    run(sql, ...params) { log.push(sql); return db.prepare(sql).run(...params); },
    exec(sql) { log.push(sql); return db.exec(sql); },
  };
}
```

Three things to note:

- **Foreign keys and a `CHECK` constraint** live in the database. The API will lean on them.
- **`seed` wraps inserts in a transaction** — much faster in SQLite than autocommitting each row.
- **`withQueryLog`** wraps the connection so every SQL string is recorded. It's a teaching
  tool, but production servers do the equivalent with tracing (Level 4).

## The server

```js title="server04.mjs"
import { ApolloServer } from "@apollo/server";
import { startStandaloneServer } from "@apollo/server/standalone";
import { openDb, seed, withQueryLog } from "./db.js";

const db = seed(openDb()); // one connection for the process

const typeDefs = /* GraphQL */ `
  type Query {
    books(search: String): [Book!]!
    book(id: ID!): Book
  }
  type Mutation {
    addReview(bookId: ID!, stars: Int!, body: String!): Review
  }
  type Book { id: ID! title: String! year: Int! author: Author! reviews: [Review!]! }
  type Author { id: ID! name: String! }
  type Review { id: ID! stars: Int! body: String! book: Book! }
`;

const resolvers = {
  Query: {
    books: (_, { search }, { sql }) =>
      search
        ? sql.all("SELECT * FROM books WHERE title LIKE ? ORDER BY id", `%${search}%`)
        : sql.all("SELECT * FROM books ORDER BY id"),
    book: (_, { id }, { sql }) => sql.get("SELECT * FROM books WHERE id = ?", id) ?? null,
  },
  Mutation: {
    addReview: (_, { bookId, stars, body }, { sql }) => {
      const { lastInsertRowid } = sql.run(
        "INSERT INTO reviews (book_id, stars, body) VALUES (?, ?, ?)", bookId, stars, body);
      return sql.get("SELECT * FROM reviews WHERE id = ?", lastInsertRowid);
    },
  },
  Book: {
    author: (b, _, { sql }) => sql.get("SELECT * FROM authors WHERE id = ?", b.author_id),
    reviews: (b, _, { sql }) => sql.all("SELECT * FROM reviews WHERE book_id = ? ORDER BY id", b.id),
  },
  Review: {
    book: (r, _, { sql }) => sql.get("SELECT * FROM books WHERE id = ?", r.book_id),
  },
};

const server = new ApolloServer({
  typeDefs,
  resolvers,
  plugins: [{
    async requestDidStart() {
      return {
        async willSendResponse({ contextValue, response }) {
          const n = contextValue.sql.log.length;
          response.http.headers.set("x-sql-statements", String(n));
          console.log(`[${contextValue.requestId}] ${n} SQL statements`);
        },
      };
    },
  }],
});

let nextId = 1;
const { url } = await startStandaloneServer(server, {
  listen: { port: 4000 },
  context: async () => ({ sql: withQueryLog(db), requestId: nextId++ }),
});
console.log(`ready at ${url}`);
```

### Why the context is built per request

The **connection** is created once per process (`const db = …` at module level): opening a
database per request would be slow, and with a networked database you'd use a pool here
instead. The **context**, though, is created inside the `context` function, so each request
gets its own `withQueryLog` wrapper with an empty `log`, and its own `requestId`. Anything
that must not leak between users — the authenticated user, per-request caches, the query log
— belongs in that function.

The plugin's `willSendResponse` hook reads the context at the end of the request, so it can
report how many statements that one request ran, both in a response header and on the server
console.

## Running it

```console
$ node server04.mjs
ready at http://localhost:4000/
```

A single book with its author and reviews:

```console
$ curl -s -D - localhost:4000/ -H 'content-type: application/json' \
    -d '{"query":"{ book(id: 1) { title year author { name } reviews { stars body } } }"}'
x-sql-statements: 3
{"data":{"book":{"title":"Dune","year":1965,"author":{"name":"Frank Herbert"},"reviews":[{"stars":5,"body":"Sand everywhere, worth it."},{"stars":4,"body":"Slow start."}]}}}
```

(Other headers omitted.) Three statements: the book, its author, its reviews. That's what
you'd write by hand.

Now the list with authors:

```console
$ curl -s -D - localhost:4000/ -H 'content-type: application/json' \
    -d '{"query":"{ books { title author { name } } }"}'
x-sql-statements: 9
{"data":{"books":[{"title":"Dune","author":{"name":"Frank Herbert"}},{"title":"Dune Messiah","author":{"name":"Frank Herbert"}},{"title":"Emma","author":{"name":"Jane Austen"}}, …]}}
```

**Nine** statements for eight books: one for the list, then one `SELECT * FROM authors WHERE
id = ?` per book — even though there are only four authors, and Frank Herbert was fetched
twice. With 100 books it would be 101 statements. That's the N+1 problem, and it's the subject
of the next two lessons. The console showed the same counts:

```text
[1] 3 SQL statements
[2] 1 SQL statements
[3] 1 SQL statements
[4] 9 SQL statements
```

## Parameters, not string building

Every query above passes values as `?` parameters. SQLite (like every database driver)
sends the SQL text and the values separately, so a value can never be interpreted as SQL:

```js title="like.mjs"
import { openDb, seed } from "./db.js";
const db = seed(openDb());
const escapeLike = (s) => s.replace(/[\\%_]/g, (c) => "\\" + c);
const search = (term) =>
  db.prepare("SELECT title FROM books WHERE title LIKE ? ESCAPE '\\' ORDER BY id").all(`%${escapeLike(term)}%`).map((r) => r.title);
console.log("dune ->", search("dune"));
console.log("%    ->", search("%"));
console.log("_    ->", search("_"));
console.log("'; DROP TABLE books; -- ->", search("'; DROP TABLE books; --"), "| books still there:", db.prepare("SELECT count(*) n FROM books").get().n);
```

```console
$ node like.mjs
dune -> [ 'Dune', 'Dune Messiah' ]
%    -> []
_    -> []
'; DROP TABLE books; -- -> [] | books still there: 8
```

That script also fixes a subtler bug that parameters alone don't. The server's original
`books(search:)` resolver was asked for `"%"`:

```console
$ curl -s localhost:4000/ -H 'content-type: application/json' -d '{"query":"{ books(search: \"%\") { title } }"}'
{"data":{"books":[{"title":"Dune"},{"title":"Dune Messiah"},{"title":"Emma"},{"title":"Persuasion"},{"title":"Neuromancer"},{"title":"Count Zero"},{"title":"The Dispossessed"},{"title":"The Left Hand of Darkness"}]}}
```

Every book. `%` and `_` are wildcards *inside* a `LIKE` pattern, and a parameter only protects
the SQL syntax, not the pattern language. On a large table, a client sending `%` or
`_%_%_%_%` can turn a cheap search into a full scan. `escapeLike` plus `ESCAPE '\'` makes the
user's text literal.

## Database errors are not API errors

Two bad mutations:

```console
$ curl -s localhost:4000/ -H 'content-type: application/json' \
    -d '{"query":"mutation { addReview(bookId: 5, stars: 9, body: \"!!\") { id } }"}'
{"errors":[{"message":"CHECK constraint failed: stars BETWEEN 1 AND 5","locations":[{"line":1,"column":12}],"path":["addReview"],"extensions":{"code":"INTERNAL_SERVER_ERROR","stacktrace":[…]}}],"data":{"addReview":null}}

$ curl -s localhost:4000/ -H 'content-type: application/json' \
    -d '{"query":"mutation { addReview(bookId: 99, stars: 3, body: \"?\") { id } }"}'
{"errors":[{"message":"FOREIGN KEY constraint failed","locations":[{"line":1,"column":12}],"path":["addReview"],"extensions":{"code":"INTERNAL_SERVER_ERROR","stacktrace":[…]}}],"data":{"addReview":null}}
```

(Stack traces trimmed; they included file paths from the server.) The constraints did their
job — no bad rows were written — but the client got raw database messages and an
`INTERNAL_SERVER_ERROR` code for what are really input mistakes. Keep the constraints as the
last line of defence, and validate first in the resolver so clients get useful errors:

```js
addReview: (_, { bookId, stars, body }, { sql }) => {
  if (!Number.isInteger(stars) || stars < 1 || stars > 5)
    throw new GraphQLError("stars must be between 1 and 5", { extensions: { code: "BAD_USER_INPUT" } });
  if (!sql.get("SELECT 1 FROM books WHERE id = ?", bookId))
    throw new GraphQLError(`No book ${bookId}`, { extensions: { code: "NOT_FOUND" } });
  // ...insert as before
},
```

[Lesson 08](08-mutation-design.md) improves on this further by returning such failures as
typed data, and [Level 4 · 06](../level-4/06-production-errors.md) masks unexpected errors so
messages like `FOREIGN KEY constraint failed` never reach clients.

A successful mutation, for comparison — 4 statements (insert, re-read the review, its book,
the book's reviews):

```console
$ curl -s localhost:4000/ -H 'content-type: application/json' \
    -d '{"query":"mutation { addReview(bookId: 5, stars: 5, body: \"Still holds up\") { id stars book { title reviews { stars } } } }"}'
{"data":{"addReview":{"id":"6","stars":5,"book":{"title":"Neuromancer","reviews":[{"stars":4},{"stars":5}]}}}}
```

## Rows as parents

The resolvers return raw rows (`{ id, title, year, author_id }`) and the schema exposes
`author` instead of `author_id`. `Book.author` reads `b.author_id` from its parent. That's
the [resolver chaining](../level-1/05-resolvers.md) from Level 1 with a database underneath,
and it's why the column naming didn't leak into the API. Also note the `id` column is an
integer but `ID` serialises it as the string `"6"`.

## How It Actually Works

`node:sqlite`'s `DatabaseSync` runs each statement on the calling thread and blocks until
SQLite returns. For an in-memory or local-file database with small queries, that's typically
faster than a round trip through a thread pool; for slow queries it would block Node's event
loop and stall every other request — the reason networked database drivers are asynchronous.
`db.prepare(sql)` compiles the SQL once into a statement object; `.all()`, `.get()` and
`.run()` bind the parameters and step through results. Bound parameters are never parsed as
SQL — that's the property that defeats injection.

Apollo Server calls your `context` function after parsing the HTTP request but before parsing
the GraphQL document, and the same object is passed as the third argument to every resolver
and to every plugin hook for that request (`requestContext.contextValue`). Because a request's
resolvers can be interleaved with other requests' resolvers whenever they `await`, anything
mutable must hang off that object rather than module scope.

## Common mistakes

- **One shared mutable context object** for all requests — user identity and caches bleed
  across requests.
- **Opening a connection per request** instead of using a process-level pool.
- **String-interpolating values into SQL.** Always parameters.
- **Forgetting that `LIKE` patterns have their own wildcards.**
- **Letting constraint errors be the validation layer**, so clients see internal messages and
  the wrong error code.
- **Forgetting `PRAGMA foreign_keys = ON`** in SQLite — foreign keys are not enforced without it.

## Exercise

1. Apply the `escapeLike` fix and the `addReview` validation to `server04.mjs`, and confirm the
   `"%"` search and both bad mutations now behave sensibly.
2. Add `Author.books` and query `{ books { author { books { title } } } }`. Count the
   statements and write down the formula for N books.
3. Wrap `addReview` in `BEGIN`/`COMMIT`/`ROLLBACK` and make it also update a
   `books.review_count` column. Make the second statement fail on purpose and prove the
   first was rolled back.
4. Log the slowest statement per request (use `performance.now()` around each call in
   `withQueryLog`).
