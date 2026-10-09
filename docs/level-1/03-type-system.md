---
description: "GraphQL's type system in practice: the five built-in scalars, object types, enums, lists and non-null, and how values are coerced on input and output."
---

# 03 · The Type System: Scalars, Objects, Lists, Non-Null & Enums

Everything a GraphQL server can return has a declared type, and every value is checked
against that type at runtime — on the way **in** (arguments and variables) and on the way
**out** (resolver results). Knowing exactly how strict each direction is saves a lot of
confusion later, so this lesson tests the rules rather than just listing them.

## The kinds of type

| Kind | SDL keyword | What it is |
|------|-------------|------------|
| Scalar | `scalar` (5 built in) | A leaf value: `Int`, `Float`, `String`, `Boolean`, `ID` |
| Object | `type` | A named set of fields, each with its own type |
| Enum | `enum` | A leaf with a fixed set of allowed names |
| Input object | `input` | A set of fields used only as argument values ([lesson 06](06-mutations-inputs.md)) |
| Interface | `interface` | A set of fields that several object types share ([Level 2 · 02](../level-2/02-interfaces-unions.md)) |
| Union | `union` | "One of these object types" ([Level 2 · 02](../level-2/02-interfaces-unions.md)) |

On top of those, two **wrapping types** modify any type: `[T]` (a list of T) and `T!`
(non-null T).

Selection rules follow from the kinds. **Leaf** types — scalars and enums — must *not* have
a sub-selection: `{ title { x } }` is invalid if `title` is a `String`. **Composite** types —
objects, interfaces, unions — *must* have one: `{ book }` on its own is invalid, because
GraphQL never returns "the whole object". You always say which fields.

## The five built-in scalars

| Scalar | Meaning | Serialized as |
|--------|---------|---------------|
| `Int` | Signed 32-bit integer (−2,147,483,648 to 2,147,483,647) | JSON number |
| `Float` | Double-precision floating point | JSON number |
| `String` | UTF-8 text | JSON string |
| `Boolean` | `true` / `false` | JSON boolean |
| `ID` | An opaque identifier; may be a string or integer on input | JSON string |

Two of these trip people up. `Int` is **32-bit**, so a database `bigint` or a millisecond
timestamp does not fit — use a `String`, a `Float` or a custom scalar
([Level 2 · 03](../level-2/03-custom-scalars.md)). `ID` signals "don't do arithmetic on this,
it's an identifier" and always comes back as a string even if your resolver returned a number.

## Testing the coercion rules

The script below gives each scalar resolver deliberately "wrong" values and sends arguments
of the wrong type. Install `@graphql-tools/schema` if you skipped it in lesson 02.

```js title="types.mjs"
import { graphql } from "graphql";
import { makeExecutableSchema } from "@graphql-tools/schema";

const typeDefs = `
  enum Format { HARDCOVER PAPERBACK EBOOK }
  type Query {
    intOut(v: String!): Int
    floatOut: Float
    strOut: String
    boolOut: Boolean
    idOut: ID
    format: Format
    badFormat: Format
    a: [String]
    b: [String!]
    c: [String]!
    d: [String!]!
    echoInt(n: Int!): Int
    echoFormat(f: Format!): String
  }
`;
const withNull = ["x", null, "z"];
const resolvers = {
  Format: { HARDCOVER: "hc", PAPERBACK: "pb", EBOOK: "eb" },
  Query: {
    intOut: (_, { v }) =>
      v === "num" ? 12 : v === "str" ? "12" : v === "float" ? 12.5
        : v === "big" ? 2 ** 31 : v === "bool" ? true : "abc",
    floatOut: () => "3.14",
    strOut: () => 42,
    boolOut: () => 0,
    idOut: () => 7,
    format: () => "pb",
    badFormat: () => "PAPERBACK",
    a: () => withNull, b: () => withNull, c: () => withNull, d: () => withNull,
    echoInt: (_, { n }) => n,
    echoFormat: (_, { f }) => `internal value: ${JSON.stringify(f)}`,
  },
};
const schema = makeExecutableSchema({ typeDefs, resolvers });

const run = async (source, variableValues) => {
  const r = await graphql({ schema, source, variableValues });
  console.log(source.trim(), variableValues ? JSON.stringify(variableValues) : "");
  console.log("  ", JSON.stringify(r.data),
    r.errors ? r.errors.map((e) => `${e.path ?? ""}: ${e.message}`) : "");
};

for (const v of ["num", "str", "float", "big", "bool", "abc"]) await run(`{ intOut(v: "${v}") }`);
await run(`{ floatOut strOut idOut }`);
await run(`{ boolOut }`);
await run(`{ format }`);
await run(`{ badFormat }`);
await run(`{ a }`); await run(`{ b }`); await run(`{ c }`); await run(`{ d }`);
await run(`{ echoInt(n: 3000000000) }`);
await run(`{ echoInt(n: "5") }`);
await run(`query($n: Int!) { echoInt(n: $n) }`, { n: "5" });
await run(`query($n: Int!) { echoInt(n: $n) }`, { n: 5.0 });
await run(`{ echoFormat(f: EBOOK) }`);
await run(`{ echoFormat(f: "EBOOK") }`);
```

### Output coercion: lenient where it's safe

```text
{ intOut(v: "num") }      {"intOut":12}
{ intOut(v: "str") }      {"intOut":12}
{ intOut(v: "float") }    {"intOut":null}  intOut: Int cannot represent non-integer value: 12.5
{ intOut(v: "big") }      {"intOut":null}  intOut: Int cannot represent non 32-bit signed integer value: 2147483648
{ intOut(v: "bool") }     {"intOut":1}
{ intOut(v: "abc") }      {"intOut":null}  intOut: Int cannot represent non-integer value: "abc"
{ floatOut strOut idOut } {"floatOut":3.14,"strOut":"42","idOut":"7"}
{ boolOut }               {"boolOut":false}
```

(Output reformatted onto one line per query; the values and messages are exactly as
printed by `graphql` 16.14.2.)

When serializing a resolver's result, `graphql-js` converts values that have an unambiguous
meaning in the target type: the string `"12"` becomes the integer `12`, `true` becomes `1`,
the number `42` becomes the string `"42"`, `0` becomes `false`. It refuses anything that
would lose information — `12.5` as an `Int`, a value outside 32 bits, `"abc"` as a number —
and reports a **field error**: that field becomes `null` and an error is added, but the rest
of the response survives.

### Enums: the name is public, the value is internal

```text
{ format }                {"format":"PAPERBACK"}
{ badFormat }             {"badFormat":null}  badFormat: Enum "Format" cannot represent value: "PAPERBACK"
{ echoFormat(f: EBOOK) }  {"echoFormat":"internal value: \"eb\""}
```

The `Format` resolver map says the public name `PAPERBACK` corresponds to the internal value
`"pb"`. So a resolver must return `"pb"`, and returning the *name* `"PAPERBACK"` is an error.
In the other direction, the argument `EBOOK` arrives in your resolver as `"eb"`. This mapping
is handy when your database stores short codes. Without an enum resolver map (the usual
case) the internal value is simply the name itself.

### Lists and non-null: where the `!` goes matters

```text
{ a }   {"a":["x",null,"z"]}
{ b }   {"b":null}   b,1: Cannot return null for non-nullable field Query.b.
{ c }   {"c":["x",null,"z"]}
{ d }   null         d,1: Cannot return null for non-nullable field Query.d.
```

All four resolvers returned `["x", null, "z"]`:

| SDL | List may be null? | Items may be null? | Result with a null item |
|-----|-------------------|--------------------|-------------------------|
| `[String]` | yes | yes | list returned as is |
| `[String!]` | yes | no | item 1 is invalid → the **whole list** becomes null |
| `[String]!` | no | yes | list returned as is |
| `[String!]!` | no | no | item invalid → list can't be null either → **`data` itself is null** |

The error's `path` (`b,1`, printed from the array `["b", 1]`) points at the exact item that
broke the rule. In the last case the null bubbled past `d` to the root, and since `Query`
fields are the top of the response, `data` became `null`. [Lesson 08](08-errors.md) covers
this "null propagation" in depth — it's the single most important reason to think hard
before adding `!`.

### Input coercion: strict

```text
{ echoInt(n: 3000000000) }                    Int cannot represent non 32-bit signed integer value: 3000000000
{ echoInt(n: "5") }                           Int cannot represent non-integer value: "5"
query($n: Int!) { echoInt(n: $n) } {"n":"5"}  Variable "$n" got invalid value "5"; Int cannot represent non-integer value: "5"
query($n: Int!) { echoInt(n: $n) } {"n":5}    {"echoInt":5}
{ echoFormat(f: "EBOOK") }                    Enum "Format" cannot represent non-enum value: "EBOOK". Did you mean the enum value "EBOOK"?
```

Inputs are not converted the way outputs are. A string `"5"` is never accepted as an `Int`,
and an enum value must be written as a bare name (`EBOOK`), not a string. Inline literals
are rejected during **validation**; variable values are rejected during **variable
coercion**, just before execution. Either way, no resolver runs and there's no `data`. The
`5.0` variable passed because JSON has no separate integer type — `5.0` and `5` are the
same JavaScript number.

The asymmetry is deliberate: the server is trusted to know what it means, so mild output
conversion is fine; client input is not trusted, so it must match exactly.

## Object types and fields with arguments

Any field — not just root fields — can take arguments:

```graphql
type Book {
  id: ID!
  title: String!
  "Cover image URL, scaled to the requested width in pixels."
  coverUrl(width: Int = 300): String
  format: Format!
}
```

`width: Int = 300` gives a **default value**: if the client omits the argument, the
resolver receives `300`. The string just before a field is a **description**; it appears in
introspection and in tools like Apollo Sandbox, so it's real documentation, not a comment.
(Comments start with `#` and are discarded by the parser.)

## How It Actually Works

Each scalar is a `GraphQLScalarType` with three functions:

- `serialize(value)` — runs on resolver **output** to produce the JSON value. For `Int` it
  accepts numbers, booleans and numeric strings, checks integrality and the 32-bit range,
  and throws a `GraphQLError` otherwise.
- `parseLiteral(ast)` — runs on **inline argument values** in the query text. For `Int` it
  only accepts an `IntValue` AST node within range; a `StringValue` node is rejected.
- `parseValue(value)` — runs on **variable values** from the JSON `variables` map. For `Int`
  it requires `typeof value === "number"` and an integer.

That's why outputs are lenient and inputs strict: they're simply different functions, and
the built-in input functions are written conservatively. When you define your own scalars in
[Level 2 · 03](../level-2/03-custom-scalars.md) you'll write these three functions yourself.

Enums work the same way: `GraphQLEnumType.serialize` maps internal value → name (failing if
no value matches) and `parseLiteral`/`parseValue` map name → internal value.

Non-null is a wrapper type, `GraphQLNonNull(inner)`. After a field's value is completed,
the executor checks: if the field type is non-null and the value is `null`, it raises
"Cannot return null for non-nullable field", and the null is pushed up to the nearest
nullable ancestor. List completion does the same check for each item when the item type is
non-null.

## Common mistakes

- **Using `Int` for large numbers.** IDs from a `bigint` column, file sizes, prices in minor
  units for large amounts, epoch milliseconds — all can exceed 2³¹−1 and will fail at
  runtime only when the large value shows up.
- **Returning the enum name instead of its internal value** when you've defined an enum
  resolver map. Or the reverse: defining a map, then comparing `args.format === "EBOOK"` in a
  resolver that now receives `"eb"`.
- **Putting `!` everywhere** because "it's never null in the database". A non-null field
  that fails takes its parent down with it.
- **Expecting `"5"` to be accepted as an `Int` argument** because the response converted
  `"12"` happily. Input and output rules are different.
- **Selecting an object without sub-fields** (`{ book }`), or selecting sub-fields of a
  scalar. Both are validation errors.

## Exercise

1. Add a field `big: Int` whose resolver returns `Date.now()`. What does the client see?
   Change the type to `Float` and then to `String` and compare.
2. Predict, then check, what `[[String!]]` does when the resolver returns
   `[["a"], ["b", null]]`, and what `[[String]!]!` does with `[null, ["a"]]`.
3. Add an enum `Status { ACTIVE ARCHIVED }` with internal values `1` and `0`. Write a
   resolver that receives the enum as an argument and returns it; confirm what the resolver
   sees and what the client sees.
4. Add a `coverUrl(width: Int = 300)` field and verify the default reaches your resolver
   when the argument is omitted, and that `coverUrl(width: null)` passes `null` rather than
   the default. (Explicit `null` and "not provided" are different things in GraphQL.)
