## Cursor Pagination Spec

This document defines a consistent, implementation-friendly contract for cursor-based pagination referenced by Hypernetix API Guidelines.

## Goals

- **Stable ordering**: No duplicates or gaps when paging.
- **Opaque cursors**: Clients never rely on internal fields.
- **Simple client contract**: One optional `cursor` and `limit` param per request.
- **Filter safety**: Cursors are bound to the query/filter they were created from.
- **Extensible**: Versioned cursor payloads for painless future changes.

## Terminology

- **Cursor**: Opaque token representing a position in a sorted result set.
- **Page**: Up to `limit` items returned for a given cursor.
- **Canonical sort**: Endpoint-defined, immutable ordering (includes a unique tiebreaker).

## Request

Query parameters every paginated endpoint should accept:

- `limit` (integer): Number of items to return.
  - Default: 25
  - Min: 1
  - Max: 200 (enforced; endpoints may choose a lower max)
- `cursor` (string, optional): Opaque token returned by a previous response. When provided, the server returns the next page after the cursor in the endpoint’s canonical sort order.

OData parameters for client-defined filtering and sorting (recommended):

A **subset of OData syntax** is employed for filtering and sorting, offering familiar, standardized query capabilities while ensuring simplicity in implementation.

- `$filter` (string, optional): OData-style filter expression. Examples:
  - `price gt 10 and startswith(name,'Pro')`
  - `status in ('paid','shipped') and created_at ge 2025-01-01T00:00:00Z`
  - Supported operators: `eq`, `ne`, `gt`, `ge`, `lt`, `le`, `and`, `or`, `not`, `in` (set membership), functions `startswith`, `endswith`, `contains`. Strings use single quotes; timestamps are RFC3339 UTC.
- `$orderby` (string, optional): OData-style order-by list. Examples:
  - `created_at desc, id desc`
  - Must include a unique tiebreaker (typically `id`) last for stable pagination; if omitted, the server appends `id` automatically.

Notes:
- Endpoints must document their canonical sort and whether it is ascending or descending.
- Only indexed fields may be used in `$filter` and `$orderby`.
- If an endpoint supports client-selectable sort or filters, the cursor must encode and validate `$orderby` and a normalized `$filter` hash; otherwise requests with mismatched params must be rejected with 400.

### Request Phases

- First request (no `cursor`):
  - Client may supply `limit`, `$orderby`, and `$filter`.
  - Server validates `$orderby` tokens (see Ordering Fields) and ensures a unique tiebreaker is last (or appends `id`).
  - Server applies `$filter` and `$orderby`, returns page, and encodes `s` (effective `$orderby`) and `o` (direction of the primary key) into the cursor along with a normalized filter hash `f` derived from `$filter`.
- Subsequent requests (with `cursor`):
  - `cursor` fully defines ordering (`s`, `o`) and filters (`f`).
  - Server ignores any provided `$orderby` when a `cursor` is present. If provided and it differs from the cursor, return 400 with `code: "ORDER_MISMATCH"`.
  - Changing filters between pages is not allowed; if detected (via `f`), return 400 with `code: "FILTER_MISMATCH"`.

## Response

Return a consistent envelope:

```
{
  "items": [ ... ],
  "page_info": {
    "next_cursor": "<opaque>",
    "prev_cursor": "<opaque>",
    "limit": 20
  }
}
```

Rules:
- `items` are in the endpoint’s canonical sort order (do not reverse for backward navigation).
- `next_cursor` points to the position immediately after the last item in `items` and may be omitted if there is no next page.
- `prev_cursor` points to the position immediately before the first item in `items` and may be omitted on the first page.
- Do not include `total_count` in cursor pagination responses.

## Canonical Sort Requirements

- Must be total and stable: combine the primary sort key with a unique tiebreaker (e.g., `created_at DESC, id DESC`).
- The unique tiebreaker must be monotonic with respect to insertion order or at least unique (UUID/ID). UUIDv7 is recommended; its lexicographic order aligns with creation time.
- Example canonical sorts:
  - Timelines: `created_at DESC, id DESC`
  - Oldest-first logs: `created_at ASC, id ASC`

## Ordering Fields

Define ordering using standardized field tokens. By default, only `created_at` and `id` are allowed for ordering. Specific APIs may add more ordering fields if they are indexed and documented in the endpoint’s allowlist.

- `created_at` (timestamp, RFC3339/UTC)
  - Recommended primary key for feeds and most lists.
  - Must be non-null and represent creation time.
  - Comparison: timestamp value; for DESC, newer first.
- `id` (string, UUIDv7)
  - Required unique tiebreaker in all canonical sorts.
  - Comparison: lexicographic on canonical UUID string. With UUIDv7, lexicographic order preserves creation-time ordering.
  - Format: lowercase hyphenated UUIDv7 string.
// Additional ordering fields may be defined per-endpoint; they must be indexed, documented, and accompanied by clear comparison semantics.

Field comparison semantics:
- Strings: compare using defined collation; default is case-insensitive NFKC with `en-US` unless endpoint specifies otherwise.
- Timestamps: compare by instant in UTC.
- Numbers: IEEE-754 comparisons; NaN not allowed; nulls not allowed in ordering fields.

Allowed `s` tokens (comma-separated): `created_at`, `id`.
- Endpoints may introduce additional domain-specific tokens if documented; include them verbatim in `s` and define their comparison semantics. All tokens must correspond to indexed fields.

## Indexed Fields & Allowed Parameters

- Filtering and ordering are allowed only on fields backed by suitable database indexes.
  - For string functions (e.g., `startswith`, `contains`), enable only when supported by an index (prefix, trigram, full-text); otherwise reject the filter.
- The set of queryable indexed fields MUST be limited. Recommended cap per endpoint: 10.
- Each endpoint MUST publish an allowlist of:
  - `$filter` fields (and supported operators per field), and
  - `$orderby` fields (with allowed directions),
  in the generated API spec.
  - Suggested OpenAPI vendor extension:

```yaml
x-pagination:
  filterFields:
    created_at: [ge, gt, le, lt, eq]
    id: [eq, in]
  orderBy:
    - created_at desc
    - created_at asc
    - id asc
```

## OpenAPI Contract Requirements

Generated OpenAPI must explicitly document the allowed filter and order parameters per endpoint.

- Parameters:
  - `$filter` (query, string): OData filter expression restricted to the endpoint’s allowlisted, indexed fields and operators.
  - `$orderby` (query, string): Comma-separated OData ordering, restricted to allowlisted, indexed fields and directions.
- Include the allowlists in vendor extensions for machine-readability, and reference them in parameter descriptions.

Example OpenAPI fragment:

```yaml
components:
  parameters:
    FilterParam:
      name: $filter
      in: query
      required: false
      schema:
        type: string
      description: |
        OData filter over allowlisted indexed fields. See x-pagination.filterFields.
    OrderByParam:
      name: $orderby
      in: query
      required: false
      schema:
        type: string
      example: created_at desc, id desc
      description: |
        OData order-by over allowlisted indexed fields. See x-pagination.orderBy.

paths:
  /v1/items:
    get:
      parameters:
        - $ref: '#/components/parameters/FilterParam'
        - $ref: '#/components/parameters/OrderByParam'
      x-pagination:
        filterFields:
          created_at: [ge, gt, le, lt, eq]
          id: [eq, in]
        orderBy:
          - created_at desc
          - created_at asc
          - id asc
```

## Cursor Format (Opaque, Versioned)

- Base64URL (no padding) encoded JSON object.
- Required fields:

```
{
  "v": 1,                 // version
  "k": [<primaryKey>, <tiebreakerKey>],
  "o": "desc|asc",       // order of the primary key
  "s": "created_at,id",  // sort keys used to build the cursor
  "f": "<filter-hash>"    // hash of normalized filters (optional but recommended)
}
```

Guidelines:
- `k` are the raw values needed for comparison in the database (e.g., ISO8601 timestamp string and id string).
- When multiple sort keys are used, `k` MUST include values for each sort key in order (e.g., `[primary, secondary, tiebreaker]`).
- Use Base64URL without padding so tokens are URL-safe.
- Treat the token as completely opaque to clients; they must not parse it.
 - For mixed-direction sorts, encode per-field direction in `s` using optional `+`/`-` prefixes (e.g., `"-score,+created_at,+id"`).

## Validation Rules

- If `cursor` is present, decode and validate:
  - Known `v`.
  - Matches endpoint’s canonical sort (`s` and `o`).
  - If filters or client sorting are allowed, ensure the current request’s normalized parameters hash to the same `f`.
- On any validation failure, return 400 with `code: "INVALID_CURSOR"`.
- If `limit` is out of bounds, return 422 with `code: "INVALID_LIMIT"`.
- Validate that all fields referenced in `$filter` and `$orderby` belong to the endpoint’s allowlist of indexed fields; otherwise return 400 with `code: "UNSUPPORTED_FILTER_FIELD"` or `code: "UNSUPPORTED_ORDERBY_FIELD"`.

## Implementation Recipe (SQL/ORM)

Assume canonical sort `created_at DESC, id DESC`.

1) Normalize and clamp `limit` to [1, MAX_LIMIT]. Use `pageSize = min(request.limit || 20, 100)`.
2) If `cursor` is provided, decode it to `(cursorCreatedAt, cursorId)`.
3) Build the predicate for forward pagination in canonical order:
   - For DESC order: `(created_at, id) < (cursorCreatedAt, cursorId)`.
   - For ASC order:  `(created_at, id) > (cursorCreatedAt, cursorId)`.
4) Apply filters first, then the cursor predicate.
5) Order by the canonical sort.
6) Fetch `pageSize + 1` rows.
7) Trim `items = rows.slice(0, pageSize)`.
8) Compute cursors:
   - `next_cursor` from the last item in `items`.
   - `prev_cursor` from the first item in `items`.
9) Optionally compute previous existence internally if needed; clients rely solely on presence/absence of cursor fields.

### Complete Pagination Examples: All 4 Scenarios

This section demonstrates all possible combinations of forward/backward navigation with ascending/descending ordering.

#### Scenario 1: Forward Pagination with DESC ordering
**Setup**: `$orderby=created_at desc, id desc` (newest first)
**Navigation**: Forward (get next/older items)

```sql
-- Given :page_size_plus_one, :cursor_created_at, :cursor_id
WHERE (
  :has_cursor IS FALSE
  OR created_at < :cursor_created_at
  OR (created_at = :cursor_created_at AND id < :cursor_id)
)
ORDER BY created_at DESC, id DESC
LIMIT :page_size_plus_one
```

#### Scenario 2: Backward Pagination with DESC ordering
**Setup**: `$orderby=created_at desc, id desc` (newest first)
**Navigation**: Backward (get previous/newer items)

```sql
-- Given :page_size_plus_one, :cursor_created_at, :cursor_id
WHERE (
  :has_cursor IS FALSE
  OR created_at > :cursor_created_at
  OR (created_at = :cursor_created_at AND id > :cursor_id)
)
ORDER BY created_at DESC, id DESC
LIMIT :page_size_plus_one
```

#### Scenario 3: Forward Pagination with ASC ordering
**Setup**: `$orderby=created_at asc, id asc` (oldest first)
**Navigation**: Forward (get next/newer items)

```sql
-- Given :page_size_plus_one, :cursor_created_at, :cursor_id
WHERE (
  :has_cursor IS FALSE
  OR created_at > :cursor_created_at
  OR (created_at = :cursor_created_at AND id > :cursor_id)
)
ORDER BY created_at ASC, id ASC
LIMIT :page_size_plus_one
```

#### Scenario 4: Backward Pagination with ASC ordering
**Setup**: `$orderby=created_at asc, id asc` (oldest first)
**Navigation**: Backward (get previous/older items)

```sql
-- Given :page_size_plus_one, :cursor_created_at, :cursor_id
WHERE (
  :has_cursor IS FALSE
  OR created_at < :cursor_created_at
  OR (created_at = :cursor_created_at AND id < :cursor_id)
)
ORDER BY created_at ASC, id ASC
LIMIT :page_size_plus_one
```

#### Mixed-Direction Sort Example
For complex sorting like `$orderby=score desc, created_at asc, id asc`:

**Forward Pagination**:
```sql
-- Given :page_size_plus_one, :cursor_score, :cursor_created_at, :cursor_id
WHERE (
  :has_cursor IS FALSE
  OR score < :cursor_score
  OR (score = :cursor_score AND created_at > :cursor_created_at)
  OR (score = :cursor_score AND created_at = :cursor_created_at AND id > :cursor_id)
)
ORDER BY score DESC, created_at ASC, id ASC
LIMIT :page_size_plus_one
```

**Backward Pagination**:
```sql
-- Given :page_size_plus_one, :cursor_score, :cursor_created_at, :cursor_id
WHERE (
  :has_cursor IS FALSE
  OR score > :cursor_score
  OR (score = :cursor_score AND created_at < :cursor_created_at)
  OR (score = :cursor_score AND created_at = :cursor_created_at AND id < :cursor_id)
)
ORDER BY score DESC, created_at ASC, id ASC
LIMIT :page_size_plus_one
```

Notes:
- Many databases support mixed-direction indexes; verify support for your engine.
- This OR-chain predicate matches the index order and remains sargable.

## Examples

### Request

```http
GET /v1/messages?limit=20&$filter=active eq true&$orderby=created_at desc, id desc
```

```http
GET /v1/messages?limit=20&cursor=eyJ2IjoxLCJrIjpbIjIwMjUtMDktMTRUMTI6MzQ6NTYuNzg5WiIsIjEyM2U0NTY3Il0sIm8iOiJkZXNjIiwicyI6ImNyZWF0ZWRfYXQsaWQiLCJmIjoiZjg2OWJhIn0
```

### Response

```json
{
  "items": [
    { "id": "018f6c9e-2c3b-7b1a-8f4a-9c3d2b1a0e5f", "created_at": "2025-09-14T12:34:57.100Z", "text": "..." }
  ],
  "page_info": {
    "next_cursor": "eyJ2IjoxLCJrIjpbIjIwMjUtMDktMTRUMTI6MzQ6NTcuMTAwWiIsIjEyM2U0NTY3Il0sIm8iOiJkZXNjIiwicyI6ImNyZWF0ZWRfYXQsaWQiLCJmIjoiZjg2OWJhIn0",
    "prev_cursor": "eyJ2IjoxLCJrIjpbIjIwMjUtMDktMTRUMTI6MzQ6NTcuMTAwWiIsIjEyM2U0NTY3Il0sIm8iOiJkZXNjIiwicyI6ImNyZWF0ZWRfYXQsaWQiLCJmIjoiZjg2OWJhIn0",
    "limit": 20
  }
}
```

## Pagination Navigation Patterns

This section illustrates the four primary pagination scenarios with concrete SQL examples.

### Scenario 1: Forward Pagination (DESC ordering)

**Setup**: `$orderby=created_at desc, id desc` (newest first)
**Forward cursor navigation**: Get next (older) items

```sql
-- From cursor pointing to item with created_at='2025-09-14T10:00:00.000Z', id='abc123'
WHERE (
  created_at < '2025-09-14T10:00:00.000Z'
  OR (created_at = '2025-09-14T10:00:00.000Z' AND id < 'abc123')
)
ORDER BY created_at DESC, id DESC
```

### Scenario 2: Backward Pagination (DESC ordering)

**Setup**: `$orderby=created_at desc, id desc` (newest first)
**Backward cursor navigation**: Get previous (newer) items

```sql
-- From same cursor, get previous page
WHERE (
  created_at > '2025-09-14T10:00:00.000Z'
  OR (created_at = '2025-09-14T10:00:00.000Z' AND id > 'abc123')
)
ORDER BY created_at DESC, id DESC
```

### Scenario 3: Forward Pagination (ASC ordering)

**Setup**: `$orderby=created_at asc, id asc` (oldest first)
**Forward cursor navigation**: Get next (newer) items

```sql
-- From cursor pointing to item with created_at='2025-09-14T10:00:00.000Z', id='abc123'
WHERE (
  created_at > '2025-09-14T10:00:00.000Z'
  OR (created_at = '2025-09-14T10:00:00.000Z' AND id > 'abc123')
)
ORDER BY created_at ASC, id ASC
```

### Scenario 4: Backward Pagination (ASC ordering)

**Setup**: `$orderby=created_at asc, id asc` (oldest first)
**Backward cursor navigation**: Get previous (older) items

```sql
-- From same cursor, get previous page
WHERE (
  created_at < '2025-09-14T10:00:00.000Z'
  OR (created_at = '2025-09-14T10:00:00.000Z' AND id < 'abc123')
)
ORDER BY created_at ASC, id ASC
```

### Key Principles

1. **Forward** means "next page in the canonical sort direction"
2. **Backward** means "previous page in the canonical sort direction"
3. **ORDER BY always remains the same** regardless of navigation direction
4. Only the **WHERE clause comparison operators are reversed** for backward navigation
5. **Results are always returned in canonical order**, never reversed

## Edge Cases & Guarantees

- Returning fewer than `limit` items is allowed (e.g., last page).
- `next_cursor` may be omitted if there is no further item; `prev_cursor` may be omitted on the very first page.
- Inserts/deletes during pagination:
  - The strictly monotonic tuple comparison `(primary, tiebreaker)` prevents duplicates and minimizes gaps.
  - Absolute consistency is not guaranteed without snapshots; this is acceptable for feed-like listings.
- Cursors are stateless and may be reused; optionally expire old versions server-side by rejecting outdated `v`.

## Do’s and Don’ts

- **Do** include a unique tiebreaker in the sort.
- **Do** over-fetch by one to compute existance of a next page.
- **Do** validate that filters/sort match what the cursor encodes.
- **Don’t** expose database IDs or raw fields as cursors; keep them opaque and versioned.
- **Don’t** provide `total_count` in cursor pagination.
- **Don’t** reverse item order for backward navigation; keep order canonical.

## Backward Navigation

Clients navigate backward by using `prev_cursor` as the `cursor` value in a new request. The server still returns results in canonical order; only the selection window moves.

## Versioning

- Start at `v = 1`.
- On breaking changes to cursor content or comparison semantics, bump `v` and reject older tokens with `INVALID_CURSOR`.
