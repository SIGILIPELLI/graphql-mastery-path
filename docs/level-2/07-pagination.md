---
description: "Paginate GraphQL lists properly: why offset pages skip and repeat, keyset cursors, Relay connections, page-size caps, cursor validation."
---

# 07 · Pagination: Offsets, Cursors & Connections

Every list field that can grow needs pagination, and it's far easier to design in from the
start than to retrofit — changing `posts: [Post!]!` into a paginated type later is a breaking
change. GraphQL doesn't mandate a pagination style, but the ecosystem has largely converged
on **cursor-based connections**. This lesson shows the bug that pushes everyone there, builds
a connection over SQLite with keyset queries, and checks the query plans.

## The setup

Seven posts, newest first, with an index matching the sort order:

```js title="pagination07.mjs"
import { makeExecutableSchema } from "@graphql-tools/schema";
import { graphql, GraphQLError } from "graphql";
import { DatabaseSync } from "node:sqlite";

const db = new DatabaseSync(":memory:");
db.exec(`CREATE TABLE posts (id INTEGER PRIMARY KEY, title TEXT NOT NULL, created_at TEXT NOT NULL);
         CREATE INDEX posts_created ON posts (created_at DESC, id DESC);`);
const ins = db.prepare("INSERT INTO posts (title, created_at) VALUES (?, ?)");
for (let i = 1; i <= 7; i++) ins.run(`Post ${i}`, `2026-01-0${i}T10:00:00Z`);

const encode = (row) => Buffer.from(JSON.stringify([row.created_at, row.id])).toString("base64url");
function decode(cursor) {
  try {
    const [createdAt, id] = JSON.parse(Buffer.from(cursor, "base64url").toString());
    if (typeof createdAt !== "string" || !Number.isInteger(id)) throw new Error();
    return { createdAt, id };
  } catch {
    throw new GraphQLError("Invalid cursor", { extensions: { code: "BAD_USER_INPUT" } });
  }
}

const MAX_PAGE = 50;

const schema = makeExecutableSchema({
  typeDefs: /* GraphQL */ `
    type Query {
      postsByOffset(offset: Int = 0, limit: Int = 3): [Post!]!
      posts(first: Int = 3, after: String): PostConnection!
    }
    type Post { id: ID! title: String! createdAt: String! }
    type PostConnection {
      edges: [PostEdge!]!
      pageInfo: PageInfo!
      totalCount: Int!
    }
    type PostEdge { cursor: String! node: Post! }
    type PageInfo { hasNextPage: Boolean! endCursor: String }
  `,
  resolvers: {
    Query: {
      postsByOffset: (_, { offset, limit }) =>
        db.prepare("SELECT * FROM posts ORDER BY created_at DESC, id DESC LIMIT ? OFFSET ?").all(limit, offset),
      posts: (_, { first, after }) => {
        if (first < 0 || first > MAX_PAGE)
          throw new GraphQLError(`first must be between 0 and ${MAX_PAGE}`, { extensions: { code: "BAD_USER_INPUT" } });
        let rows;
        if (after) {
          const c = decode(after);
          rows = db.prepare(`SELECT * FROM posts WHERE (created_at, id) < (?, ?)
                             ORDER BY created_at DESC, id DESC LIMIT ?`).all(c.createdAt, c.id, first + 1);
        } else {
          rows = db.prepare("SELECT * FROM posts ORDER BY created_at DESC, id DESC LIMIT ?").all(first + 1);
        }
        const hasNextPage = rows.length > first;
        const page = rows.slice(0, first);
        return {
          edges: page.map((row) => ({ cursor: encode(row), node: row })),
          pageInfo: { hasNextPage, endCursor: page.length ? encode(page.at(-1)) : null },
        };
      },
    },
    PostConnection: {
      totalCount: () => db.prepare("SELECT count(*) AS n FROM posts").get().n,
    },
    Post: { createdAt: (p) => p.created_at },
  },
});

const gql = async (source, variableValues) => {
  const r = await graphql({ schema, source, variableValues });
  if (r.errors) return { errors: r.errors.map((e) => e.message) };
  return r.data;
};
const titles = (list) => list.map((p) => p.title).join(", ");
const newPost = (n) => ins.run(`Post ${n}`, `2026-01-${String(n).padStart(2, "0")}T10:00:00Z`);

console.log("== offset pagination while a post is published between pages");
let p1 = await gql(`{ postsByOffset(offset: 0) { title } }`);
console.log("page 1:", titles(p1.postsByOffset));
newPost(8);
let p2 = await gql(`{ postsByOffset(offset: 3) { title } }`);
console.log("page 2:", titles(p2.postsByOffset), " <- Post 5 shown twice");

console.log("\n== cursor pagination with the same interruption");
const Q = `query($after: String) { posts(first: 3, after: $after) {
  edges { cursor node { title } } pageInfo { hasNextPage endCursor } } }`;
let c1 = await gql(Q, {});
console.log("page 1:", titles(c1.posts.edges.map((e) => e.node)), "| hasNextPage:", c1.posts.pageInfo.hasNextPage);
newPost(9);
let c2 = await gql(Q, { after: c1.posts.pageInfo.endCursor });
console.log("page 2:", titles(c2.posts.edges.map((e) => e.node)), "| hasNextPage:", c2.posts.pageInfo.hasNextPage);
let c3 = await gql(Q, { after: c2.posts.pageInfo.endCursor });
console.log("page 3:", titles(c3.posts.edges.map((e) => e.node)), "| hasNextPage:", c3.posts.pageInfo.hasNextPage, "| endCursor:", c3.posts.pageInfo.endCursor);
console.log("cursor example:", c1.posts.pageInfo.endCursor, "=", Buffer.from(c1.posts.pageInfo.endCursor, "base64url").toString());

console.log("\n== bad input");
console.log(await gql(`{ posts(first: 1000) { totalCount } }`));
console.log(await gql(`{ posts(after: "not-a-cursor") { totalCount } }`));

console.log("\n== query plans");
for (const sql of [
  "SELECT * FROM posts ORDER BY created_at DESC, id DESC LIMIT 3 OFFSET 100000",
  "SELECT * FROM posts WHERE (created_at, id) < ('2026-01-05', 5) ORDER BY created_at DESC, id DESC LIMIT 4",
]) console.log(db.prepare("EXPLAIN QUERY PLAN " + sql).all().map((r) => r.detail).join(" | "));
```

The schema offers two styles side by side: `postsByOffset(offset, limit)` returning a plain
list, and `posts(first, after)` returning a `PostConnection`.

## The offset bug

```text
== offset pagination while a post is published between pages
page 1: Post 7, Post 6, Post 5
page 2: Post 5, Post 4, Post 3  <- Post 5 shown twice
```

The client read page 1 (offset 0), then someone published Post 8, then the client asked for
offset 3. Post 8 pushed everything down by one, so Post 5 — the last item of page 1 — is now
at position 3 and appears again. Deletions cause the opposite problem: an item slides up
past the boundary and is never shown. On a feed with steady writes, infinite scroll built on
offsets visibly repeats and skips items.

There's a performance problem too, visible in SQLite's plan for a deep page:

```text
SCAN posts USING INDEX posts_created           -- ... LIMIT 3 OFFSET 100000
```

An offset can't jump: the database walks past all 100,000 skipped rows to return 3. Page 1
is cheap; page 5,000 is not.

## Cursors: "after this item", not "after N items"

A cursor identifies a *position* in the ordering by the values of the sort key. The next page
is "rows that sort after this one", which is a `WHERE` clause the index can seek to directly:

```sql
SELECT * FROM posts WHERE (created_at, id) < (?, ?)
ORDER BY created_at DESC, id DESC LIMIT ?
```

```text
SEARCH posts USING INDEX posts_created (created_at<?)
```

`SEARCH` instead of `SCAN`: SQLite seeks straight to the position, so page 5,000 costs the
same as page 1. This technique is called **keyset** (or seek) pagination. Two details matter:

- **The sort key must be unique.** Two posts can share a `created_at`, so `id` is added as a
  tie-breaker and included in the cursor. Without it, rows with equal timestamps at a page
  boundary could be skipped or repeated.
- **The row-value comparison** `(created_at, id) < (?, ?)` means "created earlier, or same
  time with a smaller id" — exactly the `ORDER BY … DESC` order. SQLite, PostgreSQL and MySQL
  support row values; on databases that don't, expand it to
  `created_at < ? OR (created_at = ? AND id < ?)`.

Now the same interruption, with cursors:

```text
== cursor pagination with the same interruption
page 1: Post 8, Post 7, Post 6 | hasNextPage: true
page 2: Post 5, Post 4, Post 3 | hasNextPage: true
page 3: Post 2, Post 1 | hasNextPage: false | endCursor: WyIyMDI2LTAxLTAxVDEwOjAwOjAwWiIsMV0
cursor example: WyIyMDI2LTAxLTA2VDEwOjAwOjAwWiIsNl0 = ["2026-01-06T10:00:00Z",6]
```

Post 9 was published after page 1 was read. Page 2 continued exactly after Post 6 — no
repeats, no gaps. Post 9 belongs at the top, where a "pull to refresh" would find it.

## The connection shape

The `PostConnection` type follows the **Relay Cursor Connections** convention, which most
GraphQL clients and tools understand:

```graphql
type PostConnection {
  edges: [PostEdge!]!
  pageInfo: PageInfo!
  totalCount: Int!
}
type PostEdge { cursor: String! node: Post! }
type PageInfo { hasNextPage: Boolean! endCursor: String }
```

- **`edges`** wrap each item with its own cursor. Edges are also where relationship data goes
  — for a `friends` connection, `edge.since` describes the friendship, not the friend.
- **`pageInfo`** tells the client whether to show "load more" and what to pass as `after`.
  The full convention also has `hasPreviousPage` and `startCursor`, and `last`/`before`
  arguments for paging backwards.
- **`totalCount`** is a common addition, not part of the core convention.

Many schemas also add a `nodes: [Post!]!` shortcut for clients that don't need per-edge
cursors.

### How `hasNextPage` is computed

The resolver asks for `first + 1` rows. If it gets more than `first`, there's another page; it
returns only the first `first`. That costs one extra row instead of a second `COUNT` query.

### `totalCount` costs extra

`totalCount` is a separate field resolver on `PostConnection`, so the `COUNT(*)` only runs when
a client selects it. That matters: counting a large filtered table can be far more expensive
than fetching one page. Some APIs omit it or return an estimate for that reason.

## Validating input

```text
== bad input
{ errors: [ 'first must be between 0 and 50' ] }
{ errors: [ 'Invalid cursor' ] }
```

- **Cap the page size.** Without `MAX_PAGE`, `first: 1000000` is a denial-of-service request.
  Combine it with query-cost limits in [Level 3 · 05](../level-3/05-security-limits.md).
- **Treat cursors as untrusted input.** They come back from the client, so decode defensively
  and check their contents' types — here a malformed cursor becomes a clean `BAD_USER_INPUT`
  rather than a JSON exception.
- **Keep cursors opaque.** Base64 tells clients "don't parse this", which keeps you free to
  change what's inside. It's not security — anyone can decode it, as the example output shows.
  Never put anything secret in a cursor, and if tampering matters, sign it (an HMAC) so the
  server can detect edits.

## Offset still has its place

Admin tables with "page 7 of 23" navigation, small bounded lists, and search results where
users jump to arbitrary pages are reasonable fits for offsets. A schema can offer both. Just
don't use offsets for feeds and infinite scroll.

## How It Actually Works

SQLite stores an index as a B-tree ordered by `(created_at DESC, id DESC)`. `LIMIT 3 OFFSET k`
has to step through `k` index entries one by one, because a B-tree has no "skip to the k-th
entry" operation — that's the `SCAN` in the plan. A keyset predicate gives the planner a key
to search for: it descends the tree in O(log n) to the first entry after the cursor and reads
the next `first + 1` entries — the `SEARCH`. PostgreSQL and MySQL behave the same way with an
index matching the `ORDER BY`. Without such an index, both styles degrade to sorting the whole
table, so the index is part of the design, not an optimisation.

On the GraphQL side, nothing special happens: `PostConnection` is an ordinary object type, and
the resolver returns a plain object `{ edges, pageInfo }` whose missing `totalCount` property
is filled in by its own resolver only if selected.

## Common mistakes

- **Returning unbounded lists** "for now". Adding pagination later breaks every client.
- **Cursors over non-unique sort keys** — missing tie-breakers cause skipped rows.
- **Using the row id as the cursor while sorting by something else.** The cursor must encode
  the sort key values.
- **No upper bound on `first`.**
- **Always computing `totalCount`** in the list resolver, even when nobody asked.
- **Paginating in memory** — fetching all rows and slicing in JavaScript gives the right API
  with none of the benefits.

## Exercise

1. Add `last: Int, before: String` for backwards pagination, plus `hasPreviousPage` and
   `startCursor`. (Hint: flip the comparison and the sort, then reverse the page.)
2. Add a `Post.author` and make `posts` filterable by author. What does the index need to
   look like now?
3. Sign cursors with an HMAC using `node:crypto` and reject tampered cursors.
4. Convert the `reviews` list on `Book` from lesson 06 into a connection. How does that
   change the DataLoader key?
