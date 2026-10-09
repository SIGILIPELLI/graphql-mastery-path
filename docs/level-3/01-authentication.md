---
description: "Authenticate GraphQL requests with JWTs: issue tokens on login, verify them in context, reject tampered, expired and alg:none tokens."
---

# 01 · Authentication: Tokens and Context

**Authentication** answers "who is making this request?"; **authorization** (next lesson)
answers "are they allowed to do this?". GraphQL has nothing built in for either — the spec
doesn't mention users at all. Authentication happens at the HTTP layer, in the context
function, and its only output is a `viewer` value that every resolver can read. This lesson
builds that with JSON Web Tokens, then attacks it four ways to show what the verification code
is actually protecting against.

## Where authentication belongs

```text
HTTP request ──► context function ──► { viewer } ──► every resolver
                 (verify token once)                  (decide what viewer may see)
```

Verify credentials **once per request**, in the context function, never inside individual
resolvers. Then make a deliberate choice about anonymous requests: here, a request with *no*
token is allowed and gets `viewer: null` (public fields still work), while a request with a
*bad* token is rejected outright with HTTP 401 — the client clearly meant to authenticate and
should know it failed.

## Issuing and verifying tokens

```bash
npm install jose
```

`jose` (6.2.12 here) is a dependency-free JWT/JOSE library that works in Node, browsers and
edge runtimes.

```js title="auth.js"
import { SignJWT, jwtVerify, errors } from "jose";
import { GraphQLError } from "graphql";

// In production: a long random secret from the environment (or an asymmetric key pair).
const SECRET = new TextEncoder().encode(process.env.JWT_SECRET ?? "dev-only-secret-change-me-0123456789");
const ISSUER = "bookstore-api";
const AUDIENCE = "bookstore-web";

export async function issueToken(user, { expiresIn = "15m" } = {}) {
  return new SignJWT({ role: user.role })
    .setProtectedHeader({ alg: "HS256" })
    .setSubject(String(user.id))
    .setIssuer(ISSUER)
    .setAudience(AUDIENCE)
    .setIssuedAt()
    .setExpirationTime(expiresIn)
    .sign(SECRET);
}

const unauthenticated = (message) =>
  new GraphQLError(message, { extensions: { code: "UNAUTHENTICATED", http: { status: 401 } } });

// Returns the viewer or null for anonymous requests. Throws only for a *bad* token.
export async function viewerFromHeader(header, users) {
  if (!header) return null;
  const match = /^Bearer (.+)$/.exec(header);
  if (!match) throw unauthenticated("Authorization header must be 'Bearer <token>'");
  try {
    const { payload } = await jwtVerify(match[1], SECRET, {
      issuer: ISSUER, audience: AUDIENCE, algorithms: ["HS256"],
    });
    const user = users.get(payload.sub);
    if (!user) throw unauthenticated("Unknown user");
    return user;
  } catch (e) {
    if (e instanceof GraphQLError) throw e;
    if (e instanceof errors.JWTExpired) throw unauthenticated("Token expired");
    throw unauthenticated("Invalid token");
  }
}
```

A JWT is three base64url segments — header, payload, signature. The signature covers the
first two, so any change to the payload invalidates it. `jwtVerify` checks the signature *and*
the claims: `exp` (not expired), `iss` (issued by us), `aud` (issued for this API). Passing
`algorithms: ["HS256"]` pins the algorithm instead of trusting whatever the token's header
claims.

Errors thrown from the context function abort the request. A `GraphQLError` with
`extensions.http.status` lets Apollo Server set the HTTP status code — 401 here.

## The server

```js title="server01.mjs"
import { ApolloServer } from "@apollo/server";
import { startStandaloneServer } from "@apollo/server/standalone";
import { scryptSync, randomBytes, timingSafeEqual } from "node:crypto";
import { issueToken, viewerFromHeader } from "./auth.js";

// Demo user store. Passwords are stored as scrypt hashes, never in plain text.
const hash = (pw, salt = randomBytes(16)) => ({ salt, key: scryptSync(pw, salt, 32) });
const users = new Map([
  ["1", { id: "1", email: "ada@example.com", role: "ADMIN", pw: hash("correct horse") }],
  ["2", { id: "2", email: "bob@example.com", role: "MEMBER", pw: hash("battery staple") }],
]);
const byEmail = (email) => [...users.values()].find((u) => u.email === email);

const typeDefs = /* GraphQL */ `
  type Query { viewer: User, publicCatalogSize: Int! }
  type Mutation { login(email: String!, password: String!): LoginPayload! }
  type User { id: ID! email: String! role: String! }
  type LoginPayload { token: String, user: User, error: String }
`;

const resolvers = {
  Query: {
    viewer: (_, __, { viewer }) => viewer,
    publicCatalogSize: () => 8,
  },
  Mutation: {
    login: async (_, { email, password }) => {
      const user = byEmail(email);
      const ok = user && timingSafeEqual(scryptSync(password, user.pw.salt, 32), user.pw.key);
      if (!ok) return { token: null, user: null, error: "Email or password is incorrect." };
      return { token: await issueToken(user), user, error: null };
    },
  },
};

const server = new ApolloServer({ typeDefs, resolvers, includeStacktraceInErrorResponses: false });
const { url } = await startStandaloneServer(server, {
  listen: { port: 4001 },
  context: async ({ req }) => ({ viewer: await viewerFromHeader(req.headers.authorization, users) }),
});
console.log(`ready at ${url}`);
```

Passwords are stored as **scrypt** hashes with a per-user random salt (`node:crypto`), and
compared with `timingSafeEqual`, which takes the same time regardless of where two buffers
first differ. Login failures are returned as data ([Level 2 · 08](../level-2/08-mutation-design.md)),
and the message doesn't say *which* of email or password was wrong.

## Exercising it

All requests below were sent with `curl` to the running server; only the status line and body
are shown.

**Anonymous** — allowed, viewer is null:

```text
HTTP/1.1 200 OK
{"data":{"viewer":null,"publicCatalogSize":8}}
```

**Wrong password** — an expected outcome, so it's data:

```text
HTTP/1.1 200 OK
{"data":{"login":{"token":null,"error":"Email or password is incorrect."}}}
```

**Correct password** returns a token. Its payload is just base64url-encoded JSON — readable by
anyone who has the token:

```text
token: eyJhbGciOiJIUzI1NiJ9.eyJyb2xlIjoiQURNSU4... (204 chars)
payload: {"role":"ADMIN","sub":"1","iss":"bookstore-api","aud":"bookstore-web","iat":1791521608,"exp":1791522508}
```

`exp − iat` = 900 seconds, the 15-minute lifetime. Never put secrets or personal data in a JWT
payload; it's signed, not encrypted.

**With the token:**

```text
HTTP/1.1 200 OK
{"data":{"viewer":{"email":"ada@example.com","role":"ADMIN"}}}
```

## Four attacks, four rejections

**1. Tampering.** Take Ada's token, replace the payload with `"sub":"2"` (become Bob) and a
far-future `exp`, keep the original signature:

```text
HTTP/1.1 401 Unauthorized
{"errors":[{"message":"Invalid token","extensions":{"code":"UNAUTHENTICATED"}}]}
```

The signature no longer matches the payload.

**2. Expired token.** Signed correctly, but with `expiresIn: "-1s"`:

```text
HTTP/1.1 401 Unauthorized
{"errors":[{"message":"Token expired","extensions":{"code":"UNAUTHENTICATED"}}]}
```

A distinct message lets clients know to refresh rather than log the user out.

**3. `alg: none`.** A historic class of JWT vulnerabilities: a token whose header says
`{"alg":"none"}` and has an empty signature, which naive libraries accepted as valid:

```text
HTTP/1.1 401 Unauthorized
{"errors":[{"message":"Invalid token","extensions":{"code":"UNAUTHENTICATED"}}]}
```

Rejected — both because `jose` doesn't accept unsecured tokens in `jwtVerify` and because we
pinned `algorithms`.

**4. Malformed header** (`Authorization: Token abc`):

```text
HTTP/1.1 401 Unauthorized
{"errors":[{"message":"Authorization header must be 'Bearer <token>'","extensions":{"code":"UNAUTHENTICATED"}}]}
```

Note that this request only asked for `publicCatalogSize`, a public field. It was still
rejected, because a broken credential is treated as an error rather than silently downgraded
to anonymous — otherwise a client bug could leave users browsing "logged out" without anyone
noticing.

## Tokens or cookies?

| | `Authorization: Bearer` header | `HttpOnly` session cookie |
|---|---|---|
| Sent automatically by browsers | no — client code adds it | yes |
| Readable by page JavaScript (XSS exposure) | yes, if stored in JS-accessible storage | no |
| CSRF exposure | none (browsers don't add it to cross-site requests) | yes — needs CSRF protection |
| Works for mobile apps and server-to-server | naturally | awkwardly |

Neither is universally better. If you use cookies for a browser app, Apollo Server's CSRF
prevention ([Level 1 · 07](../level-1/07-apollo-server-http.md)) plus `SameSite` cookies are
essential. Short-lived access tokens like these 15-minute ones are usually paired with a
longer-lived refresh mechanism; designing that is beyond this lesson, and identity providers
(Auth0, Cognito, Keycloak, Entra ID…) typically handle it — in which case you'd verify their
tokens with their public keys (`createRemoteJWKSet` in `jose`) instead of a shared secret.

## How It Actually Works

`jwtVerify` splits the token on dots, base64url-decodes the header, checks `alg` against the
allowed list, recomputes `HMAC-SHA256(secret, header + "." + payload)` and compares it to the
decoded signature in constant time. Only if that matches does it parse the payload JSON and
validate the registered claims: `exp` and `nbf` against the current clock (with optional
tolerance), `iss` and `aud` against the expected values. Any failure throws a typed error
(`JWSSignatureVerificationFailed`, `JWTExpired`, `JWTClaimValidationFailed`…), which is how
`viewerFromHeader` can tell "expired" from "invalid".

In Apollo Server, the context function runs before the GraphQL document is parsed. If it
throws a `GraphQLError`, the server skips execution, formats that error as the only entry in
`errors`, and uses `extensions.http.status` for the response status. The `http` extension
itself is stripped from the response body, which is why the output shows only `code`.

## Common mistakes

- **Verifying tokens in resolvers** instead of once in context.
- **Trusting `jwt.decode()` without verifying** — decoding only reads the payload.
- **Not pinning algorithms**, or using a short guessable HMAC secret. Use at least 32 random
  bytes, from the environment, never committed.
- **Long-lived tokens with no revocation path.** A stolen 30-day token is valid for 30 days.
- **Leaking which part of a login failed** ("no such email") — it lets attackers enumerate
  accounts. (Timing can leak it too: this demo skips the scrypt computation for unknown emails;
  a hardened version hashes a dummy password so both paths take similar time.)
- **Treating bad credentials as anonymous** silently.

## Exercise

1. Make the login path take similar time for unknown emails by hashing against a dummy salt.
   Measure both paths with `performance.now()`.
2. Add `tokenVersion` to each user and include it in the token; bump it on password change and
   reject older tokens — a simple revocation scheme.
3. Switch to an RS256 key pair (`generateKeyPair` in `jose`) so another service can verify
   tokens with only the public key.
4. Change the policy so a bad token is downgraded to anonymous for public fields only. What
   would you need to tell clients, and how?
