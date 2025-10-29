# GraphQL API Guidelines

This document provides guidelines for building GraphQL APIs as an alternative to REST.

## When to Use GraphQL

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

## Schema Design

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

## N+1 Query Prevention

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

## Directives

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

## Persisted Queries

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

## Federation

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

## Error Handling

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

## References
- [GraphQL Specification](https://spec.graphql.org/)
- [Apollo Federation](https://www.apollographql.com/docs/federation/)
- [DataLoader](https://github.com/graphql/dataloader)

