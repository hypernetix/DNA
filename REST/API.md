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

## 9. Authentication & Authorization
- **Auth**: OAuth2/OIDC Bearer tokens in `Authorization: Bearer <token>`
- **Scopes/permissions**: Document per endpoint; insufficient → 403
- **Service-to-service**: mTLS optional
- **No secrets in URLs**; short token TTLs; rotate keys; refresh tokens when needed

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

## Operational Headers (Quick Reference)
- Requests may include: `Authorization`, `Idempotency-Key`, `If-Match`, `If-None-Match`, `Accept-Encoding`, `traceparent`, `X-Request-Id`
- Responses should include: `Content-Type`, `ETag` (when cacheable), `Location` (201/202), `RateLimit-*`, `traceId`, `X-Request-Id`

## References
- RFC 9457 Problem Details: [IETF RFC 9457](https://www.rfc-editor.org/rfc/rfc9457)
- RFC 9239 RateLimit Fields: [IETF RFC 9239](https://www.rfc-editor.org/rfc/rfc9239)
- W3C Trace Context: [traceparent](https://www.w3.org/TR/trace-context/)
