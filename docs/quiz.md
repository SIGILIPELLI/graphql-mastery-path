---
title: "Final Quiz"
description: "Test what you have learned across all four levels of GraphQL Mastery Path with a 18-question final quiz."
---

# Final Quiz

18 questions drawn from all four levels of this course. You need **70% (13 out of 18)** to pass —
you can retake it as many times as you like. Your best result is saved in this browser.

<div id="mp-quiz-root"></div>

<script type="application/json" id="mp-quiz-data">
[
  {
    "question": "A client needs only a book's title and its author's name. Why is GraphQL a good fit?",
    "options": [
      "It needs no schema",
      "It's faster always",
      "It caches automatically",
      "The query selects exactly those fields in one request (no over- or under-fetching)"
    ],
    "correctIndex": 3,
    "explanation": "Client-driven shape."
  },
  {
    "question": "What does this print?\n```javascript\nconst schema = buildSchema(`\n  type Author { id: ID! name: String! books: [Book!]! }\n  type Book { id: ID! title: String! year: Int author: Author! }\n  type Query { book(id: ID!): Book books: [Book!]! }\n`);\nconst A = { id: '1', name: 'Ada', books: () => B.filter(b => b.author.id === '1') };\nconst B = [ { id: '1', title: 'Engines', year: 1843, author: A }, { id: '2', title: 'Notes', year: null, author: A } ];\nconst root = { book: ({ id }) => B.find(b => b.id === id), books: () => B };\nshow(graphqlSync({ schema, source: '{ book(id: \"2\") { title year author { name } } }', rootValue: root }));\n```\n",
    "options": [
      "{\"book\":{\"title\":\"Notes\",\"year\":null,\"author\":{\"name\":\"Ada\"}}}",
      "{\"book\":{\"title\":\"Notes\"}}",
      "{\"book\":null}",
      "Error"
    ],
    "correctIndex": 0,
    "explanation": "The response mirrors the query; nullable year is null. Verified by running it with graphql-js."
  },
  {
    "question": "What does this print?\n```javascript\nconst schema = buildSchema(`\n  type Author { id: ID! name: String! books: [Book!]! }\n  type Book { id: ID! title: String! year: Int author: Author! }\n  type Query { book(id: ID!): Book books: [Book!]! }\n`);\nconst A = { id: '1', name: 'Ada', books: () => B.filter(b => b.author.id === '1') };\nconst B = [ { id: '1', title: 'Engines', year: 1843, author: A }, { id: '2', title: 'Notes', year: null, author: A } ];\nconst root = { book: ({ id }) => B.find(b => b.id === id), books: () => B };\nshow(graphqlSync({ schema, source: '{ books { id } }', rootValue: { books: () => [{ id: '1' }, { id: null }] } }));\n```\n",
    "options": [
      "{\"books\":[{\"id\":\"1\"},{\"id\":null}]}",
      "Error",
      "{\"errors\":[\"Cannot return null for non-nullable field Book.id.\"],\"data\":null}",
      "{\"books\":[{\"id\":\"1\"}]}"
    ],
    "correctIndex": 2,
    "explanation": "A null in a non-null field propagates to the nearest nullable parent (here the whole data because books is non-null). Verified by running it with graphql-js."
  },
  {
    "question": "What does this print?\n```javascript\nconst s = buildSchema(`type Query { a: String b: String }`);\nshow(graphqlSync({ schema: s, source: '{ a b }', rootValue: { a: () => { throw new Error('boom'); }, b: () => 'ok' } }));\n```\n",
    "options": [
      "{\"data\":{\"b\":\"ok\"}}",
      "{\"errors\":[\"boom\"],\"data\":null}",
      "Error",
      "{\"errors\":[\"boom\"],\"data\":{\"a\":null,\"b\":\"ok\"}}"
    ],
    "correctIndex": 3,
    "explanation": "Nullable fields isolate failures: partial data plus errors. Verified by running it with graphql-js."
  },
  {
    "question": "What does validation say about an unknown field?\n```javascript\nconst schema = buildSchema(`\n  type Author { id: ID! name: String! books: [Book!]! }\n  type Book { id: ID! title: String! year: Int author: Author! }\n  type Query { book(id: ID!): Book books: [Book!]! }\n`);\nconst A = { id: '1', name: 'Ada', books: () => B.filter(b => b.author.id === '1') };\nconst B = [ { id: '1', title: 'Engines', year: 1843, author: A }, { id: '2', title: 'Notes', year: null, author: A } ];\nconst root = { book: ({ id }) => B.find(b => b.id === id), books: () => B };\nconst r = graphqlSync({ schema, source: '{ books { titel } }', rootValue: root });\nconsole.log(r.errors[0].message);\n```\n",
    "options": [
      "Field titel is null",
      "Cannot query field \"titel\" on type \"Book\". Did you mean \"title\"?",
      "Syntax error",
      "No errors"
    ],
    "correctIndex": 1,
    "explanation": "Validation rejects the operation before execution. Verified by running it with graphql-js."
  },
  {
    "question": "What does this print?\n```javascript\nconst s = buildSchema(`input NewBook { title: String! } type Query { x: Int } type Mutation { addBook(input: NewBook!): String! }`);\nshow(graphqlSync({ schema: s, source: 'mutation { addBook(input: { title: 5 }) }', rootValue: {} }));\n```\n",
    "options": [
      "Error",
      "{\"errors\":[\"String cannot represent a non string value: 5\"]}",
      "{\"addBook\":\"5\"}",
      "{\"addBook\":null}"
    ],
    "correctIndex": 1,
    "explanation": "Input values are validated against the schema. Verified by running it with graphql-js."
  },
  {
    "question": "What does this print?\n```javascript\nconst s = buildSchema(`type Query { greet(name: String = \"world\"): String }`);\nshow(graphqlSync({ schema: s, source: '{ a: greet b: greet(name: \"Ada\") }', rootValue: { greet: ({ name }) => `hi ${name}` } }));\n```\n",
    "options": [
      "Error",
      "{\"a\":null,\"b\":\"hi Ada\"}",
      "{\"a\":\"hi world\",\"b\":\"hi Ada\"}",
      "{\"a\":\"hi\",\"b\":\"hi Ada\"}"
    ],
    "correctIndex": 2,
    "explanation": "Aliases allow the same field twice; defaults apply. Verified by running it with graphql-js."
  },
  {
    "question": "What does __typename help clients do?",
    "options": [
      "Auth",
      "Paginate",
      "Batch",
      "Identify the concrete type at runtime, e.g. for fragments and cache normalization"
    ],
    "correctIndex": 3,
    "explanation": "Included automatically by many clients."
  },
  {
    "question": "A list of 50 books each resolves its author with one DB query. What is this and the fix?",
    "options": [
      "The N+1 problem; batch with a per-request DataLoader",
      "Over-fetching; add fields",
      "Caching; add TTL",
      "Pagination"
    ],
    "correctIndex": 0,
    "explanation": "Batching collapses 50 lookups into one."
  },
  {
    "question": "Why create DataLoaders per request?",
    "options": [
      "Cached/batched data must not leak between users or go stale across requests",
      "Style",
      "Law",
      "Speed"
    ],
    "correctIndex": 0,
    "explanation": "Put them in context."
  },
  {
    "question": "Mutations return a Boolean and clients can't update their cache. What's the better design?",
    "options": [
      "Return String",
      "Return Int",
      "Return the changed object (with id) or a payload containing it and user errors",
      "Return void"
    ],
    "correctIndex": 2,
    "explanation": "Normalized caches update from returned entities."
  },
  {
    "question": "Authorization is checked only in the UI and a user queries another user's profile. What's the problem?",
    "options": [
      "N+1",
      "CORS",
      "Validation",
      "Missing server-side object-level authorization (IDOR); enforce it in resolvers/services"
    ],
    "correctIndex": 3,
    "explanation": "Never rely on the client."
  },
  {
    "question": "A client sends a 20-level-deep nested query. What protects the server?",
    "options": [
      "Query depth/complexity limits (and timeouts and rate limiting)",
      "Introspection",
      "Caching",
      "More RAM"
    ],
    "correctIndex": 0,
    "explanation": "Cost control."
  },
  {
    "question": "Which pagination approach is stable when items are inserted during paging?",
    "options": [
      "Offset",
      "Cursor-based (Relay connections)",
      "Page numbers",
      "Limit only"
    ],
    "correctIndex": 1,
    "explanation": "Keyset cursors."
  },
  {
    "question": "A field must be removed. What's the safe process?",
    "options": [
      "Hide",
      "Delete now",
      "Rename",
      "Deprecate it, watch usage, notify clients, then remove once unused"
    ],
    "correctIndex": 3,
    "explanation": "Additive evolution."
  },
  {
    "question": "Federation needs one shared `User` across two subgraphs. How is the entity identified?",
    "options": [
      "A gateway cache",
      "Copy types",
      "Same table",
      "With @key and a reference resolver in each subgraph"
    ],
    "correctIndex": 3,
    "explanation": "Entity references."
  },
  {
    "question": "Why return errors with codes in extensions?",
    "options": [
      "Pretty",
      "Law",
      "Clients can branch on machine-readable codes instead of parsing messages",
      "Speed"
    ],
    "correctIndex": 2,
    "explanation": "extensions.code."
  },
  {
    "question": "Subscriptions work on one server but not behind a load balancer. What's likely missing?",
    "options": [
      "A shared pub/sub backend (Redis/Kafka) and sticky WebSocket handling",
      "Introspection",
      "Caching",
      "More CPU"
    ],
    "correctIndex": 0,
    "explanation": "Fan-out across instances."
  }
]
</script>

## Coding challenges

Try these on your own computer, then compare your output. Nothing you write is sent anywhere.

### Batch lookups with a DataLoader

Implement createLoader(batchFn): keys requested in the same tick are sent to batchFn once (deduplicated, results aligned with the keys) and each key is cached. Compare the number of database calls for a list of five books with and without the loader.

**Starter code**

```javascript
// npm i graphql   (run with: node batch.js)
const { graphql, buildSchema } = require('graphql');

const AUTHORS = { 1: 'Ada', 2: 'Bo', 3: 'Cy' };
let dbCalls = 0;
const fetchAuthors = async (ids) => { dbCalls++; return ids.map((id) => ({ id, name: AUTHORS[id] })); };

// A minimal DataLoader: collect keys requested in the same tick, call the batch function once,
// cache per key, and return results aligned with the keys.
function createLoader(batchFn) {
  // load(key) returns a promise. Keys requested in the same tick must be sent to batchFn ONCE
  // (deduplicated, results aligned with the key array), and repeated keys must be cached per loader.
  return { load(key) { return Promise.resolve(undefined); } };
}

const schema = buildSchema(`
  type Author { name: String! }
  type Book { title: String! author: Author! }
  type Query { books: [Book!]! }
`);
const BOOKS = [{ title: 'a', authorId: 1 }, { title: 'b', authorId: 2 }, { title: 'c', authorId: 1 }, { title: 'd', authorId: 3 }, { title: 'e', authorId: 2 }];

async function run(useLoader) {
  dbCalls = 0;
  const loader = createLoader(fetchAuthors);                       // one loader per request
  const rootValue = {
    books: () => BOOKS.map((b) => ({
      title: b.title,
      author: () => (useLoader ? loader.load(b.authorId) : fetchAuthors([b.authorId]).then((r) => r[0])),
    })),
  };
  const res = await graphql({ schema, source: '{ books { title author { name } } }', rootValue });
  console.log(useLoader ? 'with loader:   ' : 'without loader:', 'db calls =', dbCalls, '|', res.data.books.map((b) => b.author.name).join(','));
}

(async () => { await run(false); await run(true); })();
```

**Expected output**

```text
without loader: db calls = 5 | Ada,Bo,Ada,Cy,Bo
with loader:    db calls = 1 | Ada,Bo,Ada,Cy,Bo
```

<details><summary>Show solution and explanation</summary>

<pre><code>// npm i graphql   (run with: node batch.js)
const { graphql, buildSchema } = require(&#x27;graphql&#x27;);

const AUTHORS = { 1: &#x27;Ada&#x27;, 2: &#x27;Bo&#x27;, 3: &#x27;Cy&#x27; };
let dbCalls = 0;
const fetchAuthors = async (ids) =&gt; { dbCalls++; return ids.map((id) =&gt; ({ id, name: AUTHORS[id] })); };

// A minimal DataLoader: collect keys requested in the same tick, call the batch function once,
// cache per key, and return results aligned with the keys.
function createLoader(batchFn) {
  const cache = new Map();
  let queue = [];
  return {
    load(key) {
      if (cache.has(key)) return cache.get(key);
      const p = new Promise((resolve, reject) =&gt; {
        queue.push({ key, resolve, reject });
        if (queue.length === 1) {
          Promise.resolve().then(async () =&gt; {
            const batch = queue; queue = [];
            const keys = batch.map((b) =&gt; b.key);
            try {
              const results = await batchFn([...new Set(keys)]);
              const byKey = new Map(results.map((r, i) =&gt; [[...new Set(keys)][i], r]));
              batch.forEach((b) =&gt; b.resolve(byKey.get(b.key)));
            } catch (e) { batch.forEach((b) =&gt; b.reject(e)); }
          });
        }
      });
      cache.set(key, p);
      return p;
    },
  };
}

const schema = buildSchema(`
  type Author { name: String! }
  type Book { title: String! author: Author! }
  type Query { books: [Book!]! }
`);
const BOOKS = [{ title: &#x27;a&#x27;, authorId: 1 }, { title: &#x27;b&#x27;, authorId: 2 }, { title: &#x27;c&#x27;, authorId: 1 }, { title: &#x27;d&#x27;, authorId: 3 }, { title: &#x27;e&#x27;, authorId: 2 }];

async function run(useLoader) {
  dbCalls = 0;
  const loader = createLoader(fetchAuthors);                       // one loader per request
  const rootValue = {
    books: () =&gt; BOOKS.map((b) =&gt; ({
      title: b.title,
      author: () =&gt; (useLoader ? loader.load(b.authorId) : fetchAuthors([b.authorId]).then((r) =&gt; r[0])),
    })),
  };
  const res = await graphql({ schema, source: &#x27;{ books { title author { name } } }&#x27;, rootValue });
  console.log(useLoader ? &#x27;with loader:   &#x27; : &#x27;without loader:&#x27;, &#x27;db calls =&#x27;, dbCalls, &#x27;|&#x27;, res.data.books.map((b) =&gt; b.author.name).join(&#x27;,&#x27;));
}

(async () =&gt; { await run(false); await run(true); })();
</code></pre>

<p>Without batching, five books trigger five author lookups (N+1); with a loader they collapse into one batch call. A real DataLoader also handles per-key errors and max batch sizes. Verified by running the solution with graphql-js.</p>
</details>

### A query depth limit

Write depthLimit(max) as a GraphQL validation rule that reports an error when an operation's field nesting exceeds max (ignoring introspection fields and following fragments). Check five operations.

**Starter code**

```javascript
// npm i graphql   (run with: node limit.js)
const { buildSchema, parse, validate, specifiedRules, GraphQLError, Kind } = require('graphql');

const schema = buildSchema(`
  type User { name: String! friends: [User!]! }
  type Query { me: User! }
`);

// A validation rule that rejects operations nested deeper than `max` selection levels.
function depthLimit(max) {
  // return a validation rule: for each OperationDefinition compute the field nesting depth
  // (ignore __ introspection fields; follow fragments) and context.reportError(new GraphQLError(
  // `query depth ${depth} exceeds limit ${max}`, { nodes: [node] })) when it is greater than max
  return () => ({});
}

function measure(selectionSet, context, seenFragments) {
  let best = 0;
  for (const sel of selectionSet.selections) {
    if (sel.kind === Kind.FIELD) {
      if (sel.name.value.startsWith('__')) continue;
      best = Math.max(best, 1 + (sel.selectionSet ? measure(sel.selectionSet, context, seenFragments) : 0));
    } else if (sel.kind === Kind.INLINE_FRAGMENT) {
      best = Math.max(best, measure(sel.selectionSet, context, seenFragments));
    } else if (sel.kind === Kind.FRAGMENT_SPREAD && !seenFragments.has(sel.name.value)) {
      seenFragments.add(sel.name.value);
      const frag = context.getDocument().definitions.find((d) => d.kind === Kind.FRAGMENT_DEFINITION && d.name.value === sel.name.value);
      if (frag) best = Math.max(best, measure(frag.selectionSet, context, seenFragments));
    }
  }
  return best;
}

function check(source, max) {
  const errors = validate(schema, parse(source), [...specifiedRules, depthLimit(max)]);
  return errors.length ? errors.map((e) => e.message).join('; ') : 'ok';
}

console.log(check('{ me { name } }', 3));
console.log(check('{ me { friends { friends { name } } } }', 3));
console.log(check('{ me { friends { friends { friends { name } } } } }', 3));
console.log(check('query Q { me { ...F } } fragment F on User { friends { friends { friends { name } } } }', 3));
console.log(check('{ me { nope } }', 3));
```

**Expected output**

```text
ok
query depth 4 exceeds limit 3
query depth 5 exceeds limit 3
query depth 5 exceeds limit 3
Cannot query field "nope" on type "User". Did you mean "name"?
```

<details><summary>Show solution and explanation</summary>

<pre><code>// npm i graphql   (run with: node limit.js)
const { buildSchema, parse, validate, specifiedRules, GraphQLError, Kind } = require(&#x27;graphql&#x27;);

const schema = buildSchema(`
  type User { name: String! friends: [User!]! }
  type Query { me: User! }
`);

// A validation rule that rejects operations nested deeper than `max` selection levels.
function depthLimit(max) {
  return (context) =&gt; ({
    OperationDefinition(node) {
      const depth = measure(node.selectionSet, context, new Set());
      if (depth &gt; max) {
        context.reportError(new GraphQLError(`query depth ${depth} exceeds limit ${max}`, { nodes: [node] }));
      }
    },
  });
}

function measure(selectionSet, context, seenFragments) {
  let best = 0;
  for (const sel of selectionSet.selections) {
    if (sel.kind === Kind.FIELD) {
      if (sel.name.value.startsWith(&#x27;__&#x27;)) continue;
      best = Math.max(best, 1 + (sel.selectionSet ? measure(sel.selectionSet, context, seenFragments) : 0));
    } else if (sel.kind === Kind.INLINE_FRAGMENT) {
      best = Math.max(best, measure(sel.selectionSet, context, seenFragments));
    } else if (sel.kind === Kind.FRAGMENT_SPREAD &amp;&amp; !seenFragments.has(sel.name.value)) {
      seenFragments.add(sel.name.value);
      const frag = context.getDocument().definitions.find((d) =&gt; d.kind === Kind.FRAGMENT_DEFINITION &amp;&amp; d.name.value === sel.name.value);
      if (frag) best = Math.max(best, measure(frag.selectionSet, context, seenFragments));
    }
  }
  return best;
}

function check(source, max) {
  const errors = validate(schema, parse(source), [...specifiedRules, depthLimit(max)]);
  return errors.length ? errors.map((e) =&gt; e.message).join(&#x27;; &#x27;) : &#x27;ok&#x27;;
}

console.log(check(&#x27;{ me { name } }&#x27;, 3));
console.log(check(&#x27;{ me { friends { friends { name } } } }&#x27;, 3));
console.log(check(&#x27;{ me { friends { friends { friends { name } } } } }&#x27;, 3));
console.log(check(&#x27;query Q { me { ...F } } fragment F on User { friends { friends { friends { name } } } }&#x27;, 3));
console.log(check(&#x27;{ me { nope } }&#x27;, 3));
</code></pre>

<p>Depth limits stop pathological nested queries before execution. Combine them with cost analysis, timeouts and rate limiting. Verified by running the solution with graphql-js.</p>
</details>
