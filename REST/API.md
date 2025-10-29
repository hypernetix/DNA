# Hypernetix REST API Guideline

This document is an actionable, LLM-friendly playbook for building consistent, evolvable REST APIs

## Table of Contents

### Core Framework
- [1. Core Principles](#1-core-principles)
- [2. Protocol & Content](#2-protocol--content)
- [3. Resource Modeling & URLs](#3-resource-modeling--urls)
- [4. JSON Conventions](#4-json-conventions)
- [5. Pagination, Filtering, Sorting](#5-pagination-filtering-sorting)
- [6. Request Semantics](#6-request-semantics)
- [7. Error Model (Problem Details)](#7-error-model-problem-details)

### Advanced Patterns
- [8. Concurrency & Idempotency](#8-concurrency--idempotency)
- [9. Authentication & Authorization](#9-authentication--authorization)
- [10. Rate Limiting & Quotas](#10-rate-limiting--quotas)
- [11. Asynchronous Operations](#11-asynchronous-operations)
- [12. Webhooks (Outbound)](#12-webhooks-outbound)

### Implementation Details
- [13. Internationalization, Numbers & Time](#13-internationalization-numbers--time)
- [14. Caching](#14-caching)
- [15. Security & CORS](#15-security--cors)
- [16. Observability & Diagnostics](#16-observability--diagnostics)
- [17. Versioning & Deprecation](#17-versioning--deprecation)

### Reference
- [18. Canonical Status Codes](#18-canonical-status-codes)
- [19. Batch & Bulk](#19-batch--bulk)
- [20. OpenAPI & Codegen](#20-openapi--codegen)
- [21. Example Endpoints](#21-example-endpoints)
- [22. Backward Compatibility Rules](#22-backward-compatibility-rules-client-facing)
- [23. Performance & DoS](#23-performance--dos)
- [24. Documentation Style](#24-documentation-style)

### Alternative API Patterns
- [25. GraphQL](#25-graphql)
- [26. gRPC & Async APIs](#26-grpc--async-apis)
- [27. AsyncAPI & Event-Driven](#27-asyncapi--event-driven)

### Quick Reference
- [Operational Headers](#operational-headers-quick-reference)
- [References](#references)

## 1. Core Principles
- **Consistency over novelty**: Prefer one clear way to do things.
- **Explicitness**: Always specify types, units, timezones, and defaults.
- **Evolvability**: Versioned paths, idempotency, and forward-compatible schemas.
- **Observability**: Every request traceable end-to-end.
- **Security first**: HTTPS only, least privilege, safe defaults.

## 2. Protocol & Content
- **Media type**: `application/json; charset=utf-8` (request & response)
- **Errors**: Problem Details `application/problem+json` (RFC 9457)
- **Encoding**: UTF-8
- **Compression**: gzip/br when client sends `Accept-Encoding`
- **Idempotency**: `Idempotency-Key` header on unsafe methods (see §7)

## 3. Resource Modeling & URLs
- **Nouns, plural**: `/users`, `/tickets`, `/tickets/{ticketId}`
- **Hierarchy if strict ownership**: `/users/{userId}/keys`
- **Prefer top-level + filters** over deep nesting: `/tickets?assigneeId=...`
- **Identifiers**: `uuidv7` (or `ulid`). JSON field: `id`
- **Timestamps**: ISO-8601 UTC with `Z`, always include milliseconds `.SSS` (e.g., `2025-09-01T20:00:00.000Z`)
- **Standard fields**: `createdAt`, `updatedAt`, optional `deletedAt`

## 4. JSON Conventions
- **Naming**: camelCase (ergonomic in React/TS)
- **Nullability**: Prefer omitting absent fields over `null`
- **Booleans & enums**: Strongly typed; never stringly booleans
- **Money**: Integer minor units + currency code
- **Lists**: Arrays; use `[]` not `null`
- **Envelope**:

```json
{
  "data": { /* object or array */ },
  "meta": { /* optional paging, totals */ },
  "links": { /* optional self,next,prev */ }
}
```

## 5. Pagination, Filtering, Sorting

For complete cursor pagination specification including OData-style filtering and sorting, see [PAGINATION.md](PAGINATION.md).

## 6. Request Semantics
- **Create**: `POST /tickets` → 201 + `Location` + resource in body
- **Partial update**: `PATCH /tickets/{id}` (JSON Merge Patch)
- **Replace**: `PUT /tickets/{id}` (complete representation)
- **Delete**: `DELETE /tickets/{id}` → 204; if soft-delete, return 200 with `deletedAt`

## 7. Error Model (Problem Details)
- **Always** return RFC 9457 Problem Details for 4xx/5xx

```json
{
  "type": "https://api.example.com/errors/validation",
  "title": "Invalid request",
  "status": 422,
  "detail": "email is invalid",
  "instance": "https://api.example.com/req/01J...Z",
  "errors": [
    { "field": "email", "code": "format", "message": "must be a valid email" }
  ],
  "traceId": "01J...Z"
}
```

- **Mappings**: 422 (validation), 401/403 (authz), 404, 409 (conflict), 429, 5xx (no internals)

## 8. Concurrency & Idempotency
- **Optimistic locking**: Representations carry `ETag` (strong or weak). Clients send `If-Match`. On mismatch → 412.
- **Idempotency**: Clients may send `Idempotency-Key` on `POST/PATCH/DELETE`.
  - Server stores `key + request hash` for 1h and returns the original response for retries.
  - Replays add header: `Idempotency-Replayed: true`.

### Retry Strategies

**Client-Side Retries**:
- **Exponential backoff**: Start with 1s, double each retry, max 30s
- **Jitter**: Add random ±25% to prevent thundering herd
- **Max attempts**: 3-5 retries for transient errors (5xx, 429, 408, 504)
- **Circuit breaker**: Stop retrying after consecutive failures

**Retry-After Header**:
- Server includes `Retry-After: 30` (seconds) on 429/503
- Client MUST respect this value before retrying
- For 503, also check `Retry-After` in response body

**Idempotency with Retries**:
- Always include `Idempotency-Key` for state-changing operations
- Server deduplicates by `key + request hash`
- Retry with same key returns original response
- Different request with same key → 409 Conflict

**Timeout Handling**:
- Client timeout: 30s for most operations, 5min for async jobs
- Server timeout: 30s handler timeout, use async jobs for longer work
- Connection timeout: 10s for initial connection

## 9. Authentication & Authorization

### Authentication Methods

**OAuth2/OIDC (Primary)**:
- Bearer tokens in `Authorization: Bearer <token>`
- Short-lived access tokens (15-60min)
- Refresh tokens for long-term access
- PKCE for public clients
- Authorization Code flow for web apps
- Client Credentials flow for service-to-service
- Token introspection endpoint for validation

**API Keys (Alternative)**:
- Header: `X-API-Key: <key>` or `Authorization: ApiKey <key>`
- Query param: `?api_key=<key>` (less secure, avoid if possible)
- Format: `ak_live_` or `ak_test_` prefix for environment identification
- Scoped to specific resources/operations
- Include creation date and expiration in key metadata
- Support key rotation without downtime

**JWT Tokens**:
- Self-contained tokens with claims
- Verify signature and expiration
- Include `sub` (subject), `aud` (audience), `exp` (expiration), `iat` (issued at)
- Use RS256 or ES256 for signature verification
- Include `jti` (JWT ID) for revocation tracking
- Validate `nbf` (not before) claim if present

**mTLS (Service-to-Service)**:
- Mutual TLS for service authentication
- Client certificates for service identity
- Certificate-based authorization
- Certificate pinning for enhanced security
- Automatic certificate rotation

### Authorization

**Scopes & Permissions**:
- Document required scopes per endpoint
- Insufficient permissions → 403 Forbidden
- Scope format: `resource:action` (e.g., `users:read`, `tickets:write`)
- Support scope hierarchies (e.g., `users:*` includes `users:read`, `users:write`)
- Include scope descriptions in token introspection response

**Role-Based Access Control (RBAC)**:
- Roles: `admin`, `user`, `readonly`, `service`
- Permissions mapped to roles
- Hierarchical permissions (admin > user > readonly)
- Support role inheritance
- Document role-to-permission mappings in API documentation

**Attribute-Based Access Control (ABAC)**:
- Context-aware authorization
- Resource attributes + user attributes + environment
- Dynamic policy evaluation
- Examples: time-based access, IP-based restrictions, resource ownership
- Policy decision points (PDP) for centralized authorization

### Security Best Practices
- **No secrets in URLs**; short token TTLs; rotate keys; refresh tokens when needed
- API keys: 90-day rotation, immediate revocation capability
- Rate limiting per API key/token
- Audit logging for all auth events

## 10. Rate Limiting & Quotas
- **Headers** (RFC 9239):
  - `RateLimit-Limit: 100`
  - `RateLimit-Remaining: 72`
  - `RateLimit-Reset: 30`
- On 429 also include `Retry-After` (seconds). Quotas are per token by default.

## 11. Asynchronous Operations
- For long tasks return `202 Accepted` + `Location: /jobs/{jobId}`
- **Job resource** example:

```json
{
  "id": "01J...",
  "status": "queued|running|succeeded|failed|canceled",
  "percent": 35,
  "result": {},
  "error": {},
  "createdAt": "...",
  "updatedAt": "..."
}
```

- Clients poll `GET /jobs/{id}` or subscribe via SSE/WebSocket if available

## 12. Webhooks (Outbound)
- **Event shape**: `eventType`, `id`, `createdAt`, `data`
- **Delivery**: POST JSON to subscriber URL
- **Security**: `X-Signature` HMAC-SHA256 over raw body with shared secret; include `X-Timestamp` (±5 min skew)
- **Retries**: Exponential backoff for ≥24h; dead-letter queue
- **Idempotency**: Include `eventId`; receivers dedupe
- **Replay protection**: Validate `X-Timestamp` and reject old events (>5 min)
- **Signature verification**: `HMAC-SHA256(secret, timestamp + '.' + body)` compare with `X-Signature`
- **Delivery guarantees**: At-least-once delivery with retry mechanism
- **Webhook management**: CRUD endpoints for webhook subscriptions
- **Event filtering**: Allow subscribers to filter by event types
- **Webhook testing**: Provide test event endpoint for validation

## 13. Internationalization, Numbers & Time
- All timestamps UTC (`Z`), always include milliseconds `.SSS` (e.g., `2025-09-01T20:00:00.000Z`). If timezone needed, add separate `timezone` (IANA name)
- JSON numbers for typical values; use strings for high-precision decimals or use integer minor units
- Sorting/filters are locale-agnostic unless documented otherwise
- Errors are not localized; localization is handled by the UI/API client.

## 14. Caching
- Reads: `ETag` + `Cache-Control: private, max-age=30` when safe
- Mutations: `Cache-Control: no-store`
- Conditional: `If-None-Match` → 304

## 15. Security & CORS
- HTTPS only; HSTS enabled
- CORS allow-list explicit origins; example:

```
Access-Control-Allow-Origin: https://app.example.com
Access-Control-Allow-Methods: GET,POST,PUT,PATCH,DELETE,OPTIONS
Access-Control-Allow-Headers: Authorization, Content-Type, Idempotency-Key
Access-Control-Expose-Headers: ETag, Location, RateLimit-*
```

- CSRF: only relevant for cookie auth; prefer Bearer in `Authorization` for SPAs
- Content Security Policy on the app domain; avoid wildcard `*/*`

## 16. Observability & Diagnostics
- **Tracing**: accept/propagate `traceparent` (W3C). Emit `traceId` header on all responses
- **Request ID**: honor `X-Request-Id` or generate one
- **Structured logs**: JSON per request: `traceId`, `requestId`, `userId`, `path`, `status`, `durationMs`, `bytes`
- **Metrics**: RED/USE per route, with p50/p90/p99

## 17. Versioning & Deprecation
- **Path versioning**: `/v1` (breaking changes bump major)
- **Non-breaking changes**: Adding optional fields/params, new endpoints, new enum values, relaxing validation
- **Breaking changes**: Removing fields, changing types/semantics, making optional fields required, changing URL structure
- **Client compatibility**: Must ignore unknown fields, handle new enum values gracefully, not rely on field order
- **Deprecation headers**: `Deprecation: true`, `Sunset: <RFC 8594 date>`, and `Link: <doc>; rel="deprecation"`

For complete versioning strategy, compatibility rules, and deprecation practices, see [VERSIONING.md](VERSIONING.md).

## 18. Canonical Status Codes

For complete HTTP status code definitions and application error mappings, see [STATUS_CODES.md](STATUS_CODES.md).

Quick reference:
- 200 OK (read/update)
- 201 Created (+ `Location`)
- 202 Accepted (async)
- 204 No Content (delete or idempotent update without body)
- 400 Bad Request
- 401 Unauthorized / 403 Forbidden
- 404 Not Found
- 409 Conflict
- 410 Gone (for deprecated endpoints)
- 412 Precondition Failed (ETag)
- 415 Unsupported Media Type
- 422 Unprocessable Entity (validation)
- 429 Too Many Requests
- 503 Service temporarily overloaded or under maintenance
- 5xx Other Server errors

## 19. Batch & Bulk
- Batch reads via filters
- Bulk write: `POST /tickets:batch` up to N items → `207 Multi-Status` with per-item results; idempotency keys per item

## 20. OpenAPI & Codegen
- **Source of truth**: OpenAPI 3.1
- For Rust backend specifics (utoipa, serde, validator), see `RUST.md`
- Client SDK: generate TypeScript types (`openapi-typescript`) and React hooks (TanStack Query) with fetch/axios adapter
- Keep schemas DRY via shared components; provide example payloads for every operation
- All endpoints must have summary and description
- Include request/response examples for every operation
- Document all error responses with Problem Details format
- Specify required authentication and scopes per endpoint
- Include rate limit information in operation metadata
- Use `$ref` for reusable components
- Define discriminators for polymorphic types
- Include format validators (email, uuid, date-time)
- Add `x-` vendor extensions for custom metadata
- Generate type-safe clients for TypeScript, Python, Go, Java
- Include retry logic and error handling in generated code
- Support custom headers (Idempotency-Key, tracing)
- Generate mock servers for testing

## 21. Example Endpoints
- **List Tickets**

```bash
curl -sS \
  -H "Authorization: Bearer $TOKEN" \
  "https://api.example.com/v1/tickets?limit=25&after=...&status.in=open,in_progress&sort=-priority,createdAt&fields[tickets]=id,title,priority,status,createdAt"
```

```json
{
  "data": [
    { "id": "01J...", "title": "Disk full", "priority": "high", "status": "open", "createdAt": "2025-08-31T10:05:17.000Z" }
  ],
  "meta": { "limit": 25, "hasNext": true },
  "links": { "next": "...after=...", "prev": null }
}
```

- **Update with Concurrency**

```
PATCH /v1/tickets/01J...
If-Match: W/"etag-abc"
Idempotency-Key: 5b2f...

{ "status": "in_progress" }
```

- **Async Job**

```
POST /v1/reports → 202 Accepted
Location: /v1/jobs/01J...
```

## 22. Backward Compatibility Rules (Client-facing)
- Clients ignore unknown fields
- Do not rely on property order
- Treat enums laxly: unknown enum → display as string, never crash
- Handle pagination cursors generically

## 23. Performance & DoS
- Enforce max list limits and payload sizes (e.g., 1MB JSON)
- Deny N+1 by default; allow explicit `include=` with documented caps
- Timeouts: handler ≤ 30s; use async jobs for longer work
- Strict input validation with precise 422s

## 24. Documentation Style

Each endpoint must be comprehensively documented to serve both human developers and AI assistants.

### Required Documentation Elements

1. **Summary & Description**
   - One-line summary
   - Detailed purpose explanation
   - When to use this endpoint

2. **Authentication & Authorization**
   - Required authentication method
   - Required scopes/permissions

3. **Request Specification**
   - HTTP method and path
   - Path parameters (type, format, constraints)
   - Query parameters (defaults, validation)
   - Request headers (required and optional)
   - Request body schema with field descriptions

4. **Response Specification**
   - Success status codes
   - Response headers
   - Response body schema
   - Example successful responses

5. **Error Documentation**
   - All possible error status codes
   - Problem Details examples for each
   - Common error scenarios

6. **Rate Limiting**
   - Rate limit class
   - Quota consumption

7. **Code Examples**
   - curl with realistic data
   - TypeScript with generated client

### Documentation Template

**Endpoint**: `POST /v1/resources`
**Purpose**: Create new resource
**Authentication**: Required (OAuth2)
**Authorization**: `resources:write` scope
**Rate Limit**: Standard (100/hour)

**Request Body Schema**:
```json
{
  "title": "string (required, max 255 chars)",
  "description": "string (optional, max 1000 chars)",
  "priority": "enum: low|medium|high",
  "category": "string (optional)"
}
```

**Success Response** (201):
```json
{
  "data": {
    "id": "res_01JCXYZ...",
    "title": "string",
    "description": "string",
    "priority": "medium",
    "category": "string",
    "status": "active",
    "createdAt": "2024-01-15T10:30:00Z",
    "updatedAt": "2024-01-15T10:30:00Z"
  },
  "meta": { "version": "1.0" },
  "links": {
    "self": "/v1/resources/res_01JCXYZ...",
    "updates": "/v1/resources/res_01JCXYZ.../updates"
  }
}
```

**Error Responses**:
- **400**: Invalid request body
- **401**: Missing/invalid authentication  
- **403**: Insufficient permissions
- **422**: Validation errors (Problem Details)
- **429**: Rate limit exceeded

**Code Examples**:
```bash
curl -X POST https://api.example.com/v1/resources \
  -H "Authorization: Bearer eyJ..." \
  -H "Content-Type: application/json" \
  -d '{"title": "Example", "priority": "medium"}'
```

```typescript
const resource = await api.createResource({
  title: 'Example',
  priority: 'medium'
});
```

---

## 25. GraphQL

### When to Use GraphQL

**Use GraphQL when**:
- **Mobile-first applications**: Reduce bandwidth with field selection
- **Complex data relationships**: Multiple related entities in single request
- **Rapid frontend iteration**: Schema-first development with type safety
- **Real-time subscriptions**: Live data updates via WebSocket/SSE
- **Microservices aggregation**: Single endpoint for multiple backend services
- **Developer experience**: Self-documenting API with introspection

**Avoid GraphQL when**:
- **Simple CRUD APIs**: REST is more straightforward
- **Caching requirements**: HTTP caching is more mature
- **File uploads**: REST handles binary data better
- **High-frequency operations**: REST has less overhead
- **Legacy system integration**: REST is more universally supported

### Schema Design

**Type System**:
- Use descriptive, consistent naming (camelCase for fields, PascalCase for types)
- Avoid nullable fields when possible; use optional fields instead
- Group related fields into interfaces and unions
- Version schema through additive changes only

**Query Design**:
- Single entry point: `/graphql`
- Support both queries and mutations
- Use descriptive operation names
- Avoid deep nesting (max 3-4 levels)

**Schema Example**:
```graphql
type User {
  id: ID!
  email: String!
  name: String
  createdAt: DateTime!
  posts: [Post!]! @connection
}

type Post {
  id: ID!
  title: String!
  content: String
  author: User!
  publishedAt: DateTime
}

type Query {
  user(id: ID!): User
  users(first: Int, after: String): UserConnection
  posts(filter: PostFilter): [Post!]!
}
```

### N+1 Query Prevention

**DataLoader Pattern**:
- Batch database queries by field
- Cache results within single request
- Implement per-request caching

**Field-Level Resolvers**:
```typescript
// Bad: N+1 queries
const resolvers = {
  User: {
    posts: (user) => db.posts.findByUserId(user.id) // N+1
  }
};

// Good: DataLoader batching
const resolvers = {
  User: {
    posts: (user, args, context) => 
      context.loaders.postsByUser.load(user.id)
  }
};
```

**Query Analysis**:
- Implement query complexity analysis
- Set maximum query depth (e.g., 10 levels)
- Limit number of fields per query
- Use query cost analysis for rate limiting

### Directives

**Built-in Directives**:
- `@deprecated(reason: "Use newField instead")`
- `@specifiedBy(url: "https://example.com/spec")`

**Custom Directives**:
```graphql
directive @auth(requires: [Role!]!) on FIELD_DEFINITION
directive @rateLimit(max: Int, window: String) on FIELD_DEFINITION
directive @cache(maxAge: Int) on FIELD_DEFINITION

type Query {
  user(id: ID!): User @auth(requires: [USER, ADMIN])
  expensiveData: String @rateLimit(max: 10, window: "1m")
  cachedData: String @cache(maxAge: 300)
}
```

### Persisted Queries

**Query Registration**:
- Pre-register allowed queries with hashes
- Client sends query hash instead of full query
- Reduces bandwidth and enables query whitelisting

**Implementation**:
```typescript
// Server: Register queries
const persistedQueries = {
  "abc123": "query GetUser($id: ID!) { user(id: $id) { id name } }",
  "def456": "query GetUsers { users { id email } }"
};

// Client: Send hash
fetch('/graphql', {
  method: 'POST',
  body: JSON.stringify({
    id: 'abc123',
    variables: { id: 'user-123' }
  })
});
```

### Federation

**Schema Federation**:
- Split large schemas into microservices
- Use Apollo Federation or similar
- Define entity keys for cross-service references

**Entity Definition**:
```graphql
# User service
type User @key(fields: "id") {
  id: ID!
  email: String!
  name: String
}

# Post service  
type Post @key(fields: "id") {
  id: ID!
  title: String!
  author: User @requires(fields: "authorId")
  authorId: ID! @external
}
```

### Error Handling

**GraphQL Error Format**:
```json
{
  "data": null,
  "errors": [
    {
      "message": "User not found",
      "locations": [{"line": 2, "column": 3}],
      "path": ["user"],
      "extensions": {
        "code": "USER_NOT_FOUND",
        "timestamp": "2025-01-15T10:30:00.000Z"
      }
    }
  ]
}
```

**Error Categories**:
- Validation errors (400)
- Authentication errors (401)
- Authorization errors (403)
- Not found errors (404)
- Internal errors (500)

---

## 26. gRPC & Async APIs

### When to Use gRPC

**Use gRPC when**:
- **Microservices communication**: High-performance service-to-service calls
- **Polyglot environments**: Language-agnostic contracts with code generation
- **Streaming data**: Real-time data transfer (logs, metrics, events)
- **Performance critical**: Lower latency and higher throughput than REST
- **Strong typing**: Contract-first development with compile-time validation
- **Load balancing**: Built-in load balancing and health checking

**Use Async APIs when**:
- **Long-running operations**: Background processing with status polling
- **Event-driven architecture**: Loose coupling between services
- **Batch processing**: Handle large datasets asynchronously
- **Integration patterns**: Webhooks, message queues, event streams
- **Scalability**: Decouple request/response for better resource utilization

**Avoid gRPC when**:
- **Web browser clients**: Limited browser support (use gRPC-Web)
- **Simple HTTP APIs**: REST is more accessible
- **Firewall restrictions**: Some networks block gRPC traffic
- **Human-readable debugging**: Binary format is harder to debug

### Protocol Buffers Contracts

**Service Definition**:
```protobuf
syntax = "proto3";

package api.v1;

service UserService {
  rpc GetUser(GetUserRequest) returns (User);
  rpc ListUsers(ListUsersRequest) returns (stream User);
  rpc CreateUser(CreateUserRequest) returns (User);
  rpc UpdateUser(UpdateUserRequest) returns (User);
  rpc DeleteUser(DeleteUserRequest) returns (google.protobuf.Empty);
}

message User {
  string id = 1;
  string email = 2;
  string name = 3;
  google.protobuf.Timestamp created_at = 4;
  google.protobuf.Timestamp updated_at = 5;
}

message GetUserRequest {
  string id = 1;
}
```

**Field Numbering**:
- Never reuse field numbers
- Reserve numbers 1-15 for frequently used fields
- Use field numbers 19000-19999 for internal use
- Document field deprecation instead of removal

### Streaming Patterns

**Server Streaming**:
```protobuf
rpc ListUsers(ListUsersRequest) returns (stream User);
```
- Use for large datasets
- Implement backpressure handling
- Client can cancel mid-stream

**Client Streaming**:
```protobuf
rpc CreateUsers(stream CreateUserRequest) returns (CreateUsersResponse);
```
- Batch operations
- Reduce network round-trips
- Handle partial failures

**Bidirectional Streaming**:
```protobuf
rpc Chat(stream ChatMessage) returns (stream ChatMessage);
```
- Real-time communication
- WebSocket alternative
- Implement heartbeat/ping

### Backward Compatibility

**Schema Evolution Rules**:
- Add new fields with new numbers
- Never change field types
- Mark fields as `reserved` instead of deleting
- Use `oneof` for mutually exclusive fields

**Versioning Strategy**:
```protobuf
// Version in package name
package api.v1;

// Or use separate proto files
import "api/v1/user.proto";
import "api/v2/user.proto";
```

**Compatibility Testing**:
- Test old clients with new servers
- Test new clients with old servers
- Validate message serialization/deserialization

### Service Mesh Integration

**Load Balancing**:
- Round-robin, least-conn, consistent hash
- Health checks for service discovery
- Circuit breaker patterns

**Observability**:
- Distributed tracing (OpenTelemetry)
- Metrics collection (Prometheus)
- Structured logging

**Security**:
- mTLS for service-to-service communication
- JWT tokens in metadata
- Rate limiting per service

### Error Handling

**gRPC Status Codes**:
```protobuf
import "google/rpc/status.proto";

// In service implementation
return status.Error(codes.NotFound, "User not found")

// With details
st := status.New(codes.InvalidArgument, "Invalid user data")
st, _ = st.WithDetails(&errdetails.BadRequest{
  FieldViolations: []*errdetails.BadRequest_FieldViolation{
    {
      Field: "email",
      Description: "Invalid email format",
    },
  },
})
return st.Err()
```

**Error Mapping**:
- gRPC codes to HTTP status codes
- Structured error details
- Client error handling

---

## 27. AsyncAPI & Event-Driven

### When to Use Event-Driven Architecture

**Use Event-Driven when**:
- **Microservices integration**: Loose coupling between services
- **Real-time features**: Live updates, notifications, chat systems
- **Data synchronization**: Keep multiple systems in sync
- **Audit trails**: Complete history of system state changes
- **Scalability**: Independent scaling of event producers/consumers
- **Fault tolerance**: Resilient to individual service failures
- **Business workflows**: Complex multi-step processes with compensation

**Use AsyncAPI when**:
- **Event documentation**: Standardized event schema documentation
- **Code generation**: Generate clients and servers from schemas
- **Event governance**: Centralized event catalog and versioning
- **Team collaboration**: Clear contracts between event producers/consumers
- **Testing**: Mock event producers for integration testing

**Avoid Event-Driven when**:
- **Simple CRUD operations**: Direct API calls are more straightforward
- **Strong consistency requirements**: Eventual consistency may not be acceptable
- **Low-latency requirements**: Event processing adds overhead
- **Debugging complexity**: Distributed tracing is more complex
- **Small teams**: Additional complexity may not be justified

### Event Schema Design

**AsyncAPI Specification**:
```yaml
asyncapi: 3.0.0
info:
  title: User Events API
  version: 1.0.0
  description: User-related events

servers:
  production:
    host: events.example.com
    protocol: kafka
    description: Production Kafka cluster

channels:
  user.created:
    address: user.created
    messages:
      userCreated:
        $ref: '#/components/messages/UserCreated'

  user.updated:
    address: user.updated
    messages:
      userUpdated:
        $ref: '#/components/messages/UserUpdated'

operations:
  onUserCreated:
    action: receive
    channel:
      $ref: '#/channels/user.created'
    summary: Receive user created events

components:
  messages:
    UserCreated:
      name: UserCreated
      contentType: application/json
      payload:
        $ref: '#/components/schemas/UserCreatedPayload'
      headers:
        type: object
        properties:
          correlationId:
            type: string
            description: Correlation ID for tracing
    
  schemas:
    UserCreatedPayload:
      type: object
      required:
        - eventId
        - eventType
        - timestamp
        - data
      properties:
        eventId:
          type: string
          format: uuid
          description: Unique event identifier
        eventType:
          type: string
          const: user.created
        eventVersion:
          type: string
          default: "1.0"
          description: Event schema version
        timestamp:
          type: string
          format: date-time
          description: Event creation time (ISO-8601)
        source:
          type: string
          description: Service that produced the event
        data:
          $ref: '#/components/schemas/User'
        metadata:
          type: object
          properties:
            correlationId:
              type: string
            causationId:
              type: string
```

**AsyncAPI Documentation**:
- Publish AsyncAPI spec at `/asyncapi.json` or `/asyncapi.yaml`
- Generate event documentation from AsyncAPI spec
- Use AsyncAPI Studio for visual editing
- Generate code for event producers and consumers

### Event Patterns

**Event Sourcing**:
- Store events as source of truth
- Rebuild state from event stream
- Event versioning and migration

**CQRS (Command Query Responsibility Segregation)**:
- Separate read and write models
- Event handlers update read models
- Eventually consistent views

**Saga Pattern**:
- Distributed transaction management
- Compensating actions for rollbacks
- Event-driven orchestration

### Message Schemas

**Event Envelope**:
```json
{
  "eventId": "evt_01JCXYZ...",
  "eventType": "user.created",
  "eventVersion": "1.0",
  "timestamp": "2025-01-15T10:30:00.000Z",
  "source": "user-service",
  "data": {
    "id": "user_123",
    "email": "user@example.com",
    "name": "John Doe"
  },
  "metadata": {
    "correlationId": "req_abc123",
    "causationId": "cmd_def456"
  }
}
```

**Schema Evolution**:
- Additive changes only
- Version events explicitly
- Backward compatibility for consumers
- Deprecation timeline for old versions

### Event Delivery

**At-Least-Once Delivery**:
- Acknowledge after processing
- Idempotent event handlers
- Dead letter queues for failures

**Ordering Guarantees**:
- Partition by entity ID for ordering
- Sequence numbers for strict ordering
- Out-of-order handling strategies

**Replay Protection**:
- Event ID deduplication
- Timestamp-based replay windows
- Consumer group coordination

### Dead Letter Queues

**DLQ Configuration**:
```yaml
deadLetterQueue:
  maxRetries: 3
  retryDelay: "1m"
  maxAge: "24h"
  topics:
    - "user.events.dlq"
```

**Error Handling**:
- Categorize errors (retryable vs permanent)
- Exponential backoff for retries
- Manual intervention for DLQ

### Event Monitoring

**Metrics**:
- Event throughput and latency
- Consumer lag monitoring
- Error rates and DLQ depth
- Processing time per event type

**Alerting**:
- Consumer lag thresholds
- DLQ depth alerts
- Processing failure rates
- Schema validation errors

---

## Operational Headers (Quick Reference)
- Requests may include: `Authorization`, `Idempotency-Key`, `If-Match`, `If-None-Match`, `Accept-Encoding`, `traceparent`, `X-Request-Id`
- Responses should include: `Content-Type`, `ETag` (when cacheable), `Location` (201/202), `RateLimit-*`, `traceId`, `X-Request-Id`

## References
- RFC 9457 Problem Details: [IETF RFC 9457](https://www.rfc-editor.org/rfc/rfc9457)
- RFC 9239 RateLimit Fields: [IETF RFC 9239](https://www.rfc-editor.org/rfc/rfc9239)
- W3C Trace Context: [traceparent](https://www.w3.org/TR/trace-context/)
