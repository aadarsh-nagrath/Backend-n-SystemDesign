# GraphQL: A Complete Guide for Backend Engineers

## Table of Contents
1. [What GraphQL Is (and Isn't)](#what)
2. [The Type System and SDL](#types)
3. [Queries, Mutations, Subscriptions](#operations)
4. [How Execution Works: Resolvers](#execution)
5. [The N+1 Problem and DataLoader](#n1)
6. [Schema Design Best Practices](#design)
7. [Pagination: Relay Connections](#pagination)
8. [Errors](#errors)
9. [Authentication and Authorization](#auth)
10. [Security: Depth, Complexity, Introspection, Persisted Queries](#security)
11. [Caching](#caching)
12. [Federation and Schema Stitching](#federation)
13. [Subscriptions and Real-Time](#subscriptions)
14. [File Uploads](#uploads)
15. [Tooling and Server Implementations](#tooling)
16. [GraphQL vs REST vs gRPC](#compare)
17. [Example: Node.js (Apollo Server) End to End](#example)
18. [Interview Questions](#qa)

---

## 1. What GraphQL Is {#what}

GraphQL is a **query language for APIs and a runtime for executing those queries** against a typed schema. Facebook built it in 2012 for the News Feed mobile app and open-sourced it in 2015. It's now stewarded by the GraphQL Foundation (Linux Foundation).

Core ideas:
- **One endpoint** (usually `POST /graphql`). The client sends a document describing exactly the data it wants.
- **Client-specified response shape**: no over-fetching (getting fields you don't need) and no under-fetching (needing several round trips).
- **Strongly typed schema** = a contract, introspectable at runtime, which enables codegen, IDE autocomplete, and validation before execution.
- **Graph traversal**: follow relationships in one request (`user → posts → comments → author`).

What it is **not**: it's not a database query language (it doesn't talk to your DB by itself, because resolvers do), it's not tied to any storage, and it's not automatically faster than REST.

```graphql
query {
  user(id: "42") {
    name
    posts(first: 3) {
      edges { node { title commentCount } }
    }
  }
}
```
```json
{ "data": { "user": { "name": "Asha", "posts": { "edges": [ { "node": { "title": "MVCC", "commentCount": 4 } } ] } } } }
```

---

## 2. Type System and SDL {#types}

```graphql
scalar DateTime            # custom scalar
scalar URL

enum OrderStatus { PENDING PAID SHIPPED CANCELLED }

interface Node { id: ID! }                   # Relay global object identification

type User implements Node {
  id: ID!
  email: String!
  name: String
  orders(first: Int = 10, after: String, status: OrderStatus): OrderConnection!
  createdAt: DateTime!
}

type Order implements Node {
  id: ID!
  status: OrderStatus!
  total: Money!
  items: [OrderItem!]!       # non-null list of non-null items
  customer: User!
}

type Money { amount: String!, currency: String! }   # string/decimal for money, never Float

union SearchResult = User | Order | Product

input CreateOrderInput {
  items: [OrderItemInput!]!
  couponCode: String
  clientMutationId: String
}

type Query {
  node(id: ID!): Node
  me: User
  order(id: ID!): Order
  search(term: String!): [SearchResult!]!
}

type Mutation {
  createOrder(input: CreateOrderInput!): CreateOrderPayload!
}

type Subscription {
  orderStatusChanged(orderId: ID!): Order!
}

directive @auth(requires: Role = USER) on FIELD_DEFINITION | OBJECT
```
- Built-in scalars: `Int` (32-bit signed!), `Float`, `String`, `Boolean`, `ID` (serialized as a string). Use custom scalars for `DateTime`, `BigInt`, `Decimal`, `JSON`, `Email`, `URL`.
- **Nullability**: `String` is nullable, `String!` non-null. `[Post!]!` is a non-null list whose items are non-null. If a non-null field resolves to null or errors, the null **bubbles up** to the nearest nullable parent. Make fields nullable where partial failure is acceptable.
- **Interfaces and unions** for polymorphism. Query them with fragments: `... on User { email }`.
- **Input types** for arguments (they can't contain output types).
- **Directives**: `@include(if:)`, `@skip(if:)`, `@deprecated(reason:)`, `@specifiedBy`, `@oneOf` (input unions), plus custom ones.
- **Schema-first** (write SDL, attach resolvers: Apollo, graphql-tools) vs **code-first** (generate SDL from code: Pothos, Nexus, TypeGraphQL, Strawberry, Hot Chocolate, gqlgen is schema-first in Go).

---

## 3. Operations {#operations}

```graphql
# Named query with variables and fragments
query OrderPage($id: ID!, $withItems: Boolean = true) {
  order(id: $id) {
    ...OrderSummary
    items @include(if: $withItems) { sku quantity }
  }
}
fragment OrderSummary on Order { id status total { amount currency } }

# Mutation: executed serially (top-level mutation fields run in order)
mutation Checkout($input: CreateOrderInput!) {
  createOrder(input: $input) {
    order { id status }
    userErrors { field message code }
  }
}

# Subscription: long-lived stream
subscription { orderStatusChanged(orderId: "o_1") { id status } }
```
- Queries' top-level fields may execute **in parallel**. Mutations' top-level fields execute **sequentially**.
- **Aliases** fetch the same field with different args: `small: avatar(size: 64)  large: avatar(size: 512)`.
- Over HTTP: `POST` with JSON `{ "query", "variables", "operationName" }`. GET is allowed for queries (useful for CDN caching with persisted queries). The GraphQL-over-HTTP spec standardizes `application/graphql-response+json`.

---

## 4. Execution and Resolvers {#execution}

Phases: **parse** → **validate** against the schema (types, fields exist, variables match; this happens *before* any business code runs) → **execute**.

Each field has a **resolver**: `(parent, args, context, info) => value | Promise`.
- `parent`: the result of the parent field.
- `args`: field arguments.
- `context`: per-request shared object (authenticated user, DataLoaders, DB connections, request ID). **Create it per request.**
- `info`: AST/path information (useful for lookahead/projections).

Default resolver: returns `parent[fieldName]`. Execution walks the query tree breadth-wise. Each level's resolvers run, and child resolvers run for each item returned.

```js
const resolvers = {
  Query: {
    order: (_, { id }, ctx) => ctx.db.orders.findById(id),
  },
  Order: {
    customer: (order, _, ctx) => ctx.loaders.userById.load(order.customerId),   // batched
    total: (order) => ({ amount: order.totalMinor / 100 + '', currency: order.currency }),
  },
};
```

---

## 5. N+1 and DataLoader {#n1}

Query: 50 orders, each with its customer.
- Naive: 1 query for orders + **50 queries** for customers (each `Order.customer` resolver fetches individually).

**DataLoader** (Facebook pattern, implemented in every language):
- Collects all `.load(key)` calls made in the same tick of the event loop (or the same execution level), then calls your **batch function once** with all the keys: `SELECT * FROM users WHERE id = ANY($1)`.
- Caches per request (deduplicates the same key).
- The batch function must return results **in the same order as the keys**, with null or Error for missing ones.

```js
const userById = new DataLoader(async (ids) => {
  const rows = await db.query('SELECT * FROM users WHERE id = ANY($1)', [ids]);
  const map = new Map(rows.map(r => [r.id, r]));
  return ids.map(id => map.get(id) ?? null);
});
```
Create loaders **per request**, never globally, to avoid leaking data between users and stale caching.

Other approaches: query planning/lookahead to generate SQL joins (Join Monster, PostGraphile, Hasura compile GraphQL to a single SQL query), and `@defer`/`@stream` to deliver slow parts later.

---

## 6. Schema Design Best Practices {#design}

1. **Design for the client's domain**, not your database tables. Expose business concepts.
2. **Nullable by default** for fields backed by other services (partial results beat total failure). Use non-null for identity fields and guaranteed data.
3. **Mutations**: one input object and one payload type per mutation (`createOrder(input: CreateOrderInput!): CreateOrderPayload!`). The payload contains the changed object(s) + **userErrors** for expected business errors. Specific mutations (`cancelOrder`, `addItemToCart`) beat generic `updateOrder` with 30 optional fields.
4. **Global IDs** (opaque, type-encoded, e.g., base64 `Order:123`) + the `node(id:)` query enable refetching and client caches (Relay, Apollo).
5. **Evolve, don't version**: add fields freely. Deprecate with `@deprecated(reason: "Use fullName")`, monitor field usage, and remove after clients migrate. Avoid `/v2/graphql`.
6. **Avoid breaking changes**: removing fields, changing types, making nullable → non-null on inputs or non-null → nullable on outputs, and adding required arguments all break clients. Tools like GraphQL Inspector and Apollo schema checks run in CI.
7. **Pagination on every list** that can grow.
8. **Naming**: camelCase fields, PascalCase types, SCREAMING_CASE enums, verbs for mutations.
9. Don't expose raw DB IDs or internal implementation details.
10. Money as a decimal string or minor units with currency; timestamps as ISO-8601 custom scalars.

---

## 7. Pagination: Relay Connections {#pagination}

```graphql
type OrderConnection {
  edges: [OrderEdge!]!
  pageInfo: PageInfo!
  totalCount: Int          # optional; can be expensive
}
type OrderEdge { cursor: String!, node: Order! }
type PageInfo { hasNextPage: Boolean!, hasPreviousPage: Boolean!, startCursor: String, endCursor: String }

# usage
orders(first: 20, after: "Y3Vyc29yOjIwMjUtMDYtMDFUMTA6MDA6MDBaOjkwMDE=") { edges { cursor node { id } } pageInfo { hasNextPage endCursor } }
```
Cursors are opaque (base64 of the sort key + id), and the backend implements **keyset pagination** (see `databases/fundamentals/02-sql-deep-dive.md`). Fetch `first + 1` rows to compute `hasNextPage`. Cap `first` (e.g., max 100).

---

## 8. Errors {#errors}

GraphQL responses typically return **HTTP 200** even with errors (partial success). The response has `data` and an optional `errors` array:
```json
{
  "data": { "order": null },
  "errors": [{
    "message": "Order not found",
    "path": ["order"],
    "locations": [{ "line": 2, "column": 3 }],
    "extensions": { "code": "NOT_FOUND" }
  }]
}
```
Two categories:
1. **Top-level `errors`**: unexpected or system failures (resolver threw, downstream down, auth failure). Use `extensions.code` (`UNAUTHENTICATED`, `FORBIDDEN`, `BAD_USER_INPUT`, `INTERNAL_SERVER_ERROR`). Mask internal messages in production.
2. **Errors as data**: expected domain outcomes in the schema (`userErrors` in payloads, or result unions: `union CreateOrderResult = CreateOrderSuccess | OutOfStock | InvalidCoupon`). They're typed, discoverable, and force clients to handle them.

Validation errors (bad query) return before execution with no `data`. Over HTTP with the `application/graphql-response+json` media type, these get a 4xx.

Monitoring caveat: since everything is 200, **HTTP-status-based alerting misses GraphQL errors**. Instrument at the GraphQL layer (error rate per operation name and field).

---

## 9. AuthN and AuthZ {#auth}

- **Authentication** at the transport layer (HTTP middleware): validate the JWT, session, or API key, and put the user in `context`. GraphQL doesn't define auth.
- **Authorization** in the **business/domain layer**, not scattered in resolvers. The "single source of truth" principle: the same rule applies whether data is reached via `order(id)`, `user.orders`, or `node(id)`. Check access for every object returned (the IDOR risk is higher in GraphQL because there are many paths to the same object).
- Field-level authorization (e.g., `email` visible only to the user themself or admins) via directives (`@auth(requires: ADMIN)`), schema wrappers (graphql-shield), or in resolvers. Return null + error, or omit.
- Rate limiting must consider **query cost**, not just request count.

---

## 10. Security {#security}

GraphQL's flexibility is an attack surface:
| Threat | Mitigation |
|---|---|
| Deeply nested queries (`friends { friends { friends { … } } }`) → exponential work | **Max depth** limit (e.g., 10) |
| Expensive wide queries (`first: 10000` × nested lists) | **Query cost/complexity analysis**: assign cost per field and multiply by list sizes, reject over budget. Cap `first`/`last` arguments |
| Alias-based batching attacks (1000 aliases of `login(...)` to brute force in one request) | Limit aliases or root fields per operation; rate-limit by cost/operation |
| Batched operations arrays | Limit batch size |
| Introspection reveals the whole schema to attackers | Disable in production for private APIs (or restrict to authenticated developers). It's not real security, only reduced reconnaissance |
| Arbitrary queries from unknown clients | **Persisted queries / operation allowlists**: clients send a hash of a pre-registered query and the server rejects anything else. Strongest control for first-party clients |
| Field suggestion leaks ("Did you mean `passwordHash`?") | Disable suggestions in production |
| Injection in resolvers | Same as REST: parameterized queries, input validation |
| DoS via slow resolvers | Timeouts per resolver/request, circuit breakers on downstreams |
| CSRF on GET/simple POST | Require `Content-Type: application/json` or a CSRF header (Apollo's CSRF prevention) |

---

## 11. Caching {#caching}

- **HTTP caching is harder**: one POST endpoint means CDNs can't cache by URL by default. Solutions: **automatic persisted queries (APQ) over GET** (`/graphql?extensions={"persistedQuery":{"sha256Hash":"..."}}&variables=...`), so CDNs can cache by URL. Add `Cache-Control` hints per type and field (`@cacheControl(maxAge: 60, scope: PUBLIC)`), and the server computes the minimum maxAge for the response.
- **Server-side**: DataLoader (per-request), resolver-level caching (Redis) keyed by entity ID, full response caching for public queries (Apollo response cache plugin, GraphQL Yoga/Envelop response cache, Stellate edge cache).
- **Client-side normalized caches**: Apollo Client and Relay store objects by `__typename` + `id`, so updating one entity updates every view. This is a major productivity win and requires stable IDs.

---

## 12. Federation and Stitching {#federation}

With many services, you can expose one graph:
- **Apollo Federation (v2)**: each **subgraph** service owns part of the schema and can **extend entities** owned by others via `@key` directives. A **router/gateway** (Apollo Router in Rust, Cosmo Router, Hive Gateway, Grafbase) builds a **query plan**: it fetches from subgraphs, and resolves entity references with the `_entities` query, which is like a distributed join.
  ```graphql
  # orders subgraph
  type Order @key(fields: "id") { id: ID!, total: Money!, customer: User! }
  type User @key(fields: "id", resolvable: false) { id: ID! }
  # users subgraph
  type User @key(fields: "id") { id: ID!, name: String! }
  ```
- **Schema stitching** (graphql-tools): the older, more manual way of merging schemas.
- **Composite schemas spec** (GraphQL Foundation working group) aims to standardize federation.
- Governance: schema registry, composition checks in CI, ownership per type and field, usage-based breaking change detection.
- Pitfalls: query plans with many sequential subgraph hops (latency), entity resolution N+1 across services (batch `_entities`), and distributed tracing becomes essential.
- **BFF (backend-for-frontend)** is often what GraphQL effectively becomes: an aggregation layer over REST and gRPC microservices.

---

## 13. Subscriptions and Real-Time {#subscriptions}

- Transport: **WebSocket** with the `graphql-ws` protocol (the old `subscriptions-transport-ws` is deprecated), or **SSE** (graphql-sse), or multipart HTTP (Apollo's incremental delivery).
- Server: a resolver returns an **AsyncIterator**, backed by a pub/sub system (in-memory for a single node, **Redis Pub/Sub, Kafka, NATS** for multiple instances).
- Scaling: stateful connections need sticky sessions or connection-aware LB, horizontal fan-out via the pub/sub, auth on connect (`connection_init` payload) and re-auth on token expiry, plus backpressure and limits per connection.
- Alternatives: **live queries** (`@live`, re-run the query when dependencies change), polling (simple, often good enough), and `@defer`/`@stream` for incremental delivery within one request.

---

## 14. File Uploads {#uploads}

The GraphQL multipart request spec (`graphql-upload`) exists but has CSRF and complexity issues. **Preferred**: a mutation returns a **pre-signed S3 URL**, the client uploads directly to object storage, then calls a mutation with the file key. This keeps large binaries off the GraphQL server.

---

## 15. Tooling and Implementations {#tooling}

| Language | Servers |
|---|---|
| JavaScript/TypeScript | Apollo Server, GraphQL Yoga (The Guild), Mercurius (Fastify), graphql-js (reference), Pothos (code-first), NestJS GraphQL |
| Java/Kotlin | Spring for GraphQL, Netflix DGS, graphql-java, Expedia graphql-kotlin |
| Python | Strawberry, Graphene, Ariadne |
| Go | gqlgen, graphql-go |
| .NET | Hot Chocolate |
| Ruby | graphql-ruby (used by GitHub, Shopify) |
| Rust | async-graphql, Juniper |
| Instant APIs over DBs | Hasura, PostGraphile, pg_graphql (Supabase), AWS AppSync |

Clients: Apollo Client, Relay, urql, graphql-request, Apollo Kotlin/iOS. Codegen: GraphQL Code Generator. IDEs: GraphiQL, Apollo Sandbox, Altair. Linting: graphql-eslint. Observability: Apollo GraphOS, GraphQL Hive, OpenTelemetry instrumentation (spans per operation and resolver).

---

## 16. GraphQL vs REST vs gRPC {#compare}

| | REST | GraphQL | gRPC |
|---|---|---|---|
| Shape | Server-defined resources | Client-defined per query | Server-defined RPC messages |
| Transport | HTTP/1.1+, JSON | HTTP (usually POST), JSON | HTTP/2, Protobuf (binary) |
| Over/under-fetching | Common | Solved | Fixed messages |
| Caching | **Excellent** (HTTP semantics, CDN) | Harder (persisted queries/GET) | Custom |
| Typing/contract | Optional (OpenAPI) | Built-in schema | Built-in (.proto) |
| Versioning | URL/header versions | Schema evolution + deprecation | Field numbers, backward compatible |
| Real-time | SSE/WebSocket separate | Subscriptions | Bidirectional streaming |
| Error model | HTTP status codes | 200 + errors array | Status codes + details |
| Best for | Public APIs, CRUD, cacheable resources, simplicity | Client-driven UIs (mobile/web) aggregating many sources, rapidly evolving frontends | Internal service-to-service, low latency, polyglot, streaming |
| Pain points | Chattiness, many endpoints for UIs | Complexity, N+1, security/cost control, caching | Browser support (needs gRPC-Web/Connect), debugging binary |

Common architecture: **gRPC between internal services, GraphQL BFF for frontends, REST for public/partner APIs and webhooks.** See [`grpc.md`](grpc.md) and [`restapi.md`](restapi.md).

---

## 17. Example: Apollo Server + Postgres + DataLoader {#example}

```ts
import { ApolloServer } from '@apollo/server';
import { startStandaloneServer } from '@apollo/server/standalone';
import DataLoader from 'dataloader';
import depthLimit from 'graphql-depth-limit';
import pg from 'pg';

const pool = new pg.Pool({ max: 10 });

const typeDefs = /* GraphQL */ `
  type User { id: ID!, name: String!, orders(first: Int = 10): [Order!]! }
  type Order { id: ID!, status: String!, totalMinor: Int!, customer: User! }
  type Query { me: User, order(id: ID!): Order }
  type Mutation { cancelOrder(id: ID!): CancelOrderPayload! }
  type CancelOrderPayload { order: Order, userErrors: [UserError!]! }
  type UserError { field: [String!], message: String!, code: String! }
`;

const resolvers = {
  Query: {
    me: (_, __, ctx) => (ctx.userId ? ctx.loaders.user.load(ctx.userId) : null),
    order: async (_, { id }, ctx) => {
      const { rows } = await pool.query('SELECT * FROM orders WHERE id = $1', [id]);
      const order = rows[0];
      if (!order || order.customer_id !== ctx.userId) return null;     // authorization in one place (ideally a service layer)
      return order;
    },
  },
  User: {
    orders: async (user, { first }) => {
      const { rows } = await pool.query(
        'SELECT * FROM orders WHERE customer_id = $1 ORDER BY created_at DESC LIMIT $2', [user.id, Math.min(first, 100)]);
      return rows;
    },
  },
  Order: {
    totalMinor: (o) => o.total_minor,
    customer: (o, _, ctx) => ctx.loaders.user.load(o.customer_id),
  },
  Mutation: {
    cancelOrder: async (_, { id }, ctx) => {
      const { rows } = await pool.query(
        `UPDATE orders SET status = 'cancelled' WHERE id = $1 AND customer_id = $2 AND status = 'pending' RETURNING *`,
        [id, ctx.userId]);
      if (!rows[0]) return { order: null, userErrors: [{ field: ['id'], message: 'Order cannot be cancelled', code: 'NOT_CANCELLABLE' }] };
      return { order: rows[0], userErrors: [] };
    },
  },
};

const server = new ApolloServer({ typeDefs, resolvers, validationRules: [depthLimit(8)], introspection: process.env.NODE_ENV !== 'production' });

await startStandaloneServer(server, {
  context: async ({ req }) => {
    const userId = await authenticate(req.headers.authorization);   // verify JWT
    return {
      userId,
      loaders: {
        user: new DataLoader(async (ids) => {
          const { rows } = await pool.query('SELECT * FROM users WHERE id = ANY($1)', [ids]);
          const byId = new Map(rows.map((r) => [String(r.id), r]));
          return ids.map((id) => byId.get(String(id)) ?? null);
        }),
      },
    };
  },
  listen: { port: 4000 },
});
```

---

## 18. Interview Questions {#qa}

1. What problems does GraphQL solve compared to REST, and what new problems does it introduce?
2. Explain the N+1 problem in GraphQL and how DataLoader solves it. Why must loaders be per request?
3. How do you protect a GraphQL API from malicious queries?
4. How do you handle errors: top-level errors vs errors-as-data?
5. How do you version a GraphQL API?
6. How does pagination work with Relay connections, and how is it implemented on the DB?
7. How does Apollo Federation compose a graph from multiple services?
8. Why is HTTP caching harder with GraphQL, and how do persisted queries help?
9. Where should authorization logic live in a GraphQL server?
10. How would you scale GraphQL subscriptions across multiple server instances?
