# gRPC API Guidelines

This document provides guidelines for building gRPC APIs for high-performance service-to-service communication.

## When to Use gRPC

**Use gRPC when**:
- **Microservices communication**: High-performance service-to-service calls
- **Polyglot environments**: Language-agnostic contracts with code generation
- **Streaming data**: Real-time data transfer (logs, metrics, events)
- **Performance critical**: Lower latency and higher throughput than REST
- **Strong typing**: Contract-first development with compile-time validation
- **Load balancing**: Built-in load balancing and health checking

**Avoid gRPC when**:
- **Web browser clients**: Limited browser support (use gRPC-Web)
- **Simple HTTP APIs**: REST is more accessible
- **Firewall restrictions**: Some networks block gRPC traffic
- **Human-readable debugging**: Binary format is harder to debug

## Protocol Buffers Contracts

**Service Definition**:
```protobuf
syntax = "proto3";

package api.v1;

service UserService {
  rpc GetUser(GetUserRequest) returns (User);
  rpc ListUsers(ListUsersRequest) returns (stream User);
  rpc CreateUser(CreateUserRequest) returns (User);
  rpc UpdateUser(UpdateUserRequest) returns (User);
  rpc DeleteUser(DeleteUserRequest) returns (google.protobuf.Empty);
}

message User {
  string id = 1;
  string email = 2;
  string name = 3;
  google.protobuf.Timestamp created_at = 4;
  google.protobuf.Timestamp updated_at = 5;
}

message GetUserRequest {
  string id = 1;
}
```

**Field Numbering**:
- Never reuse field numbers
- Reserve numbers 1-15 for frequently used fields
- Use field numbers 19000-19999 for internal use
- Document field deprecation instead of removal

## Streaming Patterns

**Server Streaming**:
```protobuf
rpc ListUsers(ListUsersRequest) returns (stream User);
```
- Use for large datasets
- Implement backpressure handling
- Client can cancel mid-stream

**Client Streaming**:
```protobuf
rpc CreateUsers(stream CreateUserRequest) returns (CreateUsersResponse);
```
- Batch operations
- Reduce network round-trips
- Handle partial failures

**Bidirectional Streaming**:
```protobuf
rpc Chat(stream ChatMessage) returns (stream ChatMessage);
```
- Real-time communication
- WebSocket alternative
- Implement heartbeat/ping

## Backward Compatibility

**Schema Evolution Rules**:
- Add new fields with new numbers
- Never change field types
- Mark fields as `reserved` instead of deleting
- Use `oneof` for mutually exclusive fields

**Versioning Strategy**:
```protobuf
// Version in package name
package api.v1;

// Or use separate proto files
import "api/v1/user.proto";
import "api/v2/user.proto";
```

**Compatibility Testing**:
- Test old clients with new servers
- Test new clients with old servers
- Validate message serialization/deserialization

## Service Mesh Integration

**Load Balancing**:
- Round-robin, least-conn, consistent hash
- Health checks for service discovery
- Circuit breaker patterns

**Observability**:
- Distributed tracing (OpenTelemetry)
- Metrics collection (Prometheus)
- Structured logging

**Security**:
- mTLS for service-to-service communication
- JWT tokens in metadata
- Rate limiting per service

## Error Handling

**gRPC Status Codes**:
```protobuf
import "google/rpc/status.proto";

// In service implementation
return status.Error(codes.NotFound, "User not found")

// With details
st := status.New(codes.InvalidArgument, "Invalid user data")
st, _ = st.WithDetails(&errdetails.BadRequest{
  FieldViolations: []*errdetails.BadRequest_FieldViolation{
    {
      Field: "email",
      Description: "Invalid email format",
    },
  },
})
return st.Err()
```

**Error Mapping**:
- gRPC codes to HTTP status codes
- Structured error details
- Client error handling

## Async Operations

**Use Async APIs when**:
- **Long-running operations**: Background processing with status polling
- **Event-driven architecture**: Loose coupling between services
- **Batch processing**: Handle large datasets asynchronously
- **Integration patterns**: Webhooks, message queues, event streams
- **Scalability**: Decouple request/response for better resource utilization

## References
- [gRPC Documentation](https://grpc.io/docs/)
- [Protocol Buffers](https://protobuf.dev/)
- [gRPC Status Codes](https://grpc.github.io/grpc/core/md_doc_statuscodes.html)

