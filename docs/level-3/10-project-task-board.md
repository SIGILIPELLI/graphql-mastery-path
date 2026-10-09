---
description: "Level 3 project: a real-time GraphQL task board with JWT auth over HTTP and WebSockets, membership rules, live updates and revocation."
---

# 10 · Project — A Real-Time Task Board with Auth

This project combines the whole of Level 3 in one small but realistic service: a shared task
board where members create and move tasks and see each other's changes live. It needs
authentication on two transports (HTTP and WebSocket), one authorization rule enforced on
every path, payload-style mutations, subscriptions that respect permissions even when they
change mid-connection, and a test suite that drives real HTTP and WebSocket clients.

## Requirements

- Users log in with a JWT. Queries and mutations go over HTTP; live updates over WebSocket.
- A board's data — its tasks, members, events — is visible to its members only. Non-members
  get exactly what they'd get for a board that doesn't exist.
- `createTask` and `moveTask` report bad input and missing objects as `userErrors`.
- Members subscribed to a board receive `CREATED` and `MOVED` events with the actor and task.
- If a member is removed while connected, they stop receiving that board's data immediately.

## Layout

```text
taskboard/
├── schema.graphql
├── store.js            in-memory data
├── auth.js             tokens + the membership rule
├── pubsub.js           from lesson 04
├── resolvers.js
├── server.js           HTTP (Apollo) + WebSocket (graphql-ws) on one port
├── main.js
└── taskboard.test.js
```

## Schema

```graphql title="schema.graphql"
type Query {
  viewer: User
  board(id: ID!): Board
}

type Mutation {
  createTask(input: CreateTaskInput!): TaskPayload!
  moveTask(input: MoveTaskInput!): TaskPayload!
}

type Subscription {
  "Every change to tasks on a board the viewer belongs to."
  taskChanged(boardId: ID!): TaskEvent!
}

type User { id: ID! name: String! boards: [Board!]! }

type Board {
  id: ID!
  name: String!
  members: [User!]!
  tasks(status: TaskStatus): [Task!]!
}

type Task {
  id: ID!
  title: String!
  status: TaskStatus!
  assignee: User
}

enum TaskStatus { TODO DOING DONE }

type TaskEvent {
  kind: TaskEventKind!
  task: Task!
  actor: User!
}
enum TaskEventKind { CREATED MOVED }

input CreateTaskInput { boardId: ID!, title: String!, assigneeId: ID }
input MoveTaskInput { taskId: ID!, status: TaskStatus! }

type TaskPayload { task: Task, userErrors: [UserError!]! }
type UserError { field: [String!], message: String!, code: String! }
```

## Data and auth

```js title="store.js"
// In-memory data for the project. Swap for a database without touching resolvers.
export function createStore() {
  return {
    users: new Map([
      ["u1", { id: "u1", name: "Ada" }],
      ["u2", { id: "u2", name: "Bob" }],
      ["u3", { id: "u3", name: "Eve" }],
    ]),
    boards: new Map([["b1", { id: "b1", name: "Launch", memberIds: ["u1", "u2"] }]]),
    tasks: new Map([["t1", { id: "t1", boardId: "b1", title: "Write release notes", status: "TODO", assigneeId: "u2" }]]),
    nextTaskId: 2,
  };
}
```

```js title="auth.js"
import { SignJWT, jwtVerify } from "jose";
import { GraphQLError } from "graphql";

const SECRET = new TextEncoder().encode(process.env.JWT_SECRET ?? "dev-only-secret-change-me-0123456789");
const OPTS = { issuer: "taskboard", audience: "taskboard-app", algorithms: ["HS256"] };

export const issueToken = (userId, expiresIn = "15m") =>
  new SignJWT({}).setProtectedHeader({ alg: "HS256" }).setSubject(userId)
    .setIssuer(OPTS.issuer).setAudience(OPTS.audience).setIssuedAt().setExpirationTime(expiresIn).sign(SECRET);

export async function viewerFromToken(token, store) {
  if (!token) return null;
  try {
    const { payload } = await jwtVerify(token, SECRET, OPTS);
    return store.users.get(payload.sub) ?? null;
  } catch {
    throw new GraphQLError("Invalid or expired token", { extensions: { code: "UNAUTHENTICATED", http: { status: 401 } } });
  }
}

export const requireViewer = (viewer) => {
  if (!viewer) throw new GraphQLError("You must be logged in", { extensions: { code: "UNAUTHENTICATED" } });
  return viewer;
};

// The single authorization rule for this app: board data is visible to board members only.
export const isMember = (viewer, board) => Boolean(viewer && board && board.memberIds.includes(viewer.id));
```

The authorization rule is one line, `isMember`. Everything else calls it — directly or through
the two helpers below.

## Resolvers

```js title="resolvers.js"
import { GraphQLError } from "graphql";
import { isMember, requireViewer } from "./auth.js";

const userError = (field, message, code) => ({ task: null, userErrors: [{ field, message, code }] });

// Every board/task read goes through these, so no path can skip the membership rule.
const visibleBoard = (viewer, store, id) => {
  const b = store.boards.get(id);
  return isMember(viewer, b) ? b : null;
};
const visibleTask = (viewer, store, id) => {
  const t = store.tasks.get(id);
  return t && visibleBoard(viewer, store, t.boardId) ? t : null;
};

export const resolvers = {
  Query: {
    viewer: (_, __, { viewer }) => viewer,
    board: (_, { id }, { viewer, store }) => visibleBoard(viewer, store, id),
  },
  Mutation: {
    createTask: (_, { input }, { viewer, store, pubsub }) => {
      requireViewer(viewer);
      const board = visibleBoard(viewer, store, input.boardId);
      if (!board) return userError(["input", "boardId"], "No such board.", "NOT_FOUND");
      const title = input.title.trim();
      if (title.length < 1 || title.length > 200) return userError(["input", "title"], "Title must be 1–200 characters.", "INVALID");
      if (input.assigneeId && !board.memberIds.includes(input.assigneeId))
        return userError(["input", "assigneeId"], "Assignee must be a board member.", "INVALID");
      const task = { id: `t${store.nextTaskId++}`, boardId: board.id, title, status: "TODO", assigneeId: input.assigneeId ?? null };
      store.tasks.set(task.id, task);
      pubsub.publish(`board:${board.id}`, { kind: "CREATED", taskId: task.id, actorId: viewer.id });
      return { task, userErrors: [] };
    },
    moveTask: (_, { input }, { viewer, store, pubsub }) => {
      requireViewer(viewer);
      const task = visibleTask(viewer, store, input.taskId);
      if (!task) return userError(["input", "taskId"], "No such task.", "NOT_FOUND");
      if (task.status === input.status) return { task, userErrors: [] }; // no-op, no event
      task.status = input.status;
      pubsub.publish(`board:${task.boardId}`, { kind: "MOVED", taskId: task.id, actorId: viewer.id });
      return { task, userErrors: [] };
    },
  },
  Subscription: {
    taskChanged: {
      subscribe: (_, { boardId }, { viewer, store, pubsub }) => {
        requireViewer(viewer);
        if (!visibleBoard(viewer, store, boardId))
          throw new GraphQLError("No such board", { extensions: { code: "NOT_FOUND" } });
        return pubsub.subscribe(`board:${boardId}`);
      },
      // Re-check on every event: membership can change while a socket stays open.
      resolve: (event, { boardId }, { viewer, store }) => {
        if (!visibleBoard(viewer, store, boardId))
          throw new GraphQLError("Access to this board was revoked", { extensions: { code: "FORBIDDEN" } });
        return event;
      },
    },
  },
  TaskEvent: {
    task: (e, _, { store }) => store.tasks.get(e.taskId),
    actor: (e, _, { store }) => store.users.get(e.actorId),
  },
  User: {
    boards: (u, _, { viewer, store }) =>
      viewer?.id === u.id ? [...store.boards.values()].filter((b) => b.memberIds.includes(u.id)) : [],
  },
  Board: {
    members: (b, _, { store }) => b.memberIds.map((id) => store.users.get(id)),
    tasks: (b, { status }, { store }) =>
      [...store.tasks.values()].filter((t) => t.boardId === b.id && (!status || t.status === status)),
  },
  Task: {
    assignee: (t, _, { store }) => (t.assigneeId ? store.users.get(t.assigneeId) : null),
  },
};
```

How the Level 3 lessons show up:

- **Second-path safety** ([lesson 02](02-authorization.md)): `visibleBoard` and `visibleTask` are
  the only ways resolvers fetch boards and tasks. `Query.board`, both mutations, the
  subscription and its per-event `resolve` all go through them.
- **Hidden equals missing**: Eve asking for board `b1` gets `null`, and her `createTask` on it gets
  `NOT_FOUND` — the same as for a board id that doesn't exist.
- **Events carry ids, not objects** ([lesson 04](04-subscriptions.md)): the payload is
  `{ kind, taskId, actorId }`, and `TaskEvent.task`/`actor` load the current data for each
  subscriber's own selection set.
- **No-op moves publish nothing**, so clients don't re-render for changes that didn't happen.
- **Per-event authorization**: `taskChanged.resolve` re-checks membership on every event, because
  a WebSocket can stay open for hours after the check in `subscribe`.

## One server, two transports

```js title="server.js"
import { createServer } from "node:http";
import { readFileSync } from "node:fs";
import { WebSocketServer } from "ws";
import { useServer } from "graphql-ws/use/ws";
import { ApolloServer, HeaderMap } from "@apollo/server";
import { makeExecutableSchema } from "@graphql-tools/schema";
import { resolvers } from "./resolvers.js";
import { viewerFromToken } from "./auth.js";
import { PubSub } from "./pubsub.js";

const typeDefs = readFileSync(new URL("./schema.graphql", import.meta.url), "utf8");

export async function startServer({ port = 0, store }) {
  const schema = makeExecutableSchema({ typeDefs, resolvers });
  const pubsub = new PubSub();
  const apollo = new ApolloServer({ schema, includeStacktraceInErrorResponses: false });
  await apollo.start();

  // A minimal Apollo integration: Node request -> HTTPGraphQLRequest -> Node response.
  const httpServer = createServer(async (req, res) => {
    if (new URL(req.url, "http://x").pathname !== "/graphql") return res.writeHead(404).end();
    const chunks = [];
    for await (const c of req) chunks.push(c);
    const raw = Buffer.concat(chunks).toString();
    const headers = new HeaderMap();
    for (const [k, v] of Object.entries(req.headers)) if (typeof v === "string") headers.set(k, v);
    let body;
    try { body = raw ? JSON.parse(raw) : undefined; } catch { return res.writeHead(400).end("Invalid JSON"); }
    const result = await apollo.executeHTTPGraphQLRequest({
      httpGraphQLRequest: { method: req.method, headers, search: new URL(req.url, "http://x").search, body },
      context: async () => {
        const token = (req.headers.authorization ?? "").replace(/^Bearer /, "");
        return { viewer: await viewerFromToken(token, store), store, pubsub };
      },
    });
    for (const [k, v] of result.headers) res.setHeader(k, v);
    res.statusCode = result.status ?? 200;
    if (result.body.kind === "complete") return res.end(result.body.string);
    for await (const chunk of result.body.asyncIterator) res.write(chunk);
    res.end();
  });

  const wss = new WebSocketServer({ server: httpServer, path: "/graphql" });
  const wsHandle = useServer({
    schema,
    // Authenticate once per socket, in connection_init.
    onConnect: async (ctx) => {
      try { ctx.extra.viewer = await viewerFromToken(ctx.connectionParams?.token, store); return true; }
      catch { return false; }
    },
    context: (ctx) => ({ viewer: ctx.extra.viewer, store, pubsub }),
  }, wss);

  await new Promise((r) => httpServer.listen(port, r));
  const url = `http://localhost:${httpServer.address().port}/graphql`;
  return {
    url,
    wsUrl: url.replace("http", "ws"),
    pubsub,
    async stop() { await wsHandle.dispose(); await apollo.stop(); httpServer.close(); },
  };
}
```

Apollo Server's standalone helper owns its own HTTP server, which leaves nowhere to attach a
WebSocket server on the same port. So `server.js` includes a **minimal custom integration**:
read the Node request, build an `HTTPGraphQLRequest` (method, `HeaderMap`, search string, parsed
body), call `apollo.executeHTTPGraphQLRequest` with a context function, and write the returned
status, headers and body back. That's essentially what every Apollo integration package
(Express, Fastify, Lambda…) does. Body size limits and CORS, which a real framework would
provide, are left out to keep it short.

The WebSocket side authenticates **once per connection** in `onConnect`, stores the viewer on
`ctx.extra`, and builds each operation's context from it. A bad token closes the socket with
4403.

```console
$ PORT=4320 node main.js
Task board at http://localhost:4320/graphql
$ curl -s localhost:4320/graphql -H 'content-type: application/json' -H "authorization: Bearer $TOKEN" \
    -d '{"query":"{ viewer { name boards { name tasks { title status assignee { name } } } } }"}'
{"data":{"viewer":{"name":"Ada","boards":[{"name":"Launch","tasks":[{"title":"Write release notes","status":"TODO","assignee":{"name":"Bob"}}]}]}}}
```

(`$TOKEN` is the dev token `main.js` prints.)

## Tests

```js title="taskboard.test.js"
import { test, before, after } from "node:test";
import assert from "node:assert/strict";
import WebSocket from "ws";
import { createClient } from "graphql-ws";
import { startServer } from "./server.js";
import { createStore } from "./store.js";
import { issueToken } from "./auth.js";

let app, store, tokens;
before(async () => {
  store = createStore();
  app = await startServer({ store });
  tokens = { ada: await issueToken("u1"), bob: await issueToken("u2"), eve: await issueToken("u3") };
});
after(() => app.stop());

async function http(query, variables, token) {
  const res = await fetch(app.url, {
    method: "POST",
    headers: { "content-type": "application/json", ...(token ? { authorization: `Bearer ${token}` } : {}) },
    body: JSON.stringify({ query, variables }),
  });
  return { status: res.status, ...(await res.json()) };
}
const ws = (token) => createClient({ url: app.wsUrl, webSocketImpl: WebSocket, connectionParams: { token }, retryAttempts: 0 });
const tick = (ms = 50) => new Promise((r) => setTimeout(r, ms));

const CREATE = `mutation($i: CreateTaskInput!) { createTask(input: $i) { task { id title status assignee { name } } userErrors { field code } } }`;
const MOVE = `mutation($i: MoveTaskInput!) { moveTask(input: $i) { task { id status } userErrors { code } } }`;

test("members see the board; others see nothing", async () => {
  const q = `{ board(id: "b1") { name members { name } tasks { title } } }`;
  assert.deepEqual((await http(q, {}, tokens.ada)).data.board.members.map((m) => m.name), ["Ada", "Bob"]);
  assert.deepEqual((await http(q, {}, tokens.eve)).data, { board: null });
  assert.deepEqual((await http(q)).data, { board: null });
});

test("mutations require a login and a valid token", async () => {
  const anon = await http(CREATE, { i: { boardId: "b1", title: "x" } });
  assert.equal(anon.errors[0].extensions.code, "UNAUTHENTICATED");
  const forged = await http(`{ viewer { name } }`, {}, tokens.ada.slice(0, -2) + "xx");
  assert.equal(forged.status, 401);
});

test("validation and membership problems are userErrors", async () => {
  const r1 = await http(CREATE, { i: { boardId: "b1", title: "   " } }, tokens.ada);
  assert.deepEqual(r1.data.createTask.userErrors, [{ field: ["input", "title"], code: "INVALID" }]);
  const r2 = await http(CREATE, { i: { boardId: "b1", title: "Hi", assigneeId: "u3" } }, tokens.ada);
  assert.deepEqual(r2.data.createTask.userErrors, [{ field: ["input", "assigneeId"], code: "INVALID" }]);
  const r3 = await http(CREATE, { i: { boardId: "b1", title: "Sneaky" } }, tokens.eve);
  assert.deepEqual(r3.data.createTask.userErrors, [{ field: ["input", "boardId"], code: "NOT_FOUND" }]);
});

test("Bob receives Ada's changes live; Eve can't subscribe", async () => {
  const bob = ws(tokens.bob);
  const events = [];
  const stop = bob.subscribe(
    { query: `subscription { taskChanged(boardId: "b1") { kind actor { name } task { title status } } }` },
    { next: (r) => events.push(r.data?.taskChanged ?? r.errors), error: () => {}, complete: () => {} });
  await tick();

  const created = await http(CREATE, { i: { boardId: "b1", title: "Ship it", assigneeId: "u2" } }, tokens.ada);
  const id = created.data.createTask.task.id;
  await http(MOVE, { i: { taskId: id, status: "DOING" } }, tokens.ada);
  await http(MOVE, { i: { taskId: id, status: "DOING" } }, tokens.ada); // no-op: no event
  await tick();
  assert.deepEqual(events, [
    { kind: "CREATED", actor: { name: "Ada" }, task: { title: "Ship it", status: "TODO" } },
    { kind: "MOVED", actor: { name: "Ada" }, task: { title: "Ship it", status: "DOING" } },
  ]);
  stop();

  const eve = ws(tokens.eve);
  const err = await new Promise((resolve) => eve.subscribe(
    { query: `subscription { taskChanged(boardId: "b1") { kind } }` },
    { next: (r) => resolve(r.errors?.[0]?.message ?? "got data"), error: (e) => resolve(Array.isArray(e) ? e[0].message : String(e)), complete: () => resolve("completed") }));
  assert.equal(err, "No such board");
  await bob.dispose(); await eve.dispose();
});

test("a socket with a bad token is refused", async () => {
  const bad = ws("not-a-token");
  const reason = await new Promise((resolve) => bad.subscribe(
    { query: `subscription { taskChanged(boardId: "b1") { kind } }` },
    { next() {}, complete: () => resolve("completed"), error: (e) => resolve(`${e.code} ${e.reason}`) }));
  assert.equal(reason, "4403 Forbidden");
  await bad.dispose();
});

test("removing a member stops their live updates", async () => {
  const bob = ws(tokens.bob);
  const seen = [];
  bob.subscribe({ query: `subscription { taskChanged(boardId: "b1") { kind } }` },
    { next: (r) => seen.push(r.data ? r.data.taskChanged.kind : r.errors[0].extensions.code), error: () => {}, complete: () => {} });
  await tick();
  await http(CREATE, { i: { boardId: "b1", title: "Before" } }, tokens.ada);
  await tick();
  store.boards.get("b1").memberIds = ["u1"]; // Bob removed
  await http(CREATE, { i: { boardId: "b1", title: "After" } }, tokens.ada);
  await tick();
  assert.deepEqual(seen, ["CREATED", "FORBIDDEN"]);
  store.boards.get("b1").memberIds = ["u1", "u2"];
  await bob.dispose();
});
```

```console
$ node --test
✔ members see the board; others see nothing
✔ mutations require a login and a valid token
✔ validation and membership problems are userErrors
✔ Bob receives Ada's changes live; Eve can't subscribe
✔ a socket with a bad token is refused
✔ removing a member stops their live updates
ℹ tests 6
ℹ pass 6
ℹ fail 0
```

(Timings trimmed.) The suite starts the real server on a random port (`port: 0`) and uses real
`fetch` and `graphql-ws` clients, so it covers the custom HTTP integration and the socket
authentication as well as the resolvers. The short `tick()` waits give the socket time to
register a subscription before the test publishes; the whole suite still ran in well under a
second here.

### What the revocation test proves

`removing a member stops their live updates` subscribes Bob, publishes one event, removes Bob
from the board while his socket stays open, and publishes another. Bob's client saw:

```text
["CREATED", "FORBIDDEN"]
```

The second event reached his subscription, but `resolve` re-checked membership and turned it
into an error instead of data. The subscription itself stays open (each later event would be the
same error). A production system might also close it from the server side — `graphql-ws` lets you
complete an operation — and the client should treat `FORBIDDEN` as "leave this board view".

## How It Actually Works

An HTTP request flows: Node `http` → the custom handler → `executeHTTPGraphQLRequest`, which
runs CSRF prevention, the context function (token → viewer), parse, validate and execute, and
returns `{ status, headers, body }`. A WebSocket subscription flows: `connection_init` →
`onConnect` (token → viewer, stored on the socket) → `subscribe` message → `graphql-js`'s
`subscribe()` with a context built from that viewer → `taskChanged.subscribe` returns a pub/sub
iterator. When Ada's HTTP mutation publishes `{ kind, taskId, actorId }`, the in-process
`PubSub` pushes it into Bob's iterator; `graphql-js` maps it through `taskChanged.resolve` and
then executes Bob's selection set (`TaskEvent.task`, `actor`) with **Bob's** context, and
`graphql-ws` sends the result as a `next` message. Both transports share one `store` and one
`pubsub`, which is why an HTTP mutation reaches a WebSocket subscriber — and why, as in lesson
04, this design needs a shared broker before it can run as more than one process.

## Review checklist

- [ ] Every board or task read goes through `visibleBoard`/`visibleTask`.
- [ ] Hidden and missing objects are indistinguishable.
- [ ] Sockets authenticate in `connection_init`; HTTP authenticates per request.
- [ ] Subscription events are re-authorized per event.
- [ ] Events carry ids; each subscriber's selection set loads current data.

## Exercise

1. Add `addMember(boardId, userId)` (members only) and a `memberJoined` event kind. Should the
   new member's open subscriptions start receiving events immediately?
2. Close Bob's subscription from the server when he's removed, instead of sending errors, and
   update the test.
3. Add `Task.comments` and make sure a comment can't be reached by a non-member through any path.
4. Replace the in-process `PubSub` with an interface and write an implementation backed by
   PostgreSQL `LISTEN/NOTIFY` or Redis (requires running that service).
