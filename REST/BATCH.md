# Batch & Bulk Operations

This document specifies how to design and implement batch/bulk endpoints in REST APIs following the Hypernetix guidelines.

## Table of Contents
- [Endpoint Pattern](#endpoint-pattern)
- [Request Format](#request-format)
- [Response Formats](#response-formats)
- [Status Code Rules](#status-code-rules)
- [Error Format](#error-format)
- [Cross-Item Conflicts](#cross-item-conflicts)
- [Atomicity (Transactional Semantics)](#atomicity-transactional-semantics)
- [Idempotency](#idempotency)
- [Performance Limits](#performance-limits)
- [Large Batch Operations](#large-batch-operations)
- [Complete Example](#complete-example)

## Endpoint Pattern

- **Batch writes**: `POST /resources:batch`
- **Batch reads**: Use filters on collection endpoints (e.g., `GET /tickets?id.in=01J...,01K...`)
- **Response**: `207 Multi-Status` (partial success) or specific status code (all same outcome)
- **Request limit**: Default 100 items per batch (configurable per endpoint, must be documented)

## Request Format

```json
{
  "items": [
    {
      "idempotencyKey": "req-1",
      "data": {
        "title": "Fix login bug",
        "priority": "high"
      }
    },
    {
      "idempotencyKey": "req-2",
      "data": {
        "title": "Update documentation",
        "priority": "medium"
      }
    }
  ]
}
```

## Response Formats

### Partial Success

```http
HTTP/1.1 207 Multi-Status
Content-Type: application/json
```

```json
{
  "results": [
    {
      "index": 0,
      "idempotencyKey": "req-1",
      "status": 201,
      "data": {
        "id": "01J...",
        "title": "Fix login bug",
        "priority": "high",
        "status": "open",
        "createdAt": "2025-09-01T20:00:00.000Z",
        "updatedAt": "2025-09-01T20:00:00.000Z"
      },
      "headers": {
        "Location": "/v1/tickets/01J...",
        "ETag": "W/\"abc123\""
      }
    },
    {
      "index": 1,
      "idempotencyKey": "req-2",
      "status": 422,
      "error": {
        "type": "https://api.example.com/errors/validation",
        "title": "Validation failed",
        "status": 422,
        "detail": "Multiple validation errors",
        "instance": "https://api.example.com/req/01JXYZ...Z#item-1",
        "errors": [
          {
            "field": "priority",
            "code": "enum",
            "message": "must be low, medium, or high"
          }
        ],
        "traceId": "01JXYZ...Z-item-1"
      }
    }
  ],
  "meta": {
    "total": 2,
    "succeeded": 1,
    "failed": 1
  }
}
```

### All Success

```http
HTTP/1.1 200 OK
Content-Type: application/json
```

```json
{
  "results": [
    {
      "index": 0,
      "idempotencyKey": "req-1",
      "status": 201,
      "data": { "id": "01J...", "title": "..." }
    },
    {
      "index": 1,
      "idempotencyKey": "req-2",
      "status": 201,
      "data": { "id": "01K...", "title": "..." }
    }
  ],
  "meta": {
    "total": 2,
    "succeeded": 2,
    "failed": 0
  }
}
```

### All Failed (Same Error Type)

```http
HTTP/1.1 422 Unprocessable Entity
Content-Type: application/json
```

```json
{
  "results": [
    { "index": 0, "status": 422, "error": { /* Problem Details */ } },
    { "index": 1, "status": 422, "error": { /* Problem Details */ } }
  ],
  "meta": {
    "total": 2,
    "succeeded": 0,
    "failed": 2
  }
}
```

## Status Code Rules

The top-level HTTP status code reflects the aggregate outcome:

- **All succeeded** → `200 OK` (or `201 Created` if appropriate)
- **Partial success/failure** → `207 Multi-Status`
- **All failed (same error type)** → matching `4xx` (e.g., all `422` → top-level `422`)
- **All failed (mixed error types)** → `207 Multi-Status`

## Error Format

Each failed item receives **complete RFC 9457 Problem Details** for consistency with single-item error format (see [api.md#error-model](api.md#7-error-model-problem-details)).

### Required Fields
- `type` - Error type URL
- `title` - Short error description
- `status` - HTTP status code for this item
- `traceId` - Unique trace identifier for this item

### Optional Fields
- `detail` - Detailed explanation
- `instance` - Request URL with `#item-{index}` fragment
- `errors` - Array of field-level validation errors (for 422 responses)

### Example: Validation Error

```json
{
  "index": 1,
  "idempotencyKey": "req-2",
  "status": 422,
  "error": {
    "type": "https://api.example.com/errors/validation",
    "title": "Validation failed",
    "status": 422,
    "detail": "Multiple validation errors",
    "instance": "https://api.example.com/req/01JXYZ...Z#item-1",
    "errors": [
      { "field": "email", "code": "format", "message": "must be a valid email" },
      { "field": "priority", "code": "enum", "message": "must be low, medium, or high" }
    ],
    "traceId": "01JXYZ...Z-item-1"
  }
}
```

### Example: Not Found Error

```json
{
  "index": 2,
  "idempotencyKey": "req-3",
  "status": 404,
  "error": {
    "type": "https://api.example.com/errors/not-found",
    "title": "Resource not found",
    "status": 404,
    "detail": "Ticket with id '01JOLD...' does not exist",
    "instance": "https://api.example.com/req/01JXYZ...Z#item-2",
    "traceId": "01JXYZ...Z-item-2"
  }
}
```

## Cross-Item Conflicts

Since cross-item conflicts (duplicates within the batch, constraint violations across items) are typically rare, use simple batch-level error reporting when detected.

### Pre-validation Approach

Stop processing and return single Problem Details:

```http
HTTP/1.1 400 Bad Request
Content-Type: application/problem+json
```

```json
{
  "type": "https://api.example.com/errors/batch-conflict",
  "title": "Duplicate items in batch",
  "status": 400,
  "detail": "Items at indices 1 and 3 have duplicate email addresses",
  "conflicts": [
    {
      "type": "duplicate",
      "field": "email",
      "value": "user@example.com",
      "itemIndices": [1, 3]
    }
  ],
  "traceId": "01JXYZ...Z"
}
```

### Conflicts with Existing Resources

Conflicts with existing resources (not within the batch) are treated as per-item 409 errors:

```json
{
  "index": 2,
  "idempotencyKey": "req-3",
  "status": 409,
  "error": {
    "type": "https://api.example.com/errors/conflict",
    "title": "Resource conflict",
    "status": 409,
    "detail": "A ticket with title 'Fix login bug' already exists",
    "existingResourceId": "01HOLD...",
    "traceId": "01JXYZ...Z-item-2"
  }
}
```

## Atomicity (Transactional Semantics)

Default behavior is **endpoint-specific** and must be documented for each batch operation.

### Best-Effort Endpoints (Default)

Most endpoints use **best-effort** semantics: each item is processed independently, and successes are committed even if other items fail.

**Request:**
```json
{
  "items": [ /* ... */ ]
}
```

**Response:** `207 Multi-Status` (or other status based on aggregate outcome) with per-item results showing mix of successes and failures.

**Use Cases:**
- Bulk user imports (independent records)
- Creating multiple independent tickets
- Batch notifications
- Log ingestion

### Atomic Endpoints (All-or-Nothing)

Some endpoints enforce **transactional** semantics: if any item fails, all changes are rolled back.

Document clearly in endpoint specification that atomic behavior applies.

**Response on Failure:**

```http
HTTP/1.1 422 Unprocessable Entity
Content-Type: application/problem+json
```

```json
{
  "type": "https://api.example.com/errors/batch-failed",
  "title": "Batch operation failed",
  "status": 422,
  "detail": "Transaction rolled back due to validation failure at item 3",
  "failedItemIndex": 3,
  "itemError": {
    "errors": [
      { "field": "amount", "code": "range", "message": "must be positive" }
    ]
  },
  "traceId": "01JXYZ...Z"
}
```

**Use Cases:**
- Financial transactions
- Creating related records that must coexist (parent + children)
- Critical business operations requiring consistency

### Client-Controlled Atomicity (Optional)

Some endpoints may support both modes via request parameter for maximum flexibility:

**Request:**
```json
{
  "atomic": true,
  "items": [ /* ... */ ]
}
```

- `"atomic": false` (or omitted) → best-effort (default)
- `"atomic": true` → all-or-nothing

## Idempotency

Batch operations support **per-item idempotency** to enable safe retries.

### Request with Idempotency Keys

```json
{
  "items": [
    {
      "idempotencyKey": "user-action-123-item-0",
      "data": { "title": "Ticket 1", "priority": "high" }
    },
    {
      "idempotencyKey": "user-action-123-item-1",
      "data": { "title": "Ticket 2", "priority": "medium" }
    }
  ]
}
```

### Server Behavior

- Each item's `idempotencyKey` is matched independently
- Only **successful (2xx) outcomes** are cached and replayed with the original response
- Error responses (4xx/5xx) are **NOT cached**; retries re-execute to allow fresh validation, permission checks, and recovery from transient failures
- Mix of new and replayed items is allowed in a single batch
- **Idempotency retention** (tiered by operation criticality):
  - Minimum default: 1 hour (sufficient for network retry protection)
  - Important operations: 24h-7d (e.g., bulk notifications, report generation)
  - Critical operations: Permanent via DB uniqueness on `idempotencyKey` (e.g., payment processing, invoice creation) → return `409 Conflict` per item if key exists after cache expiry

### Replayed Item Indication

```json
{
  "index": 0,
  "idempotencyKey": "user-action-123-item-0",
  "status": 201,
  "data": { "id": "01J...", "title": "..." },
  "meta": {
    "idempotencyReplayed": true
  }
}
```

## Performance Limits

- **Default maximum batch size**: 100 items. Specific endpoints MAY configure different limits but MUST document the limit clearly.
- **Timeout**: 30s default (same as single operations)
- **Payload size**: 1MB limit applies to entire batch request
- **Rate limiting**: Batch counts as single request; consider separate quotas for batch vs single-item operations

## Large Batch Operations

For operations requiring more than 100 items:

### Option 1: Client-Side Chunking (Recommended)

Clients split large batches into multiple smaller requests (50-100 items each):
- Maintains full error detail per item
- Better progress feedback
- Natural retry boundaries
- Simpler server implementation

### Option 2: Asynchronous Processing

For very large batches (1000+ items), use async job pattern:

**Submit Job:**
```http
POST /v1/tickets:batch
Content-Type: application/json

{ "items": [ /* 1000+ items */ ] }
```

**Response:**
```http
HTTP/1.1 202 Accepted
Location: /v1/jobs/01JXYZ...
```

**Poll Job Status:**
```http
GET /v1/jobs/01JXYZ...
```

```json
{
  "id": "01JXYZ...",
  "status": "running",
  "progress": {
    "total": 1500,
    "processed": 850,
    "succeeded": 820,
    "failed": 30
  },
  "resultsUrl": "/v1/jobs/01JXYZ.../results"
}
```

**Paginate Results:**
```http
GET /v1/jobs/01JXYZ.../results?limit=100&after=...
```

See [api.md#asynchronous-operations](api.md#11-asynchronous-operations) for complete async job specifications.

## Complete Example

### Request

```bash
curl -X POST https://api.example.com/v1/tickets:batch \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -H "traceparent: 00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01" \
  -d '{
    "items": [
      {
        "idempotencyKey": "req-1",
        "data": {
          "title": "Fix login bug",
          "priority": "high",
          "assigneeId": "01JUSR..."
        }
      },
      {
        "idempotencyKey": "req-2",
        "data": {
          "title": "Update docs",
          "priority": "low"
        }
      },
      {
        "idempotencyKey": "req-3",
        "data": {
          "title": "Invalid ticket",
          "priority": "invalid-value"
        }
      }
    ]
  }'
```

### Response (Partial Success)

```http
HTTP/1.1 207 Multi-Status
Content-Type: application/json
RateLimit-Limit: 100
RateLimit-Remaining: 99
RateLimit-Reset: 60
traceId: 01JXYZ...Z
```

```json
{
  "results": [
    {
      "index": 0,
      "idempotencyKey": "req-1",
      "status": 201,
      "data": {
        "id": "01JTKT...",
        "title": "Fix login bug",
        "priority": "high",
        "status": "open",
        "assigneeId": "01JUSR...",
        "createdAt": "2025-09-01T20:00:00.000Z",
        "updatedAt": "2025-09-01T20:00:00.000Z"
      },
      "headers": {
        "Location": "/v1/tickets/01JTKT...",
        "ETag": "W/\"abc123\""
      }
    },
    {
      "index": 1,
      "idempotencyKey": "req-2",
      "status": 201,
      "data": {
        "id": "01JTKU...",
        "title": "Update docs",
        "priority": "low",
        "status": "open",
        "createdAt": "2025-09-01T20:00:01.000Z",
        "updatedAt": "2025-09-01T20:00:01.000Z"
      },
      "headers": {
        "Location": "/v1/tickets/01JTKU...",
        "ETag": "W/\"def456\""
      }
    },
    {
      "index": 2,
      "idempotencyKey": "req-3",
      "status": 422,
      "error": {
        "type": "https://api.example.com/errors/validation",
        "title": "Validation failed",
        "status": 422,
        "detail": "Invalid priority value",
        "instance": "https://api.example.com/req/01JXYZ...Z#item-2",
        "errors": [
          {
            "field": "priority",
            "code": "enum",
            "message": "must be low, medium, or high"
          }
        ],
        "traceId": "01JXYZ...Z-item-2"
      }
    }
  ],
  "meta": {
    "total": 3,
    "succeeded": 2,
    "failed": 1
  }
}
```
