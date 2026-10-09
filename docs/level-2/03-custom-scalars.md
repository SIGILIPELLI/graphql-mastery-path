---
description: "Write a DateTime scalar from scratch with serialize, parseValue and parseLiteral, see each path fail, then compare with the graphql-scalars library."
---

# 03 · Custom Scalars

The five built-in scalars cover strings, numbers, booleans and ids. Everything else —
timestamps, email addresses, URLs, decimals, JSON blobs — either gets squeezed into `String`
or gets a **custom scalar**. A custom scalar is three small functions that decide how a value
crosses the boundary between your server and the wire, in both directions. Write them
carelessly and invalid data slips through; write them well and every resolver can trust its
inputs. This lesson builds a `DateTime` scalar by hand, exercises all three functions, finds a
bug in it, and compares it with a maintained library.

## Three functions, three directions

| Function | Called when | Converts |
|---|---|---|
| `serialize(value)` | a resolver returned a value for a field of this type | internal value → JSON |
| `parseValue(value)` | a client sent this type as a **variable** | JSON → internal value |
| `parseLiteral(ast)` | a client wrote this type **inline** in the query text | AST node → internal value |

Inputs arrive two ways because a value in the query text (`after: "2026-03-10T00:00:00Z"`)
is part of the parsed document, an AST node with a `kind`, whereas a variable arrives as
already-parsed JSON. Both must produce the same internal value.

## A DateTime scalar

```js title="scalars03.mjs"
import { makeExecutableSchema } from "@graphql-tools/schema";
import { graphql, GraphQLScalarType, GraphQLError, Kind } from "graphql";

const ISO = /^\d{4}-\d{2}-\d{2}T\d{2}:\d{2}:\d{2}(\.\d+)?(Z|[+-]\d{2}:\d{2})$/;

function toDate(value, where) {
  if (typeof value !== "string" || !ISO.test(value) || Number.isNaN(Date.parse(value))) {
    throw new Error(`DateTime ${where} must be an ISO-8601 string with a timezone, got: ${JSON.stringify(value)}`);
  }
  return new Date(value);
}

const DateTime = new GraphQLScalarType({
  name: "DateTime",
  description: "An instant in time, serialised as an ISO-8601 string in UTC.",
  // internal value -> JSON (output)
  serialize(value) {
    if (value instanceof Date && !Number.isNaN(value.getTime())) return value.toISOString();
    if (typeof value === "number") return new Date(value).toISOString(); // epoch millis from a DB
    throw new GraphQLError(`DateTime cannot represent value: ${JSON.stringify(value)}`);
  },
  // JSON variable -> internal value (input via variables)
  parseValue(value) {
    return toDate(value, "variable");
  },
  // AST literal in the query text -> internal value (input via literals)
  parseLiteral(ast) {
    if (ast.kind !== Kind.STRING) throw new Error(`DateTime literal must be a string, got ${ast.kind}`);
    return toDate(ast.value, "literal");
  },
});

const events = [
  { id: "1", name: "Launch", at: new Date("2026-03-01T09:00:00Z") },
  { id: "2", name: "Retro", at: Date.parse("2026-03-15T16:30:00Z") }, // stored as epoch millis
  { id: "3", name: "Broken", at: "next tuesday" },
];

const schema = makeExecutableSchema({
  typeDefs: /* GraphQL */ `
    scalar DateTime
    type Event { id: ID! name: String! at: DateTime }
    type Query { events(after: DateTime): [Event!]! }
  `,
  resolvers: {
    DateTime,
    Query: {
      events: (_, { after }) => {
        if (after) console.log("   resolver got:", after instanceof Date ? `Date(${after.toISOString()})` : typeof after);
        return events.filter((e) => !after || new Date(e.at) > after);
      },
    },
  },
});

const run = async (label, source, variableValues) => {
  const r = await graphql({ schema, source, variableValues });
  console.log(`--- ${label}\n${JSON.stringify(r)}`);
};
```

In SDL, a custom scalar is just `scalar DateTime`. The behaviour comes from the
`GraphQLScalarType` object, which `makeExecutableSchema` attaches when you put it in the
resolver map under the same name.

The internal representation is a JavaScript `Date`. The wire representation is an ISO-8601
string, always normalised to UTC. Resolvers never see strings; clients never see `Date`
objects.

## Output: `serialize`

```js
await run("output: Date, epoch millis, and junk", `{ events { name at } }`);
```

```text
--- output: Date, epoch millis, and junk
{"errors":[{"message":"DateTime cannot represent value: \"next tuesday\"","locations":[{"line":1,"column":17}],"path":["events",2,"at"]}],"data":{"events":[{"name":"Launch","at":"2026-03-01T09:00:00.000Z"},{"name":"Retro","at":"2026-03-15T16:30:00.000Z"},{"name":"Broken","at":null}]}}
```

`serialize` accepted both representations our "database" uses (a `Date` and epoch
milliseconds) and produced identical formats. The corrupt row became a field error on exactly
one field, because `at` is nullable. A scalar that passed `"next tuesday"` through would have
handed every client a string they can't parse — failing loudly here is the point.

## Input: literals and variables

```js
await run("input literal", `{ events(after: "2026-03-10T00:00:00+05:30") { name } }`);
await run("input variable", `query($t: DateTime) { events(after: $t) { name } }`, { t: "2026-03-02T00:00:00Z" });
```

```text
   resolver got: Date(2026-03-09T18:30:00.000Z)
--- input literal
{"data":{"events":[{"name":"Retro"}]}}
   resolver got: Date(2026-03-02T00:00:00.000Z)
--- input variable
{"data":{"events":[{"name":"Retro"}]}}
```

Both paths gave the resolver a real `Date`. The literal had a `+05:30` offset, and it arrived
as the correct UTC instant. (The `Broken` row is silently filtered out here because
`new Date("next tuesday")` is an invalid date and any comparison with it is `false`.)

## Rejecting bad input

```js
await run("bad literal (no timezone)", `{ events(after: "2026-03-10T00:00:00") { name } }`);
await run("bad literal (number)", `{ events(after: 1700000000) { name } }`);
await run("bad variable", `query($t: DateTime) { events(after: $t) { name } }`, { t: "yesterday" });
```

```text
--- bad literal (no timezone)
{"errors":[{"message":"Expected value of type \"DateTime\", found \"2026-03-10T00:00:00\"; DateTime literal must be an ISO-8601 string with a timezone, got: \"2026-03-10T00:00:00\"","locations":[{"line":1,"column":17}]}]}
--- bad literal (number)
{"errors":[{"message":"Expected value of type \"DateTime\", found 1700000000; DateTime literal must be a string, got IntValue","locations":[{"line":1,"column":17}]}]}
--- bad variable
{"errors":[{"message":"Variable \"$t\" got invalid value \"yesterday\"; Expected type \"DateTime\". DateTime variable must be an ISO-8601 string with a timezone, got: \"yesterday\"","locations":[{"line":1,"column":7}]}]}
```

None of these reached the resolver; there's no `data` key. Rejecting timestamps **without** a
timezone is deliberate: `"2026-03-10T00:00:00"` means a different instant in Delhi and in
Denver, and JavaScript would silently interpret it in the *server's* local time zone.

### A detail about the error type

Notice the input functions throw a plain `Error`, but `serialize` throws `GraphQLError`. That
choice was made after a first version threw `GraphQLError` everywhere and got this for the
no-timezone literal:

```text
{"errors":[{"message":"DateTime literal must be an ISO-8601 string with a timezone, got: \"2026-03-10T00:00:00\""}]}
```

No `locations`, and no "Expected value of type" prefix. When validation catches an error from
`parseLiteral`, it passes a `GraphQLError` through as-is (assuming you built it with the
details you wanted) but wraps any other error with the type name and the location of the
literal. For input functions, throwing a plain `Error` gives clients a better message.

## The bug: February 30th

```js
await run("Feb 30 sneaks through", `{ events(after: "2026-02-30T09:00:00Z") { name } }`);
```

```text
   resolver got: Date(2026-03-02T09:00:00.000Z)
--- Feb 30 sneaks through
{"data":{"events":[{"name":"Retro"}]}}
```

The regex checks the *format*, and `Date.parse` doesn't reject impossible days — it rolls
February 30th over to March 2nd. A client with a date-picker bug would get results for a date
it never asked for. Writing a correct date validator is fiddly (month lengths, leap years,
leap seconds, offset ranges), which is the best argument for the next section.

## Using `graphql-scalars`

The [`graphql-scalars`](https://the-guild.dev/graphql/scalars) package (2.0.0 here) provides
dozens of tested scalars. Each export is a `GraphQLScalarType` you plug into the resolver map:

```js title="gqlscalars.mjs"
import { makeExecutableSchema } from "@graphql-tools/schema";
import { graphql } from "graphql";
import { EmailAddressResolver, PositiveIntResolver, URLResolver, DateTimeResolver } from "graphql-scalars";

const schema = makeExecutableSchema({
  typeDefs: `scalar EmailAddress scalar PositiveInt scalar URL scalar DateTime
    type Query { check(email: EmailAddress, qty: PositiveInt, site: URL, at: DateTime): String }`,
  resolvers: {
    EmailAddress: EmailAddressResolver, PositiveInt: PositiveIntResolver, URL: URLResolver, DateTime: DateTimeResolver,
    Query: { check: (_, args) => JSON.stringify(Object.fromEntries(Object.entries(args).map(([k, v]) => [k, v?.constructor?.name + ":" + (v instanceof Date ? v.toISOString() : String(v))]))) },
  },
});
for (const src of [
  `{ check(email: "ada@example.com", qty: 3, site: "https://example.com/a", at: "2026-03-01T09:00:00Z") }`,
  `{ check(email: "not-an-email") }`,
  `{ check(qty: 0) }`,
  `{ check(at: "2026-02-30T09:00:00Z") }`,
]) console.log(JSON.stringify(await graphql({ schema, source: src })));
```

```text
{"data":{"check":"{\"email\":\"String:ada@example.com\",\"qty\":\"Number:3\",\"site\":\"URL:https://example.com/a\",\"at\":\"Date:2026-03-01T09:00:00.000Z\"}"}}
{"errors":[{"message":"Value is not a valid email address: not-an-email","locations":[{"line":1,"column":16}]}]}
{"errors":[{"message":"Value is not a positive number: 0"}]}
{"errors":[{"message":"DateTime cannot represent an invalid date-time-string 2026-02-30T09:00:00Z.","locations":[{"line":1,"column":13}]}]}
```

Its `DateTime` rejects February 30th. `URL` parses into a real `URL` object and `DateTime`
into a `Date`, while `EmailAddress` stays a string. Note that `PositiveInt`'s error has no
`locations` — the same `GraphQLError` pass-through effect described above. Library scalars
aren't perfect, but they've met far more edge cases than a scalar written this afternoon.

## When to create a scalar — and when not to

Good candidates: values with a strict, well-known wire format that every client must parse the
same way (timestamps, dates, UUIDs, URLs, decimals as strings, email addresses).

Poor candidates:

- **Structured data.** If a "scalar" has internal fields (`Money`, `Address`), make it an
  object type so clients can select parts of it and the schema documents them.
- **A `JSON` scalar to avoid designing types.** It works, but the client gets no
  validation, no autocomplete and no field-level evolution — it opts that part of the API out
  of GraphQL.
- **Business rules** like "a username must be unique" — that's resolver logic, not format.

Also remember: a custom scalar is opaque to clients. Introspection tells them only the name and
description. Use the description (and, from the October 2021 spec on, the `@specifiedBy(url:)`
directive) to say exactly what format to send.

## How It Actually Works

`serialize` is called in `completeLeafValue` during execution: the executor passes the
resolver's return value to `returnType.serialize`, and a thrown error becomes a field error at
that path. A `null`/`undefined` return never reaches `serialize` — nulls are handled before.

`parseLiteral` runs twice for literal arguments. First during **validation**:
`ValuesOfCorrectTypeRule` calls it to check the literal is acceptable, which is why bad
literals are request errors with no `data`. Then during **execution**, `valueFromAST` calls it
again to produce the value passed to your resolver. (Keep `parseLiteral` free of side
effects for that reason.) If a literal contains variables — say inside an input object —
`parseLiteral` receives the variables map as its second argument.

`parseValue` runs in `coerceVariableValues` before execution starts; errors there are also
request errors. If you don't provide `parseLiteral`, `graphql-js` defaults to converting the
AST to a plain JS value with `valueFromASTUntyped` and passing it to `parseValue`, so a single
`parseValue` can cover both paths if you don't need AST-specific checks.

## Common mistakes

- **Implementing `serialize` only.** The scalar works in responses and silently accepts
  anything as input (the default `parseValue` is the identity function).
- **Different behaviour between `parseValue` and `parseLiteral`**, so a query works inline but
  fails with variables, or vice versa.
- **Accepting timezone-less timestamps**, then interpreting them in server-local time.
- **Format checks without semantic checks** — the February 30th bug.
- **Returning strings from resolvers** for a `DateTime` field and doing the formatting there; let
  `serialize` own the wire format so it's consistent everywhere.

## Exercise

1. Fix the February 30th bug in the hand-written scalar: after parsing, check that
   `toISOString()` of the result starts with the same calendar date as the input (careful with
   offsets — normalise first).
2. Write a `Date` scalar (calendar date, no time, `YYYY-MM-DD`) whose internal value is a string.
   Why might a string be a better internal value than a JavaScript `Date` here?
3. Write a `Cursor` scalar that base64-encodes an internal `{ id }` object on output and decodes
   it on input. You'll reuse it in [lesson 07](07-pagination.md).
4. Add `@specifiedBy(url: "https://scalars.graphql.org/andimarek/date-time")` to `scalar DateTime`
   and find where it shows up in introspection.
