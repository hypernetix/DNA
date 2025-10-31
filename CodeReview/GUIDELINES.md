# Code Review Guidelines

This document provides comprehensive guidelines for effective code reviews.

## Purpose of Code Reviews

**Primary Goals**:
- **Quality**: Catch bugs, design issues, and maintainability problems
- **Knowledge sharing**: Spread domain knowledge across the team
- **Consistency**: Ensure adherence to standards and best practices
- **Learning**: Mentor junior developers and learn from peers
- **Collective ownership**: Build shared responsibility for codebase

**Not the Goal**:
- Nitpicking style (use automated formatters)
- Asserting authority
- Finding every possible issue (tests catch many)

## Review Checklist

### Functionality
- [ ] Does the code do what it's supposed to do?
- [ ] Are edge cases handled correctly?
- [ ] Are error conditions handled appropriately?
- [ ] Is input validation comprehensive?
- [ ] Are there any obvious bugs or logic errors?

### Design & Architecture
- [ ] Does the code follow established patterns?
- [ ] Is the code in the right place (proper layer/module)?
- [ ] Are abstractions appropriate (not over/under-engineered)?
- [ ] Is the code extensible for future requirements?
- [ ] Are dependencies reasonable and minimal?

### Testing
- [ ] Are there tests for new functionality?
- [ ] Do tests cover edge cases and error paths?
- [ ] Are tests deterministic and fast?
- [ ] Is test coverage adequate (80%+ for critical paths)?
- [ ] Are integration/contract tests included where needed?

### Security
- [ ] Is user input validated and sanitized?
- [ ] Are authentication and authorization checks present?
- [ ] Are secrets/credentials properly managed?
- [ ] Is sensitive data encrypted/hashed appropriately?
- [ ] Are there any SQL injection, XSS, or CSRF vulnerabilities?

### Performance
- [ ] Are there any obvious performance issues (N+1 queries)?
- [ ] Are database queries optimized with proper indexes?
- [ ] Is caching used appropriately?
- [ ] Are large datasets paginated?
- [ ] Are expensive operations done asynchronously?

### API Design
- [ ] Does the API follow REST/GraphQL/gRPC guidelines?
- [ ] Are request/response formats consistent?
- [ ] Is versioning handled correctly?
- [ ] Are error responses following Problem Details (RFC 9457)?
- [ ] Is documentation (OpenAPI/AsyncAPI) updated?

### Readability & Maintainability
- [ ] Is the code self-documenting with clear names?
- [ ] Are complex sections commented appropriately?
- [ ] Is the code DRY (Don't Repeat Yourself)?
- [ ] Are functions/methods focused and small?
- [ ] Is nesting depth reasonable (< 4 levels)?

### Documentation
- [ ] Are public APIs documented?
- [ ] Are complex algorithms explained?
- [ ] Is the README updated if needed?
- [ ] Are breaking changes documented?
- [ ] Are migration guides provided for API changes?

### Observability
- [ ] Are appropriate log levels used?
- [ ] Are errors logged with context?
- [ ] Are metrics/traces added for critical paths?
- [ ] Are request IDs propagated?
- [ ] Are rate limits and quotas logged?

### Dependencies
- [ ] Are new dependencies justified?
- [ ] Are dependency versions pinned?
- [ ] Are licenses compatible?
- [ ] Are dependencies actively maintained?
- [ ] Are security vulnerabilities checked?

## Review Heuristics

### The "Why" Test
Ask "Why?" for every significant decision:
- Why this approach over alternatives?
- Why this data structure?
- Why this abstraction?

If the answer isn't clear from code or comments, request clarification.

### The "What If" Test
Consider edge cases:
- What if the input is empty/null/negative?
- What if the database is down?
- What if the request times out?
- What if there are 1 million records?

### The "Future You" Test
Will you understand this code in 6 months?
- Are names descriptive?
- Is the logic clear?
- Are side effects obvious?

### The "Delete" Test
Can any code be removed?
- Unused functions/variables
- Dead code paths
- Unnecessary abstractions
- Redundant comments

### The "Extract" Test
Should any code be extracted?
- Repeated logic → function
- Complex conditions → named boolean
- Long functions → smaller functions
- Magic numbers → named constants

## Common Code Smells

### Design Smells

**God Object**:
```rust
// Bad: Class does too much
struct UserManager {
    fn create_user() {}
    fn send_email() {}
    fn process_payment() {}
    fn generate_report() {}
}

// Good: Single responsibility
struct UserService { fn create_user() {} }
struct EmailService { fn send_email() {} }
struct PaymentService { fn process_payment() {} }
```

**Feature Envy**:
```rust
// Bad: Method uses another object's data more than its own
impl Order {
    fn calculate_discount(&self, customer: &Customer) -> f64 {
        if customer.is_premium() {
            customer.get_discount_rate() * self.total
        } else {
            0.0
        }
    }
}

// Good: Move logic to where data lives
impl Customer {
    fn calculate_discount(&self, order_total: f64) -> f64 {
        if self.is_premium() {
            self.get_discount_rate() * order_total
        } else {
            0.0
        }
    }
}
```

**Primitive Obsession**:
```rust
// Bad: Using primitives for domain concepts
fn create_user(email: String, age: i32) -> Result<User> {
    if !email.contains('@') {
        return Err("Invalid email");
    }
    if age < 0 || age > 150 {
        return Err("Invalid age");
    }
    // ...
}

// Good: Domain types with validation
struct Email(String);
impl Email {
    fn new(s: String) -> Result<Self> {
        if s.contains('@') { Ok(Email(s)) }
        else { Err("Invalid email") }
    }
}

struct Age(u8);
impl Age {
    fn new(n: u8) -> Result<Self> {
        if n <= 150 { Ok(Age(n)) }
        else { Err("Invalid age") }
    }
}

fn create_user(email: Email, age: Age) -> User {
    // Validation already done!
}
```

### Implementation Smells

**Long Method**:
```rust
// Bad: 100+ line method
fn process_order(order: Order) -> Result<()> {
    // Validate order (20 lines)
    // Calculate totals (30 lines)
    // Apply discounts (25 lines)
    // Process payment (30 lines)
    // Send confirmation (15 lines)
}

// Good: Extract smaller methods
fn process_order(order: Order) -> Result<()> {
    validate_order(&order)?;
    let total = calculate_total(&order);
    let discounted = apply_discounts(total, &order.customer);
    process_payment(discounted)?;
    send_confirmation(&order)?;
    Ok(())
}
```

**Deep Nesting**:
```rust
// Bad: 5+ levels of nesting
if user.is_authenticated() {
    if user.has_permission("write") {
        if resource.is_available() {
            if !resource.is_locked() {
                if quota.has_capacity() {
                    // Do work
                }
            }
        }
    }
}

// Good: Guard clauses
if !user.is_authenticated() { return Err("Not authenticated"); }
if !user.has_permission("write") { return Err("No permission"); }
if !resource.is_available() { return Err("Resource unavailable"); }
if resource.is_locked() { return Err("Resource locked"); }
if !quota.has_capacity() { return Err("Quota exceeded"); }

// Do work
```

**Magic Numbers**:
```rust
// Bad: Unexplained constants
if age > 18 && score >= 750 && balance > 1000.0 {
    approve_loan();
}

// Good: Named constants
const MIN_AGE: u8 = 18;
const MIN_CREDIT_SCORE: u16 = 750;
const MIN_BALANCE: f64 = 1000.0;

if age > MIN_AGE && score >= MIN_CREDIT_SCORE && balance > MIN_BALANCE {
    approve_loan();
}
```

**Shotgun Surgery**:
```rust
// Bad: One change requires modifying many files
// Adding a field requires changes in:
// - models/user.rs
// - dto/user_dto.rs
// - mappers/user_mapper.rs
// - validators/user_validator.rs
// - database/user_repository.rs

// Good: Colocate related code, use macros/codegen
#[derive(Serialize, Deserialize, Validate, FromRow)]
struct User {
    #[validate(email)]
    email: String,
    // Single source of truth
}
```

### Testing Smells

**Test Interdependence**:
```rust
// Bad: Tests depend on execution order
#[test]
fn test_1_create_user() {
    DB.insert(user);  // Global state!
}

#[test]
fn test_2_get_user() {
    let user = DB.get(1);  // Depends on test_1
    assert!(user.is_some());
}

// Good: Independent tests
#[test]
fn test_create_user() {
    let db = TestDb::new();  // Fresh state
    db.insert(user);
}

#[test]
fn test_get_user() {
    let db = TestDb::new();
    db.insert(test_user());  // Setup own data
    let user = db.get(1);
    assert!(user.is_some());
}
```

**Obscure Test**:
```rust
// Bad: Unclear what's being tested
#[test]
fn test_user() {
    let u = User::new("a@b.c", "x", 25, true, vec![1,2], None);
    assert!(u.validate());
}

// Good: Clear intent
#[test]
fn test_user_with_valid_email_passes_validation() {
    let user = User::builder()
        .email("alice@example.com")
        .name("Alice")
        .age(25)
        .build();
    
    assert!(user.validate().is_ok());
}
```

## Review Comments Best Practices

### Tone and Language

**Be Kind and Constructive**:
```
❌ "This code is terrible"
✅ "This approach might cause issues with X. Consider using Y instead."

❌ "Why didn't you just..."
✅ "Have you considered...? It might simplify this."

❌ "This is wrong"
✅ "This might not handle edge case X. What do you think about...?"
```

**Use Questions, Not Commands**:
```
❌ "Change this to use a HashMap"
✅ "Would a HashMap be more efficient here given the lookup patterns?"

❌ "Add error handling"
✅ "What happens if this API call fails? Should we handle that?"
```

**Praise Good Code**:
```
✅ "Nice use of the builder pattern here!"
✅ "Great test coverage for edge cases"
✅ "This abstraction makes the code much clearer"
```

### Comment Categories

**Use Prefixes to Indicate Severity**:
- `[blocking]` - Must be fixed before merge
- `[suggestion]` - Nice to have, author decides
- `[question]` - Seeking clarification
- `[nit]` - Minor style issue (optional)
- `[praise]` - Positive feedback

**Examples**:
```
[blocking] This SQL query is vulnerable to injection. Use parameterized queries.

[suggestion] Consider extracting this logic into a separate function for reusability.

[question] Why did we choose approach X over Y here?

[nit] Missing trailing comma (auto-formatter will fix)

[praise] Excellent error handling with detailed context!
```

## Review Process

### Before Submitting for Review

**Author Checklist**:
- [ ] Self-review: Read your own diff
- [ ] Tests pass locally
- [ ] Linter/formatter run
- [ ] Documentation updated
- [ ] Commit messages are clear
- [ ] PR description explains "why"

**PR Description Template**:
```markdown
## Description
Brief description of changes

## Motivation
Why is this change needed?

## Changes
- List of specific changes
- Breaking changes highlighted

## Testing
How was this tested?

## Screenshots (if UI changes)

## Checklist
- [ ] Tests added/updated
- [ ] Documentation updated
- [ ] Breaking changes documented
```

### During Review

**Reviewer Responsibilities**:
1. Review within 24 hours (or set expectations)
2. Focus on high-level issues first
3. Ask questions rather than demand changes
4. Approve when good enough (not perfect)
5. Test locally if complex changes

**Review Priorities** (in order):
1. Correctness (bugs, logic errors)
2. Security vulnerabilities
3. Performance issues
4. Design and architecture
5. Readability and maintainability
6. Style and formatting (lowest priority)

### After Review

**Addressing Feedback**:
- Respond to all comments (even if just "Done")
- Ask for clarification if unclear
- Push back respectfully if you disagree
- Mark conversations as resolved when addressed

**Re-review**:
- Request re-review after significant changes
- Don't request re-review for trivial fixes

## Review Metrics

**Healthy Metrics**:
- Review turnaround: < 24 hours
- PR size: < 400 lines of code
- Review time: 30-60 minutes per PR
- Comments per PR: 5-15
- Approval rate: 70-80% after first review

**Red Flags**:
- PRs sitting for days without review
- PRs with 1000+ lines
- Rubber-stamp approvals (no comments)
- Hostile or dismissive comments

## Tools and Automation

**Automated Checks** (run before human review):
- Linting (clippy, eslint)
- Formatting (rustfmt, prettier)
- Tests (unit, integration)
- Security scanning (cargo-audit, Snyk)
- Coverage reports

**Review Tools**:
- GitHub/GitLab/Bitbucket PR interface
- Review apps for UI changes
- Diff tools for complex changes

## References
- [Google Engineering Practices](https://google.github.io/eng-practices/review/)
- [Code Review Best Practices](https://www.kevinlondon.com/2015/05/05/code-review-best-practices.html)
- [Refactoring: Improving the Design of Existing Code](https://martinfowler.com/books/refactoring.html)

