---
description: "Look inside graphql-js: lexing, the AST and visitors, the 27 validation rules, measured phase costs, and a 15-line toy executor that mirrors the real one."
---

# 01 · Inside graphql-js: Parse, Validate, Execute

Every server in this course — Apollo Server, Yoga, the federation gateway — hands the real
work to `graphql-js`. Level 4 begins by opening it up. Knowing what each phase does and costs
explains things you've already met (why validation errors come all at once, why non-null
errors climb upward, why persisted documents save time) and is the foundation for the
plugins, tracing and federation that follow. All measurements are from `graphql` 16.14.2 on
Node 26.3, on one laptop.

```js title="internals01.mjs"
import {
  Lexer, Source, TokenKind, parse, print, visit, validate, specifiedRules, execute, buildSchema,
} from "graphql";

// 1. Lexing: the query as tokens
const src = new Source(`query Q($id: ID!) { book(id: $id) { title ...F } } fragment F on Book { year }`);
const lexer = new Lexer(src);
const tokens = [];
for (let t = lexer.advance(); t.kind !== TokenKind.EOF; t = lexer.advance()) tokens.push(t.value ?? t.kind);
console.log(`tokens (${tokens.length}):`, tokens.join(" "));

// 2. Parsing: tokens -> AST, and visiting it
const doc = parse(src);
const kinds = {};
visit(doc, { enter(node) { kinds[node.kind] = (kinds[node.kind] ?? 0) + 1; } });
console.log("AST node kinds:", JSON.stringify(kinds));

// A visitor can also *transform*: rename every field "year" to "publishedYear"
const renamed = visit(doc, { Field(node) { return node.name.value === "year" ? { ...node, name: { ...node.name, value: "publishedYear" } } : undefined; } });
console.log("transformed:", print(renamed).replace(/\s+/g, " "));

// 3. Validation: the rule list
console.log(`\nspecifiedRules: ${specifiedRules.length} rules, e.g.`, specifiedRules.slice(0, 4).map((r) => r.name).join(", "), "...");

// 4. Timing the phases on a big query
const schema = buildSchema(`
  type Query { items(n: Int!): [Item!]! }
  type Item { id: ID! a: Int! b: Int! c: Int! child: Item }
`);
const rootValue = {
  items: ({ n }) => Array.from({ length: n }, (_, i) => ({ id: i, a: i, b: i * 2, c: i * 3, child: { id: -i, a: 0, b: 0, c: 0, child: null } })),
};
const fields = Array.from({ length: 40 }, (_, i) => `f${i}: a`).join(" ");
const bigQuery = `{ items(n: 2000) { id ${fields} b c child { id a b c } } }`;

function time(fn, runs = 30) {
  fn(); // warm up
  const ts = [];
  for (let i = 0; i < runs; i++) { const t = performance.now(); fn(); ts.push(performance.now() - t); }
  ts.sort((x, y) => x - y);
  return ts[Math.floor(runs / 2)];
}
const parsed = parse(bigQuery);
const tParse = time(() => parse(bigQuery));
const tValidate = time(() => validate(schema, parsed));
const tExecute = time(() => execute({ schema, document: parsed, rootValue }), 10);
const result = execute({ schema, document: parsed, rootValue });
const tSerialize = time(() => JSON.stringify(result), 10);
console.log(`\nquery: ${bigQuery.length} chars, result: ${JSON.stringify(result).length} bytes, 2000 items x (44 fields + a 4-field child)`);
console.log(`median ms  parse=${tParse.toFixed(3)}  validate=${tValidate.toFixed(3)}  execute=${tExecute.toFixed(1)}  JSON.stringify=${tSerialize.toFixed(1)}`);

// 5. Execution is just a recursive walk — a toy executor for plain objects
function toyExecute(selectionSet, value) {
  if (value === null || typeof value !== "object") return value;
  if (Array.isArray(value)) return value.map((v) => toyExecute(selectionSet, v));
  const out = {};
  for (const sel of selectionSet.selections) {
    const key = sel.alias?.value ?? sel.name.value;
    const raw = typeof value[sel.name.value] === "function" ? value[sel.name.value]() : value[sel.name.value];
    out[key] = sel.selectionSet ? toyExecute(sel.selectionSet, raw) : raw;
  }
  return out;
}
const tiny = parse(`{ shelf { name books { t: title } } }`);
console.log("\ntoy executor:", JSON.stringify(toyExecute(tiny.definitions[0].selectionSet,
  { shelf: () => ({ name: "Sci-fi", books: [{ title: "Dune" }, { title: "Neuromancer" }] }) })));
```

## Phase 1: lexing and parsing

```text
tokens (30): query Q ( $ id : ID ! ) { book ( id : $ id ) { title ... F } } fragment F on Book { year }
AST node kinds: {"Document":1,"OperationDefinition":1,"Name":11,"VariableDefinition":1,"Variable":2,"NonNullType":1,"NamedType":2,"SelectionSet":3,"Field":3,"Argument":1,"FragmentSpread":1,"FragmentDefinition":1}
```

The **lexer** turns text into tokens, skipping whitespace, commas (which are insignificant in
GraphQL — `{ a, b }` and `{ a b }` are identical) and comments. The **parser** is a
hand-written recursive-descent parser: `parseDocument` calls `parseDefinition`, which calls
`parseOperationDefinition`, and so on, each consuming tokens and returning a plain-object AST
node with a `kind` and a `loc` (start/end offsets, used for error `locations`). Eleven `Name`
nodes for one short query shows how fine-grained the tree is.

## The visitor

`visit(ast, visitor)` walks the tree depth-first and calls `enter`/`leave` functions — either
generic, or keyed by node kind (`Field(node) {…}`). Returning a new node from a visitor
**replaces** it, producing a new tree without mutating the original:

```text
transformed: query Q($id: ID!) { book(id: $id) { title ...F } } fragment F on Book { publishedYear }
```

Almost every GraphQL tool is a visitor: validation rules, `print`, code generators, the depth
and cost analysers from [Level 3 · 05](../level-3/05-security-limits.md), query-rewriting
gateways.

## Phase 2: validation

```text
specifiedRules: 27 rules, e.g. ExecutableDefinitionsRule, UniqueOperationNamesRule, LoneAnonymousOperationRule, SingleFieldSubscriptionsRule ...
```

Each rule corresponds to a section of the spec's Validation chapter. `validate()` combines all
of their visitors with `visitInParallel` and walks the document **once**, with a `TypeInfo`
instance tracking the current parent type, field definition and expected input type at each
node. That shared `TypeInfo` is what lets `FieldsOnCorrectTypeRule` know which type a field is
being selected on. Errors are collected, not thrown — the reason clients get every problem at
once.

## Phase 3: execution

Execution has two steps you've seen piecemeal:

1. **Prepare**: pick the operation (by `operationName`), coerce variables, build an execution
   context with the schema, root value, context value and fragments map.
2. **Execute**: for the root type, `collectFields` flattens the selection set (expanding fragments,
   applying `@skip`/`@include`, merging same-named fields), then `executeFields` (or
   `executeFieldsSerially` for mutations) resolves each field and calls `completeValue`, which
   recurses according to the return type.

The toy executor at the end of the script captures the core idea in fifteen lines — walk the
selection set, call or read each field on the parent value, recurse for sub-selections, map over
lists:

```text
toy executor: {"shelf":{"name":"Sci-fi","books":[{"t":"Dune"},{"t":"Neuromancer"}]}}
```

What the real executor adds on top: type-directed completion (scalar serialisation, abstract type
resolution), argument coercion, error capture with paths, non-null propagation, and promise
handling — while staying synchronous when nothing is async.

## What each phase costs

A deliberately heavy query — 2,000 items, each with 44 fields (40 of them aliases) plus a nested
child — producing a 958 KB response:

```text
query: 318 chars, result: 958472 bytes, 2000 items x (44 fields + a 4-field child)
median ms  parse=0.038  validate=0.295  execute=26.0  JSON.stringify=4.6
```

| Phase | Median | Scales with |
|---|---|---|
| parse | 0.04 ms | query text length |
| validate | 0.3 ms | query size × schema lookups |
| execute | 26 ms | **number of field values produced** |
| serialise | 4.6 ms | response size |

Parsing and validation depend on the *document*, which is small and repeats across requests;
execution depends on the *data*. That's why servers cache parsed-and-validated documents by query
text (Apollo Server's document store, persisted queries) — it removes a fraction of a millisecond
per request, worthwhile at high request rates — but the big wins for slow requests are almost
always in resolvers and data access. Here, with trivial in-memory resolvers, execution of ~100,000
field values took about 26 ms; with real I/O behind fields, the executor's own overhead becomes a
small share.

## How It Actually Works

A few implementation details that explain observable behaviour:

- **`collectFields` merges by response key.** `{ a a }` resolves `a` once; `{ x: a y: a }` resolves
  it twice. That's why aliases multiply cost but duplicates don't.
- **`completeValue` is the only place types are enforced.** Resolvers can return anything; the
  executor checks it against the return type — `null` against non-null, iterables for lists,
  `serialize` for scalars.
- **Results are built with `Object.create(null)`** objects — the null-prototype objects that
  surprised `deepStrictEqual` in [Level 1 · 10](../level-1/10-project-library-api.md).
- **Sync stays sync.** `execute` checks each result with `isPromise`; only if something returned a
  promise does it switch to promise-based assembly for that subtree. A fully synchronous query
  returns a plain object, not a promise — as `phases.mjs` showed in [Level 1 · 02](../level-1/02-first-schema.md).
- **Errors are values until they reach a nullable field** — the `handleFieldError` re-throw described
  in [Level 1 · 08](../level-1/08-errors.md).

## Common mistakes

- **Optimising parsing** when the time is in resolvers. Measure the phases first.
- **Re-parsing on every request** in a hand-rolled server; cache `parse` + `validate` results keyed
  by query text (bounded LRU).
- **Skipping `validate`** when calling `execute` directly. `execute` assumes a valid document;
  invalid ones can produce confusing errors or partial behaviour.
- **Mutating AST nodes** in visitors instead of returning replacements; ASTs are often shared and
  cached.

## Exercise

1. Write a visitor that returns the list of all field names (with their parent types, using
   `TypeInfo` and `visitWithTypeInfo`) a query selects. This is the basis of field-usage tracking
   ([lesson 05](05-schema-evolution.md)).
2. Extend the toy executor with aliases on lists, `@skip(if: true)`, and error capture that records
   a `path`.
3. Make every resolver in the timing script `async` and re-measure `execute`. How much does promise
   handling add for 100,000 values?
4. Implement a 100-entry LRU cache for `parse` + `validate` and measure the speed-up for repeated
   small queries.
