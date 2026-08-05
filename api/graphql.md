# GraphQL

GraphQL is a query language for APIs and a runtime for executing those queries against your data. Facebook created it in 2012 (open-sourced in 2015) to solve a concrete problem: their mobile apps were making dozens of REST calls per screen, pulling back way more data than they needed on some endpoints and not enough on others. GraphQL replaces "many fixed-shape endpoints" with "one endpoint, client-specified shape."

## TL;DR
- Single endpoint (typically `/graphql`), single HTTP method (POST, usually) — the query itself, not the URL, determines what data comes back.
- Client asks for exactly the fields it needs, nested arbitrarily deep, in one round trip — no over-fetching, no under-fetching, no N+1 *client-side* requests.
- Strongly typed schema is the contract between client and server; it's introspectable, so tools (GraphiQL, Apollo Studio, codegen) can auto-generate docs and typed clients.
- Three operation types: **queries** (read), **mutations** (write), **subscriptions** (real-time push).
- Resolvers — one function per field — do the actual data fetching; naive resolver nesting causes the N+1 query problem, solved with **DataLoader**-style batching.
- The trade-off: you gain flexible, precise data-fetching but lose free HTTP-level caching (every request is a POST to the same URL) and gain new risks (malicious clients can ask for arbitrarily deep/expensive queries).

## The Problem It Solves

Consider a mobile app screen showing a user's profile with their 5 most recent posts and each post's comment count. With a typical REST API:

```
GET /users/123          → { id, name, email, avatar, bio, ... } — 20 fields, you needed 3
GET /users/123/posts?limit=5   → posts, but each post object has 15 fields, you needed 2
GET /posts/1/comments/count
GET /posts/2/comments/count
GET /posts/3/comments/count
... (N more round trips)
```

This is REST's classic **over-fetching** (getting a bloated `User` object when you only wanted `name` and `avatar`) combined with **under-fetching** (needing follow-up requests to get comment counts, because that's not embedded in the posts endpoint) — often forcing a chain of N+1 sequential requests from the client.

GraphQL collapses this into one request, one round trip:

```graphql
query {
  user(id: "123") {
    name
    avatar
    posts(limit: 5) {
      title
      commentCount
    }
  }
}
```

The server resolves exactly this shape and returns exactly this shape — nothing more.

## How It Works: The Core Pieces

```
Client                     Server
  |                           |
  | POST /graphql             |
  |  { query: "..." }         |
  |-------------------------->|
  |                     Parse query against Schema
  |                     Validate types/fields exist
  |                     Execute: call Resolvers per field
  |                     Resolvers fetch from DB/APIs/caches
  |<---------------------------|
  |  { data: {...}, errors: [...] }
```

1. **Schema** — a typed contract (written in Schema Definition Language, SDL) describing every type, field, query, and mutation the API supports.
2. **Query** — the client sends a query string describing the exact shape of data it wants.
3. **Resolvers** — server-side functions, one per field, responsible for fetching that field's value. The GraphQL engine walks the query tree and calls the matching resolver for each requested field.
4. **Execution** — the engine resolves the whole tree (often in parallel where fields are independent), assembles a JSON response matching the query's shape, and returns `{ data, errors }`.

## 🟢 Beginner

### The Type System and Schema

Everything in GraphQL starts with the schema — it's the single source of truth for what's queryable. Defined in SDL:

```graphql
type User {
  id: ID!
  name: String!
  email: String!
  age: Int
  isActive: Boolean!
  posts: [Post!]!
}

type Post {
  id: ID!
  title: String!
  content: String
  author: User!
  commentCount: Int!
  createdAt: String!
}

type Query {
  user(id: ID!): User
  users(limit: Int, offset: Int): [User!]!
  post(id: ID!): Post
}

type Mutation {
  createUser(name: String!, email: String!): User!
  updatePost(id: ID!, title: String): Post!
  deletePost(id: ID!): Boolean!
}
```

**Scalar types** (built-in): `Int`, `Float`, `String`, `Boolean`, `ID` (a unique identifier, serialized as a string but semantically distinct). You can also define custom scalars (e.g., `DateTime`, `Email`) with custom serialization logic.

**Type modifiers**:
- `String` — nullable string (can be `null`).
- `String!` — non-null string (the `!` means this field is *guaranteed* to have a value; if the resolver returns null, that's a schema-breaking error).
- `[String]` — nullable list of nullable strings.
- `[String!]!` — non-null list of non-null strings — the most common "give me a real array of real values" shape.

**Object types** define the shape of your domain models (`User`, `Post`). **`Query`** and **`Mutation`** are special root types — every field on `Query` is an entry point for reading data; every field on `Mutation` is an entry point for writing data.

**Enums** and **interfaces/unions** round out the type system:

```graphql
enum Role {
  ADMIN
  EDITOR
  VIEWER
}

interface Node {
  id: ID!
}

type User implements Node {
  id: ID!
  name: String!
}

union SearchResult = User | Post
```

### Queries

A query reads data. The client picks exactly the fields it wants:

```graphql
query GetUser {
  user(id: "123") {
    name
    email
    posts {
      title
    }
  }
}
```

Response:
```json
{
  "data": {
    "user": {
      "name": "Alice",
      "email": "alice@example.com",
      "posts": [
        { "title": "Hello World" },
        { "title": "GraphQL Basics" }
      ]
    }
  }
}
```

Queries can take **arguments** (like `id: "123"` above), use **variables** to avoid string-interpolating values into the query text, and use **aliases** to rename fields or fetch the same field twice with different arguments:

```graphql
query GetUserById($userId: ID!) {
  user(id: $userId) {
    name
  }
}
```
```json
{ "userId": "123" }
```

```graphql
query CompareUsers {
  alice: user(id: "1") { name }
  bob: user(id: "2") { name }
}
```

**Fragments** let you reuse a set of fields across multiple queries:

```graphql
fragment UserFields on User {
  id
  name
  email
}

query {
  user(id: "1") {
    ...UserFields
    posts { title }
  }
}
```

### Mutations

Mutations write data — create, update, delete. Syntactically similar to queries, but semantically the entry point for side effects, and (by convention, not enforcement) executed serially rather than in parallel when multiple mutations appear in one request.

```graphql
mutation CreatePost($title: String!, $content: String!) {
  createPost(title: $title, content: $content) {
    id
    title
    createdAt
  }
}
```
Variables:
```json
{ "title": "New Post", "content": "Hello GraphQL" }
```
Response:
```json
{
  "data": {
    "createPost": {
      "id": "42",
      "title": "New Post",
      "createdAt": "2026-08-05T10:00:00Z"
    }
  }
}
```

Convention: mutations return the object they just modified, so the client can immediately update its local cache without a follow-up query.

### Subscriptions

Subscriptions provide real-time updates — the client opens a persistent connection (typically WebSocket) and the server pushes data whenever an event occurs.

```graphql
subscription OnCommentAdded($postId: ID!) {
  commentAdded(postId: $postId) {
    id
    text
    author {
      name
    }
  }
}
```

Every time a new comment is added to that post, the server pushes a new payload matching this shape down the open connection. Server-side, a subscription resolver typically hooks into a pub/sub system (Redis pub/sub, an in-memory event emitter, Kafka) rather than a database query.

## 🟡 Intermediate

### Resolvers: How Execution Actually Works

Each field in the schema has a corresponding **resolver function**. When a query comes in, the GraphQL engine walks the query's structure and, for each requested field, calls that field's resolver — passing along the parent object, the field's arguments, shared context (e.g., the authenticated user, a DB connection), and info about the query.

Conceptually (pseudocode, JS-flavored — exact API varies by library like `graphql-js`, Apollo Server, etc.):

```js
const resolvers = {
  Query: {
    user: (parent, args, context, info) => {
      return db.users.findById(args.id);
    },
    users: (parent, args, context) => {
      return db.users.findMany({ limit: args.limit, offset: args.offset });
    },
  },
  User: {
    // Called for each User object, to resolve its `posts` field
    posts: (parent, args, context) => {
      return db.posts.findByAuthorId(parent.id);
    },
  },
  Post: {
    author: (parent, args, context) => {
      return db.users.findById(parent.authorId);
    },
    commentCount: (parent, args, context) => {
      return db.comments.countByPostId(parent.id);
    },
  },
  Mutation: {
    createUser: (parent, args, context) => {
      return db.users.create({ name: args.name, email: args.email });
    },
  },
};
```

Key things to understand:
- **A resolver runs per field, per object instance.** If a query returns 10 users and asks for each one's `posts`, the `User.posts` resolver runs 10 times — once per user.
- **`parent`** is the already-resolved value of the enclosing field (e.g., inside `Post.author`, `parent` is the `Post` object, so you can read `parent.authorId`).
- If a resolver isn't explicitly defined for a field, most GraphQL implementations fall back to a default resolver that just reads the matching property off the parent object (`parent[fieldName]`) — this is why simple scalar fields on `User` in the example above don't need explicit resolvers.
- Resolvers for independent fields execute concurrently where the runtime supports it (e.g., `Promise.all`-style parallelism in JS implementations), which is part of why GraphQL servers can be efficient despite the tree of function calls.

### The N+1 Query Problem

This is the single most important gotcha to understand before shipping a GraphQL API backed by a database.

Take this query:

```graphql
query {
  posts {
    title
    author {
      name
    }
  }
}
```

With the naive resolvers above: `Query.posts` runs **1** query to fetch, say, 50 posts. Then, for *each* of those 50 posts, `Post.author` resolver runs independently, each firing its own `SELECT * FROM users WHERE id = ?` — that's **50 more queries**. Total: 51 queries to render one screen. This is the **N+1 problem**: 1 query for the list, N queries for each item's related data.

It's not a GraphQL-specific bug conceptually (ORMs have the exact same issue with naive lazy-loading), but GraphQL's resolver-per-field model makes it easy to write code that triggers it without realizing, because each resolver looks innocent in isolation — the N+1 pattern only becomes visible when you look at the aggregate query log for a single request.

### DataLoader: Batching and Caching

**DataLoader** (originally a Facebook library, now the standard pattern regardless of exact implementation/language) fixes this by **batching** and **caching** requests for the same type of data within a single tick of the event loop / a single request.

The core idea:
1. Instead of each resolver hitting the DB immediately, it calls `loader.load(id)`, which returns a promise but doesn't fire the query yet.
2. DataLoader collects all the `.load(id)` calls that happen synchronously (i.e., within the same execution frame, before the event loop yields).
3. Once that batch window closes, DataLoader calls a single **batch function** you provide, passing it the full array of collected keys — this fires exactly one query for all of them (`SELECT * FROM users WHERE id IN (...)`).
4. DataLoader distributes the batched results back to each individual `.load(id)` call's promise.
5. Within the same request, calling `.load(id)` again for an ID already fetched returns the cached result instead of re-querying.

```js
const DataLoader = require('dataloader');

// Batch function: takes an array of user IDs, returns a matching array of users (same order!)
const userLoader = new DataLoader(async (userIds) => {
  const users = await db.users.findByIds(userIds); // ONE query: WHERE id IN (...)
  // DataLoader requires the output array to align 1:1 with the input keys array
  const usersById = {};
  users.forEach(u => { usersById[u.id] = u; });
  return userIds.map(id => usersById[id] || null);
});

const resolvers = {
  Post: {
    author: (parent, args, context) => {
      return context.userLoader.load(parent.authorId); // batched + cached automatically
    },
  },
};
```

With this in place, the same query that previously fired 51 queries now fires **2**: one for the posts, one batched `IN (...)` query for all 50 authors' IDs collected across all 50 `Post.author` resolver calls.

⚠️ **Gotchas with DataLoader**:
- **Create a new DataLoader instance per request**, not a global singleton — otherwise its per-request cache leaks stale data or, worse, leaks one user's data into another user's request in a shared server process.
- The batch function's output array must be the same length as, and in the same order as, the input keys array — DataLoader matches them positionally, not by searching.
- DataLoader only batches calls that happen synchronously before the event loop's microtask queue flushes — if you `await` something before calling `.load()` for a sibling field, it may miss the batch window and fire its own separate batch.

### Pagination Approaches

**Offset-based** (simple, familiar from REST/SQL, but has real problems at scale):

```graphql
query {
  posts(limit: 10, offset: 20) {
    title
  }
}
```

Problems: if items are inserted/deleted between requests, offset pagination can skip or duplicate items across pages ("page drift"). Also gets slower on large offsets in most databases (`OFFSET 100000` still has to scan/skip those rows).

**Cursor-based (Relay-style connections)** — GraphQL's de facto standard for robust pagination, formalized by the [Relay Cursor Connections spec](https://relay.dev/graphql/connections.htm). Instead of a numeric offset, the client passes an opaque cursor (usually a base64-encoded pointer, like an encoded ID or timestamp) marking "give me items after this one":

```graphql
type PostConnection {
  edges: [PostEdge!]!
  pageInfo: PageInfo!
}

type PostEdge {
  cursor: String!
  node: Post!
}

type PageInfo {
  hasNextPage: Boolean!
  hasPreviousPage: Boolean!
  startCursor: String
  endCursor: String
}

type Query {
  posts(first: Int, after: String, last: Int, before: String): PostConnection!
}
```

Query:
```graphql
query {
  posts(first: 10, after: "Y3Vyc29yOjEw") {
    edges {
      cursor
      node {
        title
      }
    }
    pageInfo {
      hasNextPage
      endCursor
    }
  }
}
```

The client stores `endCursor` from the response and passes it as `after` on the next request. Because the cursor is tied to a specific item (not a shifting numeric position), inserts/deletes elsewhere in the list don't cause skips or duplicates. This is the pattern GitHub's GraphQL API and Shopify's Storefront API both use for every list field.

| | Offset-based | Cursor-based (Relay) |
|---|---|---|
| Simplicity | Simple, familiar | More setup (`edges`/`node`/`pageInfo` boilerplate) |
| Jump to arbitrary page | Yes (`offset=500`) | No — sequential only |
| Stable under concurrent writes | No (page drift) | Yes |
| Performance on large datasets | Degrades with large offsets | Consistent (index-seek on cursor value) |
| Standard in GraphQL ecosystem | Less common now | The convention (Relay, GitHub, Shopify) |

## 🔴 Advanced

### GraphQL vs REST: The Real Trade-offs

| Aspect | REST | GraphQL |
|---|---|---|
| Endpoints | Many, resource-shaped (`/users`, `/posts`) | One (`/graphql`) |
| Response shape | Fixed per endpoint | Client-specified per request |
| Over-fetching | Common (fixed fields) | Solved (ask for exactly what you need) |
| Under-fetching | Common (needs follow-up requests) | Solved (nest related data in one query) |
| HTTP caching | Native — GET is cacheable by URL via CDNs/browsers | Hard — everything's usually a POST to the same URL, so standard HTTP caching doesn't apply |
| Versioning | Often needs `/v1`, `/v2` | Usually avoided — add fields, deprecate old ones (`@deprecated`), evolve the schema in place |
| Complexity/cost predictability | Predictable (fixed query per endpoint) | Unpredictable — a single query can be arbitrarily expensive |
| Tooling/introspection | Requires separate OpenAPI spec, can drift from reality | Self-describing — the schema *is* always accurate, since it's what the server actually executes against |
| File uploads | Native (multipart/form-data) | Awkward — not natively part of the spec, needs extensions (`graphql-multipart-request-spec`) or a separate REST endpoint |
| Learning curve | Lower, ubiquitous | Higher — schema design, resolvers, N+1 awareness |

**What you gain**: precise, client-driven data fetching in one round trip; a strongly typed, self-documenting, introspectable contract; smooth schema evolution without versioning; great fit for complex/nested/graph-shaped data (which is exactly why it's popular for apps with rich relational UIs).

**What you lose**: free HTTP caching. A `GET /users/123` REST response can be cached by the browser, a CDN, or an intermediate proxy purely based on the URL and `Cache-Control` headers — no application code involved. A GraphQL request is (almost always) a `POST /graphql` with a query in the body; the URL never changes, so URL-based caching infrastructure is blind to it. You have to build caching yourself, at the application layer (see below).

### Query Complexity / Depth Attacks and Mitigation

Because clients construct arbitrary queries against your schema, a malicious or just poorly written client can ask for something catastrophically expensive:

```graphql
query EvilQuery {
  user(id: "1") {
    friends {
      friends {
        friends {
          friends {
            friends {
              name  # exponential fan-out — could be millions of resolver calls
            }
          }
        }
      }
    }
  }
}
```

Or simply request a huge page size on multiple nested connections at once, or send the same expensive query 1000 times in a single request via aliases:

```graphql
query {
  a1: expensiveField
  a2: expensiveField
  a3: expensiveField
  # ... repeated hundreds of times
}
```

**Mitigations**:
- **Depth limiting** — reject queries nested beyond N levels (e.g., `graphql-depth-limit`). Simple, but can be too blunt for legitimately deep schemas.
- **Query complexity/cost analysis** — assign a "cost" to each field (scalar fields cheap, fields that trigger DB calls or list fields more expensive, often multiplied by requested `limit`/`first` arguments) and reject queries whose total estimated cost exceeds a budget before execution even starts. Both Apollo Server and most major GraphQL server frameworks support cost-analysis plugins/middleware for this.
- **Timeouts** — hard-cap execution time server-side regardless of what the query looks like statically.
- **Pagination limits** — enforce a max `first`/`limit` argument value server-side (don't trust the client's requested page size).
- **Persisted queries / allowlisting** — for production apps with known clients, only allow a pre-registered, hashed set of queries (the client sends a hash, not the full query text) rather than accepting arbitrary query strings from the wire. This closes off the "arbitrary query" attack surface entirely for internal/first-party clients, at the cost of flexibility for ad-hoc/public API consumers.
- **Rate limiting** — still applies, just needs to be cost-aware rather than purely request-count-based, since one GraphQL request can do the work of 50 REST requests.

### Caching Difficulty vs REST

REST's caching story is essentially free: `GET /users/123` with `Cache-Control: max-age=3600` — any CDN, reverse proxy, or browser cache honors it without your application knowing or caring. GraphQL breaks this because:
1. Requests are typically POST (not cacheable by HTTP semantics at all — POST responses aren't cached by default).
2. Every request goes to the same URL (`/graphql`), so URL-keyed caches can't differentiate one query from another.
3. Two different queries can return overlapping data (e.g., `user(id: "1") { name }` and `user(id: "1") { name email }`) that a naive cache can't recognize as related.

**How production GraphQL APIs actually solve this**:
- **Persisted queries over GET** — if the query is a pre-registered hash sent as a GET query parameter (`GET /graphql?queryId=abc123&variables=...`), it becomes URL-cacheable again by CDNs, recovering standard HTTP caching.
- **Normalized client-side caching** — Apollo Client and Relay cache individual objects by type+ID (e.g., `User:123`) in a normalized store, not by request/response shape. When any query returns a `User:123`, the cache updates that one entry, and every component reading `User:123` (even from a different query) sees the update — this is a different caching model than HTTP caching, solving staleness at the client rather than the network layer.
- **Server-side response/field caching** — cache at the resolver or data-source level (e.g., cache the result of `db.users.findById` in Redis) rather than caching the whole GraphQL response.
- **Automatic Persisted Queries (APQ)** — Apollo's specific protocol combining query hashing with a fallback to registering unseen queries on the fly.

### Authentication and Authorization

Handled in resolver `context`, not via URL-based route guards like REST middleware:

```js
const server = new ApolloServer({
  typeDefs,
  resolvers,
  context: ({ req }) => {
    const token = req.headers.authorization || '';
    const user = verifyToken(token); // null if invalid/absent
    return { user, dataSources: { userLoader, postLoader } };
  },
});

const resolvers = {
  Mutation: {
    deletePost: (parent, args, context) => {
      if (!context.user) throw new Error('Not authenticated');
      if (!context.user.canDelete(args.id)) throw new Error('Forbidden');
      return db.posts.delete(args.id);
    },
  },
};
```

Field-level authorization is possible too — a resolver for a sensitive field (e.g., `User.email`) can check `context.user` and return `null` or throw, independent of whether the rest of the query succeeds.

### Error Handling

GraphQL always returns HTTP 200 in most implementations (even for application-level errors) — errors are communicated in the response body's `errors` array, separate from `data`, and a response can contain *both* partial data and errors simultaneously (e.g., one field failed but sibling fields resolved fine):

```json
{
  "data": {
    "user": {
      "name": "Alice",
      "email": null
    }
  },
  "errors": [
    {
      "message": "Not authorized to view email",
      "path": ["user", "email"],
      "extensions": { "code": "FORBIDDEN" }
    }
  ]
}
```

This is a deliberate design choice — it lets clients render whatever data did come back rather than treating the entire response as failed. It does mean clients can't rely on HTTP status codes for error handling the way they would with REST; they must inspect the `errors` array.

### Schema Design Best Practices

- **Design for the client, not the database.** Don't just mirror your DB tables 1:1 into GraphQL types — model the schema around how consumers actually think about and query the data.
- **Use `@deprecated` instead of versioning**: `field: String @deprecated(reason: "Use newField instead")` — deprecated fields keep working while signaling migration paths, avoiding the `/v1` vs `/v2` split REST APIs often need.
- **Prefer non-null (`!`) by default where a field logically can't be absent** — it pushes null-handling responsibility to the resolver/data layer instead of every client having to defensively null-check everything.
- **Use input types for mutation arguments** once they grow past 2-3 fields, for readability and reuse:
  ```graphql
  input CreatePostInput {
    title: String!
    content: String!
    tags: [String!]
  }
  type Mutation {
    createPost(input: CreatePostInput!): Post!
  }
  ```
- **Avoid deeply nested mutations** — keep mutations focused on one primary side effect; compose multiple mutations client-side rather than building one giant mutation that does everything.

## Real-World Examples

### GitHub API v4 (GraphQL)
GitHub's v4 API is GraphQL, explicitly built alongside their existing REST v3 API, aimed at clients that need to pull related data (a repo, its issues, each issue's comments and labels) in one request instead of chaining REST calls. It uses Relay-style cursor pagination throughout (`first`, `after`, `edges`, `node`, `pageInfo`) and enforces a **point-based query cost limit** per request/hour rather than a flat request-count rate limit — exactly the "cost analysis" mitigation described above, because a GraphQL query's cost varies wildly by shape.

```graphql
query {
  repository(owner: "octocat", name: "Hello-World") {
    issues(first: 10, states: OPEN) {
      edges {
        node {
          title
          comments(first: 5) {
            nodes {
              body
              author { login }
            }
          }
        }
      }
    }
  }
}
```

### Shopify Storefront API
Shopify's Storefront API is GraphQL-first for building custom storefronts — fetching a product, its variants, images, and pricing in one query instead of assembling it from multiple REST calls. Like GitHub, it uses Relay connections for pagination and enforces query-cost-based rate limiting (`requestedQueryCost` / `throttleStatus` in every response) rather than a simple per-minute request cap.

### Others worth knowing
- **Facebook/Meta** — the origin of GraphQL, still used extensively across their app graph.
- **Netflix** (via GraphQL Federation-style patterns), **Airbnb**, **Twitter (X)** internal APIs — all adopted GraphQL for mobile-client data-fetching efficiency reasons similar to Facebook's original motivation.

## Common Pitfalls / Gotchas

- **Not solving N+1 queries** — the single most common production issue; always design with DataLoader-style batching in mind from the start, not as an afterthought once a query is slow.
- **Trusting query complexity implicitly** — without depth/cost limiting, your schema is an open invitation to accidental or malicious resource exhaustion.
- **Treating GraphQL as automatically faster than REST** — it isn't; it reduces round trips and payload waste, but a badly resolved GraphQL query (N+1, no caching) can be *slower* than a well-designed set of REST endpoints.
- **Ignoring HTTP caching loss** — teams that migrate from REST to GraphQL and don't build a replacement caching strategy (persisted queries, normalized client cache, server-side field caching) often see a real performance regression at the CDN/edge layer.
- **Overusing nullable fields** — making everything nullable "to be safe" pushes tedious null-checks onto every client and loses the schema's ability to express real guarantees.
- **No file upload story out of the box** — plan for this early (multipart spec extension or a separate signed-upload-URL REST/pre-signed-S3 flow) rather than fighting the spec later.
- **Forgetting mutations aren't guaranteed sequential across implementations/transports** — don't assume ordering guarantees you haven't verified for your specific server.
- **Subscriptions at scale** — a naive subscription implementation (in-memory pub/sub) doesn't survive horizontal scaling across multiple server instances; production setups typically need a shared pub/sub backend (Redis, Kafka) so an event published on one instance reaches subscribers connected to another.

## Quick Reference

| Concept | Purpose |
|---|---|
| Query | Read data |
| Mutation | Write data (create/update/delete) |
| Subscription | Real-time push over a persistent connection |
| Schema (SDL) | Typed contract: types, queries, mutations available |
| Resolver | Function that fetches the value for one field |
| `context` | Shared per-request data (auth, loaders, DB connections) passed to every resolver |
| DataLoader | Batches + caches per-request data fetches to avoid N+1 |
| Fragment | Reusable named set of fields |
| Variables | Parameterize a query without string interpolation |
| Relay connection (`edges`/`node`/`pageInfo`) | Standard cursor-based pagination pattern |
| `@deprecated` | Mark a field as deprecated without breaking clients (avoids versioning) |
| Persisted queries | Pre-registered, hashed queries — restores cacheability + limits attack surface |

## Further Reading
- [graphql.org](https://graphql.org) — the official spec and docs.
- [Relay Cursor Connections Specification](https://relay.dev/graphql/connections.htm)
- [GitHub GraphQL API docs](https://docs.github.com/en/graphql)
- [Shopify Storefront API docs](https://shopify.dev/docs/api/storefront)
- `graphql-depth-limit`, `graphql-cost-analysis` — common complexity-mitigation libraries in the Node ecosystem.
- Apollo Server / Apollo Client docs — widely used reference implementation for both server and normalized client caching.
