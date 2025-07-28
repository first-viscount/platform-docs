# ADR-002: Apache Kafka for Event Streaming

## Status

**Accepted** - January 16, 2024

## Context

With our microservices architecture, we need a robust messaging system to enable asynchronous communication between services. The system must handle event streaming, ensure message delivery, and support both real-time and batch processing patterns.

### Requirements

- **Event Streaming**: Support for publishing and consuming business events
- **Scalability**: Handle thousands of messages per second with growth potential
- **Reliability**: Ensure message delivery with minimal data loss
- **Ordering**: Maintain event ordering within logical partitions
- **Replay**: Ability to replay events for recovery and new consumer onboarding
- **Integration**: Easy integration with Spring Boot microservices
- **Monitoring**: Comprehensive observability into message flows

### Event Types

- **Domain Events**: OrderCreated, InventoryReserved, PaymentProcessed
- **Integration Events**: Cross-service communication and coordination
- **Audit Events**: Complete audit trail of system changes
- **Analytics Events**: Business intelligence and reporting data

## Alternatives Considered

### 1. Amazon SQS/SNS

**Pros:**
- Fully managed service
- High availability and durability
- Native AWS integration
- Simple pub/sub model
- Cost-effective for low volumes

**Cons:**
- Limited message ordering guarantees
- No native event replay capability
- Message retention limited to 14 days
- Vendor lock-in to AWS
- Less suitable for event streaming patterns

### 2. RabbitMQ

**Pros:**
- Mature and stable
- Rich routing capabilities
- Good Spring integration
- Multiple messaging patterns
- Strong consistency guarantees

**Cons:**
- Not designed for high-throughput streaming
- Limited horizontal scaling
- Complex cluster management
- No built-in event replay
- Memory-based storage limitations

### 3. Apache Pulsar

**Pros:**
- Built-in multi-tenancy
- Geographic replication
- Tiered storage architecture
- Strong durability guarantees
- Schema registry integration

**Cons:**
- Relatively new with smaller ecosystem
- More complex architecture
- Higher operational overhead
- Less mature tooling
- Steeper learning curve

### 4. Apache Kafka

**Pros:**
- Excellent performance and scalability
- Built-in event replay capabilities
- Strong ordering guarantees within partitions
- Rich ecosystem and tooling
- Mature Spring integration
- Distributed, fault-tolerant architecture

**Cons:**
- Requires operational expertise
- Complex configuration options
- At-least-once delivery semantics require idempotency
- Higher resource requirements
- Zookeeper dependency (traditional setup)

## Decision

We will use **Apache Kafka** as our primary messaging and event streaming platform.

### Deployment Architecture

- **Kafka Cluster**: 3-node cluster for high availability
- **Zookeeper**: 3-node ensemble for coordination (migrating to KRaft in future)
- **Schema Registry**: Confluent Schema Registry for event schema management
- **Connect**: Kafka Connect for external system integration
- **Monitoring**: Prometheus + JMX for comprehensive monitoring

### Topic Strategy

```
Topic Naming Convention: {domain}.{entity}.{event-type}
Examples:
- order.order.created
- order.order.updated
- order.order.cancelled
- inventory.stock.reserved
- inventory.stock.released
- delivery.shipment.created
- notification.message.sent
```

### Partitioning Strategy

- **Order Events**: Partitioned by customer ID for ordering
- **Inventory Events**: Partitioned by product ID
- **Delivery Events**: Partitioned by shipment ID
- **Notification Events**: Partitioned by customer ID

## Rationale

### Why Kafka Over Alternatives

1. **Event Streaming First**: Kafka is designed specifically for event streaming use cases
2. **Replay Capability**: Essential for:
   - Recovery from failures
   - Adding new consumers to existing event streams
   - Rebuilding read models from event history
   - Debugging and audit trail analysis

3. **Performance**: Proven ability to handle millions of messages per second
4. **Ecosystem**: Excellent Spring Boot integration and rich tooling
5. **Scalability**: Horizontal scaling through partitioning
6. **Durability**: Configurable replication and persistence

### Event-Driven Architecture Benefits

1. **Loose Coupling**: Services communicate through events, not direct API calls
2. **Temporal Decoupling**: Producers and consumers don't need to be online simultaneously
3. **Scalability**: Consumers can scale independently based on processing requirements
4. **Resilience**: Failed consumers can resume from last processed offset

## Implementation Strategy

### Message Structure

```json
{
  "eventId": "550e8400-e29b-41d4-a716-446655440000",
  "eventType": "OrderCreated",
  "eventVersion": "1.0",
  "timestamp": "2024-01-15T10:00:00Z",
  "source": "order-service",
  "correlationId": "corr-123e4567-e89b-12d3-a456-426614174000",
  "data": {
    "orderId": "ord-770e8400-e29b-41d4-a716-446655440000",
    "customerId": "cust-123e4567-e89b-12d3-a456-426614174000",
    "totalAmount": 1299.99,
    "items": [...]
  },
  "metadata": {
    "userId": "user-456",
    "sessionId": "session-789"
  }
}
```

### Producer Configuration

```yaml
spring:
  kafka:
    producer:
      bootstrap-servers: kafka1:9092,kafka2:9092,kafka3:9092
      key-serializer: org.apache.kafka.common.serialization.StringSerializer
      value-serializer: org.springframework.kafka.support.serializer.JsonSerializer
      acks: all                    # Wait for all replicas
      retries: 3                   # Retry failed sends
      batch-size: 16384           # Batch size for efficiency
      linger-ms: 5                # Wait time for batching
      enable-idempotence: true    # Exactly-once semantics
```

### Consumer Configuration

```yaml
spring:
  kafka:
    consumer:
      bootstrap-servers: kafka1:9092,kafka2:9092,kafka3:9092
      group-id: ${spring.application.name}
      auto-offset-reset: earliest
      enable-auto-commit: false   # Manual offset management
      key-deserializer: org.apache.kafka.common.serialization.StringDeserializer
      value-deserializer: org.springframework.kafka.support.serializer.JsonDeserializer
      max-poll-records: 100       # Batch processing size
```

### Topic Configuration

```bash
# Order events topic
kafka-topics.sh --create \
  --topic order.order.created \
  --partitions 12 \
  --replication-factor 3 \
  --config retention.ms=2592000000 \  # 30 days
  --config cleanup.policy=compact

# Inventory events topic  
kafka-topics.sh --create \
  --topic inventory.stock.reserved \
  --partitions 6 \
  --replication-factor 3 \
  --config retention.ms=1209600000    # 14 days
```

## Consequences

### Positive

- **Event Replay**: Critical for recovery and new service onboarding
- **High Throughput**: Can handle current and projected message volumes
- **Ordering Guarantees**: Within-partition ordering for related events
- **Durability**: Configurable replication ensures no message loss
- **Ecosystem**: Rich tooling for monitoring, testing, and operations
- **Community**: Large, active community and extensive documentation

### Negative

- **Complexity**: Requires Kafka operational expertise
- **Resource Usage**: Higher memory and disk requirements than simple queues
- **At-Least-Once**: Requires idempotent consumers for exactly-once processing
- **Latency**: Slightly higher latency than direct service calls
- **Schema Evolution**: Requires careful schema versioning strategy

### Mitigation Strategies

1. **Operational Complexity**:
   - Use managed Kafka service (Confluent Cloud) for production
   - Implement comprehensive monitoring and alerting
   - Automate common operational tasks
   - Document runbooks for common scenarios

2. **Resource Requirements**:
   - Right-size brokers based on throughput requirements
   - Use tiered storage for cost optimization
   - Monitor and tune JVM settings
   - Implement retention policies to manage disk usage

3. **Consumer Idempotency**:
   - Design all consumers to be idempotent
   - Use database unique constraints
   - Implement deduplication patterns
   - Document idempotency requirements

4. **Schema Evolution**:
   - Use Schema Registry for validation
   - Follow backward compatibility rules
   - Version all event schemas
   - Plan migration strategies for breaking changes

## Event Catalog

### Order Service Events

```
order.order.created      - New order placed
order.order.paid         - Payment completed
order.order.shipped      - Order shipped
order.order.delivered    - Order delivered
order.order.cancelled    - Order cancelled
order.order.refunded     - Refund processed
```

### Inventory Service Events

```
inventory.stock.reserved     - Inventory reserved
inventory.stock.released     - Reservation released
inventory.stock.adjusted     - Manual stock adjustment
inventory.stock.low          - Low stock alert
inventory.movement.recorded  - Stock movement logged
```

### Delivery Service Events

```
delivery.shipment.created    - Shipping label created
delivery.shipment.picked_up  - Package picked up
delivery.shipment.in_transit - Package in transit
delivery.shipment.delivered  - Package delivered
delivery.shipment.exception  - Delivery exception
```

### Notification Service Events

```
notification.sent            - Notification sent
notification.delivered       - Notification delivered
notification.opened          - Email/SMS opened
notification.failed          - Delivery failed
```

## Monitoring and Observability

### Key Metrics

- **Producer Metrics**: Send rate, error rate, batch size
- **Consumer Metrics**: Consumption rate, lag, processing time
- **Broker Metrics**: Throughput, disk usage, replication lag
- **Topic Metrics**: Message count, size, retention

### Alerting Thresholds

- Consumer lag > 1000 messages
- Error rate > 1%
- Disk usage > 80%
- Replication lag > 5 seconds

### Distributed Tracing

- Correlation IDs for request tracing across services
- Kafka headers for trace propagation
- Integration with Jaeger/Zipkin for visualization

## Related Decisions

- [ADR-001: Microservices Architecture](./adr-001-microservices.md)
- [ADR-005: Platform Coordination Service](./adr-005-platform-coordination.md)

## Migration Strategy

### Phase 1: Infrastructure Setup
- Deploy Kafka cluster
- Configure monitoring and alerting
- Set up Schema Registry

### Phase 2: Core Events
- Implement order lifecycle events
- Add inventory reservation events
- Deploy notification events

### Phase 3: Advanced Patterns
- Implement event sourcing for audit
- Add analytics events
- Integrate external systems via Kafka Connect

### Phase 4: Optimization
- Fine-tune configurations
- Implement tiered storage
- Add advanced monitoring

## References

- [Apache Kafka Documentation](https://kafka.apache.org/documentation/)
- [Confluent Platform Documentation](https://docs.confluent.io/)
- [Spring for Apache Kafka](https://spring.io/projects/spring-kafka)
- [Event Streaming Patterns](https://www.confluent.io/blog/event-streaming-patterns/)
- [Building Event-Driven Microservices](https://www.oreilly.com/library/view/building-event-driven-microservices/9781492057888/)