# ADR-005: Platform Coordination Service - Orchestration vs Choreography

## Status

**Accepted** - January 19, 2024

## Context

With our microservices architecture and Apache Kafka messaging system, we need to coordinate complex business processes that span multiple services. These processes, such as order fulfillment, require reliable coordination, compensation handling, and observability across service boundaries.

### Requirements

- **Process Coordination**: Manage multi-step business workflows across services
- **Failure Handling**: Graceful handling of failures with proper compensation
- **Observability**: Clear visibility into workflow progress and status
- **Consistency**: Ensure business invariants are maintained across services
- **Scalability**: Handle thousands of concurrent workflows
- **Flexibility**: Support evolving business processes and requirements

### Business Processes

1. **Order Fulfillment**:
   - Inventory reservation → Payment processing → Order confirmation → Fulfillment → Shipping → Notification

2. **Return Processing**:
   - Return authorization → Item inspection → Refund processing → Inventory update → Customer notification

3. **Inventory Replenishment**:
   - Stock level monitoring → Supplier ordering → Delivery tracking → Inventory update

4. **Customer Onboarding**:
   - Account creation → Verification → Welcome sequence → Preference setup

### Coordination Challenges

- **Distributed State**: Process state spans multiple services and databases
- **Partial Failures**: Some steps may succeed while others fail
- **Compensation**: Rolling back completed steps when later steps fail
- **Temporal Coupling**: Managing dependencies between asynchronous operations
- **Business Logic**: Keeping complex business rules centralized and maintainable

## Alternatives Considered

### 1. Pure Choreography (Event-Driven)

**Approach**: Services react to domain events without central coordination.

**Implementation Example**:
```
OrderCreated → InventoryService reserves stock → InventoryReserved
InventoryReserved → PaymentService processes → PaymentProcessed
PaymentProcessed → FulfillmentService prepares → OrderPrepared
OrderPrepared → DeliveryService ships → OrderShipped
OrderShipped → NotificationService notifies → CustomerNotified
```

**Pros:**
- High autonomy for individual services
- Natural loose coupling through events
- Excellent scalability and resilience
- No single point of failure
- Aligns well with microservices principles

**Cons:**
- Difficult to track overall process state
- Complex error handling and compensation
- Business logic scattered across services
- No central point for monitoring workflows
- Challenging to implement complex business rules
- Hard to understand complete process flow

### 2. Pure Orchestration (Central Coordinator)

**Approach**: Central service orchestrates all steps in business processes.

**Implementation**: Dedicated Platform Coordination Service that:
- Manages workflow definitions and execution
- Calls services synchronously or asynchronously
- Handles failures and compensation
- Maintains complete process state

**Pros:**
- Clear central control and monitoring
- Explicit workflow definitions
- Easier error handling and compensation
- Complete visibility into process state
- Centralized business logic
- Simpler debugging and troubleshooting

**Cons:**
- Central point of failure
- Potential performance bottleneck
- Risk of creating distributed monolith
- Services become less autonomous
- Coordinator service complexity

### 3. Hybrid Approach (Saga Pattern)

**Approach**: Combine orchestration for complex workflows with choreography for simple interactions.

**Implementation**: 
- Use orchestration (Platform Coordination Service) for complex multi-step processes
- Use choreography for simple point-to-point interactions
- Implement Saga pattern for distributed transaction management

**Pros:**
- Best of both approaches
- Flexibility to choose appropriate pattern
- Saga pattern for distributed transactions
- Maintains service autonomy where possible
- Central coordination where needed

**Cons:**
- More complex architecture
- Requires careful design decisions
- Two coordination patterns to understand
- Potential for inconsistent approaches

## Decision

We will implement a **Hybrid Approach** using the **Saga Pattern** with a dedicated **Platform Coordination Service** for orchestration of complex workflows, while maintaining choreography for simple service interactions.

### Architecture Components

#### 1. Platform Coordination Service
- **Purpose**: Orchestrate complex multi-service workflows
- **Technology**: Spring Boot with state machine implementation
- **Database**: PostgreSQL for workflow state persistence
- **Port**: 8081

#### 2. Saga Implementation
- **Pattern**: Both Orchestration-Based and Choreography-Based Sagas
- **Compensation**: Automatic rollback with compensating transactions
- **State Management**: Event sourcing for saga state

#### 3. Event Choreography
- **Simple Interactions**: Direct service-to-service communication via Kafka
- **Domain Events**: Services publish events for loosely coupled integration

### Workflow Categories

#### Complex Workflows (Orchestration)
- **Order Fulfillment**: Multi-step process requiring coordination
- **Return Processing**: Complex approval and refund workflows
- **Customer Onboarding**: Multi-service account setup process
- **Inventory Replenishment**: Supplier integration workflows

#### Simple Interactions (Choreography)
- **Inventory Updates**: Stock level changes
- **Audit Logging**: Cross-cutting audit events
- **Analytics Events**: Business intelligence data collection
- **Cache Invalidation**: Data consistency notifications

## Rationale

### Why Hybrid Over Pure Approaches

1. **Complexity Management**:
   - Orchestration for complex workflows provides clarity and control
   - Choreography for simple interactions maintains loose coupling
   - Clear boundaries between coordination patterns

2. **Failure Handling**:
   - Saga pattern provides robust compensation mechanisms
   - Central coordinator can implement sophisticated retry logic
   - Event-based patterns handle simple failures naturally

3. **Observability**:
   - Orchestrated workflows provide complete visibility
   - Choreographed interactions rely on distributed tracing
   - Hybrid approach gives observability where most needed

4. **Business Logic**:
   - Complex business rules centralized in coordination service
   - Simple business logic remains in domain services
   - Clear separation of concerns

5. **Evolution Path**:
   - Start with orchestration for critical workflows
   - Migrate to choreography as services mature
   - Flexibility to choose appropriate pattern per use case

## Implementation Strategy

### Saga Orchestrator Design

```java
@Component
public class OrderFulfillmentSaga {
    
    @Autowired
    private InventoryService inventoryService;
    
    @Autowired
    private PaymentService paymentService;
    
    @Autowired
    private FulfillmentService fulfillmentService;
    
    @SagaStart
    public void startOrderFulfillment(OrderCreatedEvent event) {
        SagaTransaction.begin()
            .step("reserveInventory")
                .invoke(() -> inventoryService.reserveInventory(event.getOrderItems()))
                .compensate(() -> inventoryService.releaseReservation(event.getOrderId()))
            .step("processPayment")
                .invoke(() -> paymentService.authorizePayment(event.getPaymentDetails()))
                .compensate(() -> paymentService.voidPayment(event.getPaymentId()))
            .step("confirmOrder")
                .invoke(() -> orderService.confirmOrder(event.getOrderId()))
                .compensate(() -> orderService.cancelOrder(event.getOrderId()))
            .step("prepareFulfillment")
                .invoke(() -> fulfillmentService.prepareOrder(event.getOrderId()))
                .compensate(() -> fulfillmentService.cancelPreparation(event.getOrderId()))
            .execute();
    }
    
    @SagaCompensation
    public void compensateOrderFulfillment(OrderFulfillmentFailedEvent event) {
        // Automatic compensation handling
        log.info("Compensating order fulfillment for order: {}", event.getOrderId());
    }
}
```

### Workflow State Management

```yaml
# Workflow Definition
workflows:
  order-fulfillment:
    name: "Order Fulfillment Process"
    version: "1.0"
    steps:
      - name: "inventory-reservation"
        service: "inventory-service"
        timeout: "30s"
        retry:
          maxAttempts: 3
          backoffMultiplier: 2
        compensation: "release-inventory"
        
      - name: "payment-processing"
        service: "payment-service"
        timeout: "2m"
        retry:
          maxAttempts: 2
          backoffMultiplier: 1.5
        compensation: "void-payment"
        dependencies: ["inventory-reservation"]
        
      - name: "order-confirmation"
        service: "order-service"
        timeout: "10s"
        dependencies: ["payment-processing"]
        compensation: "cancel-order"
```

### Event Choreography Patterns

```java
// Simple choreography example
@EventListener
public class InventoryEventHandler {
    
    @KafkaListener(topics = "order.order.created")
    public void handleOrderCreated(OrderCreatedEvent event) {
        // Simple reactive behavior - no coordination needed
        inventoryService.checkAvailability(event.getItems());
    }
    
    @KafkaListener(topics = "inventory.stock.low")
    public void handleLowStock(LowStockEvent event) {
        // Trigger replenishment workflow (orchestrated)
        coordinationService.startReplenishmentWorkflow(event.getProductId());
    }
}
```

### Monitoring and Observability

```java
@RestController
public class WorkflowController {
    
    @GetMapping("/workflows/{workflowId}/status")
    public WorkflowStatus getWorkflowStatus(@PathVariable String workflowId) {
        return WorkflowStatus.builder()
            .workflowId(workflowId)
            .status(sagaManager.getStatus(workflowId))
            .currentStep(sagaManager.getCurrentStep(workflowId))
            .completedSteps(sagaManager.getCompletedSteps(workflowId))
            .compensationPlan(sagaManager.getCompensationPlan(workflowId))
            .estimatedCompletion(sagaManager.getEstimatedCompletion(workflowId))
            .build();
    }
    
    @PostMapping("/workflows/{workflowId}/compensate")
    public ResponseEntity<Void> compensateWorkflow(@PathVariable String workflowId) {
        sagaManager.compensate(workflowId);
        return ResponseEntity.accepted().build();
    }
}
```

## Consequences

### Positive

- **Flexibility**: Can choose optimal coordination pattern per use case
- **Observability**: Complete visibility into complex workflows
- **Reliability**: Robust error handling and compensation mechanisms
- **Maintainability**: Clear separation between simple and complex coordination
- **Scalability**: Orchestrator can be scaled independently
- **Business Logic Clarity**: Complex rules centralized and explicit
- **Debugging**: Easier to trace and debug complex workflows

### Negative

- **Architectural Complexity**: Two coordination patterns to understand and maintain
- **Service Dependencies**: Platform Coordination Service becomes critical dependency
- **Coordination Overhead**: Additional network calls for orchestrated workflows
- **State Management**: Complex state persistence requirements for sagas
- **Pattern Confusion**: Potential for inconsistent application of patterns
- **Learning Curve**: Team must understand both orchestration and choreography

### Mitigation Strategies

1. **Complexity Management**:
   - Clear guidelines on when to use each pattern
   - Standard workflow templates for common processes
   - Comprehensive documentation and training
   - Code reviews to ensure consistent pattern application

2. **Service Dependencies**:
   - High availability deployment for coordination service
   - Circuit breakers and fallback mechanisms
   - Regular disaster recovery testing
   - Monitoring and alerting for service health

3. **State Management**:
   - Event sourcing for complete audit trail
   - Snapshot strategies for large sagas
   - Database backup and recovery procedures
   - State machine visualization tools

4. **Pattern Application**:
   - Architecture decision tree for pattern selection
   - Regular architecture reviews
   - Refactoring guidelines for pattern migration
   - Team training and knowledge sharing

## Workflow Decision Matrix

| Workflow Characteristic | Orchestration | Choreography |
|------------------------|---------------|--------------|
| Multiple services (>3) | ✓ | ✗ |
| Complex business rules | ✓ | ✗ |
| Compensation required | ✓ | ✗ |
| Strong consistency needed | ✓ | ✗ |
| High performance critical | ✗ | ✓ |
| Simple reactive behavior | ✗ | ✓ |
| Service autonomy priority | ✗ | ✓ |
| Event-driven integration | ✗ | ✓ |

## Implementation Phases

### Phase 1: Foundation (Weeks 1-2)
- Implement Platform Coordination Service
- Basic saga framework
- Order fulfillment workflow
- Monitoring and observability

### Phase 2: Core Workflows (Weeks 3-4)
- Return processing workflow
- Inventory replenishment workflow
- Customer onboarding workflow
- Compensation mechanisms

### Phase 3: Choreography Integration (Weeks 5-6)
- Simple event-driven interactions
- Audit and analytics events
- Cache invalidation patterns
- Performance optimization

### Phase 4: Advanced Features (Weeks 7-8)
- Workflow templates and customization
- Advanced monitoring and alerting
- Performance tuning
- Documentation and training

### Phase 5: Production Hardening (Weeks 9-10)
- Load testing and performance validation
- Disaster recovery procedures
- Security hardening
- Go-live preparation

## Monitoring and Metrics

### Orchestration Metrics
- Workflow completion rate
- Average workflow duration
- Compensation frequency
- Step failure rates
- Service dependency health

### Choreography Metrics
- Event processing latency
- Event ordering violations
- Dead letter queue depth
- Service coupling metrics
- Event schema evolution impact

### Business Metrics
- Order fulfillment time
- Customer satisfaction impact
- Revenue recognition accuracy
- Operational efficiency gains
- Error resolution time

## Security Considerations

### Workflow Security
- Secure workflow state storage
- Encrypted inter-service communication
- Authentication and authorization for workflow operations
- Audit logging for compliance

### Event Security
- Event payload encryption
- Message integrity verification
- Access control for event topics
- Sensitive data handling policies

## Related Decisions

- [ADR-001: Microservices Architecture](./adr-001-microservices.md)
- [ADR-002: Apache Kafka for Event Streaming](./adr-002-kafka-messaging.md)
- [ADR-003: Multi-Repository Structure](./adr-003-repository-structure.md)
- [ADR-004: Testing Strategy](./adr-004-testing-strategy.md)

## References

- [Saga Pattern Documentation](https://microservices.io/patterns/data/saga.html)
- [Orchestration vs Choreography](https://camunda.com/blog/2023/02/orchestration-vs-choreography/)
- [Event-Driven Architecture Patterns](https://docs.microsoft.com/en-us/azure/architecture/guide/architecture-styles/event-driven)
- [Distributed Saga Pattern](https://developers.redhat.com/articles/2021/09/21/distributed-transaction-patterns-microservices-compared)
- [Spring State Machine](https://spring.io/projects/spring-statemachine)
- [Microservices Patterns by Chris Richardson](https://microservices.io/book)