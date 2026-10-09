---
description: "Level 1 project: build a library loans API with Apollo Server — schema, resolvers, business rules as GraphQL errors, and an automated test suite."
---

# 10 · Project — A Library Catalog API with Tests

Time to put Level 1 together. You'll build a small but complete API for a lending library:
books, members and loans, with real rules (a book can only be lent once at a time, members
may hold at most two books) and a test suite that exercises it through GraphQL itself. It's
deliberately in-memory — a database arrives in [Level 2](../level-2/index.md) — so all the
attention goes to the schema, resolvers and tests.

## What you're building

- Queries: list books (optionally only available ones), look up a book or member.
- A member's loans, active or historical.
- Mutations: `checkOut(bookId, memberId)` and `returnBook(loanId)`.
- Rule violations reported as GraphQL errors with stable `extensions.code` values.
- Seven tests using `node:test`, run without opening a network port.

Project layout:

```text
library/
├── package.json        {"type": "module"}, deps: @apollo/server, graphql
├── schema.graphql
├── data.js             seed data + constants
├── resolvers.js
├── server.js           builds the ApolloServer (no listening)
├── main.js             starts HTTP
└── library.test.js
```

Splitting `server.js` (build the server) from `main.js` (listen on a port) is what makes
the tests easy: tests import `createServer()` and never touch HTTP.

## Step 1 — the schema

Design the schema first, from what a client screen needs: a catalog page with availability,
a member page with their loans, and two actions.

```graphql title="schema.graphql"
type Query {
  books(available: Boolean): [Book!]!
  book(id: ID!): Book
  member(id: ID!): Member
}

type Mutation {
  checkOut(bookId: ID!, memberId: ID!): Loan
  returnBook(loanId: ID!): Loan
}

type Book {
  id: ID!
  title: String!
  author: String!
  available: Boolean!
  currentLoan: Loan
}

type Member {
  id: ID!
  name: String!
  loans(active: Boolean = true): [Loan!]!
}

type Loan {
  id: ID!
  book: Book!
  member: Member!
  dueDate: String!
  returned: Boolean!
}
```

Decisions worth calling out:

- `Book.available` is a **computed** field, not stored. It's derived from loans, so it can't
  get out of sync.
- `Book.currentLoan` is nullable — most books aren't lent out.
- `Member.loans(active: Boolean = true)` defaults to what the UI wants most often but lets a
  client pass `false` for history or `null` for everything (absent ≠ null, from
  [lesson 06](06-mutations-inputs.md)).
- Both mutations return a nullable `Loan`, so a failure nulls only that field.

## Step 2 — data

```js title="data.js"
export function createStore() {
  return {
    books: [
      { id: "b1", title: "Dune", author: "Frank Herbert" },
      { id: "b2", title: "Emma", author: "Jane Austen" },
      { id: "b3", title: "Neuromancer", author: "William Gibson" },
    ],
    members: [
      { id: "m1", name: "Ada" },
      { id: "m2", name: "Grace" },
    ],
    loans: [],
    nextLoanId: 1,
  };
}

export const MAX_ACTIVE_LOANS = 2;
export const LOAN_DAYS = 14;
```

`createStore()` is a factory rather than a module-level object so each test can start from a
fresh copy.

## Step 3 — resolvers

```js title="resolvers.js"
import { GraphQLError } from "graphql";
import { MAX_ACTIVE_LOANS, LOAN_DAYS } from "./data.js";

const activeLoanFor = (store, bookId) =>
  store.loans.find((l) => l.bookId === bookId && !l.returned);

function fail(message, code) {
  throw new GraphQLError(message, { extensions: { code } });
}

export const resolvers = {
  Query: {
    books: (_, { available }, { store }) =>
      available == null
        ? store.books
        : store.books.filter((b) => !activeLoanFor(store, b.id) === available),
    book: (_, { id }, { store }) => store.books.find((b) => b.id === id) ?? null,
    member: (_, { id }, { store }) => store.members.find((m) => m.id === id) ?? null,
  },
  Mutation: {
    checkOut: (_, { bookId, memberId }, { store, now }) => {
      const book = store.books.find((b) => b.id === bookId);
      if (!book) fail(`Book ${bookId} not found`, "NOT_FOUND");
      const member = store.members.find((m) => m.id === memberId);
      if (!member) fail(`Member ${memberId} not found`, "NOT_FOUND");
      if (activeLoanFor(store, bookId)) fail(`${book.title} is already checked out`, "UNAVAILABLE");
      const active = store.loans.filter((l) => l.memberId === memberId && !l.returned);
      if (active.length >= MAX_ACTIVE_LOANS)
        fail(`${member.name} already has ${MAX_ACTIVE_LOANS} books`, "LOAN_LIMIT");
      const due = new Date(now());
      due.setUTCDate(due.getUTCDate() + LOAN_DAYS);
      const loan = {
        id: `l${store.nextLoanId++}`,
        bookId,
        memberId,
        dueDate: due.toISOString().slice(0, 10),
        returned: false,
      };
      store.loans.push(loan);
      return loan;
    },
    returnBook: (_, { loanId }, { store }) => {
      const loan = store.loans.find((l) => l.id === loanId);
      if (!loan) return null;
      if (loan.returned) fail(`Loan ${loanId} was already returned`, "ALREADY_RETURNED");
      loan.returned = true;
      return loan;
    },
  },
  Book: {
    available: (book, _, { store }) => !activeLoanFor(store, book.id),
    currentLoan: (book, _, { store }) => activeLoanFor(store, book.id) ?? null,
  },
  Member: {
    loans: (member, { active }, { store }) =>
      store.loans.filter((l) => l.memberId === member.id && (active == null || l.returned === !active)),
  },
  Loan: {
    book: (loan, _, { store }) => store.books.find((b) => b.id === loan.bookId),
    member: (loan, _, { store }) => store.members.find((m) => m.id === loan.memberId),
  },
};
```

Things to notice:

- **Every resolver reads the store from `context`.** Nothing is imported as global state,
  so tests can inject their own store.
- **The clock is in context too** (`now`). Due dates depend on the current time; injecting
  it makes the due-date test deterministic.
- `Loan` rows store `bookId`/`memberId`; the `Loan.book` and `Loan.member` resolvers turn
  them into objects only when a client asks. That's the parent-chaining from
  [lesson 05](05-resolvers.md).
- Business-rule failures throw `GraphQLError` with a `code`. "Return a loan that doesn't
  exist" returns `null` with no error instead — a deliberate contrast: that one has a
  natural "nothing" answer.

## Step 4 — the server and entry point

```js title="server.js"
import { readFileSync } from "node:fs";
import { ApolloServer } from "@apollo/server";
import { resolvers } from "./resolvers.js";

export const typeDefs = readFileSync(new URL("./schema.graphql", import.meta.url), "utf8");

export function createServer() {
  return new ApolloServer({ typeDefs, resolvers });
}
```

```js title="main.js"
import { startStandaloneServer } from "@apollo/server/standalone";
import { createServer } from "./server.js";
import { createStore } from "./data.js";

const store = createStore();
const { url } = await startStandaloneServer(createServer(), {
  listen: { port: Number(process.env.PORT ?? 4000) },
  context: async () => ({ store, now: () => Date.now() }),
});
console.log(`Library API at ${url}`);
```

`readFileSync` with `new URL(…, import.meta.url)` loads the schema relative to the module
file, so it works no matter which directory you run `node` from.

Smoke test it:

```console
$ node main.js
Library API at http://localhost:4000/
$ curl -s localhost:4000/ -H 'content-type: application/json' \
    -d '{"query":"mutation { checkOut(bookId: \"b3\", memberId: \"m2\") { id dueDate book { title } } }"}'
{"data":{"checkOut":{"id":"l1","dueDate":"2026-10-23","book":{"title":"Neuromancer"}}}}
$ curl -s localhost:4000/ -H 'content-type: application/json' \
    -d '{"query":"{ books { title available currentLoan { member { name } } } }"}'
{"data":{"books":[{"title":"Dune","available":true,"currentLoan":null},{"title":"Emma","available":true,"currentLoan":null},{"title":"Neuromancer","available":false,"currentLoan":{"member":{"name":"Grace"}}}]}}
```

(The due date is 14 days after the day it was run — 9 October 2026 here. Yours will differ.)

## Step 5 — tests

Apollo Server's `executeOperation` runs a request through the full pipeline — parsing,
validation, your context value, execution, error formatting — without HTTP. That's the right
level for most API tests: it checks the schema and resolvers together, the way clients
use them.

```js title="library.test.js"
import { test, beforeEach } from "node:test";
import assert from "node:assert/strict";
import { createServer } from "./server.js";
import { createStore } from "./data.js";

const server = createServer();
let store;
const FIXED_NOW = Date.UTC(2026, 0, 10);

beforeEach(() => { store = createStore(); });

async function gql(query, variables) {
  const res = await server.executeOperation(
    { query, variables },
    { contextValue: { store, now: () => FIXED_NOW } },
  );
  assert.equal(res.body.kind, "single");
  // graphql-js builds result objects with a null prototype; a JSON round trip
  // turns them into plain objects so deepStrictEqual compares values only.
  return JSON.parse(JSON.stringify(res.body.singleResult));
}

const CHECK_OUT = `mutation($b: ID!, $m: ID!) { checkOut(bookId: $b, memberId: $m) { id dueDate book { title available } member { name } } }`;

test("all books are available initially", async () => {
  const { data, errors } = await gql(`{ books(available: true) { id } }`);
  assert.equal(errors, undefined);
  assert.deepEqual(data.books.map((b) => b.id), ["b1", "b2", "b3"]);
});

test("checking out a book sets a due date 14 days out and marks it unavailable", async () => {
  const { data, errors } = await gql(CHECK_OUT, { b: "b1", m: "m1" });
  assert.equal(errors, undefined);
  assert.deepEqual(data.checkOut, {
    id: "l1",
    dueDate: "2026-01-24",
    book: { title: "Dune", available: false },
    member: { name: "Ada" },
  });
  const after = await gql(`{ books(available: false) { title currentLoan { member { name } } } }`);
  assert.deepEqual(after.data.books, [{ title: "Dune", currentLoan: { member: { name: "Ada" } } }]);
});

test("a book can't be checked out twice", async () => {
  await gql(CHECK_OUT, { b: "b1", m: "m1" });
  const { data, errors } = await gql(CHECK_OUT, { b: "b1", m: "m2" });
  assert.equal(data.checkOut, null);
  assert.equal(errors[0].extensions.code, "UNAVAILABLE");
});

test("members are limited to two active loans", async () => {
  await gql(CHECK_OUT, { b: "b1", m: "m1" });
  await gql(CHECK_OUT, { b: "b2", m: "m1" });
  const { errors } = await gql(CHECK_OUT, { b: "b3", m: "m1" });
  assert.equal(errors[0].extensions.code, "LOAN_LIMIT");
  assert.match(errors[0].message, /Ada already has 2 books/);
});

test("returning frees the book and moves the loan to history", async () => {
  await gql(CHECK_OUT, { b: "b2", m: "m2" });
  const ret = await gql(`mutation { returnBook(loanId: "l1") { returned book { available } } }`);
  assert.deepEqual(ret.data.returnBook, { returned: true, book: { available: true } });
  const member = await gql(`{ member(id: "m2") { active: loans { id } history: loans(active: false) { id } all: loans(active: null) { id } } }`);
  assert.deepEqual(member.data.member, { active: [], history: [{ id: "l1" }], all: [{ id: "l1" }] });
  const again = await gql(`mutation { returnBook(loanId: "l1") { id } }`);
  assert.equal(again.errors[0].extensions.code, "ALREADY_RETURNED");
});

test("unknown ids", async () => {
  const q = await gql(`{ book(id: "nope") { title } member(id: "nope") { name } }`);
  assert.deepEqual(q.data, { book: null, member: null });
  const m = await gql(CHECK_OUT, { b: "nope", m: "m1" });
  assert.equal(m.errors[0].extensions.code, "NOT_FOUND");
  const r = await gql(`mutation { returnBook(loanId: "nope") { id } }`);
  assert.deepEqual(r.data, { returnBook: null });
  assert.equal(r.errors, undefined);
});

test("invalid queries are rejected before resolvers run", async () => {
  const { data, errors } = await gql(`{ books { isbn } }`);
  assert.equal(data, undefined);
  assert.equal(errors[0].extensions.code, "GRAPHQL_VALIDATION_FAILED");
});
```

```console
$ node --test
✔ all books are available initially
✔ checking out a book sets a due date 14 days out and marks it unavailable
✔ a book can't be checked out twice
✔ members are limited to two active loans
✔ returning frees the book and moves the loan to history
✔ unknown ids
✔ invalid queries are rejected before resolvers run
ℹ tests 7
ℹ pass 7
ℹ fail 0
```

(Timings trimmed from the output.)

### The null-prototype surprise

The first version of this test file returned `res.body.singleResult` directly, and the
"unknown ids" test failed with a baffling diff:

```text
✖ unknown ids
  AssertionError [ERR_ASSERTION]: Expected values to be strictly deep-equal:
  + actual - expected

  + [Object: null prototype] {
  - {
      book: null,
      member: null
    }
```

The values are identical. The difference is that `graphql-js` builds result objects with
`Object.create(null)` — objects with no prototype, so a field named `constructor` or
`__proto__` can't collide with anything inherited. `assert.deepStrictEqual` compares
prototypes too, so it rejects them. The JSON round trip in `gql()` produces plain objects;
it's also a fair simulation of what a real client receives after HTTP serialisation.

## How It Actually Works

`executeOperation(request, { contextValue })` calls the same internal function the HTTP
integrations use, minus the HTTP-specific parts (no CSRF check, no body parsing). It returns
a `GraphQLResponse` whose `body` is either `{ kind: "single", singleResult }` or, for
incremental delivery with `@defer`, `{ kind: "incremental", … }` — hence the
`assert.equal(res.body.kind, "single")` guard. Because we pass `contextValue` directly, the
server's `context` function isn't involved; that's exactly what lets each test inject a fresh
store and a fixed clock.

The ordering of mutation fields matters in one test only implicitly: each `gql` call is its
own operation and awaited in sequence, so the store changes are visible to the next call.
Within a single request, serial execution of mutation fields would give the same guarantee.

## Review checklist

Before moving on, check your version against these:

- [ ] Every rule violation has a distinct `extensions.code` that a test asserts on.
- [ ] No resolver imports mutable state directly; it all comes from `context`.
- [ ] Computed fields (`available`, `currentLoan`) are derived, not stored.
- [ ] Tests are independent — run any one alone with `node --test --test-name-pattern "twice"`.
- [ ] The schema file is the single source of truth for the API surface.

## Common mistakes

- **Testing resolvers as plain functions only.** It misses schema mistakes (wrong
  nullability, a misspelled field that silently falls back to the default resolver).
  Test through `executeOperation`.
- **Sharing one store across tests**, so the order tests run in changes their results.
- **Asserting on error messages** instead of codes. Messages get reworded.
- **Using `Date.now()` inside resolvers** and then writing date tests that pass only on the
  day you wrote them.

## Exercise

1. Add `renew(loanId: ID!): Loan` that extends the due date by 14 days, at most once per
   loan. Add the field the client needs to show "renewed", and tests for both the success and
   the "already renewed" case.
2. Add `overdueLoans: [Loan!]!` to `Query`, using `now` from context. Test it with a fixed
   clock set after a due date.
3. Add a `search(text: String!): [Book!]!` query that matches title or author case-
   insensitively.
4. Break something on purpose — make `Loan.book` return `undefined` — and see which tests
   catch it and what error the client receives.
