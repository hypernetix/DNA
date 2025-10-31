# AsyncAPI & Event-Driven Architecture Guidelines

This document provides guidelines for building event-driven systems using AsyncAPI specification.

## When to Use Event-Driven Architecture

**Use Event-Driven when**:
- **Microservices integration**: Loose coupling between services
- **Real-time features**: Live updates, notifications, chat systems
- **Data synchronization**: Keep multiple systems in sync
- **Audit trails**: Complete history of system state changes
- **Scalability**: Independent scaling of event producers/consumers
- **Fault tolerance**: Resilient to individual service failures
- **Business workflows**: Complex multi-step processes with compensation

**Use AsyncAPI when**:
- **Event documentation**: Standardized event schema documentation
- **Code generation**: Generate clients and servers from schemas
- **Event governance**: Centralized event catalog and versioning
- **Team collaboration**: Clear contracts between event producers/consumers
- **Testing**: Mock event producers for integration testing

**Avoid Event-Driven when**:
- **Simple CRUD operations**: Direct API calls are more straightforward
- **Strong consistency requirements**: Eventual consistency may not be acceptable
- **Low-latency requirements**: Event processing adds overhead
- **Debugging complexity**: Distributed tracing is more complex
- **Small teams**: Additional complexity may not be justified

## Event Schema Design

**AsyncAPI Specification**:
```yaml
asyncapi: 3.0.0
info:
  title: User Events API
  version: 1.0.0
  description: User-related events

servers:
  production:
    host: events.example.com
    protocol: kafka
    description: Production Kafka cluster

channels:
  user.created:
    address: user.created
    messages:
      userCreated:
        $ref: '#/components/messages/UserCreated'

  user.updated:
    address: user.updated
    messages:
      userUpdated:
        $ref: '#/components/messages/UserUpdated'

operations:
  onUserCreated:
    action: receive
    channel:
      $ref: '#/channels/user.created'
    summary: Receive user created events

components:
  messages:
    UserCreated:
      name: UserCreated
      contentType: application/json
      payload:
        $ref: '#/components/schemas/UserCreatedPayload'
      headers:
        type: object
        properties:
          correlationId:
            type: string
            description: Correlation ID for tracing
    
  schemas:
    UserCreatedPayload:
      type: object
      required:
        - eventId
        - eventType
        - timestamp
        - data
      properties:
        eventId:
          type: string
          format: uuid
          description: Unique event identifier
        eventType:
          type: string
          const: user.created
        eventVersion:
          type: string
          default: "1.0"
          description: Event schema version
        timestamp:
          type: string
          format: date-time
          description: Event creation time (ISO-8601)
        source:
          type: string
          description: Service that produced the event
        data:
          $ref: '#/components/schemas/User'
        metadata:
          type: object
          properties:
            correlationId:
              type: string
            causationId:
              type: string
```

**AsyncAPI Documentation**:
- Publish AsyncAPI spec at `/asyncapi.json` or `/asyncapi.yaml`
- Generate event documentation from AsyncAPI spec
- Use AsyncAPI Studio for visual editing
- Generate code for event producers and consumers

## Event Patterns

**Event Sourcing**:
- Store events as source of truth
- Rebuild state from event stream
- Event versioning and migration

**CQRS (Command Query Responsibility Segregation)**:
- Separate read and write models
- Event handlers update read models
- Eventually consistent views

**Saga Pattern**:
- Distributed transaction management
- Compensating actions for rollbacks
- Event-driven orchestration

## Message Schemas

**Event Envelope**:
```json
{
  "eventId": "evt_01JCXYZ...",
  "eventType": "user.created",
  "eventVersion": "1.0",
  "timestamp": "2025-01-15T10:30:00.000Z",
  "source": "user-service",
  "data": {
    "id": "user_123",
    "email": "user@example.com",
    "name": "John Doe"
  },
  "metadata": {
    "correlationId": "req_abc123",
    "causationId": "cmd_def456"
  }
}
```

**Schema Evolution**:
- Additive changes only
- Version events explicitly
- Backward compatibility for consumers
- Deprecation timeline for old versions

## Event Delivery

**At-Least-Once Delivery**:
- Acknowledge after processing
- Idempotent event handlers
- Dead letter queues for failures

**Ordering Guarantees**:
- Partition by entity ID for ordering
- Sequence numbers for strict ordering
- Out-of-order handling strategies

**Replay Protection**:
- Event ID deduplication
- Timestamp-based replay windows
- Consumer group coordination

## Dead Letter Queues

**DLQ Configuration**:
```yaml
deadLetterQueue:
  maxRetries: 3
  retryDelay: "1m"
  maxAge: "24h"
  topics:
    - "user.events.dlq"
```

**Error Handling**:
- Categorize errors (retryable vs permanent)
- Exponential backoff for retries
- Manual intervention for DLQ

## Event Monitoring

**Metrics**:
- Event throughput and latency
- Consumer lag monitoring
- Error rates and DLQ depth
- Processing time per event type

**Alerting**:
- Consumer lag thresholds
- DLQ depth alerts
- Processing failure rates
- Schema validation errors

## References
- [AsyncAPI Specification](https://www.asyncapi.com/docs/reference/specification/v3.0.0)
- [Event-Driven Architecture](https://martinfowler.com/articles/201701-event-driven.html)
- [CQRS Pattern](https://martinfowler.com/bliki/CQRS.html)
- [Saga Pattern](https://microservices.io/patterns/data/saga.html)

