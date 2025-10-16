# Rust Backend — Implementation Guide

This file contains Rust-specific guidance and examples.

## Backend Stack & Libraries
- Routing/middleware: `axum`, `tower-http` (CORS, compression, timeouts)
- JSON & validation: `serde`, `validator`
- OpenAPI: `utoipa`, `utoipa-swagger-ui` (optional docs UI)
- Observability: `tracing`, `tracing-subscriber`, `tracing-opentelemetry`
- DB access: `sqlx` or `sea-orm`; use query builders for filters/sorts safely
- IDs & time: `uuid` (v7) or `ulid`; `time` crate for UTC (`OffsetDateTime`)

## OpenAPI Generation
- Annotate handlers with `utoipa::path`
- Export OpenAPI 3.1 JSON at `/v1/openapi.json`
- Validate in CI with an OpenAPI linter

## Data Types
```rust
use serde::{Deserialize, Serialize};
use time::OffsetDateTime;
use uuid::Uuid; // enable the v7 feature in Cargo.toml

#[derive(Debug, Serialize, Deserialize, ToSchema)]
#[serde(rename_all = "camelCase")]
pub struct Ticket {
    pub id: Uuid,
    pub title: String,
    pub priority: TicketPriority,
    pub status: TicketStatus,
    #[serde(with = "time::serde::rfc3339")]
    pub created_at: OffsetDateTime,
    #[serde(with = "time::serde::rfc3339")]
    pub updated_at: OffsetDateTime,
    #[serde(with = "time::serde::rfc3339::option")]
    #[serde(default, skip_serializing_if = "Option::is_none")]
    pub deleted_at: Option<OffsetDateTime>,
}

#[derive(Debug, Serialize, Deserialize, ToSchema)]
#[serde(rename_all = "camelCase")]
pub enum TicketPriority { Low, Medium, High }

#[derive(Debug, Serialize, Deserialize, ToSchema)]
#[serde(rename_all = "camelCase")]
pub enum TicketStatus { Open, InProgress, Resolved, Closed }
```

## Error Handling (Problem Details)

```rust
use serde::Serialize;
use utoipa::ToSchema;

/// RFC 9457 Problem Details
#[derive(Debug, Serialize, ToSchema)]
#[serde(rename_all = "camelCase")]
pub struct Problem {
    #[schema(example = "https://api.example.com/errors/validation")]
    pub r#type: String,
    #[schema(example = "Invalid request")]
    pub title: String,
    pub status: u16,
    #[serde(skip_serializing_if = "Option::is_none")]
    pub detail: Option<String>,
    #[serde(skip_serializing_if = "Option::is_none")]
    pub instance: Option<String>,
    #[serde(rename = "traceId")]
    pub trace_id: String,
    #[serde(skip_serializing_if = "Option::is_none")]
    pub errors: Option<Vec<ValidationError>>,
}

#[derive(Debug, Serialize, ToSchema)]
pub struct ValidationError {
    pub field: String,
    pub code: String,
    pub message: String,
}
```

## axum Handler with utoipa
```rust
use axum::{extract::Query, http::HeaderMap, response::{IntoResponse, Json}, Json};
use serde::{Deserialize, Serialize};
use std::collections::HashMap;
use utoipa::{IntoParams, ToSchema};

#[derive(Debug, Deserialize, IntoParams)]
pub struct ListParams {
    // We can capture all query params into a HashMap to handle filters dynamically
    #[serde(flatten)]
    pub filters: HashMap<String, String>,
}

#[derive(Debug, Serialize, ToSchema)]
#[serde(rename_all = "camelCase")]
pub struct Envelope<T> { pub data: T, pub meta: Meta, pub links: Links }

#[derive(Debug, Serialize, ToSchema)]
#[serde(rename_all = "camelCase")]
pub struct Meta { pub limit: u16, pub has_next: bool, pub has_prev: bool }

#[derive(Debug, Serialize, ToSchema)]
pub struct Links { pub next: Option<String>, pub prev: Option<String> }

/// List tickets with cursor pagination
#[utoipa::path(
    get,
    path = "/v1/tickets",
    params(ListParams),
    responses(
        (status = 200, description = "List tickets", body = Envelope<Vec<Ticket>>),
        (status = 422, description = "Validation error", body = Problem)
    ),
    security(("oauth2" = []))
)]
pub async fn list_tickets(Query(params): Query<ListParams>) -> impl IntoResponse {
    // Example of extracting params from the filters map
    let limit = params.filters.get("limit").and_then(|s| s.parse().ok()).unwrap_or(25);
    let after = params.filters.get("after").cloned();
    // status.in=open,in_progress -> ["open", "in_progress"]
    let statuses: Option<Vec<_>> = params.filters.get("status.in").map(|s| s.split(',').collect());

    // ... database logic to fetch tickets based on filters ...
    let tickets: Vec<Ticket> = vec![]; // Placeholder

    let response = Envelope {
        data: tickets,
        meta: Meta { limit, has_next: false, has_prev: false },
        links: Links { next: None, prev: None },
    };

    // Build response with headers
    let mut headers = HeaderMap::new();
    headers.insert("traceId", "01J...".parse().unwrap());
    (headers, Json(response))
}
```

## Idempotency & ETags (server hints)
- Persist `(idempotency_key, request_fingerprint, response_hash, expires_at)`
- On replay with same fingerprint: return stored response + `Idempotency-Replayed: true`
- For writes, compute and return `ETag`; clients send `If-Match` for concurrency

## Timestamps
- Use `time::OffsetDateTime` for timestamp fields.
- To ensure RFC3339 / ISO-8601 formatting with milliseconds, use `#[serde(with = "time::serde::rfc3339")]` for required fields and `#[serde(with = "time::serde::rfc3339::option")]` for optional ones.
