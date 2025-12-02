### HTTP status codes and application error codes

This document outlines the usage of HTTP status codes and application-level error codes according to the Hypernetix API Guidelines. Ensure that responses remain consistent across all endpoints.

### Success

- 200 OK: Standard successful GET/PUT/PATCH responses.
- 201 Created: Resource successfully created. Include `Location` header.
- 202 Accepted: Async operation accepted; returns operation status handle.
- 204 No Content: Successful mutation with no body.

### Client errors

- 400 Bad Request
  - Malformed syntax, invalid parameters, or invalid cursor.
  - Error codes:
    - Pagination: `INVALID_CURSOR`, `ORDER_MISMATCH`, `FILTER_MISMATCH`, `UNSUPPORTED_FILTER_FIELD`, `UNSUPPORTED_ORDERBY_FIELD`
    - Field projection: `INVALID_FIELD`, `TOO_MANY_FIELDS`, `FIELD_SELECTION_MISMATCH`
    - General: `INVALID_QUERY`
- 401 Unauthorized
  - Missing/invalid/expired auth.
  - Error codes: `UNAUTHENTICATED`, `TOKEN_EXPIRED`.
- 403 Forbidden
  - Authenticated but not authorized.
  - Error codes: `FORBIDDEN`, `INSUFFICIENT_PERMISSIONS`.
- 404 Not Found
  - Resource or route not found.
  - Error codes: `NOT_FOUND`.
- 405 Method Not Allowed
  - HTTP method not supported for this resource. Include `Allow` header.
  - Error codes: `METHOD_NOT_ALLOWED`.
- 406 Not Acceptable
  - The requested `Accept` media type is not supported.
  - Error codes: `NOT_ACCEPTABLE`.
- 409 Conflict
  - Resource state conflict (duplicate, version conflict, invariant violation).
  - Error codes: `CONFLICT`, `VERSION_CONFLICT`, `DUPLICATE`.
- 412 Precondition Failed
  - ETag preconditions failed.
  - Error codes: `PRECONDITION_FAILED`.
- 413 Payload Too Large / 414 URI Too Long
  - Request exceeds limits (body or URL length).
  - Error codes: `REQUEST_TOO_LARGE`, `URI_TOO_LONG`.
- 415 Unsupported Media Type
  - Content-Type not supported.
  - Error codes: `UNSUPPORTED_MEDIA_TYPE`.
- 422 Unprocessable Entity
  - Validation failed, semantically invalid input.
  - Error codes: `INVALID_LIMIT`, `VALIDATION_ERROR`, `SCHEMA_MISMATCH`.
- 428 Precondition Required
  - The request is required to be conditional or include a specific precondition (e.g., `If-Match`, `Idempotency-Key`) per API policy.
  - Error codes: `PRECONDITION_REQUIRED`.
- 429 Too Many Requests
  - Rate limit exceeded; include standard rate limit headers.
  - Error codes: `RATE_LIMITED`.
- 431 Request Header Fields Too Large
  - Request header fields are too large. Reduce header sizes (e.g., cookies) and retry.
  - Error codes: `HEADERS_TOO_LARGE`.

### Server errors

- 503 Service Unavailable
  - Service temporarily overloaded or under maintenance. May also be used for rate limiting in extreme cases.
  - Error codes: `SERVICE_UNAVAILABLE`, `RATE_LIMITED_SEVERE`.

### Applicability by HTTP method

- GET
  - Success: 200
  - Typical errors: 400, 401, 403, 404, 406, 429, 431, 503
- POST
  - Success: 201 (created), 202 (async), 200 (idempotent replay)
  - Typical errors: 400, 401, 403, 404, 405, 406, 409, 415, 422, 428 (require Idempotency-Key when mandated), 429, 431, 503
- PUT
  - Success: 200 (replaced), 204 (no body), 201 (upsert if supported)
  - Typical errors: 400, 401, 403, 404, 405, 406, 412 (ETag), 415, 422, 428 (require If-Match), 429, 431, 503
- PATCH
  - Success: 200 (updated), 204 (no body)
  - Typical errors: 400, 401, 403, 404, 405, 406, 412 (ETag), 415, 422, 428 (require If-Match), 429, 431, 503
- DELETE
  - Success: 204 (deleted), 202 (async delete), 200 (soft-delete payload)
  - Typical errors: 400, 401, 403, 404, 405, 406, 412 (ETag), 409 (conflict), 428 (require If-Match), 429, 431, 503
