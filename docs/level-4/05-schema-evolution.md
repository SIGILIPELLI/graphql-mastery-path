---
description: "Evolve a live GraphQL schema safely: expand–contract, deprecation, per-client field usage, and checking real operations before removal."
---

# 05 · Schema Evolution & Breaking-Change Detection

GraphQL APIs are famously "versionless": instead of `/v2`, one schema evolves continuously.
That only works if you can answer, before every change, *will this break someone?* — and for
removals, *is anyone still using this?* [Level 2 · 01](../level-2/01-schema-design.md) showed
which changes are breaking in principle. This lesson is about operating that at scale:
replacing a field in stages, measuring who still uses the old one, and validating real client
operations against the proposed schema.

## The expand–contract pattern

Every breaking change can be split into non-breaking steps:

1. **Expand** — add the new field alongside the old one. Nothing breaks.
2. **Migrate** — deprecate the old field, update clients, watch usage fall.
3. **Contract** — remove the old field once usage reaches zero (or an agreed cut-off).

Here the change is replacing `priceCents: Int!` with a `price: Money!` object (the design from
Level 2 · 01). Steps 1 and 2 are the "v2" schema:

```js title="evolution05.mjs"
import { ApolloServer, HeaderMap } from "@apollo/server";
import { TypeInfo, visit, visitWithTypeInfo, getNamedType, buildSchema, parse, validate, findBreakingChanges, findDangerousChanges } from "graphql";

// v2 of the schema: priceCents is deprecated in favour of price { amount currency }
const typeDefsV2 = /* GraphQL */ `
  type Query { products: [Product!]! }
  type Product {
    id: ID!
    name: String!
    priceCents: Int! @deprecated(reason: "Use price { amount currency }. Removal planned after all clients migrate.")
    price: Money!
  }
  type Money { amount: String!  currency: String! }
`;
const products = [{ id: "1", name: "Lamp", cents: 2499 }];
const resolvers = {
  Query: { products: () => products },
  Product: {
    priceCents: (p) => p.cents,
    price: (p) => ({ amount: (p.cents / 100).toFixed(2), currency: "USD" }),
  },
};

// Field-usage tracking: which client uses which Type.field
const usage = new Map(); // "Type.field" -> Map(client -> count)
const usagePlugin = {
  async requestDidStart({ request }) {
    const client = request.http?.headers.get("apollographql-client-name") ?? "unknown";
    return {
      async didResolveOperation({ document, schema }) {
        const typeInfo = new TypeInfo(schema);
        visit(document, visitWithTypeInfo(typeInfo, {
          Field() {
            const parent = typeInfo.getParentType();
            const field = typeInfo.getFieldDef();
            if (!parent || !field || field.name.startsWith("__")) return;
            const key = `${parent.name}.${field.name}`;
            const perClient = usage.get(key) ?? new Map();
            perClient.set(client, (perClient.get(client) ?? 0) + 1);
            usage.set(key, perClient);
          },
        }));
      },
    };
  },
};

const server = new ApolloServer({ typeDefs: typeDefsV2, resolvers, plugins: [usagePlugin] });
await server.start();

const send = (client, query) => server.executeOperation(
  { query, http: { method: "POST", headers: new HeaderMap([["apollographql-client-name", client]]), search: "", body: {} } });

// Simulated traffic: an old mobile build, the migrated web app, a partner script
for (let i = 0; i < 3; i++) await send("ios-3.2", "{ products { name priceCents } }");
for (let i = 0; i < 5; i++) await send("web", "{ products { name price { amount currency } } }");
await send("partner-feed", "{ products { id priceCents price { currency } } }");

console.log("usage of deprecated and replacement fields:");
for (const key of ["Product.priceCents", "Product.price", "Money.amount"])
  console.log(`  ${key.padEnd(20)} ${JSON.stringify(Object.fromEntries(usage.get(key) ?? []))}`);

// Deprecation is visible to tools via introspection
const intro = await server.executeOperation({ query: `{ __type(name: "Product") { fields(includeDeprecated: true) { name isDeprecated deprecationReason } } }` });
console.log("\nintrospection:", JSON.stringify(intro.body.singleResult.data.__type.fields.filter((f) => f.isDeprecated)));

// v3 proposal: remove priceCents
const typeDefsV3 = typeDefsV2.replace(/\s*priceCents: Int! @deprecated\([^)]*\)/, "");
const v2 = buildSchema(typeDefsV2), v3 = buildSchema(typeDefsV3);
console.log("\nv2 -> v3 breaking:", findBreakingChanges(v2, v3).map((c) => c.description));

// Check the real operations clients have sent against v3
const known = { "ios-3.2": "{ products { name priceCents } }", web: "{ products { name price { amount currency } } }", "partner-feed": "{ products { id priceCents price { currency } } }" };
console.log("client operations against v3:");
for (const [client, q] of Object.entries(known)) {
  const errs = validate(v3, parse(q));
  console.log(`  ${client.padEnd(13)} ${errs.length ? "BREAKS: " + errs[0].message : "ok"}`);
}

// Adding an enum value or a union member is "dangerous", not breaking
const e1 = buildSchema(`type Query { s: Status } enum Status { OPEN CLOSED }`);
const e2 = buildSchema(`type Query { s: Status } enum Status { OPEN CLOSED ARCHIVED }`);
console.log("\nadd enum value -> dangerous:", findDangerousChanges(e1, e2).map((c) => c.description));
await server.stop();
```

## Deprecation is a signal, not a mechanism

```text
introspection: [{"name":"priceCents","isDeprecated":true,"deprecationReason":"Use price { amount currency }. Removal planned after all clients migrate."}]
```

`@deprecated` changes nothing at runtime — `priceCents` still resolves. It tells tools: IDEs
strike the field through, code generators mark it `@deprecated` in TypeScript, linters can warn.
Write the reason as an instruction ("use X instead"), and since the October 2021 spec edition you
can also deprecate arguments and input fields.

## Who still uses it?

The `usagePlugin` records, for each operation, every `Type.field` it selects — using `TypeInfo`
so it knows each field's parent type even inside fragments — keyed by the client's name from the
`apollographql-client-name` header (a convention Apollo's clients send; any header you control
works). After some simulated traffic:

```text
usage of deprecated and replacement fields:
  Product.priceCents   {"ios-3.2":3,"partner-feed":1}
  Product.price        {"web":5,"partner-feed":1}
  Money.amount         {"web":5}
```

Now the removal decision has names attached: the web app has migrated; an old iOS build and a
partner's script still read `priceCents`. The partner is mid-migration — it reads both. Usage is
tracked at **validation time** (`didResolveOperation`), so it counts fields *requested*, which is
what matters for breakage — even a field that errored or was null was depended on.

In production this data comes from your tracing pipeline or a schema registry (Apollo GraphOS
records field usage per client from its usage reporting; that wasn't run here). The principle is
identical: collect per-field, per-client usage over a long enough window to include rarely used
clients — monthly jobs, old app versions.

## Contract: check before you remove

```text
v2 -> v3 breaking: [
  'Standard scalar Int was removed because it is not referenced anymore.',
  'Product.priceCents was removed.'
]
```

`findBreakingChanges` flags the removal — and a surprise: `priceCents` was the schema's only `Int`
field, so the built-in `Int` scalar disappeared from the schema too. Built-in scalars only appear in
a schema when referenced. It doesn't matter here, but a client that declared a variable `$n: Int`
in some unrelated operation would now fail validation. The tool's report is worth reading in full.

Static diffing says *something* could break. Validating **real operations** says *what* breaks:

```text
client operations against v3:
  ios-3.2       BREAKS: Cannot query field "priceCents" on type "Product". Did you mean "price"?
  web           ok
  partner-feed  BREAKS: Cannot query field "priceCents" on type "Product". Did you mean "price"?
```

If you collect clients' operations (from trusted-document manifests, [Level 3 · 06](../level-3/06-caching-persisted-queries.md),
or from traffic), this check turns "is removing this field safe?" into a CI job with a precise
answer. Schema registries automate exactly this ("operation checks" against recent traffic).

## Dangerous changes

```text
add enum value -> dangerous: [ 'ARCHIVED was added to enum type Status.' ]
```

Additions that are valid for every existing query can still break client *code*: a `switch` with
no default branch, an exhaustive TypeScript check, a union member the UI can't render. Report
these to client teams; don't block on them.

## A practical policy

- Every schema change runs breaking/dangerous checks in CI against the production schema.
- Breaking changes require a recorded usage check (zero usage for N days, or named sign-off).
- Deprecations carry a reason and, where it helps, a target date in the reason text.
- Mobile apps get the longest windows — old versions live for months or years.
- When usage can't reach zero (an abandoned partner integration), make an explicit decision and
  communicate it; don't let a deprecated field linger forever by default.

## How It Actually Works

`TypeInfo` is a stack machine: `visitWithTypeInfo` calls `typeInfo.enter(node)` before your
visitor and `typeInfo.leave(node)` after it. On entering a `Field`, it looks up the field on the
current parent type and pushes the field definition and its return type; on entering a selection
set, it pushes the named type of the current field as the new parent type; inline fragments and
fragment definitions push their type condition. So inside your `Field` handler,
`getParentType()` and `getFieldDef()` are correct even for fields deep inside fragments — the
same mechanism the validation rules use.

`findBreakingChanges` compares the two schemas' type maps: removed types (including built-in
scalars that are no longer referenced, hence the `Int` message), changed kinds, and for object,
interface and input types, field-by-field comparisons with the variance rules from Level 2 · 01.
`findDangerousChanges` runs the same walk looking for additions that existing operations can't
reject: new enum values, new union members, new implementations of interfaces, changed argument
defaults.

## Common mistakes

- **Removing a deprecated field on a schedule** without checking usage.
- **Measuring usage from resolver calls**, which miss fields that were requested but errored or
  short-circuited.
- **Short usage windows** that miss monthly jobs and old mobile versions.
- **Renaming in place** (`priceCents` → `price`) instead of expand–contract.
- **Not identifying clients**, which turns "who uses this?" into guesswork.

## Exercise

1. Extend the usage plugin to record **argument** usage too (`Query.products(first:)`), so you can
   tell whether an argument can be removed or made required.
2. Store usage with timestamps and write a report: "fields with zero usage in the last 30 days".
3. Turn the operation check into a CLI: given `schema.graphql` and a folder of `.graphql` operations
   per client, exit non-zero and name the clients that would break.
4. Deprecate an argument (`products(sort: String @deprecated(reason: "..."))`) and check how it appears
   in introspection with `args(includeDeprecated: true)`.
