---
description: "Authorize GraphQL access at the right level — operation, object and field — avoid the second-path bypass, and choose between errors, nulls and filtering."
---

# 02 · Authorization: Field, Object & Role Rules

With a `viewer` in the context ([lesson 01](01-authentication.md)), every resolver knows who's
asking. Authorization is deciding what they get. GraphQL makes this both easier and riskier
than REST: easier because one schema describes everything, riskier because a client can reach
the same object through many paths in one query — and every path must enforce the same
rules. This lesson implements the three levels of rules, then shows the most common
authorization bug in GraphQL APIs and the structural fix.

## Three levels

| Level | Question | Example | Typical response when denied |
|---|---|---|---|
| Operation / field-on-root | may this viewer call this at all? | only admins list users | error (`UNAUTHENTICATED` / `FORBIDDEN`) |
| Object | may this viewer see *this* object? | private posts visible to owner and admins | filter it out, or `null` |
| Field | may this viewer see *this field* of an object they can see? | email visible to self and admins | `null` for that field |

## Rules as plain functions

```js title="authz.js"
import { GraphQLError } from "graphql";

// All rules in one place, as plain functions of (viewer, object).
export const can = {
  readPost: (viewer, post) =>
    post.visibility === "PUBLIC" || (viewer && (viewer.role === "ADMIN" || viewer.id === post.authorId)),
  editPost: (viewer, post) => viewer && (viewer.role === "ADMIN" || viewer.id === post.authorId),
  seeEmail: (viewer, user) => viewer && (viewer.role === "ADMIN" || viewer.id === user.id),
  listUsers: (viewer) => viewer?.role === "ADMIN",
};

export const forbidden = (message = "Not allowed") =>
  new GraphQLError(message, { extensions: { code: "FORBIDDEN" } });
export const unauthenticated = () =>
  new GraphQLError("You must be logged in", { extensions: { code: "UNAUTHENTICATED" } });
```

Keeping every rule in one module, as pure functions of `(viewer, object)`, has three benefits:
the rules are unit-testable without GraphQL, they can be reused by REST endpoints or background
jobs, and reviewing "who can read a post?" means reading one line.

## Applying them

```js title="server02.mjs"
import { ApolloServer } from "@apollo/server";
import { makeExecutableSchema } from "@graphql-tools/schema";
import { can, forbidden, unauthenticated } from "./authz.js";

const users = [
  { id: "u1", name: "Ada", email: "ada@example.com", role: "ADMIN" },
  { id: "u2", name: "Bob", email: "bob@example.com", role: "MEMBER" },
  { id: "u3", name: "Cy", email: "cy@example.com", role: "MEMBER" },
];
const posts = [
  { id: "p1", title: "Hello world", visibility: "PUBLIC", authorId: "u2" },
  { id: "p2", title: "Bob's draft", visibility: "PRIVATE", authorId: "u2" },
  { id: "p3", title: "Cy's notes", visibility: "PRIVATE", authorId: "u3" },
];

export const schema = makeExecutableSchema({
  typeDefs: /* GraphQL */ `
    type Query {
      posts: [Post!]!
      post(id: ID!): Post
      users: [User!]!
    }
    type Mutation { renamePost(id: ID!, title: String!): Post }
    type Post { id: ID! title: String! visibility: String! author: User! }
    type User {
      id: ID!
      name: String!
      "Visible to the user themselves and to admins; null otherwise."
      email: String
    }
  `,
  resolvers: {
    Query: {
      // object-level: filter what the viewer may read
      posts: (_, __, { viewer }) => posts.filter((p) => can.readPost(viewer, p)),
      // a hidden object looks exactly like a missing one
      post: (_, { id }, { viewer }) => {
        const p = posts.find((x) => x.id === id);
        return p && can.readPost(viewer, p) ? p : null;
      },
      // operation-level: whole field requires a role
      users: (_, __, { viewer }) => {
        if (!viewer) throw unauthenticated();
        if (!can.listUsers(viewer)) throw forbidden("Only admins can list users");
        return users;
      },
    },
    Mutation: {
      renamePost: (_, { id, title }, { viewer }) => {
        if (!viewer) throw unauthenticated();
        const p = posts.find((x) => x.id === id);
        if (!p || !can.readPost(viewer, p)) return null;          // don't reveal it exists
        if (!can.editPost(viewer, p)) throw forbidden("You can only rename your own posts");
        p.title = title;
        return p;
      },
    },
    Post: { author: (p) => users.find((u) => u.id === p.authorId) },
    // field-level: redact instead of failing the whole object
    User: { email: (u, _, { viewer }) => (can.seeEmail(viewer, u) ? u.email : null) },
  },
});

const server = new ApolloServer({ schema, includeStacktraceInErrorResponses: false });
const as = (userId) => ({ viewer: users.find((u) => u.id === userId) ?? null });

async function run(label, who, query) {
  const res = await server.executeOperation({ query }, { contextValue: as(who) });
  console.log(`--- ${label} (as ${who ?? "anonymous"})\n${JSON.stringify(res.body.singleResult)}`);
}

const LIST = `{ posts { id title author { name email } } }`;
await run("post list", null, LIST);
await run("post list", "u2", LIST);
await run("post list", "u1", LIST);
await run("someone else's private post", "u2", `{ post(id: "p3") { title } }`);
await run("a post that doesn't exist", "u2", `{ post(id: "p999") { title } }`);
await run("users", null, `{ users { name } }`);
await run("users", "u2", `{ users { name } }`);
await run("users", "u1", `{ users { name email } }`);
await run("rename own post", "u2", `mutation { renamePost(id: "p2", title: "Bob's post") { title } }`);
await run("rename someone else's public post", "u3", `mutation { renamePost(id: "p1", title: "pwned") { title } }`);
await run("rename someone else's private post", "u3", `mutation { renamePost(id: "p2", title: "pwned") { title } }`);
```

### Object level: filtered lists, invisible objects

```text
--- post list (as anonymous)
{"data":{"posts":[{"id":"p1","title":"Hello world","author":{"name":"Bob","email":null}}]}}
--- post list (as u2)
{"data":{"posts":[{"id":"p1","title":"Hello world","author":{"name":"Bob","email":"bob@example.com"}},{"id":"p2","title":"Bob's draft","author":{"name":"Bob","email":"bob@example.com"}}]}}
--- post list (as u1)
{"data":{"posts":[{"id":"p1",…},{"id":"p2",…},{"id":"p3","title":"Cy's notes","author":{"name":"Cy","email":"cy@example.com"}}]}}
```

(The admin's output is abbreviated.) Same query, three different results. Lists are
**filtered** — an error for each hidden item would be noise.

For a single lookup, a hidden object returns exactly what a missing one returns:

```text
--- someone else's private post (as u2)
{"data":{"post":null}}
--- a post that doesn't exist (as u2)
{"data":{"post":null}}
```

If the first case returned `FORBIDDEN`, anyone could probe ids to learn which private posts
exist. Making hidden and missing indistinguishable leaks nothing.

### Field level: redaction

`author.email` was `null` for the anonymous viewer and filled in for Bob (his own email) and the
admin. The rest of the `User` object was still returned. That's why `email` is nullable in the
schema, and its description says when it's `null` — a field that can be redacted *must* be
nullable, or a denied read would null out the whole parent ([Level 1 · 08](../level-1/08-errors.md)).

### Operation level: explicit errors

```text
--- users (as anonymous)
{"errors":[{"message":"You must be logged in",…,"extensions":{"code":"UNAUTHENTICATED"}}],"data":null}
--- users (as u2)
{"errors":[{"message":"Only admins can list users",…,"extensions":{"code":"FORBIDDEN"}}],"data":null}
--- users (as u1)
{"data":{"users":[{"name":"Ada","email":"ada@example.com"},{"name":"Bob","email":"bob@example.com"},{"name":"Cy","email":"cy@example.com"}]}}
```

Here an error is right: the client asked for something it can't have and needs to know why. The
two codes mean different things to a client — `UNAUTHENTICATED` means "log in and retry",
`FORBIDDEN` means "logging in again won't help".

### Mutations: check readability before writability

```text
--- rename own post (as u2)
{"data":{"renamePost":{"title":"Bob's post"}}}
--- rename someone else's public post (as u3)
{"errors":[{"message":"You can only rename your own posts",…,"extensions":{"code":"FORBIDDEN"}}],"data":{"renamePost":null}}
--- rename someone else's private post (as u3)
{"data":{"renamePost":null}}
```

Cy can *see* Bob's public post, so telling her she can't edit it leaks nothing. Bob's private
post is invisible to her, so she gets the same `null` as for a non-existent id. The order of
checks in `renamePost` — exists and readable first, then editable — produces that.

## The second-path bug

Here's the bug that shows up in real GraphQL APIs. Version 1 puts the read rule in
`Query.posts`; later someone adds `User.posts` and forgets it:

```js title="bypass02.mjs"
import { makeExecutableSchema } from "@graphql-tools/schema";
import { graphql } from "graphql";
import { can } from "./authz.js";

const users = [{ id: "u2", name: "Bob" }, { id: "u3", name: "Cy" }];
const posts = [
  { id: "p1", title: "Hello world", visibility: "PUBLIC", authorId: "u2" },
  { id: "p3", title: "Cy's notes", visibility: "PRIVATE", authorId: "u3" },
];
const typeDefs = `type Query { posts: [Post!]! users: [User!]! }
  type Post { id: ID! title: String! author: User! }
  type User { name: String! posts: [Post!]! }`;

// v1: the check lives in Query.posts only
const v1 = makeExecutableSchema({ typeDefs, resolvers: {
  Query: { posts: (_, __, { viewer }) => posts.filter((p) => can.readPost(viewer, p)), users: () => users },
  Post: { author: (p) => users.find((u) => u.id === p.authorId) },
  User: { posts: (u) => posts.filter((p) => p.authorId === u.id) },          // forgot the check
}});

// v2: every path goes through one data-access function that applies the rule
const visiblePosts = (viewer, where = () => true) => posts.filter((p) => where(p) && can.readPost(viewer, p));
const v2 = makeExecutableSchema({ typeDefs, resolvers: {
  Query: { posts: (_, __, { viewer }) => visiblePosts(viewer), users: () => users },
  Post: { author: (p) => users.find((u) => u.id === p.authorId) },
  User: { posts: (u, _, { viewer }) => visiblePosts(viewer, (p) => p.authorId === u.id) },
}});

const q = `{ posts { title } users { name posts { title } } }`;
for (const [name, schema] of [["v1", v1], ["v2", v2]])
  console.log(name, JSON.stringify((await graphql({ schema, source: q, contextValue: { viewer: null } })).data));
```

```console
$ node bypass02.mjs
v1 {"posts":[{"title":"Hello world"}],"users":[{"name":"Bob","posts":[{"title":"Hello world"}]},{"name":"Cy","posts":[{"title":"Cy's notes"}]}]}
v2 {"posts":[{"title":"Hello world"}],"users":[{"name":"Bob","posts":[{"title":"Hello world"}]},{"name":"Cy","posts":[]}]}
```

In v1 an anonymous client can't see "Cy's notes" through `posts` — but can through
`users { posts }`. Every edge in a graph is another way to reach an object, and a schema grows
edges over time (`Comment.post`, `Notification.subject`, `search` results, `node(id:)`…).

v2 fixes it structurally: **all post reads go through one data-access function that applies the
rule**. Resolvers can't forget a check they never perform. In a larger codebase this is a model
or repository layer (`Posts.visibleTo(viewer)`), and DataLoaders sit on top of it — loaders are
created per request, so they can carry the viewer too.

## Directives and middleware

You'll also see authorization expressed in the schema (`email: String @auth(requires: SELF)`)
and implemented by a schema transform, or applied as resolver middleware. That's the next
lesson. Those are good for coarse, declarative rules (roles, logged-in-only) and make the
policy visible in the SDL. Object-level rules that depend on data usually still belong in the
data layer, for the second-path reason above.

## How It Actually Works

Nothing in GraphQL execution knows about authorization; it's all ordinary resolver code. What
makes the patterns work is execution order and null handling. A list resolver that filters
returns fewer items, and the executor never calls child resolvers for items that weren't
returned — so filtering at the parent protects every field below. Returning `null` from a
nullable object field has the same effect for a single object. A field resolver that returns
`null` (the email redaction) affects only that leaf. Throwing turns into a field error at that
path and, through non-null propagation, may null out parents; that's why operation-level checks
sit on nullable root fields or are expected to null `data` entirely.

## Common mistakes

- **Authorizing in only one resolver** when the object is reachable from several — the
  second-path bug.
- **Non-null fields that can be redacted.** A denied `email: String!` nulls the whole user.
- **`FORBIDDEN` for invisible objects**, which confirms they exist.
- **Trusting role claims in the token** for decisions that change when roles change. Tokens
  live for minutes; load roles from the database when that matters.
- **Relying on the client** to hide fields. The server is the only enforcement point.
- **Authorization in the gateway only.** In federated setups ([Level 4](../level-4/index.md)),
  each subgraph must still enforce its own data rules.

## Exercise

1. Add `Comment` with a `post` field and a `comments` list on `Post`. Make sure a comment on a
   private post is invisible through every path. Write a test for each path.
2. Add an `EDITOR` role that can rename any post but not read private ones. Where does the rule
   change, and does `renamePost`'s check order still make sense?
3. Add `node(id: ID!): Node` and make it respect `can.readPost`.
4. Write unit tests for `can` covering anonymous, owner, other member and admin for every rule.
