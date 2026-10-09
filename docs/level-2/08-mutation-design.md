---
description: "Design GraphQL mutations clients can rely on: payload types, userErrors versus result unions, constraint failures, evolving outcomes."
---

# 08 · Mutation Design: Payloads & Errors as Data

[Level 1 · 06](../level-1/06-mutations-inputs.md) introduced mutations; this lesson is about
designing them so they age well. The central question: when a user does something that
can't succeed — an email already registered, a handle with spaces in it — is that an
*error* or a *result*? GraphQL's `errors` array is built for things that went wrong with the
request or the server. A user typing an invalid email isn't that; it's an expected outcome a
UI must render next to a form field. Treating such outcomes as **data in the schema** is now
the mainstream approach, and there are two popular ways to do it. This lesson implements both
side by side for a registration mutation.

## The shared parts

A `users` table with two unique columns, a validation function, and an insert that turns a
uniqueness violation into a value instead of an exception:

```js title="mutation08.mjs"
import { makeExecutableSchema } from "@graphql-tools/schema";
import { graphql } from "graphql";
import { DatabaseSync } from "node:sqlite";

const db = new DatabaseSync(":memory:");
db.exec(`CREATE TABLE users (id INTEGER PRIMARY KEY, email TEXT NOT NULL UNIQUE, handle TEXT NOT NULL UNIQUE);
         INSERT INTO users (email, handle) VALUES ('ada@example.com', 'ada');`);

function validate({ email, handle }) {
  const problems = [];
  if (!/^[^@\s]+@[^@\s]+\.[^@\s]+$/.test(email)) problems.push({ field: ["email"], message: "Enter a valid email address." });
  if (!/^[a-z0-9_]{3,20}$/.test(handle)) problems.push({ field: ["handle"], message: "Handles are 3–20 lowercase letters, digits or underscores." });
  return problems;
}

function insertUser(input) {
  try {
    const { lastInsertRowid } = db.prepare("INSERT INTO users (email, handle) VALUES (?, ?)").run(input.email, input.handle);
    return { user: db.prepare("SELECT * FROM users WHERE id = ?").get(lastInsertRowid) };
  } catch (e) {
    // node:sqlite reports constraint failures with errcode 2067 (SQLITE_CONSTRAINT_UNIQUE)
    if (e.errcode === 2067) return { taken: e.message.includes("users.email") ? "email" : "handle" };
    throw e; // anything else is a genuine server error
  }
}

const typeDefs = /* GraphQL */ `
  type User { id: ID! email: String! handle: String! }
  input RegisterInput { email: String!, handle: String! }

  # Style A: payload with a userErrors list
  type RegisterPayload { user: User, userErrors: [UserError!]! }
  type UserError { field: [String!], message: String!, code: UserErrorCode! }
  enum UserErrorCode { INVALID TAKEN }

  # Style B: a union of outcomes
  union RegisterResult = RegisterSuccess | InputInvalid | AlreadyTaken
  type RegisterSuccess { user: User! }
  type InputInvalid { problems: [FieldProblem!]! }
  type FieldProblem { field: [String!]!, message: String! }
  type AlreadyTaken { field: String!, message: String! }

  type Query { userCount: Int! }
  type Mutation {
    registerA(input: RegisterInput!): RegisterPayload!
    registerB(input: RegisterInput!): RegisterResult!
  }
`;

const resolvers = {
  Query: { userCount: () => db.prepare("SELECT count(*) AS n FROM users").get().n },
  Mutation: {
    registerA: (_, { input }) => {
      const problems = validate(input);
      if (problems.length) return { user: null, userErrors: problems.map((p) => ({ ...p, code: "INVALID" })) };
      const r = insertUser(input);
      if (r.taken) return { user: null, userErrors: [{ field: [r.taken], message: `That ${r.taken} is already registered.`, code: "TAKEN" }] };
      return { user: r.user, userErrors: [] };
    },
    registerB: (_, { input }) => {
      const problems = validate(input);
      if (problems.length) return { __typename: "InputInvalid", problems };
      const r = insertUser(input);
      if (r.taken) return { __typename: "AlreadyTaken", field: r.taken, message: `That ${r.taken} is already registered.` };
      return { __typename: "RegisterSuccess", user: r.user };
    },
  },
  // __typename on each returned object lets the default type resolver pick the member
};

const schema = makeExecutableSchema({ typeDefs, resolvers });
const run = async (label, source, variableValues) =>
  console.log(`--- ${label}\n${JSON.stringify((await graphql({ schema, source, variableValues })))}`);

const A = `mutation($i: RegisterInput!) { registerA(input: $i) { user { id handle } userErrors { field message code } } }`;
const B = `mutation($i: RegisterInput!) { registerB(input: $i) {
  __typename
  ... on RegisterSuccess { user { id handle } }
  ... on InputInvalid { problems { field message } }
  ... on AlreadyTaken { field message }
} }`;

await run("A ok", A, { i: { email: "grace@example.com", handle: "grace" } });
await run("A invalid", A, { i: { email: "nope", handle: "X" } });
await run("A taken", A, { i: { email: "ada@example.com", handle: "ada2" } });
await run("B ok", B, { i: { email: "linus@example.com", handle: "linus" } });
await run("B invalid", B, { i: { email: "nope", handle: "ok_handle" } });
await run("B taken", B, { i: { email: "new@example.com", handle: "ada" } });
await run("B old client that only knows RegisterSuccess", `mutation($i: RegisterInput!) { registerB(input: $i) { ... on RegisterSuccess { user { id } } } }`,
  { i: { email: "x@example.com", handle: "grace" } });
await run("count", `{ userCount }`);
```

Notice `insertUser` checks the error code, not just "some error happened". `node:sqlite`
exposes SQLite's extended result code as `errcode`; 2067 is `SQLITE_CONSTRAINT_UNIQUE`. Any
*other* failure is re-thrown, because a disk error or a bug is not something the user can fix
and should be a real GraphQL error (and logged).

Relying on the unique constraint rather than "check first, then insert" also avoids a race:
two requests registering the same handle at the same instant would both pass a `SELECT`
check, but only one can win the `INSERT`.

## Style A: payload with `userErrors`

```graphql
type RegisterPayload { user: User, userErrors: [UserError!]! }
type UserError { field: [String!], message: String!, code: UserErrorCode! }
```

Every mutation returns its own `XxxPayload` type containing the changed object (nullable) and
a list of user errors. This style is used by large public APIs (Shopify's Admin API is the
well-known example).

```text
--- A ok
{"data":{"registerA":{"user":{"id":"2","handle":"grace"},"userErrors":[]}}}
--- A invalid
{"data":{"registerA":{"user":null,"userErrors":[{"field":["email"],"message":"Enter a valid email address.","code":"INVALID"},{"field":["handle"],"message":"Handles are 3–20 lowercase letters, digits or underscores.","code":"INVALID"}]}}}
--- A taken
{"data":{"registerA":{"user":null,"userErrors":[{"field":["email"],"message":"That email is already registered.","code":"TAKEN"}]}}}
```

Strengths: one uniform shape for every mutation; several problems reported at once (both
fields were invalid above); `field` is a path (`["input", "address", "zip"]` for nested
inputs) a form library can map to inputs. Weakness: nothing in the type system says *which*
codes a given mutation can produce, or that `user` is non-null exactly when `userErrors` is
empty — clients must trust the convention.

## Style B: a union of outcomes

```graphql
union RegisterResult = RegisterSuccess | InputInvalid | AlreadyTaken
type RegisterSuccess { user: User! }
type InputInvalid { problems: [FieldProblem!]! }
type AlreadyTaken { field: String!, message: String! }
```

Each outcome is its own type with exactly the fields that make sense for it.

```text
--- B ok
{"data":{"registerB":{"__typename":"RegisterSuccess","user":{"id":"3","handle":"linus"}}}}
--- B invalid
{"data":{"registerB":{"__typename":"InputInvalid","problems":[{"field":["email"],"message":"Enter a valid email address."}]}}}
--- B taken
{"data":{"registerB":{"__typename":"AlreadyTaken","field":"handle","message":"That handle is already registered."}}}
```

Strengths: the schema documents every possible outcome; `user` is `User!` on success, so no
null check; typed clients get an exhaustive `switch (result.__typename)`. The resolvers return
a `__typename` property on each object, which the default type resolver uses (as seen in
[lesson 02](02-interfaces-unions.md)) — no `__resolveType` needed.

Weakness: evolution. Here's a client written before `AlreadyTaken` existed, which only selects
the success case, hitting it:

```text
--- B old client that only knows RegisterSuccess
{"data":{"registerB":{}}}
```

An empty object. No error, no indication of what happened. Old clients can't know about
outcome types added later, so they must treat "a `__typename` I don't recognise" — or, as
here, no fields at all — as a generic failure. A common mitigation is to have all error
outcomes implement a shared interface, so even old clients can select `... on Error { message }`:

```graphql
interface Error { message: String! }
type AlreadyTaken implements Error { message: String!, field: String! }
```

## Which to choose

| | `userErrors` payload | Result union |
|---|---|---|
| Possible outcomes visible in schema | no — by convention | yes |
| Multiple problems at once | natural | needs a list inside one outcome type |
| Null checks on success | `user` is nullable | `user` is non-null |
| Adding a new failure kind | new enum value (dangerous change) | new union member (dangerous change) |
| Client code | check list length | switch on `__typename` |

Both are far better than throwing. Pick one per API and use it consistently; mixing styles
across mutations is what really hurts client developers.

## Other payload conventions

- **Always return a payload object, never the bare entity.** `registerA: RegisterPayload!`
  can grow new fields (a `warnings` list, a `viewer` for cache refreshes) without breaking
  anyone. `register: User` can't.
- **One input argument** (`input: RegisterInput!`) keeps mutation signatures uniform and lets
  clients send a single variable.
- **Return what the client needs to update its cache**: the changed object with its `id`,
  and for deletions the deleted `id` (`deletedPostId: ID`), because the object no longer
  exists to be queried.
- **Name for intent.** `publishPost` and `archivePost` beat a generic `updatePost(status:)`,
  because each can have its own rules, permissions and outcome types.

## When an error *is* right

Keep the `errors` array for things the user can't fix by changing their input: not
authenticated, not authorized (arguably — some APIs model this as data too), rate limited,
the database is down, a bug. Those are typically handled generically by the client's network
layer rather than by a specific form.

## How It Actually Works

Both styles are just ordinary output types; nothing in `graphql-js` treats them specially. In
style B, `completeAbstractValue` calls the union's type resolver; since we didn't define
`__resolveType`, `graphql-tools` leaves the default, which reads `value.__typename` from the
returned object. The empty `{}` for the old client happens during field collection: the
runtime type `AlreadyTaken` doesn't match the fragment's type condition `RegisterSuccess`, so
the collected field set is empty and the result object has no keys.

In `insertUser`, `node:sqlite` throws an `Error` whose `errcode` is SQLite's extended result
code. The base code for constraint failures is 19 (`SQLITE_CONSTRAINT`); extended codes refine
it as `19 | (n << 8)`, and `n = 8` gives 2067 for a `UNIQUE` violation (versus 787 for a
foreign key). PostgreSQL drivers expose the equivalent as SQLSTATE `23505`.

## Common mistakes

- **Throwing for validation failures**, forcing clients to parse error messages to find which
  field was wrong.
- **Catching every database error as "taken".** Check the specific code and re-throw the rest.
- **Check-then-insert without a unique constraint** — a race that lets duplicates through.
- **Messages without codes.** Clients need a stable machine-readable value; messages are for
  display and get rewritten.
- **Union outcomes without a fallback** in clients, so a new outcome type renders as nothing.

## Exercise

1. Add `interface Error { message: String! }` and make `InputInvalid` and `AlreadyTaken`
   implement it. Rewrite the "old client" query to handle unknown errors generically.
2. Add `changeHandle(input: ChangeHandleInput!)` in style A. What new error codes does it need?
3. Make style A report *both* "email invalid" and "handle taken" in one response. What has to
   change in the order of checks?
4. Add a `clientMutationId: String` that's echoed back in the payload, and explain what problem
   it was designed to solve for clients with optimistic UIs.
