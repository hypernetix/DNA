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
pub struct Resource {
    pub id: Uuid,
    pub title: String,
    pub priority: ResourcePriority,
    pub status: ResourceStatus,
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
pub enum ResourcePriority { Low, Medium, High }

#[derive(Debug, Serialize, Deserialize, ToSchema)]
#[serde(rename_all = "camelCase")]
pub enum ResourceStatus { Open, InProgress, Resolved, Closed }
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

### Custom Error Types with thiserror

```rust
use thiserror::Error;

#[derive(Debug, Error)]
pub enum ResourceError {
    #[error("Resource not found: {id}")]
    NotFound { id: Uuid },
    
    #[error("Validation failed: {0}")]
    Validation(#[from] validator::ValidationErrors),
    
    #[error("Database error: {0}")]
    Database(#[from] sqlx::Error),
    
    #[error("Concurrent modification detected")]
    Conflict,
}

impl From<ResourceError> for Problem {
    fn from(err: ResourceError) -> Self {
        match err {
            ResourceError::NotFound { id } => Problem {
                r#type: "https://api.example.com/errors/not-found".to_string(),
                title: "Resource not found".to_string(),
                status: 404,
                detail: Some(format!("Resource {} not found", id)),
                instance: None,
                trace_id: get_current_trace_id(),
                errors: None,
            },
            ResourceError::Validation(e) => Problem {
                r#type: "https://api.example.com/errors/validation".to_string(),
                title: "Validation failed".to_string(),
                status: 422,
                detail: Some("Request validation failed".to_string()),
                instance: None,
                trace_id: get_current_trace_id(),
                errors: Some(convert_validation_errors(e)),
            },
            // ... other variants
        }
    }
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

/// List resources with cursor pagination
#[utoipa::path(
    get,
    path = "/v1/resources",
    params(ListParams),
    responses(
        (status = 200, description = "List resources", body = Envelope<Vec<Resource>>),
        (status = 422, description = "Validation error", body = Problem)
    ),
    security(("oauth2" = []))
)]
pub async fn list_resources(Query(params): Query<ListParams>) -> impl IntoResponse {
    // Example of extracting params from the filters map
    let limit = params.filters.get("limit").and_then(|s| s.parse().ok()).unwrap_or(25);
    let after = params.filters.get("after").cloned();
    // status.in=open,in_progress -> ["open", "in_progress"]
    let statuses: Option<Vec<_>> = params.filters.get("status.in").map(|s| s.split(',').collect());

    // ... database logic to fetch resources based on filters ...
    let resources: Vec<Resource> = vec![]; // Placeholder

    let response = Envelope {
        data: resources,
        meta: Meta { limit, has_next: false, has_prev: false },
        links: Links { next: None, prev: None },
    };

    // Build response with headers
    let mut headers = HeaderMap::new();
    headers.insert("traceId", "01J...".parse().unwrap());
    (headers, Json(response))
}
```

## Async Patterns

```rust
// Basic async function
pub async fn fetch_user(id: Uuid, db: &DbPool) -> Result<User, ApiError> {
    let user = sqlx::query_as!(User, "SELECT * FROM users WHERE id = $1", id)
        .fetch_one(db)
        .await?;
    
    Ok(user)
}

// Parallel async operations
use tokio::try_join;

pub async fn get_dashboard_data(
    user_id: Uuid,
    db: &DbPool,
) -> Result<DashboardData, ApiError> {
    let (user, resources, notifications) = try_join!(
        fetch_user(user_id, db),
        fetch_user_resources(user_id, db),
        fetch_user_notifications(user_id, db),
    )?;
    
    Ok(DashboardData { user, resources, notifications })
}

// Timeout handling
use tokio::time::{timeout, Duration};

pub async fn fetch_with_timeout(id: Uuid, db: &DbPool) -> Result<User, ApiError> {
    timeout(Duration::from_secs(5), fetch_user(id, db))
        .await
        .map_err(|_| ApiError::Timeout)?
}
```

## Idempotency & ETags (server hints)
- Persist `(idempotency_key, request_fingerprint, response_hash, expires_at)`
- On replay with same fingerprint: return stored response + `Idempotency-Replayed: true`
- For writes, compute and return `ETag`; clients send `If-Match` for concurrency

## Timestamps
- Use `time::OffsetDateTime` for timestamp fields.
- To ensure RFC3339 / ISO-8601 formatting with milliseconds, use `#[serde(with = "time::serde::rfc3339")]` for required fields and `#[serde(with = "time::serde::rfc3339::option")]` for optional ones.

## Test-Driven Development

### Test Organization
Tests should mirror implementation structure in a dedicated `tests/` directory:

```
src/
├── lib.rs
├── handlers/
│   └── resources.rs
└── models/
    └── resource.rs

tests/
├── common/
│   └── mod.rs
├── resources_test.rs
└── integration_test.rs
```

### Unit Tests
```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn test_resource_validation_success() {
        let request = CreateResourceRequest {
            title: "Test resource".to_string(),
            description: "Test description".to_string(),
            priority: ResourcePriority::High,
        };
        
        let result = validate_resource_request(&request);
        assert!(result.is_ok());
    }

    #[test]
    fn test_resource_validation_empty_title() {
        let request = CreateResourceRequest {
            title: "".to_string(),
            description: "Test".to_string(),
            priority: ResourcePriority::High,
        };
        
        let result = validate_resource_request(&request);
        assert!(result.is_err());
    }
}
```

### Integration Tests
```rust
// tests/resources_test.rs
use axum::http::StatusCode;

#[tokio::test]
async fn test_create_resource_endpoint() {
    let app = create_test_app().await;
    
    let response = app
        .post("/v1/resources")
        .json(&json!({
            "title": "Test resource",
            "description": "Test description",
            "priority": "high"
        }))
        .header("Authorization", "Bearer test-token")
        .send()
        .await;
    
    assert_eq!(response.status(), StatusCode::CREATED);
    assert!(response.headers().get("Location").is_some());
}
```

### Test Coverage Requirements
- All public functions must have tests
- Test both success and error paths
- Include edge cases and boundary conditions
- Test async operations thoroughly

## Code Documentation Standards

```rust
/// Creates a new resource with validation
///
/// # Arguments
///
/// * `request` - Resource creation data
/// * `db` - Database connection pool
///
/// # Returns
///
/// Returns the created resource with assigned ID
///
/// # Errors
///
/// * `ResourceError::Validation` - Invalid input data
/// * `ResourceError::Database` - Database operation failed
///
/// # Examples
///
/// ```
/// # use myapp::create_resource;
/// # async fn example(db: &DbPool) -> Result<(), Box<dyn std::error::Error>> {
/// let request = CreateResourceRequest {
///     title: "Bug report".to_string(),
///     description: "Found a bug".to_string(),
///     priority: ResourcePriority::High,
/// };
/// let resource = create_resource(request, db).await?;
/// # Ok(())
/// # }
/// ```
pub async fn create_resource(
    request: CreateResourceRequest,
    db: &DbPool,
) -> Result<Resource, ResourceError> {
    // Implementation
}
```