---
description: "Define and implement custom GraphQL schema directives — @auth, @lower, @truncate — with mapSchema, and learn what directives can't do."
---

# 03 · Custom Schema Directives

A **directive** is an annotation starting with `@`. You've used two kinds already:
`@include`/`@skip`, which clients put in *queries*, and `@deprecated`, which servers put in the
*schema*. This lesson is about the second kind — **schema directives** you define yourself —
and how to give them behaviour. They're a tidy way to declare cross-cutting rules right in the
SDL (`stats: Stats @auth(requires: ADMIN)`), but they're also easy to over-use, so the lesson
ends with their limits.

## Two families of directive

| | Executable directives | Type system (schema) directives |
|---|---|---|
| Written in | operations (queries, fragments) | SDL |
| Locations | `QUERY`, `FIELD`, `FRAGMENT_SPREAD`, `INLINE_FRAGMENT`, … | `OBJECT`, `FIELD_DEFINITION`, `ARGUMENT_DEFINITION`, `ENUM_VALUE`, … |
| Examples | `@include(if:)`, `@skip(if:)`, `@defer` | `@deprecated`, `@specifiedBy`, `@oneOf`, federation's `@key` |
| Interpreted by | the executor, per request | whatever code reads the schema |

A schema directive by itself **does nothing**. `graphql-js` parses it, validates its location
and arguments, and stores it on the AST node. Behaviour comes from code that walks the schema
and acts on it — typically by wrapping resolvers once, at startup.

## Declaring directives

```graphql
enum Role { MEMBER ADMIN }

directive @auth(requires: Role = MEMBER) on OBJECT | FIELD_DEFINITION
directive @lower on FIELD_DEFINITION
directive @truncate(length: Int!) on FIELD_DEFINITION
```

A declaration gives the name, typed arguments (with defaults) and the locations where it may
appear. Using `@lower` on a type, or `@truncate` without `length`, is a schema validation error.

## Implementing them with `mapSchema`

`@graphql-tools/utils` (12.0.3 here) provides `mapSchema`, which rebuilds a schema while letting
you replace types and fields, and `getDirective`, which reads a directive's arguments from a
schema element:

```js title="directives03.mjs"
import { makeExecutableSchema } from "@graphql-tools/schema";
import { mapSchema, getDirective, MapperKind } from "@graphql-tools/utils";
import { graphql, defaultFieldResolver, GraphQLError, printSchema } from "graphql";

const typeDefs = /* GraphQL */ `
  enum Role { MEMBER ADMIN }

  directive @auth(requires: Role = MEMBER) on OBJECT | FIELD_DEFINITION
  directive @lower on FIELD_DEFINITION
  directive @truncate(length: Int!) on FIELD_DEFINITION

  type Query {
    articles: [Article!]!
    stats: Stats @auth(requires: ADMIN)
  }

  type Article {
    id: ID!
    title: String! @lower
    summary: String! @truncate(length: 24)
    "Drafts are visible to logged-in members only."
    draftNotes: String @auth
  }

  type Stats @auth(requires: ADMIN) {
    articleCount: Int!
    revenueCents: Int!
  }
`;

const resolvers = {
  Query: {
    articles: () => [
      { id: "1", title: "Directives In Depth", summary: "How schema directives transform fields at build time.", draftNotes: "add diagrams" },
    ],
    stats: () => ({ articleCount: 1, revenueCents: 4200 }),
  },
};

const rank = { MEMBER: 1, ADMIN: 2 };

function authDirective(schema) {
  const typeRequirement = new Map(); // type name -> role from an OBJECT-level @auth
  return mapSchema(schema, {
    [MapperKind.OBJECT_TYPE]: (type) => {
      const auth = getDirective(schema, type, "auth")?.[0];
      if (auth) typeRequirement.set(type.name, auth.requires);
      return type;
    },
    [MapperKind.OBJECT_FIELD]: (field, fieldName, typeName) => {
      const auth = getDirective(schema, field, "auth")?.[0];
      const requires = auth?.requires ?? typeRequirement.get(typeName);
      if (!requires) return field;
      const { resolve = defaultFieldResolver } = field;
      return {
        ...field,
        resolve(source, args, context, info) {
          const viewer = context.viewer;
          if (!viewer) throw new GraphQLError("You must be logged in", { extensions: { code: "UNAUTHENTICATED" } });
          if (rank[viewer.role] < rank[requires])
            throw new GraphQLError(`Requires ${requires}`, { extensions: { code: "FORBIDDEN" } });
          return resolve(source, args, context, info);
        },
      };
    },
  });
}

function formatDirectives(schema) {
  return mapSchema(schema, {
    [MapperKind.OBJECT_FIELD]: (field) => {
      const lower = getDirective(schema, field, "lower")?.[0];
      const truncate = getDirective(schema, field, "truncate")?.[0];
      if (!lower && !truncate) return field;
      const { resolve = defaultFieldResolver } = field;
      return {
        ...field,
        async resolve(...args) {
          let value = await resolve(...args);
          if (typeof value !== "string") return value;
          if (lower) value = value.toLowerCase();
          if (truncate && value.length > truncate.length) value = value.slice(0, truncate.length - 1) + "…";
          return value;
        },
      };
    },
  });
}

let schema = makeExecutableSchema({ typeDefs, resolvers });
schema = formatDirectives(authDirective(schema));

const run = async (label, viewer, source) =>
  console.log(`--- ${label}\n${JSON.stringify(await graphql({ schema, source, contextValue: { viewer } }))}`);

await run("anonymous", null, `{ articles { title summary draftNotes } }`);
await run("member", { role: "MEMBER" }, `{ articles { title draftNotes } stats { articleCount } }`);
await run("admin", { role: "ADMIN" }, `{ stats { articleCount revenueCents } }`);

console.log("\n--- introspection: are directives visible to clients?");
const r = await graphql({ schema, source: `{ __type(name: "Article") { fields { name } } __schema { directives { name } } }` });
console.log(JSON.stringify(r.data));
console.log("\n--- printSchema keeps them in SDL:");
console.log(printSchema(schema).split("\n").filter((l) => l.includes("@")).join("\n"));
```

How it works:

- `authDirective` visits every object type first, remembering types annotated with `@auth`. For
  every field, it uses the field's own `@auth` or inherits the type's. If either exists, it
  replaces the field's `resolve` with a wrapper that checks the viewer and then calls the
  original resolver — or `defaultFieldResolver` for fields that had none.
- `formatDirectives` wraps resolvers to post-process string results.
- The two transforms are **composed** at startup:
  `formatDirectives(authDirective(schema))`. The result is an ordinary schema; at request time
  there's no directive processing at all, just the wrapped resolvers.

## Results

```text
--- anonymous
{"errors":[{"message":"You must be logged in","locations":[{"line":1,"column":28}],"path":["articles",0,"draftNotes"],"extensions":{"code":"UNAUTHENTICATED"}}],"data":{"articles":[{"title":"directives in depth","summary":"How schema directives t…","draftNotes":null}]}}
--- member
{"errors":[{"message":"Requires ADMIN","locations":[{"line":1,"column":33}],"path":["stats"],"extensions":{"code":"FORBIDDEN"}}],"data":{"articles":[{"title":"directives in depth","draftNotes":"add diagrams"}],"stats":null}}
--- admin
{"data":{"stats":{"articleCount":1,"revenueCents":4200}}}
```

- `@lower` lower-cased the title and `@truncate(length: 24)` cut the summary to 23 characters plus
  an ellipsis.
- `draftNotes @auth` used the default `requires: MEMBER`: `null` plus an error for anonymous,
  visible for a member. Because `draftNotes` is nullable, the rest of the article survived.
- `stats` is protected twice: the field has `@auth(requires: ADMIN)`, and the `Stats` type has
  it too, so every field of `Stats` is protected even if it's reached through some other path
  later. That type-level inheritance is a partial answer to the second-path bug from
  [lesson 02](02-authorization.md) — for role rules, not data-dependent ones.

## What clients see

```text
--- introspection: are directives visible to clients?
{"__type":{"fields":[{"name":"id"},{"name":"title"},{"name":"summary"},{"name":"draftNotes"}]},"__schema":{"directives":[{"name":"auth"},{"name":"lower"},{"name":"truncate"},{"name":"include"},{"name":"skip"},{"name":"deprecated"},{"name":"specifiedBy"},{"name":"oneOf"}]}}

--- printSchema keeps them in SDL:
directive @auth(requires: Role = MEMBER) on OBJECT | FIELD_DEFINITION
directive @lower on FIELD_DEFINITION
directive @truncate(length: Int!) on FIELD_DEFINITION
```

Introspection lists the directive **definitions** (alongside the built-ins — `graphql` 16.14
includes `@oneOf`), but not **where they're applied**. Standard introspection has no way to say
"`draftNotes` has `@auth`". The same is true for `printSchema` on the executable schema: the
definitions are printed, the usages are not. So directives are documentation for *your team*
reading the SDL source, not for API consumers. If clients need to know a field requires login,
say so in its description, as `draftNotes` does.

(Federation is the exception that proves the rule: its subgraphs expose their SDL *with*
applied directives through a special `_service { sdl }` field precisely because standard
introspection drops them. See [Level 4 · 03](../level-4/03-federation-concepts.md).)

## When to use a directive — and when not to

Good fits:

- **Coarse, declarative policies** that are the same everywhere: logged-in only, role
  requirements, rate-limit buckets, cache hints (Apollo's `@cacheControl`, used in
  [lesson 06](06-caching-persisted-queries.md)).
- **Metadata for tools**: federation keys, ownership tags, data-classification labels read by
  linters or code generators.

Poor fits:

- **Data-dependent authorization** ("owner or admin"). The directive would need to know how to
  find the owner of every type — that logic belongs in the data layer.
- **Business logic** (`@computePrice`). It hides behaviour in a place nobody looks for it.
- **Formatting that clients should control.** `@lower` here is a teaching example; in real APIs,
  a field argument (`title(case: LOWER)`) or plain client-side formatting is clearer.

## How It Actually Works

When `buildASTSchema` (used by `makeExecutableSchema`) creates a `GraphQLField` from SDL, it
keeps the original AST node as `field.astNode`, directives included. `getDirective(schema,
field, "auth")` looks at `astNode.directives` (and `extensionASTNodes`), finds those named
`auth`, and turns their argument AST into values with `getArgumentValues`, applying defaults from
the directive definition — which is how a bare `@auth` produced `requires: "MEMBER"`.

`mapSchema` walks every named type, calls your mapper for each matching `MapperKind`, and then
builds a new `GraphQLSchema` from the results, rewiring all type references so that a field
returning `Stats` points at the *new* `Stats` object. That rewiring is why you must use the
schema `mapSchema` returns rather than mutating the original.

Execution never sees the directives. The wrapped `resolve` functions are just closures created
at startup that captured `requires` and the original resolver.

## Common mistakes

- **Declaring a directive and expecting it to do something.** Without a transform, it's inert.
- **Forgetting fields without resolvers.** Wrap `field.resolve ?? defaultFieldResolver`, or
  directive-protected default fields stay unprotected.
- **Mutating the schema instead of using the result of `mapSchema`.**
- **Order-dependent transforms.** Here formatting wraps auth; if a transform short-circuited
  (say, a cache), putting it outside auth would serve cached data to unauthorized users.
- **Putting `@auth` on a non-null field** — a denied read nulls out the parent.

## Exercise

1. Add `@auth` support for `INTERFACE` types, so every implementing type's fields inherit it.
2. Write `@rateLimit(perMinute: Int!)` that counts calls per viewer in the context and throws
   `RATE_LIMITED` past the limit. Where should the counter live so it works across requests,
   and what does that imply for multiple server instances?
3. Replace `@lower` with a field argument `title(case: Case = ORIGINAL)` and compare the
   client-facing documentation of both designs.
4. Swap the order of the two transforms and explain whether any observable behaviour changes.
