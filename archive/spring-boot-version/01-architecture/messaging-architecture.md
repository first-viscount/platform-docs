# Messaging Architecture

## Overview

This document describes the event-driven messaging architecture using Apache Kafka for the First Viscount e-commerce platform. It covers event patterns, topic design, message schemas, and coordination strategies.

## Why Apache Kafka?

### Evaluation Criteria

| Criteria | Kafka | RabbitMQ | Pulsar | NATS | Redis Streams |
|----------|-------|----------|--------|------|---------------|
| Spring Boot Integration | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐ | ⭐⭐ | ⭐⭐⭐ |
| Performance | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ |
| Message Ordering | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐ | ⭐⭐⭐ |
| Durability | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐ | ⭐⭐⭐ |
| E-commerce Fit | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐ | ⭐⭐⭐ |

### Key Advantages

1. **Perfect Spring Boot Integration**
   - Spring Kafka provides excellent abstraction
   - Auto-configuration support
   - Extensive documentation

2. **E-commerce Workload Optimization**
   - High throughput for order events
   - Guaranteed message ordering per partition
   - Built for event sourcing patterns

3. **Operational Excellence**
   - Horizontal scalability
   - Built-in replication
   - Mature ecosystem and tooling

## Kafka Architecture

### Cluster Configuration

```yaml
# Production Configuration
kafka:
  cluster:
    brokers: 3
    replication-factor: 2
    min-in-sync-replicas: 2
    
  topics:
    order-events:
      partitions: 6
      retention: 7d
      
    inventory-events:
      partitions: 3
      retention: 3d
      
    delivery-events:
      partitions: 3
      retention: 3d
      
    notification-events:
      partitions: 2
      retention: 1d
      
    platform-commands:
      partitions: 6
      retention: 1d
```

### Topic Design

```
┌─────────────────────────────────────────────────────────────┐
│                     Kafka Topics                             │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Domain Events                    Commands                   │
│  ┌─────────────────┐           ┌─────────────────┐         │
│  │ order-events    │           │ order-commands  │         │
│  │ • OrderCreated  │           │ • CreateOrder   │         │
│  │ • OrderUpdated  │           │ • CancelOrder   │         │
│  │ • OrderCanceled │           │ • RefundOrder   │         │
│  └─────────────────┘           └─────────────────┘         │
│                                                              │
│  ┌─────────────────┐           ┌─────────────────┐         │
│  │inventory-events │           │inventory-commands│         │
│  │ • StockReserved │           │ • ReserveStock  │         │
│  │ • StockReleased │           │ • ReleaseStock  │         │
│  │ • StockUpdated  │           │ • UpdateStock   │         │
│  └─────────────────┘           └─────────────────┘         │
│                                                              │
│  Platform Events                Dead Letter Queue            │
│  ┌─────────────────┐           ┌─────────────────┐         │
│  │platform-events  │           │ dlq-events      │         │
│  │ • WorkflowStart │           │ • Failed Events │         │
│  │ • SagaCompleted │           │ • Retry Logic   │         │
│  └─────────────────┘           └─────────────────┘         │
└─────────────────────────────────────────────────────────────┘
```

## Event Patterns

### Event Types

#### 1. Domain Events
Events that represent business facts that have occurred.

```java
@Value
@Builder
public class OrderCreatedEvent implements DomainEvent {
    String eventId;
    String orderId;
    String customerId;
    List<OrderItem> items;
    BigDecimal totalAmount;
    Instant timestamp;
    
    @Override
    public String getAggregateId() {
        return orderId;
    }
    
    @Override
    public String getEventType() {
        return "OrderCreated";
    }
}
```

#### 2. Command Events
Requests to perform an action.

```java
@Value
@Builder
public class ReserveInventoryCommand implements Command {
    String commandId;
    String orderId;
    List<InventoryReservation> reservations;
    Duration timeout;
    
    @Override
    public String getTargetAggregateId() {
        return orderId;
    }
}
```

#### 3. Integration Events
Events for cross-service communication.

```java
@Value
@Builder
public class InventoryReservedEvent implements IntegrationEvent {
    String eventId;
    String orderId;
    String reservationId;
    List<ReservedItem> items;
    Instant expiresAt;
    Instant timestamp;
}
```

### Event Schema Registry

Using Confluent Schema Registry for schema evolution:

```json
{
  "type": "record",
  "name": "OrderCreatedEvent",
  "namespace": "com.firstviscount.events.order",
  "fields": [
    {"name": "eventId", "type": "string"},
    {"name": "orderId", "type": "string"},
    {"name": "customerId", "type": "string"},
    {"name": "items", "type": {
      "type": "array",
      "items": {
        "type": "record",
        "name": "OrderItem",
        "fields": [
          {"name": "productId", "type": "string"},
          {"name": "quantity", "type": "int"},
          {"name": "price", "type": "double"}
        ]
      }
    }},
    {"name": "totalAmount", "type": "double"},
    {"name": "timestamp", "type": "long", "logicalType": "timestamp-millis"}
  ]
}
```

## Platform Coordination Patterns

### Saga Orchestration

The Platform Coordination Service manages complex workflows using the Saga pattern.

```java
@Component
@Slf4j
public class OrderFulfillmentSaga {
    
    @Autowired
    private KafkaTemplate<String, Command> commandPublisher;
    
    @SagaOrchestrationStart
    public void handle(OrderCreatedEvent event) {
        log.info("Starting order fulfillment saga for order: {}", event.getOrderId());
        
        // Step 1: Reserve Inventory
        ReserveInventoryCommand command = ReserveInventoryCommand.builder()
            .orderId(event.getOrderId())
            .reservations(mapToReservations(event.getItems()))
            .timeout(Duration.ofMinutes(10))
            .build();
            
        commandPublisher.send("inventory-commands", event.getOrderId(), command);
    }
    
    @SagaEventHandler
    public void handle(InventoryReservedEvent event) {
        log.info("Inventory reserved for order: {}", event.getOrderId());
        
        // Step 2: Process Payment
        ProcessPaymentCommand command = ProcessPaymentCommand.builder()
            .orderId(event.getOrderId())
            .amount(getOrderAmount(event.getOrderId()))
            .build();
            
        commandPublisher.send("payment-commands", event.getOrderId(), command);
    }
    
    @SagaEventHandler
    public void handle(PaymentProcessedEvent event) {
        log.info("Payment processed for order: {}", event.getOrderId());
        
        // Step 3: Schedule Delivery
        ScheduleDeliveryCommand command = ScheduleDeliveryCommand.builder()
            .orderId(event.getOrderId())
            .shippingDetails(getShippingDetails(event.getOrderId()))
            .build();
            
        commandPublisher.send("delivery-commands", event.getOrderId(), command);
    }
    
    @SagaEventHandler
    public void handle(PaymentFailedEvent event) {
        log.warn("Payment failed for order: {}, initiating compensation", event.getOrderId());
        
        // Compensating transaction
        ReleaseInventoryCommand command = ReleaseInventoryCommand.builder()
            .orderId(event.getOrderId())
            .reason("Payment failed")
            .build();
            
        commandPublisher.send("inventory-commands", event.getOrderId(), command);
    }
}
```

### Event Choreography

For simpler workflows, services coordinate through events without central orchestration.

```java
@Component
@KafkaListener(topics = "order-events")
public class InventoryEventHandler {
    
    @EventHandler
    public void on(OrderCreatedEvent event) {
        // Automatically reserve inventory when order is created
        inventoryService.reserveItems(event.getItems());
        
        // Publish result
        if (reservationSuccessful) {
            publisher.publish(InventoryReservedEvent.builder()
                .orderId(event.getOrderId())
                .reservationId(reservation.getId())
                .build());
        } else {
            publisher.publish(InventoryReservationFailedEvent.builder()
                .orderId(event.getOrderId())
                .reason("Insufficient stock")
                .build());
        }
    }
}
```

## Message Publishing

### Producer Configuration

```java
@Configuration
public class KafkaProducerConfig {
    
    @Bean
    public ProducerFactory<String, Object> producerFactory() {
        Map<String, Object> props = new HashMap<>();
        props.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
        props.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, StringSerializer.class);
        props.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, JsonSerializer.class);
        
        // Reliability settings
        props.put(ProducerConfig.ACKS_CONFIG, "all");
        props.put(ProducerConfig.RETRIES_CONFIG, 3);
        props.put(ProducerConfig.ENABLE_IDEMPOTENCE_CONFIG, true);
        
        // Performance settings
        props.put(ProducerConfig.COMPRESSION_TYPE_CONFIG, "snappy");
        props.put(ProducerConfig.BATCH_SIZE_CONFIG, 16384);
        props.put(ProducerConfig.LINGER_MS_CONFIG, 10);
        
        return new DefaultKafkaProducerFactory<>(props);
    }
}
```

### Event Publisher Service

```java
@Service
@Slf4j
public class EventPublisher {
    
    private final KafkaTemplate<String, Object> kafkaTemplate;
    private final MeterRegistry meterRegistry;
    
    public CompletableFuture<SendResult<String, Object>> publish(
            String topic, String key, Object event) {
        
        log.debug("Publishing event to topic: {}, key: {}, type: {}", 
            topic, key, event.getClass().getSimpleName());
        
        return kafkaTemplate.send(topic, key, event)
            .whenComplete((result, ex) -> {
                if (ex == null) {
                    meterRegistry.counter("events.published", 
                        "topic", topic,
                        "type", event.getClass().getSimpleName()
                    ).increment();
                } else {
                    meterRegistry.counter("events.failed",
                        "topic", topic,
                        "type", event.getClass().getSimpleName()
                    ).increment();
                    log.error("Failed to publish event", ex);
                }
            });
    }
}
```

## Message Consumption

### Consumer Configuration

```java
@Configuration
@EnableKafka
public class KafkaConsumerConfig {
    
    @Bean
    public ConsumerFactory<String, Object> consumerFactory() {
        Map<String, Object> props = new HashMap<>();
        props.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
        props.put(ConsumerConfig.GROUP_ID_CONFIG, "${spring.application.name}");
        props.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class);
        props.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, JsonDeserializer.class);
        
        // Reliability settings
        props.put(ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG, false);
        props.put(ConsumerConfig.AUTO_OFFSET_RESET_CONFIG, "earliest");
        props.put(ConsumerConfig.ISOLATION_LEVEL_CONFIG, "read_committed");
        
        // Performance settings
        props.put(ConsumerConfig.MAX_POLL_RECORDS_CONFIG, 100);
        props.put(ConsumerConfig.FETCH_MIN_BYTES_CONFIG, 1024);
        
        return new DefaultKafkaConsumerFactory<>(props);
    }
    
    @Bean
    public ConcurrentKafkaListenerContainerFactory<String, Object> 
            kafkaListenerContainerFactory() {
        
        ConcurrentKafkaListenerContainerFactory<String, Object> factory = 
            new ConcurrentKafkaListenerContainerFactory<>();
        factory.setConsumerFactory(consumerFactory());
        factory.setConcurrency(3);
        factory.setAckMode(ContainerProperties.AckMode.MANUAL);
        factory.setErrorHandler(new SeekToCurrentErrorHandler(
            new DeadLetterPublishingRecoverer(kafkaTemplate()), 
            new FixedBackOff(1000L, 3)
        ));
        
        return factory;
    }
}
```

### Event Consumer Pattern

```java
@Component
@Slf4j
public class OrderEventConsumer {
    
    @KafkaListener(
        topics = "order-events",
        groupId = "inventory-service",
        containerFactory = "kafkaListenerContainerFactory"
    )
    public void handleOrderEvent(
            @Payload OrderEvent event,
            @Header(KafkaHeaders.RECEIVED_TOPIC) String topic,
            @Header(KafkaHeaders.RECEIVED_PARTITION_ID) int partition,
            @Header(KafkaHeaders.OFFSET) long offset,
            Acknowledgment acknowledgment) {
        
        try {
            log.info("Received event: {} from partition: {} offset: {}", 
                event.getEventType(), partition, offset);
            
            // Process event based on type
            switch (event.getEventType()) {
                case "OrderCreated":
                    handleOrderCreated((OrderCreatedEvent) event);
                    break;
                case "OrderCancelled":
                    handleOrderCancelled((OrderCancelledEvent) event);
                    break;
                default:
                    log.warn("Unknown event type: {}", event.getEventType());
            }
            
            // Acknowledge successful processing
            acknowledgment.acknowledge();
            
        } catch (Exception e) {
            log.error("Error processing event", e);
            // Don't acknowledge - message will be retried
            throw e;
        }
    }
}
```

## Error Handling

### Dead Letter Queue Pattern

```java
@Component
public class DeadLetterQueueHandler {
    
    @KafkaListener(topics = "dlq-events")
    public void handleDeadLetter(
            @Payload String payload,
            @Header(KafkaHeaders.RECEIVED_TOPIC) String originalTopic,
            @Header(KafkaHeaders.EXCEPTION_MESSAGE) String exceptionMessage) {
        
        log.error("Dead letter received from topic: {}, error: {}", 
            originalTopic, exceptionMessage);
        
        // Store in database for manual investigation
        deadLetterRepository.save(DeadLetter.builder()
            .originalTopic(originalTopic)
            .payload(payload)
            .errorMessage(exceptionMessage)
            .timestamp(Instant.now())
            .build());
        
        // Alert operations team
        alertingService.sendAlert(AlertLevel.HIGH, 
            "Dead letter event from " + originalTopic);
    }
}
```

### Retry Configuration

```java
@Configuration
public class RetryConfiguration {
    
    @Bean
    public RetryTemplate retryTemplate() {
        RetryTemplate template = new RetryTemplate();
        
        // Exponential backoff
        ExponentialBackOffPolicy backOffPolicy = new ExponentialBackOffPolicy();
        backOffPolicy.setInitialInterval(1000);
        backOffPolicy.setMultiplier(2);
        backOffPolicy.setMaxInterval(10000);
        template.setBackOffPolicy(backOffPolicy);
        
        // Max attempts
        SimpleRetryPolicy retryPolicy = new SimpleRetryPolicy();
        retryPolicy.setMaxAttempts(3);
        template.setRetryPolicy(retryPolicy);
        
        return template;
    }
}
```

## Message Ordering Guarantees

### Partition Key Strategy

```java
@Service
public class OrderEventPublisher {
    
    public void publishOrderEvent(OrderEvent event) {
        // Use orderId as partition key to ensure all events 
        // for an order go to the same partition
        String partitionKey = event.getOrderId();
        
        kafkaTemplate.send("order-events", partitionKey, event);
    }
    
    public void publishCustomerEvent(CustomerEvent event) {
        // Use customerId to group customer events
        String partitionKey = event.getCustomerId();
        
        kafkaTemplate.send("customer-events", partitionKey, event);
    }
}
```

## Monitoring & Observability

### Kafka Metrics

```java
@Component
public class KafkaMetricsCollector {
    
    @Autowired
    private MeterRegistry meterRegistry;
    
    @EventListener
    public void handleKafkaEvent(ConsumerStoppedEvent event) {
        meterRegistry.counter("kafka.consumer.stopped",
            "container", event.getContainer().getListenerId()
        ).increment();
    }
    
    @Scheduled(fixedDelay = 60000)
    public void collectLagMetrics() {
        kafkaAdmin.describeConsumerGroups()
            .forEach((groupId, description) -> {
                description.members().forEach(member -> {
                    long lag = calculateLag(member);
                    meterRegistry.gauge("kafka.consumer.lag",
                        Tags.of("group", groupId, "topic", member.topic()),
                        lag
                    );
                });
            });
    }
}
```

### Event Tracing

```java
@Aspect
@Component
public class EventTracingAspect {
    
    @Around("@annotation(KafkaListener)")
    public Object traceKafkaListener(ProceedingJoinPoint joinPoint) throws Throwable {
        Span span = tracer.nextSpan()
            .name("kafka-consumer")
            .tag("event.type", getEventType(joinPoint))
            .start();
            
        try (Tracer.SpanInScope ws = tracer.withSpanInScope(span)) {
            return joinPoint.proceed();
        } finally {
            span.end();
        }
    }
}
```

## Performance Optimization

### Batch Processing

```java
@KafkaListener(topics = "high-volume-events", batch = "true")
public void handleBatch(List<Event> events) {
    log.info("Processing batch of {} events", events.size());
    
    // Process in bulk for better performance
    List<ProcessedEvent> processed = events.stream()
        .map(this::processEvent)
        .collect(Collectors.toList());
        
    // Bulk save to database
    repository.saveAll(processed);
}
```

### Compression

```yaml
spring:
  kafka:
    producer:
      compression-type: snappy
    consumer:
      properties:
        fetch.max.bytes: 52428800 # 50MB
```

## Testing Strategies

### Embedded Kafka for Integration Tests

```java
@SpringBootTest
@EmbeddedKafka(
    partitions = 1,
    topics = {"test-events"},
    brokerProperties = {
        "listeners=PLAINTEXT://localhost:9092",
        "port=9092"
    }
)
class EventIntegrationTest {
    
    @Autowired
    private KafkaTemplate<String, Object> kafkaTemplate;
    
    @Test
    void testEventPublishing() {
        // Given
        OrderCreatedEvent event = createTestEvent();
        
        // When
        SendResult<String, Object> result = kafkaTemplate
            .send("test-events", event.getOrderId(), event)
            .get(10, TimeUnit.SECONDS);
        
        // Then
        assertThat(result.getRecordMetadata().hasOffset()).isTrue();
    }
}
```

### Event Flow Testing

```java
@Test
void testOrderFulfillmentFlow() {
    // Publish initial event
    publisher.publish(orderCreatedEvent());
    
    // Verify inventory reserved
    await().atMost(5, SECONDS).until(() -> 
        eventStore.findByType("InventoryReserved").size() == 1
    );
    
    // Verify payment processed
    await().atMost(5, SECONDS).until(() -> 
        eventStore.findByType("PaymentProcessed").size() == 1
    );
    
    // Verify delivery scheduled
    await().atMost(5, SECONDS).until(() -> 
        eventStore.findByType("DeliveryScheduled").size() == 1
    );
}
```

## Best Practices

### 1. Event Design
- Use past tense for event names (OrderCreated, not CreateOrder)
- Include all necessary data in events (avoid chatty communication)
- Version events for backward compatibility

### 2. Error Handling
- Implement proper retry logic
- Use dead letter queues for failed messages
- Monitor consumer lag

### 3. Performance
- Use appropriate partition counts
- Enable compression for large messages
- Batch processing where applicable

### 4. Operations
- Monitor consumer lag
- Set up alerts for critical events
- Regular backup of Kafka topics

## References

- [Microservices Design](./microservices-design.md)
- [Platform Coordination Patterns](./service-specifications/platform-coordination-service.md)
- [Event-Driven Development Guide](../05-development-guides/event-driven-development.md)