# DNA - Development Norms & Architecture

These are opinionated, concise, and LLM-friendly guidelines designed to create consistent and adaptable APIs and microservices. The core principles are stack-agnostic, with specific guidance available for Rust backends and React frontends.

## Why (Rationale)
- **Consistency**: One way to do common things reduces cognitive load.
- **Explicitness**: Types, units, timezones, and defaults are always stated.
- **Evolvability**: Versioned APIs, forward-compatible schemas, idempotent writes.
- **Observability**: End-to-end tracing, structured logs, measurable SLIs/SLOs.
- **Security first**: HTTPS, least privilege, safe defaults.

## What (Scope)
- Stack-agnostic core: protocol, JSON shape, pagination/filter/sort, errors, caching, CORS, rate limiting
- Concurrency and idempotency, async jobs, webhooks, uploads
- OpenAPI source-of-truth and client codegen (any backend/frontend)
- Optional stack-specific recommendations: Rust backend and React frontend

## Start Here
- Core API rules (any stack): see [API.md](./REST/API.md)
- Backend (Rust) specifics (optional): see [RUST.md](./languages/RUST.md)
- Frontend (React) usage patterns (optional): see [REACT.md](./languages/REACT.md)

## Key Decisions & Defaults
- **JSON**: camelCase; envelope `{ data, meta, links }`; omit absent fields (avoid nulls)
- **Timestamps**: ISO-8601 UTC with `Z`, always include milliseconds (e.g., `2025-09-01T20:00:00.000Z`)
- **Filtering**: `field.op=value` (omit `op` for eq). Example: `status.in=open,in_progress&createdAt.gte=...`
- **Pagination**: cursor-first (`limit`, `after`/`before`); default `25`, max `200`
- **Errors**: RFC 9457 Problem Details (`application/problem+json`)
- **Concurrency**: `ETag` + `If-Match` (412 on mismatch)
- **Idempotency**: `Idempotency-Key` on POST/PATCH/DELETE; retention 1h with replay detection
- **Rate limits**: RFC 9239 headers (`RateLimit-*`); 429 includes `Retry-After`
- **OpenAPI**: 3.1 as the source-of-truth; generate TS types and React hooks

## How to Adopt

### Option 1: Git Submodule (Recommended)
Keep DNA guidelines synchronized across projects:

```bash
# Add DNA as a submodule to your project
git submodule add https://github.com/hypernetix/DNA.git docs/DNA
git submodule update --init

# Reference the guidelines in your project
ln -s docs/DNA/REST/API.md API_GUIDELINES.md
```

### Option 2: Copy Guidelines
For standalone projects or when you need customized versions:

```bash
# Copy the guidelines you need
curl -o API_GUIDELINES.md https://raw.githubusercontent.com/hypernetix/DNA/main/REST/API.md
curl -o docs/rust-api-guide.md https://raw.githubusercontent.com/hypernetix/DNA/main/languages/RUST.md
```

### Step-by-Step Implementation
1) **Setup guidelines**: Use submodule or copy approach above
2) **Configure AI assistants**: Update your `.cursorrules` and `.windsurfrules` (see below)
3) **Define resources**: Model your domain per [API.md](./REST/API.md) sections 3-4
4) **Implement handlers**: Follow stack-specific patterns ([RUST.md](./languages/RUST.md), [REACT.md](./languages/REACT.md))
5) **Generate OpenAPI**: Publish specification at `/v1/openapi.json`
6) **Generate clients**: Create TypeScript types and React hooks from OpenAPI
7) **Add observability**: Implement tracing, request IDs, metrics, and rate limiting
8) **Document examples**: Provide curl and TypeScript examples for each endpoint

### AI Assistant Configuration

#### Cursor Rules (`.cursorrules`)
```markdown
# API Development Guidelines

## REST API Standards
Follow Hypernetix DNA guidelines for all API development:
- Use the guidelines from docs/DNA/REST/API.md as the primary reference
- JSON envelope format: `{ data, meta, links }`
- camelCase naming for JSON fields
- ISO-8601 timestamps with milliseconds: `2025-01-14T10:30:15.123Z`
- Cursor-based pagination with `limit`, `after`, `before` parameters
- Filter syntax: `field.op=value` (e.g., `status.in=open,urgent`)
- RFC 9457 Problem Details for all errors
- ETags for optimistic concurrency control
- Idempotency-Key header for POST/PATCH/DELETE operations

## Code Generation Requirements
When generating API code:
1. Always include comprehensive validation using the `validator` crate
2. Implement proper error handling with Problem Details format
3. Add OpenAPI documentation with `utoipa` annotations
4. Include request/response examples in documentation
5. Use UUID v7 for all resource identifiers
6. Implement soft deletes with `deletedAt` timestamps
7. Add proper CORS configuration for web clients
8. Include tracing and observability headers

## Client Code Requirements
For React/TypeScript clients:
1. Use TanStack Query for all API interactions
2. Implement proper error handling with typed Problem Details
3. Add retry logic with exponential backoff
4. Include optimistic updates for better UX
5. Generate types from OpenAPI specifications
6. Implement proper cache invalidation strategies

Refer to docs/DNA/REST/ for complete implementation examples and patterns.
```

#### Windsurf Rules (`.windsurfrules`)
```yaml
api_standards:
  framework: "Hypernetix API Guidelines"
  reference: "docs/DNA/REST/API.md"

conventions:
  json_format:
    envelope: "{ data, meta, links }"
    naming: "camelCase"
    timestamps: "ISO-8601 with milliseconds (.SSS)"

  pagination:
    type: "cursor-based"
    params: ["limit", "after", "before"]
    defaults: { limit: 25, max: 200 }

  filtering:
    syntax: "field.op=value"
    operators: ["eq", "ne", "lt", "lte", "gt", "gte", "in", "nin", "contains"]
    examples: ["status.in=open,urgent", "createdAt.gte=2025-01-01T00:00:00.000Z"]

  errors:
    format: "RFC 9457 Problem Details"
    content_type: "application/problem+json"
    required_fields: ["type", "title", "status", "traceId"]

  concurrency:
    method: "ETags with If-Match headers"
    conflict_status: 412

  idempotency:
    header: "Idempotency-Key"
    methods: ["POST", "PATCH", "DELETE"]
    retention: "1 hour"

code_generation:
  rust:
    validation: "validator crate with derive macros"
    openapi: "utoipa with comprehensive documentation"
    error_handling: "thiserror with Problem Details"
    database: "sqlx with proper query building"

  typescript:
    client: "TanStack Query with custom hooks"
    types: "Generated from OpenAPI specs"
    error_handling: "Typed Problem Details with retry logic"
    optimization: "Optimistic updates and cache management"

required_patterns:
  - "UUID v7 for all resource identifiers"
  - "Soft deletes with deletedAt timestamps"
  - "Request tracing with traceparent headers"
  - "Rate limiting with RateLimit-* headers"
  - "CORS configuration for web clients"
  - "Comprehensive request/response examples"
```

### Integration Examples

#### Makefile Integration
```makefile
# Update Hypernetix Guidelines
update-guidelines:
	git submodule update --remote docs/DNA

# Validate API against guidelines
validate-api:
	@echo "Checking API compliance with Hypernetix API guidelines..."
	# Add your validation scripts here
```

#### CI/CD Integration
```yaml
# .github/workflows/api-compliance.yml
name: API Compliance Check
on: [push, pull_request]
jobs:
  validate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          submodules: true
      - name: Check API Guideline Compliance
        run: |
          # Validate OpenAPI spec against API Guideline patterns
          # Check for required headers, envelope format, etc.
```

## References
- Problem Details (RFC 9457): https://www.rfc-editor.org/rfc/rfc9457
- RateLimit Fields (RFC 9239): https://www.rfc-editor.org/rfc/rfc9239
- W3C Trace Context: https://www.w3.org/TR/trace-context/
- JSON Merge Patch (RFC 7396): https://www.rfc-editor.org/rfc/rfc7396
- Sunset Header (RFC 8594): https://www.rfc-editor.org/rfc/rfc8594
