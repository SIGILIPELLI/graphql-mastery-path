---
description: "GraphQL subscriptions over WebSockets with graphql-ws: async iterators, a pub/sub, per-room filtering, connection auth and cleanup."
---

# 04 · Subscriptions over WebSockets

Queries and mutations are request/response. A **subscription** is a long-lived operation: the
client subscribes once and the server pushes a result every time something happens — a chat
message, a status change, a price tick. This lesson builds a working chat backend and a Node
client that talk the `graphql-transport-ws` protocol via the `graphql-ws` library (6.3.0 with
`ws` 8.22.0 here), and watches every stage of a subscription's life on the server.

## The shape of a subscription

```graphql
type Subscription {
  messagePosted(room: String!): Message!
}
```

A subscription operation must select **exactly one** root field. Its resolver is an object
with two functions:

- `subscribe(parent, args, context, info)` returns an **AsyncIterable** — a source of events.
- `resolve(payload, args, context, info)` (optional) maps each event to the field's value.
  Without it, the default resolver reads `payload[fieldName]`.

For every value the iterable yields, the server runs the subscription's selection set against it
— exactly like a query — and sends the result to the client.

## A minimal pub/sub

Mutations need a way to tell subscriptions that something happened. In one process, that's an
event bus that hands out async iterators:

```js title="pubsub.js"
// A minimal in-process pub/sub that hands out async iterators.
export class PubSub {
  #subscribers = new Map(); // topic -> Set<push function>

  publish(topic, payload) {
    for (const push of this.#subscribers.get(topic) ?? []) push(payload);
  }

  subscribe(topic) {
    const queue = [];
    let wake = null;
    let done = false;
    const push = (payload) => {
      queue.push(payload);
      wake?.();
    };
    const subs = this.#subscribers.get(topic) ?? new Set();
    subs.add(push);
    this.#subscribers.set(topic, subs);
    const cleanup = () => {
      done = true;
      subs.delete(push);
      wake?.();
    };
    return {
      [Symbol.asyncIterator]() { return this; },
      async next() {
        while (!done && queue.length === 0) await new Promise((r) => (wake = r));
        wake = null;
        if (done) return { value: undefined, done: true };
        return { value: queue.shift(), done: false };
      },
      async return() { cleanup(); return { value: undefined, done: true }; },
    };
  }

  listenerCount(topic) { return this.#subscribers.get(topic)?.size ?? 0; }
}
```

The important method is `return()`. When a client unsubscribes or disconnects, the server calls
`return()` on the iterator; that's the only chance to remove the listener. Without it, every
closed tab would leave a listener behind forever.

## The server

```js title="server04.mjs"
import { createServer } from "node:http";
import { WebSocketServer } from "ws";
import { useServer } from "graphql-ws/use/ws";
import { makeExecutableSchema } from "@graphql-tools/schema";
import { PubSub } from "./pubsub.js";

export const pubsub = new PubSub();
const messages = [];

export const schema = makeExecutableSchema({
  typeDefs: /* GraphQL */ `
    type Message { id: ID! room: String! author: String! text: String! }
    type Query { messages(room: String!): [Message!]! }
    type Mutation { post(room: String!, text: String!): Message! }
    type Subscription {
      messagePosted(room: String!): Message!
      tick(count: Int!): Int!
    }
  `,
  resolvers: {
    Query: { messages: (_, { room }) => messages.filter((m) => m.room === room) },
    Mutation: {
      post: (_, { room, text }, { user }) => {
        const msg = { id: String(messages.length + 1), room, author: user ?? "anonymous", text };
        messages.push(msg);
        pubsub.publish(`room:${room}`, msg); // publish AFTER the write succeeds
        return msg;
      },
    },
    Subscription: {
      messagePosted: {
        // subscribe returns an AsyncIterable; each value becomes one event
        subscribe: (_, { room }) => pubsub.subscribe(`room:${room}`),
        resolve: (payload) => payload,
      },
      tick: {
        subscribe: async function* (_, { count }) {
          for (let i = 1; i <= count; i++) {
            await new Promise((r) => setTimeout(r, 50));
            yield { tick: i };
          }
        },
      },
    },
  },
});

export function start(port) {
  const httpServer = createServer((req, res) => { res.writeHead(404).end(); });
  const wss = new WebSocketServer({ server: httpServer, path: "/graphql" });
  const handle = useServer(
    {
      schema,
      // connectionParams arrive in the first message (connection_init), not as HTTP headers
      context: (ctx) => ({ user: ctx.connectionParams?.user ?? null }),
      onConnect: (ctx) => {
        if (ctx.connectionParams?.token === "bad") return false; // reject the connection
        return true;
      },
      onSubscribe: (_ctx, id, payload) => console.log(`  [server] subscribe ${id.slice(0, 8)}: ${payload.query.replace(/\s+/g, " ").slice(0, 40)}`),
      onComplete: (_ctx, id) => console.log(`  [server] complete ${id.slice(0, 8)}`),
    },
    wss,
  );
  return new Promise((resolve) => httpServer.listen(port, () => resolve({ httpServer, wss, handle })));
}
```

Things to notice:

- `messagePosted.subscribe` builds the topic name from the `room` argument, so each subscriber
  only receives messages for its room — the filtering happens by topic, not by checking every
  message against every subscriber.
- `tick` shows the other common pattern: `subscribe` is an **async generator**. When it
  returns, the subscription completes. No `resolve` is needed because each yielded value has a
  `tick` key matching the field name.
- The mutation publishes **after** the write. Publishing first risks notifying subscribers
  about a change that then fails.
- WebSocket context comes from `connectionParams` — a JSON payload the client sends in its first
  message. Browsers can't set custom headers on WebSocket connections, so tokens usually travel
  this way. `onConnect` returning `false` rejects the connection.

## The client

```js title="client04.mjs"
import WebSocket from "ws";
import { createClient } from "graphql-ws";
import { start, pubsub } from "./server04.mjs";

const { httpServer, handle } = await start(4310);
const url = "ws://localhost:4310/graphql";

const alice = createClient({ url, webSocketImpl: WebSocket, connectionParams: { user: "alice" }, lazy: false });
const bob = createClient({ url, webSocketImpl: WebSocket, connectionParams: { user: "bob" }, lazy: false });

// 1. A finite subscription, consumed with the async-iterator API
const ticks = [];
for await (const result of alice.iterate({ query: "subscription Ticks { tick(count: 3) }" })) ticks.push(result.data.tick);
console.log("ticks:", ticks);

// 2. Room-filtered chat: bob listens to #general only
const received = [];
const stop = bob.subscribe(
  { query: `subscription Room { messagePosted(room: "general") { author text } }` },
  { next: (r) => received.push(r.data.messagePosted), error: (e) => console.log("error", e), complete: () => console.log("  [bob] complete") },
);
await new Promise((r) => setTimeout(r, 100)); // let the subscription register
console.log("listeners on room:general:", pubsub.listenerCount("room:general"));

const mutate = (client, query) => new Promise((resolve, reject) => {
  let result;
  client.subscribe({ query }, { next: (r) => (result = r), error: reject, complete: () => resolve(result) });
});
await mutate(alice, `mutation { post(room: "general", text: "hi bob") { id } }`);
await mutate(alice, `mutation { post(room: "random", text: "not for bob") { id } }`);
await mutate(alice, `mutation { post(room: "general", text: "still there?") { id } }`);
await new Promise((r) => setTimeout(r, 100));
console.log("bob received:", JSON.stringify(received));

stop(); // client sends "complete"; server calls return() on the iterator
await new Promise((r) => setTimeout(r, 100));
console.log("listeners after unsubscribe:", pubsub.listenerCount("room:general"));

// 3. A rejected connection
const evil = createClient({ url, webSocketImpl: WebSocket, connectionParams: { token: "bad" }, retryAttempts: 0 });
await new Promise((resolve) => evil.subscribe({ query: "subscription { tick(count: 1) }" },
  { next() {}, complete: resolve, error: (e) => { console.log("rejected:", e.code, e.reason); resolve(); } }));

// 4. Validation errors arrive before any event
await new Promise((resolve) => alice.subscribe({ query: "subscription { nope }" },
  { next() {}, complete: resolve, error: (e) => { console.log("invalid:", JSON.stringify(e)); resolve(); } }));

await alice.dispose(); await bob.dispose(); await evil.dispose();
await handle.dispose();
httpServer.close();
```

Output, with the server's log lines interleaved:

```console
$ node client04.mjs
  [server] subscribe 8b2ab345: subscription Ticks { tick(count: 3) }
  [server] complete 8b2ab345
ticks: [ 1, 2, 3 ]
  [server] subscribe f402d4b3: subscription Room { messagePosted(room: 
listeners on room:general: 1
  [server] subscribe cbb4c994: mutation { post(room: "general", text: "
  [server] complete cbb4c994
  [server] subscribe 43324dff: mutation { post(room: "random", text: "n
  [server] complete 43324dff
  [server] subscribe a63b62ae: mutation { post(room: "general", text: "
  [server] complete a63b62ae
bob received: [{"author":"alice","text":"hi bob"},{"author":"alice","text":"still there?"}]
  [bob] complete
  [server] complete f402d4b3
listeners after unsubscribe: 0
rejected: 4403 Forbidden
  [server] subscribe e97a24b5: subscription { nope }
invalid: [{"message":"Cannot query field \"nope\" on type \"Subscription\".","locations":[{"line":1,"column":16}]}]
```

(Operation ids are random UUIDs, shortened by the log statement.) Walking through it:

1. **`tick`** produced three events and then completed by itself, because the generator ended.
   `client.iterate` turns a subscription into a `for await` loop.
2. **Room filtering.** Bob subscribed to `general`; Alice posted twice to `general` and once to
   `random`. Bob received exactly the two `general` messages, each with `author: "alice"` — taken
   from *Alice's* connection context when her mutation ran.
3. **Mutations over the socket.** `graphql-ws` can carry queries and mutations too; each is a
   one-shot "subscription" that yields a single result and completes, which is why the server
   logs `subscribe` then `complete` for each. Many apps send mutations over HTTP instead and use
   the socket only for subscriptions; both work.
4. **Cleanup.** After `stop()`, the listener count dropped from 1 to 0 — `return()` ran.
5. **Rejected connection.** The socket was closed with code `4403 Forbidden` before any operation
   ran.
6. **Validation.** An invalid subscription failed with ordinary GraphQL validation errors, before
   any event source was created.

## Scaling beyond one process

The in-process `PubSub` only reaches subscribers connected to the **same** server process. With
two instances behind a load balancer, Alice's mutation on instance A never reaches Bob on
instance B. Production systems replace the event bus with a shared broker — Redis pub/sub, NATS,
Kafka, PostgreSQL `LISTEN/NOTIFY`, or a managed service — while the resolver code stays the
same shape: `subscribe` returns an async iterator fed by the broker. That wasn't run here; it
needs infrastructure beyond this lesson.

Other operational facts to plan for:

- **Long-lived connections** need load balancers and proxies configured for WebSockets and long
  idle timeouts; `graphql-ws` has ping/pong keep-alives for this.
- **Each connection holds memory** on the server. Thousands of idle sockets per instance are
  normal; millions need dedicated infrastructure.
- **Authorization happens per event** too. A subscription authorized at subscribe time keeps
  running if the user's permissions change later. Re-check in `resolve` for sensitive data.
- **Server-Sent Events** (`graphql-sse`, or GraphQL Yoga's built-in support) are a simpler
  alternative when traffic is server-to-client only: plain HTTP, easier through proxies.

## How It Actually Works

The `graphql-transport-ws` protocol runs over one WebSocket with JSON messages. The client
opens the socket with sub-protocol `graphql-transport-ws` and sends `connection_init` (carrying
`connectionParams`); the server answers `connection_ack` or closes the socket (4403 here, from
`onConnect` returning `false`). For each operation, the client sends `subscribe` with a unique
`id` and `{ query, variables }`; the server sends `next` messages with results, then `complete`.
Either side can send `complete` for an id to stop it. `ping`/`pong` keep the connection alive.

On the server, `graphql-ws` calls `graphql-js`'s `subscribe()` for subscription operations. That
function validates the document, calls the root field's `subscribe` resolver to get the event
stream (`createSourceEventStream`), and then returns a new async iterator that **maps** each
source event through `execute()` with the event as the root value. When the client completes or
disconnects, `graphql-ws` calls `return()` on that mapped iterator, which calls `return()` on
your source iterator — the cleanup you saw in step 4.

## Common mistakes

- **No `return()` cleanup** in custom iterators — a memory leak per disconnected client.
- **Filtering every event in `resolve`** instead of using fine-grained topics, so every
  subscriber is woken for every event.
- **Publishing before the write commits.**
- **Expecting HTTP headers on WebSockets** — use `connectionParams`.
- **Using the in-memory pub/sub with more than one instance.**
- **Huge payloads in events.** Publish ids or small objects and let the selection set fetch the
  rest, so each subscriber gets exactly the fields it asked for.

## Exercise

1. Add `typing(room: String!): String!` that publishes a user name when they start typing, and
   throttle it so a user can publish at most once per second.
2. Reject subscriptions to rooms whose names start with `private-` unless `connectionParams.user`
   is in an allow-list, using `onSubscribe`.
3. Add a `messageCount(room:)` subscription that emits the current count immediately on subscribe
   and then after every post. (Hint: an async generator that yields once before looping.)
4. Kill the client process mid-subscription (`process.exit()` instead of `stop()`) and confirm via
   `listenerCount` that the server still cleans up.
