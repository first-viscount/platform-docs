# Platform Coordination Service Specification

## Overview

The Platform Coordination Service is the orchestration brain of the First Viscount e-commerce platform. It manages complex workflows, coordinates distributed transactions using the Saga pattern, and ensures business processes complete successfully or are properly compensated in case of failures.

### Key Responsibilities
- Orchestrate multi-service workflows (order fulfillment, returns, etc.)
- Manage distributed transactions using Saga pattern
- Handle compensation logic for failed transactions
- Monitor workflow execution and provide visibility
- Coordinate complex business processes
- Ensure eventual consistency across services

## API Specification

### Base URL
```
http://platform-coordination:8081/api/v1
```

### Endpoints

#### Start Order Workflow
```http
POST /workflows/order/start
Content-Type: application/json
Authorization: Bearer {token}

{
  "orderId": "550e8400-e29b-41d4-a716-446655440000",
  "customerId": "123e4567-e89b-12d3-a456-426614174000",
  "items": [
    {
      "productId": "789e0123-e89b-12d3-a456-426614174000",
      "quantity": 2,
      "price": 29.99
    }
  ],
  "paymentMethod": {
    "type": "CREDIT_CARD",
    "token": "tok_1234567890"
  },
  "shippingAddress": {
    "street": "123 Main St",
    "city": "New York",
    "state": "NY",
    "zipCode": "10001",
    "country": "US"
  }
}

Response:
{
  "workflowId": "660e8400-e29b-41d4-a716-446655440000",
  "status": "STARTED",
  "orderId": "550e8400-e29b-41d4-a716-446655440000",
  "estimatedCompletion": "2024-01-15T10:30:00Z",
  "currentStep": "INVENTORY_RESERVATION"
}
```

#### Get Workflow Status
```http
GET /workflows/{workflowId}/status
Authorization: Bearer {token}

Response:
{
  "workflowId": "660e8400-e29b-41d4-a716-446655440000",
  "type": "ORDER_FULFILLMENT",
  "status": "IN_PROGRESS",
  "currentStep": "PAYMENT_PROCESSING",
  "steps": [
    {
      "name": "INVENTORY_RESERVATION",
      "status": "COMPLETED",
      "startedAt": "2024-01-15T10:00:00Z",
      "completedAt": "2024-01-15T10:00:05Z"
    },
    {
      "name": "PAYMENT_PROCESSING",
      "status": "IN_PROGRESS",
      "startedAt": "2024-01-15T10:00:06Z"
    },
    {
      "name": "DELIVERY_SCHEDULING",
      "status": "PENDING"
    }
  ],
  "metadata": {
    "orderId": "550e8400-e29b-41d4-a716-446655440000",
    "customerId": "123e4567-e89b-12d3-a456-426614174000"
  }
}
```

#### Cancel Workflow
```http
POST /workflows/{workflowId}/cancel
Content-Type: application/json
Authorization: Bearer {token}

{
  "reason": "Customer requested cancellation",
  "initiatedBy": "CUSTOMER"
}

Response:
{
  "workflowId": "660e8400-e29b-41d4-a716-446655440000",
  "status": "CANCELLING",
  "compensationSteps": [
    "RELEASE_INVENTORY",
    "REFUND_PAYMENT",
    "CANCEL_DELIVERY"
  ]
}
```

#### List Active Workflows
```http
GET /workflows/active?page=0&size=20
Authorization: Bearer {token}

Response:
{
  "content": [
    {
      "workflowId": "660e8400-e29b-41d4-a716-446655440000",
      "type": "ORDER_FULFILLMENT",
      "status": "IN_PROGRESS",
      "startedAt": "2024-01-15T10:00:00Z",
      "metadata": {
        "orderId": "550e8400-e29b-41d4-a716-446655440000"
      }
    }
  ],
  "totalElements": 45,
  "totalPages": 3,
  "number": 0
}
```

#### Get Workflow Metrics
```http
GET /workflows/metrics
Authorization: Bearer {token}

Response:
{
  "activeWorkflows": 45,
  "completedToday": 1234,
  "failedToday": 12,
  "averageCompletionTime": "PT2M30S",
  "successRate": 99.1,
  "workflowsByType": {
    "ORDER_FULFILLMENT": 40,
    "RETURN_PROCESSING": 3,
    "INVENTORY_REPLENISHMENT": 2
  }
}
```

## Event Contracts

### Published Events

#### WorkflowStartedEvent
```json
{
  "eventId": "770e8400-e29b-41d4-a716-446655440000",
  "workflowId": "660e8400-e29b-41d4-a716-446655440000",
  "workflowType": "ORDER_FULFILLMENT",
  "correlationId": "550e8400-e29b-41d4-a716-446655440000",
  "metadata": {
    "orderId": "550e8400-e29b-41d4-a716-446655440000",
    "customerId": "123e4567-e89b-12d3-a456-426614174000"
  },
  "timestamp": "2024-01-15T10:00:00Z"
}
```

#### SagaStepCompletedEvent
```json
{
  "eventId": "880e8400-e29b-41d4-a716-446655440000",
  "workflowId": "660e8400-e29b-41d4-a716-446655440000",
  "stepName": "INVENTORY_RESERVATION",
  "status": "SUCCESS",
  "duration": "PT5S",
  "result": {
    "reservationId": "990e8400-e29b-41d4-a716-446655440000"
  },
  "timestamp": "2024-01-15T10:00:05Z"
}
```

#### WorkflowCompletedEvent
```json
{
  "eventId": "aa0e8400-e29b-41d4-a716-446655440000",
  "workflowId": "660e8400-e29b-41d4-a716-446655440000",
  "workflowType": "ORDER_FULFILLMENT",
  "status": "COMPLETED",
  "duration": "PT3M",
  "completedSteps": [
    "INVENTORY_RESERVATION",
    "PAYMENT_PROCESSING",
    "DELIVERY_SCHEDULING",
    "NOTIFICATION_SENDING"
  ],
  "timestamp": "2024-01-15T10:03:00Z"
}
```

#### WorkflowFailedEvent
```json
{
  "eventId": "bb0e8400-e29b-41d4-a716-446655440000",
  "workflowId": "660e8400-e29b-41d4-a716-446655440000",
  "workflowType": "ORDER_FULFILLMENT",
  "failedStep": "PAYMENT_PROCESSING",
  "error": {
    "code": "PAYMENT_DECLINED",
    "message": "Insufficient funds",
    "details": {}
  },
  "compensationSteps": [
    "RELEASE_INVENTORY"
  ],
  "timestamp": "2024-01-15T10:01:30Z"
}
```

### Consumed Events

The Platform Coordination Service consumes events from all other services to coordinate workflows:

- `OrderCreatedEvent` - Triggers order fulfillment workflow
- `InventoryReservedEvent` - Proceeds to payment processing
- `PaymentProcessedEvent` - Proceeds to delivery scheduling
- `DeliveryScheduledEvent` - Proceeds to notification
- `*FailedEvent` - Triggers compensation logic

## Data Model

### Database Schema

```sql
-- Workflow instances
CREATE TABLE workflows (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    workflow_type VARCHAR(100) NOT NULL,
    correlation_id VARCHAR(200),
    status VARCHAR(50) NOT NULL,
    current_step VARCHAR(200),
    context JSONB NOT NULL,
    metadata JSONB,
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    completed_at TIMESTAMP,
    
    INDEX idx_status (status),
    INDEX idx_type_status (workflow_type, status),
    INDEX idx_correlation_id (correlation_id),
    INDEX idx_created_at (created_at)
);

-- Workflow steps
CREATE TABLE workflow_steps (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    workflow_id UUID NOT NULL REFERENCES workflows(id),
    step_name VARCHAR(100) NOT NULL,
    status VARCHAR(50) NOT NULL,
    retry_count INT DEFAULT 0,
    input JSONB,
    output JSONB,
    error JSONB,
    started_at TIMESTAMP,
    completed_at TIMESTAMP,
    
    INDEX idx_workflow_id (workflow_id),
    INDEX idx_status (status)
);

-- Saga instances
CREATE TABLE sagas (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    saga_type VARCHAR(100) NOT NULL,
    workflow_id UUID REFERENCES workflows(id),
    state JSONB NOT NULL,
    compensation_state JSONB,
    version INT NOT NULL DEFAULT 1,
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    
    INDEX idx_workflow_id (workflow_id),
    INDEX idx_saga_type (saga_type)
);

-- Compensation logs
CREATE TABLE compensation_logs (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    saga_id UUID NOT NULL REFERENCES sagas(id),
    step_name VARCHAR(100) NOT NULL,
    action VARCHAR(100) NOT NULL,
    status VARCHAR(50) NOT NULL,
    details JSONB,
    executed_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    
    INDEX idx_saga_id (saga_id)
);

-- Workflow events (event sourcing)
CREATE TABLE workflow_events (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    workflow_id UUID NOT NULL REFERENCES workflows(id),
    event_type VARCHAR(100) NOT NULL,
    payload JSONB NOT NULL,
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    
    INDEX idx_workflow_id (workflow_id),
    INDEX idx_created_at (created_at),
    INDEX idx_event_type (event_type)
);
```

### Entity Models

```java
@Entity
@Table(name = "workflows")
public class Workflow {
    @Id
    @GeneratedValue
    private UUID id;
    
    @Enumerated(EnumType.STRING)
    private WorkflowType workflowType;
    
    private String correlationId;
    
    @Enumerated(EnumType.STRING)
    private WorkflowStatus status;
    
    private String currentStep;
    
    @Type(type = "jsonb")
    private Map<String, Object> context;
    
    @Type(type = "jsonb")
    private Map<String, Object> metadata;
    
    @OneToMany(mappedBy = "workflow", cascade = CascadeType.ALL)
    private List<WorkflowStep> steps;
    
    @Version
    private Long version;
    
    private Instant createdAt;
    private Instant updatedAt;
    private Instant completedAt;
}
```

## Configuration

### Application Properties

```yaml
spring:
  application:
    name: platform-coordination-service
  
  datasource:
    url: jdbc:postgresql://${DB_HOST:localhost}:5432/coordination_db
    username: ${DB_USER:coordination_user}
    password: ${DB_PASSWORD:password}
    hikari:
      maximum-pool-size: 20
      minimum-idle: 5
  
  kafka:
    bootstrap-servers: ${KAFKA_BROKERS:localhost:9092}
    consumer:
      group-id: platform-coordination
      auto-offset-reset: earliest
      enable-auto-commit: false
    producer:
      acks: all
      retries: 3
      
workflow:
  timeouts:
    order-fulfillment: PT10M
    payment-processing: PT2M
    inventory-reservation: PT1M
    
  retry:
    max-attempts: 3
    backoff-multiplier: 2
    initial-interval: PT1S
    max-interval: PT30S
    
saga:
  compensation:
    timeout: PT5M
    max-retries: 5
```

### Environment Variables

```bash
# Database
DB_HOST=postgres
DB_USER=coordination_user
DB_PASSWORD=secure_password

# Kafka
KAFKA_BROKERS=kafka1:9092,kafka2:9092,kafka3:9092

# Service Discovery
EUREKA_SERVER=http://eureka:8761/eureka

# Security
JWT_SECRET=your-secret-key
SERVICE_ACCOUNT_KEY=/secrets/service-account.json

# Monitoring
PROMETHEUS_ENDPOINT=http://prometheus:9090
JAEGER_ENDPOINT=http://jaeger:14250
```

## Dependencies

### External Services
- **Order Service** - Order information and updates
- **Inventory Service** - Stock reservation and release
- **Payment Service** - Payment processing
- **Delivery Service** - Shipping coordination
- **Notification Service** - Customer communications

### Libraries
```xml
<dependencies>
    <!-- Spring Boot -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
    
    <!-- Spring Kafka -->
    <dependency>
        <groupId>org.springframework.kafka</groupId>
        <artifactId>spring-kafka</artifactId>
    </dependency>
    
    <!-- Temporal (Workflow Engine) -->
    <dependency>
        <groupId>io.temporal</groupId>
        <artifactId>temporal-sdk</artifactId>
        <version>1.18.0</version>
    </dependency>
    
    <!-- Database -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-jpa</artifactId>
    </dependency>
    <dependency>
        <groupId>org.postgresql</groupId>
        <artifactId>postgresql</artifactId>
    </dependency>
    
    <!-- Monitoring -->
    <dependency>
        <groupId>io.micrometer</groupId>
        <artifactId>micrometer-registry-prometheus</artifactId>
    </dependency>
</dependencies>
```

## Testing

### Unit Tests
```java
@Test
void shouldStartOrderWorkflow() {
    // Given
    OrderWorkflowRequest request = createOrderWorkflowRequest();
    
    // When
    Workflow workflow = coordinationService.startOrderWorkflow(request);
    
    // Then
    assertThat(workflow.getStatus()).isEqualTo(WorkflowStatus.IN_PROGRESS);
    assertThat(workflow.getCurrentStep()).isEqualTo("INVENTORY_RESERVATION");
    verify(eventPublisher).publish(any(WorkflowStartedEvent.class));
}
```

### Integration Tests
```java
@SpringBootTest
@Testcontainers
class OrderWorkflowIntegrationTest {
    
    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:15");
    
    @Container
    static KafkaContainer kafka = new KafkaContainer(DockerImageName.parse("confluentinc/cp-kafka:7.5.0"));
    
    @Test
    void shouldCompleteOrderWorkflowEndToEnd() {
        // Start workflow
        WorkflowResponse response = startOrderWorkflow();
        
        // Simulate service responses
        publishInventoryReservedEvent(response.getWorkflowId());
        publishPaymentProcessedEvent(response.getWorkflowId());
        publishDeliveryScheduledEvent(response.getWorkflowId());
        
        // Verify completion
        await().atMost(30, SECONDS).until(() -> 
            getWorkflowStatus(response.getWorkflowId()).equals("COMPLETED")
        );
    }
}
```

## Deployment

### Kubernetes Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: platform-coordination-service
  namespace: ecommerce
spec:
  replicas: 3
  selector:
    matchLabels:
      app: platform-coordination-service
  template:
    metadata:
      labels:
        app: platform-coordination-service
    spec:
      containers:
      - name: platform-coordination
        image: firstviscount/platform-coordination-service:latest
        ports:
        - containerPort: 8081
        env:
        - name: SPRING_PROFILES_ACTIVE
          value: "production"
        - name: DB_HOST
          valueFrom:
            secretKeyRef:
              name: db-secrets
              key: host
        resources:
          requests:
            memory: "512Mi"
            cpu: "500m"
          limits:
            memory: "1Gi"
            cpu: "1000m"
        livenessProbe:
          httpGet:
            path: /actuator/health/liveness
            port: 8081
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /actuator/health/readiness
            port: 8081
          initialDelaySeconds: 20
          periodSeconds: 5
```

## Monitoring

### Health Checks

```java
@Component
public class WorkflowHealthIndicator implements HealthIndicator {
    
    @Override
    public Health health() {
        try {
            long activeWorkflows = workflowRepository.countByStatus(WorkflowStatus.IN_PROGRESS);
            long stuckWorkflows = workflowRepository.countStuckWorkflows(Duration.ofMinutes(30));
            
            if (stuckWorkflows > 10) {
                return Health.down()
                    .withDetail("stuck_workflows", stuckWorkflows)
                    .build();
            }
            
            return Health.up()
                .withDetail("active_workflows", activeWorkflows)
                .withDetail("database", "Connected")
                .build();
                
        } catch (Exception e) {
            return Health.down()
                .withException(e)
                .build();
        }
    }
}
```

### Metrics

Key metrics exposed:
- `workflow.started.total` - Total workflows started
- `workflow.completed.total` - Total workflows completed
- `workflow.failed.total` - Total workflows failed
- `workflow.duration` - Workflow execution time
- `saga.compensation.total` - Total compensations executed
- `workflow.active` - Currently active workflows

### Alerts

```yaml
groups:
  - name: platform-coordination
    rules:
      - alert: HighWorkflowFailureRate
        expr: rate(workflow_failed_total[5m]) > 0.1
        for: 5m
        annotations:
          summary: "High workflow failure rate"
          
      - alert: StuckWorkflows
        expr: workflow_stuck_count > 10
        for: 10m
        annotations:
          summary: "Too many stuck workflows"
          
      - alert: ServiceDown
        expr: up{job="platform-coordination"} == 0
        for: 1m
        annotations:
          summary: "Platform coordination service is down"
```

## Best Practices

1. **Idempotency** - All workflow steps must be idempotent
2. **Timeouts** - Configure appropriate timeouts for each step
3. **Compensation** - Always implement compensation logic
4. **Monitoring** - Track workflow metrics and set up alerts
5. **Testing** - Test both happy path and failure scenarios
6. **Versioning** - Version workflows for backward compatibility