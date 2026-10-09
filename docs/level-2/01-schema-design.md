---
description: "Design GraphQL schemas around client needs: naming, IDs, money and time fields, nullability rules backed by a tested matrix of safe and breaking changes."
---

# 01 · Schema Design: Nullability, Naming & Client-Driven Shapes

Level 1 was about mechanics. Level 2 starts with the decision that outlives all of them: the
shape of the schema. Resolvers can be rewritten in an afternoon; a published field that
fifty mobile app versions depend on is effectively permanent. This lesson works through one
realistic design, then puts numbers — well, `graphql-js` verdicts — behind the rules about
nullability and evolution.

## The same data, two schemas

A shop's order-history screen shows, for each order: number, status, date, total, the
products with thumbnails and quantities, and a tracking link if shipped.

Here's the schema you get by exposing database tables directly:

```graphql title="db-shaped.graphql"
type Query {
  orders(user_id: Int): [orders]
  order_items(order_id: Int): [order_items]
  products(id: Int): [products]
}
type orders { id: Int, user_id: Int, status: String, created: String, total_cents: Int }
type order_items { id: Int, order_id: Int, product_id: Int, qty: Int, price_cents: Int }
type products { id: Int, name: String, image_path: String }
```

It's valid GraphQL, and it's a bad API. The screen needs three round trips (orders, then
items per order, then products per item) — the exact problem GraphQL exists to remove.
`user_id` as an argument means any caller can ask for anyone's orders. Every field is
nullable, so clients must null-check everything. `total_cents` and `image_path` leak storage
decisions, and `status: String` gives clients no list of possible values.

Now designed from the screen backwards:

```graphql title="client-shaped.graphql"
type Query {
  viewer: User
  order(id: ID!): Order
}

type User {
  id: ID!
  displayName: String!
  orders(first: Int = 10, status: OrderStatus): [Order!]!
}

"""A placed order. Totals are computed server-side and always include tax."""
type Order {
  id: ID!
  number: String!
  status: OrderStatus!
  placedAt: String!
  lines: [OrderLine!]!
  total: Money!
  "Null until the carrier has scanned the parcel."
  tracking: Shipment
}

type OrderLine {
  product: Product!
  quantity: Int!
  lineTotal: Money!
}

type Product {
  id: ID!
  name: String!
  imageUrl(width: Int = 400): String
}

type Money {
  amount: String!
  currency: String!
  formatted: String!
}

type Shipment { carrier: String! trackingUrl: String! }

enum OrderStatus { PENDING PAID SHIPPED DELIVERED CANCELLED }
```

And the screen's entire data need is one query, which validates against it:

```graphql
query OrderHistory {
  viewer {
    displayName
    orders(first: 5) {
      number status placedAt
      total { formatted }
      lines { quantity product { name imageUrl(width: 120) } }
      tracking { trackingUrl }
    }
  }
}
```

(Both schemas were checked with `validateSchema`, and the query with `validate` — no
errors.) Note that `orders(first:)` here is a plain list for simplicity; real pagination
gets its own lesson ([07](07-pagination.md)).

## The design rules behind it

**Start from `viewer`, not from ids the client must already know.** "The logged-in user's
orders" is `viewer.orders`; the server decides who the viewer is from the request context,
so there's no `userId` argument to tamper with.

**Model relationships as fields, not foreign keys.** `OrderLine.product` returns a
`Product`; there's no `productId` the client must look up separately. You can still expose an
id when clients genuinely need it, but the edge comes first.

**Name for the domain, consistently.** `camelCase` fields, `PascalCase` types,
`SCREAMING_SNAKE` enum values — that's the overwhelming convention and what generated client
code expects. Pick one verb set for mutations (`createX`/`updateX`/`deleteX`, or
domain verbs like `placeOrder`, `cancelOrder`) and stick to it.

**Use `ID` for identifiers.** It serialises as a string, which keeps you free to change
from integers to UUIDs or to opaque global ids (`"T3JkZXI6NDI="`) without a breaking change.

**Make awkward values into types.** Money as `Int` cents or as `Float` pushes formatting and
rounding bugs into every client. A `Money` object can carry the exact amount (as a decimal
string), currency, and a server-formatted display string. Times deserve care too:
`placedAt: String!` here is an ISO-8601 timestamp by documentation only — a custom `DateTime`
scalar ([lesson 03](03-custom-scalars.md)) makes that enforceable.

**Put arguments where the variation is.** `imageUrl(width:)` lets each screen ask for the
size it renders, instead of the schema guessing.

**Use enums for closed sets.** `OrderStatus` documents every value and lets clients switch
exhaustively. But see the evolution matrix below — adding a value later is "dangerous".

**Document meaning, not type.** `"Null until the carrier has scanned the parcel."` tells a
client developer *why* `tracking` can be null, which a type signature can't.

## Nullability, with evidence

Lesson [08 of Level 1](../level-1/08-errors.md) showed how `!` affects error blast radius. The
other half of the decision is evolution: which changes can you make later without breaking
clients? Here's a matrix run through `graphql-js`'s own checkers:

```js title="evolve.mjs"
import { buildSchema, findBreakingChanges, findDangerousChanges } from "graphql";

function check(label, before, after) {
  const a = buildSchema(before), b = buildSchema(after);
  const breaking = findBreakingChanges(a, b).map((c) => c.type);
  const dangerous = findDangerousChanges(a, b).map((c) => c.type);
  console.log(`${label.padEnd(44)} breaking=${JSON.stringify(breaking)} dangerous=${JSON.stringify(dangerous)}`);
}

const q = (field, arg = "") => `type Query { f${arg}: ${field} } input In { x: Int y: Int } enum Status { OPEN CLOSED }`;

check("output: String  -> String!", q("String"), q("String!"));
check("output: String! -> String", q("String!"), q("String"));
check("arg:    Int     -> Int!", q("Int", "(a: Int)"), q("Int", "(a: Int!)"));
check("arg:    Int!    -> Int", q("Int", "(a: Int!)"), q("Int", "(a: Int)"));
check("add optional arg", q("Int"), q("Int", "(a: Int)"));
check("add required arg", q("Int"), q("Int", "(a: Int!)"));
check("add required arg with default", q("Int"), q("Int", "(a: Int! = 1)"));
check("add enum value (output)", q("Status"), q("Status").replace("CLOSED", "CLOSED ARCHIVED"));
check("output: [String] -> [String!]!", q("[String]"), q("[String!]!"));
check("change arg default", q("Int", "(a: Int = 1)"), q("Int", "(a: Int = 2)"));
check("add required input field", q("Int", "(i: In)"), q("Int", "(i: In)").replace("y: Int }", "y: Int z: Int! }"));
check("output: Int -> Float", q("Int"), q("Float"));
```

```console
$ node evolve.mjs
output: String  -> String!                   breaking=[] dangerous=[]
output: String! -> String                    breaking=["FIELD_CHANGED_KIND"] dangerous=[]
arg:    Int     -> Int!                      breaking=["ARG_CHANGED_KIND"] dangerous=[]
arg:    Int!    -> Int                       breaking=[] dangerous=[]
add optional arg                             breaking=[] dangerous=["OPTIONAL_ARG_ADDED"]
add required arg                             breaking=["REQUIRED_ARG_ADDED"] dangerous=[]
add required arg with default                breaking=[] dangerous=["OPTIONAL_ARG_ADDED"]
add enum value (output)                      breaking=[] dangerous=["VALUE_ADDED_TO_ENUM"]
output: [String] -> [String!]!               breaking=[] dangerous=[]
change arg default                           breaking=[] dangerous=["ARG_DEFAULT_VALUE_CHANGE"]
add required input field                     breaking=["REQUIRED_INPUT_FIELD_ADDED"] dangerous=[]
output: Int -> Float                         breaking=["FIELD_CHANGED_KIND"] dangerous=[]
```

The pattern is a mirror image:

- **Outputs** may become *stricter* (nullable → non-null) safely, because a client that handled
  `null` still works when it never arrives. Loosening (non-null → nullable) breaks clients that
  assumed a value.
- **Inputs** may become *looser* (required → optional) safely. Tightening breaks callers that
  omitted the value.

So for output fields, **nullable is the reversible choice**. Start nullable where you're
unsure; you can add `!` later. You can't take it away. For arguments, start required where
you're unsure; you can relax later.

"Dangerous" means valid clients won't fail validation, but behaviour might change: a client
`switch` on `Status` with no default branch meets `ARCHIVED` and does who-knows-what. Tell
client teams to always handle unknown enum values.

## How It Actually Works

`findBreakingChanges` compares the two schemas type by type. For each field present in both,
it calls `isChangeSafeForObjectOrInterfaceField(oldType, newType)`, which unwraps the
types recursively: a change is safe if the new type is the same named type, or if the new
type is a non-null wrapper around something safe (adding `!` to an output), or if both are
lists whose item types are safe. Arguments and input fields use the inverse function,
`isChangeSafeForInputObjectFieldOrFieldArg`, which allows *removing* a non-null wrapper but
not adding one. Removed types, fields, enum values and union members are reported as
breaking outright. That's the whole algorithm — small enough to read in an afternoon in
`utilities/findBreakingChanges.js`.

What it can't see is *semantics*: changing what a field means (prices suddenly excluding tax)
breaks clients without changing a single type. That's what descriptions and deprecations are
for.

## Common mistakes

- **Mirroring the database.** The schema becomes a remote SQL interface, and every storage
  change becomes an API change.
- **Non-null everywhere** for "stronger types". It's the one direction you can't undo, and
  it maximises error blast radius.
- **`userId` arguments for "my" data.** Use `viewer` (or `me`) and get identity from context.
- **Floats for money**, and timestamps as ambiguous strings without a documented format.
- **Booleans that later need a third state** (`isActive` → active/suspended/banned). If a
  state machine is plausible, start with an enum.
- **Generic names** like `data`, `info`, `items` on many types; they make queries unreadable
  and generated types confusing.

## Exercise

1. Take a screen from an app you use (a playlist, a GitHub pull request page) and write its
   query first, then the schema that serves it.
2. Run the matrix for `[String!]! → [String]!` and `[String]! → [String!]!` and explain both
   results.
3. Redesign `Money` so that totals in a multi-currency basket can't be accidentally added
   together on the client. What does the schema have to say about currency?
4. Find three things in `db-shaped.graphql` that would be breaking to fix once published.
