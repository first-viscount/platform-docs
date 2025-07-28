# Event-Driven Messaging Architecture

## Overview

First Viscount uses Redpanda (Kafka-compatible) for asynchronous event-driven communication between services. This document outlines our messaging patterns, event schemas, and implementation guidelines.

## Technology Stack

### Redpanda Configuration
```yaml
# docker-compose.yml
redpanda:
  image: docker.redpanda.com/redpandadata/redpanda:latest
  ports:
    - "9092:9092"
    - "9644:9644"  # Admin API
  command:
    - redpanda
    - start
    - --smp 2
    - --memory 2G
    - --overprovisioned
    - --node-id 0
    - --kafka-addr PLAINTEXT://0.0.0.0:9092
    - --advertise-kafka-addr PLAINTEXT://localhost:9092
```

### Client Configuration
```python
# FastAPI service configuration
from aiokafka import AIOKafkaProducer, AIOKafkaConsumer
import json

# Producer setup
producer = AIOKafkaProducer(
    bootstrap_servers='localhost:9092',
    value_serializer=lambda v: json.dumps(v).encode(),
    key_serializer=lambda k: k.encode() if k else None,
    acks='all',  # Wait for all replicas
    enable_idempotence=True  # Exactly-once semantics
)

# Consumer setup  
consumer = AIOKafkaConsumer(
    'order-events',
    bootstrap_servers='localhost:9092',
    group_id='inventory-service',
    value_deserializer=lambda v: json.loads(v.decode()),
    enable_auto_commit=False,  # Manual commit for reliability
    auto_offset_reset='earliest'
)
```

## Event Patterns

### Event Naming Convention
- **Format**: `<Entity><Action>` (e.g., `OrderCreated`, `InventoryReserved`)
- **Tense**: Past tense for events (something that happened)
- **Namespace**: Include service prefix for clarity if needed

### Event Structure
```python
# Base event schema
class BaseEvent:
    event_id: str          # UUID for idempotency
    event_type: str        # e.g., "OrderCreated"
    event_version: str     # e.g., "1.0"
    timestamp: datetime    # When event occurred
    correlation_id: str    # Trace requests across services
    source_service: str    # Service that produced event
    
# Example: Order event
class OrderCreatedEvent(BaseEvent):
    event_type: str = "OrderCreated"
    data: OrderData
    
class OrderData:
    order_id: str
    customer_id: str
    items: List[OrderItem]
    total_amount: Decimal
    status: str
```

## Topic Strategy

### Topic Naming
- **Pattern**: `<domain>-events` (e.g., `order-events`, `inventory-events`)
- **Partitioning**: By entity ID for ordering guarantees
- **Retention**: 7 days default, 30 days for audit events

### Topic Configuration
```python
# Topic creation
from kafka.admin import KafkaAdminClient, NewTopic

admin = KafkaAdminClient(bootstrap_servers='localhost:9092')

topics = [
    NewTopic(name='order-events', num_partitions=3, replication_factor=1),
    NewTopic(name='inventory-events', num_partitions=3, replication_factor=1),
    NewTopic(name='product-events', num_partitions=3, replication_factor=1),
    NewTopic(name='platform-events', num_partitions=1, replication_factor=1)
]
```

## Event Flow Examples

### Order Creation Flow
```
1. Customer places order via API
2. Order Service validates and creates order
3. Order Service publishes "OrderCreated" to order-events
4. Inventory Service consumes "OrderCreated"
5. Inventory Service reserves items
6. Inventory Service publishes "InventoryReserved" to inventory-events
7. Order Service consumes "InventoryReserved"
8. Order Service updates order status
```

### Compensation Flow (Saga Pattern)
```
1. Order Service publishes "OrderCreated"
2. Inventory Service fails to reserve (insufficient stock)
3. Inventory Service publishes "InventoryReservationFailed"
4. Order Service consumes failure event
5. Order Service publishes "OrderCancelled"
6. All services react to cancellation
```

## Implementation Patterns

### Producer Pattern
```python
class EventProducer:
    def __init__(self, bootstrap_servers: str):
        self.producer = None
        self.bootstrap_servers = bootstrap_servers
    
    async def start(self):
        self.producer = AIOKafkaProducer(
            bootstrap_servers=self.bootstrap_servers,
            value_serializer=lambda v: json.dumps(v).encode()
        )
        await self.producer.start()
    
    async def publish_event(self, topic: str, event: BaseEvent):
        # Add metadata
        event.event_id = str(uuid.uuid4())
        event.timestamp = datetime.utcnow()
        event.source_service = "order-service"
        
        # Publish with key for partitioning
        key = event.data.order_id if hasattr(event.data, 'order_id') else None
        await self.producer.send(topic, value=event.dict(), key=key)
```

### Consumer Pattern
```python
class EventConsumer:
    def __init__(self, topics: List[str], group_id: str):
        self.consumer = None
        self.topics = topics
        self.group_id = group_id
        self.handlers = {}
    
    def register_handler(self, event_type: str, handler: Callable):
        self.handlers[event_type] = handler
    
    async def start(self):
        self.consumer = AIOKafkaConsumer(
            *self.topics,
            bootstrap_servers='localhost:9092',
            group_id=self.group_id,
            value_deserializer=lambda v: json.loads(v.decode())
        )
        await self.consumer.start()
    
    async def consume_events(self):
        async for msg in self.consumer:
            event = msg.value
            event_type = event.get('event_type')
            
            if event_type in self.handlers:
                try:
                    await self.handlers[event_type](event)
                    await self.consumer.commit()
                except Exception as e:
                    # Log error, potentially dead letter queue
                    logger.error(f"Failed to process {event_type}: {e}")
```

## Error Handling

### Retry Strategy
```python
@retry(
    stop=stop_after_attempt(3),
    wait=wait_exponential(multiplier=1, min=4, max=10),
    retry=retry_if_exception_type(KafkaError)
)
async def publish_with_retry(producer, topic, event):
    await producer.send(topic, event)
```

### Dead Letter Queue
```python
async def process_with_dlq(event, handler, dlq_producer):
    try:
        await handler(event)
    except Exception as e:
        # Send to DLQ for manual processing
        await dlq_producer.send(
            'dead-letter-queue',
            value={
                'original_event': event,
                'error': str(e),
                'timestamp': datetime.utcnow().isoformat()
            }
        )
```

## Monitoring & Observability

### Key Metrics
- **Lag**: Consumer group lag per partition
- **Throughput**: Events/second per topic
- **Errors**: Failed event processing rate
- **Latency**: End-to-end event processing time

### Tracing Events
```python
# Include trace context in events
event.trace_id = get_current_trace_id()
event.span_id = get_current_span_id()

# Log event publishing
logger.info(
    "Event published",
    event_type=event.event_type,
    event_id=event.event_id,
    trace_id=event.trace_id
)
```

## Best Practices

### Do's
- ✅ Use idempotent event handlers
- ✅ Include correlation IDs for tracing
- ✅ Version your events from day one
- ✅ Use meaningful partition keys
- ✅ Implement proper error handling
- ✅ Monitor consumer lag

### Don'ts
- ❌ Don't put large payloads in events (use references)
- ❌ Don't assume event ordering across partitions
- ❌ Don't skip events (process or dead letter)
- ❌ Don't use synchronous processing for slow operations
- ❌ Don't forget about event replay scenarios

## Schema Evolution

### Backward Compatible Changes
- Adding optional fields
- Adding new event types
- Adding new topics

### Breaking Changes (Require Version Bump)
- Removing fields
- Changing field types
- Renaming fields
- Changing topic structure

### Migration Strategy
```python
# Support multiple versions
def handle_order_event(event):
    version = event.get('event_version', '1.0')
    
    if version == '1.0':
        return handle_order_v1(event)
    elif version == '2.0':
        return handle_order_v2(event)
    else:
        raise ValueError(f"Unknown version: {version}")
```

---

*This messaging architecture enables loose coupling while maintaining data consistency through well-designed event flows.*