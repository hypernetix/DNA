# Testing Strategy

This document outlines comprehensive testing practices for building reliable, maintainable systems.

## Test Pyramid

The test pyramid guides the distribution of tests across different levels:

```
        /\
       /  \
      / E2E \
     /______\
    /        \
   /Integration\
  /____________\
 /              \
/   Unit Tests   \
/__________________\
```

**Distribution**:
- **Unit Tests**: 70% - Fast, isolated, test single units
- **Integration Tests**: 20% - Test component interactions
- **E2E Tests**: 10% - Test complete user workflows

**Rationale**:
- Unit tests are fast and cheap to maintain
- Integration tests catch interface issues
- E2E tests validate critical user paths
- Inverse pyramid leads to slow, brittle test suites

## Unit Testing

**Characteristics**:
- Test single function/method in isolation
- No external dependencies (database, network, filesystem)
- Fast execution (< 100ms per test)
- Deterministic results

**Best Practices**:
- Follow AAA pattern: Arrange, Act, Assert
- One assertion per test (or closely related assertions)
- Test behavior, not implementation
- Use descriptive test names: `test_<scenario>_<expected_behavior>`

**Example (Rust)**:
```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn test_calculate_discount_applies_percentage_correctly() {
        // Arrange
        let price = 100.0;
        let discount_percent = 10.0;
        
        // Act
        let result = calculate_discount(price, discount_percent);
        
        // Assert
        assert_eq!(result, 90.0);
    }

    #[test]
    fn test_calculate_discount_handles_zero_discount() {
        let result = calculate_discount(100.0, 0.0);
        assert_eq!(result, 100.0);
    }

    #[test]
    fn test_calculate_discount_rejects_negative_discount() {
        let result = calculate_discount(100.0, -10.0);
        assert!(result.is_err());
    }
}
```

## Integration Testing

**Characteristics**:
- Test interactions between components
- May use real external dependencies (database, message queue)
- Slower than unit tests (< 5s per test)
- Test API contracts, database queries, message handling

**Best Practices**:
- Use test containers for dependencies (Testcontainers)
- Clean state between tests (transactions, fixtures)
- Test realistic scenarios
- Verify side effects (database writes, events published)

**Example (API Integration Test)**:
```rust
#[tokio::test]
async fn test_create_user_stores_in_database() {
    // Arrange
    let app = TestApp::spawn().await;
    let payload = json!({
        "email": "test@example.com",
        "name": "Test User"
    });

    // Act
    let response = app
        .post("/v1/users")
        .json(&payload)
        .send()
        .await
        .expect("Failed to execute request");

    // Assert
    assert_eq!(response.status(), StatusCode::CREATED);
    
    let user = app.db.get_user_by_email("test@example.com").await;
    assert!(user.is_some());
    assert_eq!(user.unwrap().name, "Test User");
}
```

## Contract Testing (Consumer-Driven Contracts)

**Purpose**: Verify API contracts between services without requiring both services to be running.

**Tools**: Pact, Spring Cloud Contract, Postman Contract Testing

**Provider Side**:
```rust
// Verify provider honors consumer contracts
#[tokio::test]
async fn test_provider_honors_user_service_contract() {
    let pact_broker_url = "http://pact-broker:9292";
    
    PactVerifier::new()
        .provider("UserService")
        .pact_broker(pact_broker_url, None)
        .verify()
        .await
        .expect("Provider verification failed");
}
```

**Consumer Side**:
```rust
// Define expected contract
#[tokio::test]
async fn test_get_user_contract() {
    let mock_server = PactBuilder::new("OrderService", "UserService")
        .interaction("get user by id", |i| {
            i.given("user exists")
                .request
                .method("GET")
                .path("/v1/users/123")
                .response
                .status(200)
                .json_body(json_pattern!({
                    "id": "123",
                    "email": like!("user@example.com"),
                    "name": like!("John Doe")
                }))
        })
        .start_mock_server();

    let client = UserClient::new(mock_server.url());
    let user = client.get_user("123").await.unwrap();
    
    assert_eq!(user.id, "123");
}
```

**Benefits**:
- Independent service testing
- Early detection of breaking changes
- Living documentation of API contracts
- Faster feedback than E2E tests

## Integration Environments

**Environment Hierarchy**:
1. **Local**: Developer machine with mocked dependencies
2. **Development**: Shared environment for integration testing
3. **Staging**: Production-like environment for final validation
4. **Production**: Live environment

**Best Practices**:
- Use infrastructure as code (Terraform, Pulumi)
- Maintain environment parity
- Automate environment provisioning
- Use feature flags for gradual rollouts
- Test data management strategy

**Environment Configuration**:
```yaml
# config/environments.yml
local:
  database_url: "postgres://localhost:5432/app_dev"
  redis_url: "redis://localhost:6379"
  external_apis: "mocked"

development:
  database_url: "${DATABASE_URL}"
  redis_url: "${REDIS_URL}"
  external_apis: "sandbox"

staging:
  database_url: "${DATABASE_URL}"
  redis_url: "${REDIS_URL}"
  external_apis: "production"
  
production:
  database_url: "${DATABASE_URL}"
  redis_url: "${REDIS_URL}"
  external_apis: "production"
```

## Deterministic Tests

**Principles**:
- Same input → Same output (always)
- No flaky tests
- No time-dependent behavior
- No random data without seeding

**Common Issues**:
- **Time**: Use dependency injection for clock
- **Random**: Seed random generators
- **Concurrency**: Use synchronization primitives
- **External APIs**: Mock or use contract tests

**Example (Time Injection)**:
```rust
// Bad: Non-deterministic
fn create_user(email: &str) -> User {
    User {
        email: email.to_string(),
        created_at: Utc::now(), // Non-deterministic!
    }
}

// Good: Deterministic
trait Clock {
    fn now(&self) -> DateTime<Utc>;
}

fn create_user(email: &str, clock: &impl Clock) -> User {
    User {
        email: email.to_string(),
        created_at: clock.now(),
    }
}

// Test with fixed clock
struct FixedClock(DateTime<Utc>);
impl Clock for FixedClock {
    fn now(&self) -> DateTime<Utc> { self.0 }
}

#[test]
fn test_create_user_sets_timestamp() {
    let fixed_time = Utc.ymd(2025, 1, 15).and_hms(10, 30, 0);
    let clock = FixedClock(fixed_time);
    
    let user = create_user("test@example.com", &clock);
    
    assert_eq!(user.created_at, fixed_time);
}
```

## Property-Based Testing

**Concept**: Generate random inputs and verify properties hold for all inputs.

**Tools**: QuickCheck (Rust), Hypothesis (Python), fast-check (TypeScript)

**Example (Rust with proptest)**:
```rust
use proptest::prelude::*;

proptest! {
    #[test]
    fn test_discount_never_negative(
        price in 0.0..10000.0f64,
        discount in 0.0..100.0f64
    ) {
        let result = calculate_discount(price, discount);
        prop_assert!(result >= 0.0);
    }

    #[test]
    fn test_discount_never_exceeds_original_price(
        price in 0.0..10000.0f64,
        discount in 0.0..100.0f64
    ) {
        let result = calculate_discount(price, discount);
        prop_assert!(result <= price);
    }

    #[test]
    fn test_serialization_roundtrip(user: User) {
        let json = serde_json::to_string(&user)?;
        let deserialized: User = serde_json::from_str(&json)?;
        prop_assert_eq!(user, deserialized);
    }
}
```

**When to Use**:
- Testing invariants (e.g., sorted list stays sorted)
- Serialization/deserialization roundtrips
- Parser correctness
- Mathematical properties
- State machine transitions

## Fuzzing

**Purpose**: Find edge cases and crashes by generating random/malformed inputs.

**Tools**: cargo-fuzz (Rust), AFL, libFuzzer

**Example (Rust)**:
```rust
// fuzz/fuzz_targets/parse_json.rs
#![no_main]
use libfuzzer_sys::fuzz_target;

fuzz_target!(|data: &[u8]| {
    if let Ok(s) = std::str::from_utf8(data) {
        // Should never panic, only return error
        let _ = serde_json::from_str::<MyStruct>(s);
    }
});
```

**Run fuzzer**:
```bash
cargo install cargo-fuzz
cargo fuzz run parse_json
```

**When to Use**:
- Parsing untrusted input (JSON, XML, Protocol Buffers)
- Security-critical code
- Complex state machines
- File format handlers

## Mutation Testing

**Concept**: Modify code (mutate) and verify tests catch the changes.

**Purpose**: Measure test suite effectiveness.

**Tools**: cargo-mutants (Rust), Stryker (JavaScript), PIT (Java)

**Example**:
```rust
// Original code
fn is_adult(age: u32) -> bool {
    age >= 18
}

// Mutation 1: Change >= to >
fn is_adult(age: u32) -> bool {
    age > 18  // Should be caught by test with age=18
}

// Mutation 2: Change 18 to 17
fn is_adult(age: u32) -> bool {
    age >= 17  // Should be caught by boundary test
}
```

**Run mutation testing**:
```bash
cargo install cargo-mutants
cargo mutants
```

**Interpreting Results**:
- **Killed mutants**: Tests caught the mutation ✅
- **Survived mutants**: Tests didn't catch the mutation ❌
- **Target**: 80%+ mutation score

## Automated Test Generation

**Approaches**:

### 1. OpenAPI-Based Test Generation
Generate API tests from OpenAPI specification:

```typescript
// Generate tests from OpenAPI spec
import { generateTests } from 'openapi-test-generator';

const tests = generateTests('openapi.yaml', {
  baseUrl: 'http://localhost:3000',
  auth: { type: 'bearer', token: 'test-token' }
});

// Generates tests for:
// - All endpoints
// - Valid request examples
// - Invalid request validation
// - Response schema validation
```

### 2. Snapshot Testing
Capture output and detect unintended changes:

```rust
#[test]
fn test_user_serialization_snapshot() {
    let user = User {
        id: "123".to_string(),
        email: "test@example.com".to_string(),
        name: "Test User".to_string(),
    };
    
    let json = serde_json::to_string_pretty(&user).unwrap();
    insta::assert_snapshot!(json);
}
```

### 3. Contract-First Test Generation
Generate tests from Pact contracts:

```bash
# Generate provider verification tests
pact-stub-server --pact-dir ./pacts --port 8080

# Generate consumer tests
pact-mock-service --pact-dir ./pacts --port 8081
```

### 4. Model-Based Test Generation
Generate tests from state machines:

```rust
// Define state machine
#[derive(Debug, Clone)]
enum State { New, Active, Suspended, Closed }

#[derive(Debug, Clone)]
enum Event { Activate, Suspend, Close, Reopen }

// Generate all valid state transitions
fn generate_transition_tests() -> Vec<(State, Event, State)> {
    vec![
        (State::New, Event::Activate, State::Active),
        (State::Active, Event::Suspend, State::Suspended),
        (State::Active, Event::Close, State::Closed),
        (State::Suspended, Event::Reopen, State::Active),
        // ... auto-generate all valid transitions
    ]
}
```

## Test Organization

**Structure**:
```
tests/
├── unit/           # Unit tests (fast, isolated)
├── integration/    # Integration tests (with dependencies)
├── contract/       # Contract tests (CDC)
├── e2e/           # End-to-end tests (full system)
└── fixtures/      # Test data and helpers
```

**Naming Conventions**:
- Unit: `test_<function>_<scenario>_<expected>`
- Integration: `test_<feature>_<scenario>`
- E2E: `test_<user_story>_<happy_path|edge_case>`

## Continuous Testing

**CI Pipeline**:
```yaml
# .github/workflows/test.yml
name: Test Suite
on: [push, pull_request]

jobs:
  unit:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Run unit tests
        run: cargo test --lib
        
  integration:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:15
      redis:
        image: redis:7
    steps:
      - uses: actions/checkout@v4
      - name: Run integration tests
        run: cargo test --test '*'
        
  contract:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Verify contracts
        run: cargo test --features contract-testing
        
  mutation:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Run mutation tests
        run: cargo mutants --timeout 300
```

## Test Quality Metrics

**Coverage**:
- Line coverage: 80%+ target
- Branch coverage: 70%+ target
- Mutation score: 80%+ target

**Performance**:
- Unit tests: < 10 minutes total
- Integration tests: < 30 minutes total
- E2E tests: < 1 hour total

**Reliability**:
- Zero flaky tests
- Deterministic results
- Fast feedback (< 15 minutes for critical path)

## References
- [Test Pyramid](https://martinfowler.com/articles/practical-test-pyramid.html)
- [Contract Testing](https://pactflow.io/blog/what-is-contract-testing/)
- [Property-Based Testing](https://hypothesis.works/articles/what-is-property-based-testing/)
- [Mutation Testing](https://en.wikipedia.org/wiki/Mutation_testing)

