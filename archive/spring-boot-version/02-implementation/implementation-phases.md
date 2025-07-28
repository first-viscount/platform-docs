# Implementation Phases

This document outlines the 5-week phased rollout plan for implementing the First Viscount microservices platform.

## Overview

The implementation follows a strategic phased approach to minimize risk and ensure each component is properly tested before moving to the next phase.

### Phase Timeline

| Phase | Week | Focus | Deliverables |
|-------|------|-------|--------------|
| Phase 1 | Week 1 | Foundation & Infrastructure | Core setup, CI/CD, shared libraries |
| Phase 2 | Week 2 | Platform Coordination & Product Catalog | Core services, event bus |
| Phase 3 | Week 3 | Inventory & Order Management | Transactional services |
| Phase 4 | Week 4 | Delivery & Notifications | Supporting services |
| Phase 5 | Week 5 | Integration & Testing | End-to-end testing, performance tuning |

## Phase 1: Foundation & Infrastructure (Week 1)

### Goals
- Set up development environment
- Establish CI/CD pipelines
- Create shared libraries
- Deploy infrastructure services

### Day-by-Day Breakdown

#### Day 1-2: Repository Setup
```bash
# Create GitHub organization
# Set up repositories
for service in platform-libs platform-coordination product-catalog inventory order delivery notification platform-gateway platform-infrastructure; do
  gh repo create firstviscount/${service}-service --private
done

# Clone service template
gh repo clone firstviscount/service-template

# Initialize each repository with template
```

**Deliverables:**
- [ ] GitHub organization created
- [ ] All repositories initialized
- [ ] Branch protection rules configured
- [ ] Team access configured

#### Day 3-4: Shared Libraries
```xml
<!-- platform-libs/pom.xml -->
<modules>
  <module>common-core</module>
  <module>common-messaging</module>
  <module>common-security</module>
  <module>common-testing</module>
  <module>bom</module>
  <module>parent</module>
</modules>
```

**Key Components:**
```java
// common-core
- BaseEntity.java
- BaseException.java
- ErrorResponse.java
- ValidationUtils.java

// common-messaging
- EventPublisher.java
- EventListener.java
- KafkaConfig.java
- CloudEvent.java

// common-security
- JwtTokenProvider.java
- SecurityConfig.java
- AuthenticationFilter.java
```

#### Day 5: Infrastructure Services
```yaml
# docker-compose-infrastructure.yml
version: '3.8'
services:
  postgres:
    image: postgres:15-alpine
    environment:
      POSTGRES_PASSWORD: devpassword
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data

  mongodb:
    image: mongo:6
    ports:
      - "27017:27017"
    volumes:
      - mongo_data:/data/db

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"

  kafka:
    image: confluentinc/cp-kafka:7.4.0
    depends_on:
      - zookeeper
    ports:
      - "9092:9092"
    environment:
      KAFKA_BROKER_ID: 1
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://localhost:9092

  zookeeper:
    image: confluentinc/cp-zookeeper:7.4.0
    ports:
      - "2181:2181"
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181
```

### Phase 1 Checklist
- [ ] Development environment setup complete
- [ ] All repositories created and configured
- [ ] Shared libraries implemented and published
- [ ] Infrastructure services running
- [ ] CI/CD pipelines configured
- [ ] Documentation structure in place

## Phase 2: Platform Coordination & Product Catalog (Week 2)

### Goals
- Implement Platform Coordination Service
- Implement Product Catalog Service
- Set up event publishing
- Basic API Gateway configuration

### Platform Coordination Service

#### Day 1-2: Core Implementation
```java
@SpringBootApplication
@EnableDiscoveryClient
@EnableKafka
public class PlatformCoordinationApplication {
    public static void main(String[] args) {
        SpringApplication.run(PlatformCoordinationApplication.class, args);
    }
}

// Saga Orchestrator
@Service
@Slf4j
public class SagaOrchestrator {
    
    private final Map<String, SagaDefinition> sagas = new ConcurrentHashMap<>();
    
    public String startSaga(String sagaType, Map<String, Object> context) {
        String sagaId = UUID.randomUUID().toString();
        SagaInstance instance = new SagaInstance(sagaId, sagaType, context);
        
        SagaDefinition definition = sagas.get(sagaType);
        executeSaga(instance, definition);
        
        return sagaId;
    }
}
```

#### Day 3: Product Catalog Service
```java
// Product Entity
@Entity
@Table(name = "products")
@EntityListeners(AuditingEntityListener.class)
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class Product {
    
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @Column(nullable = false, unique = true)
    private String sku;
    
    @Column(nullable = false)
    private String name;
    
    @Column(length = 1000)
    private String description;
    
    @Column(nullable = false, precision = 10, scale = 2)
    private BigDecimal price;
    
    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "category_id")
    private Category category;
    
    @Column(nullable = false)
    private Integer stockQuantity = 0;
    
    @Column(nullable = false)
    private Boolean active = true;
    
    @CreatedDate
    private Instant createdAt;
    
    @LastModifiedDate
    private Instant updatedAt;
}
```

#### Day 4-5: Event Integration
```java
// Product Events
@Component
@Slf4j
public class ProductEventPublisher {
    
    private final KafkaTemplate<String, CloudEvent> kafkaTemplate;
    
    @EventListener
    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    public void handleProductCreated(ProductCreatedEvent event) {
        CloudEvent cloudEvent = CloudEvent.builder()
            .id(UUID.randomUUID().toString())
            .source("product-catalog-service")
            .type("com.firstviscount.product.created.v1")
            .data(event.getProduct())
            .build();
            
        kafkaTemplate.send("product-events", cloudEvent);
        log.info("Published product created event: {}", event.getProduct().getId());
    }
}
```

### Phase 2 Deliverables
- [ ] Platform Coordination Service deployed
- [ ] Saga orchestration working
- [ ] Product Catalog Service deployed
- [ ] Product CRUD operations implemented
- [ ] Event publishing configured
- [ ] Basic integration tests passing

## Phase 3: Inventory & Order Management (Week 3)

### Goals
- Implement Inventory Management Service
- Implement Order Management Service
- Integrate with Platform Coordination
- Implement distributed transactions

### Inventory Management Service

#### Day 1-2: Core Implementation
```java
// Inventory Tracking
@Entity
@Table(name = "inventory")
public class Inventory {
    
    @Id
    private Long productId;
    
    @Column(nullable = false)
    private Integer availableQuantity;
    
    @Column(nullable = false)
    private Integer reservedQuantity;
    
    @Version
    private Long version; // Optimistic locking
    
    public boolean canReserve(int quantity) {
        return availableQuantity >= quantity;
    }
    
    public void reserve(int quantity) {
        if (!canReserve(quantity)) {
            throw new InsufficientInventoryException(productId, quantity, availableQuantity);
        }
        availableQuantity -= quantity;
        reservedQuantity += quantity;
    }
    
    public void confirmReservation(int quantity) {
        reservedQuantity -= quantity;
    }
    
    public void cancelReservation(int quantity) {
        reservedQuantity -= quantity;
        availableQuantity += quantity;
    }
}
```

#### Day 3-4: Order Management Service
```java
// Order Aggregate
@Entity
@Table(name = "orders")
public class Order {
    
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @Column(nullable = false, unique = true)
    private String orderNumber;
    
    @Column(nullable = false)
    private Long customerId;
    
    @Enumerated(EnumType.STRING)
    private OrderStatus status = OrderStatus.PENDING;
    
    @OneToMany(cascade = CascadeType.ALL, orphanRemoval = true)
    @JoinColumn(name = "order_id")
    private List<OrderItem> items = new ArrayList<>();
    
    @Column(nullable = false, precision = 10, scale = 2)
    private BigDecimal totalAmount;
    
    @Embedded
    private ShippingAddress shippingAddress;
    
    @Embedded
    private PaymentDetails paymentDetails;
    
    // State machine transitions
    public void confirmPayment(String transactionId) {
        if (status != OrderStatus.PENDING) {
            throw new InvalidOrderStateException();
        }
        this.paymentDetails.setTransactionId(transactionId);
        this.status = OrderStatus.PAYMENT_CONFIRMED;
    }
    
    public void markAsProcessing() {
        if (status != OrderStatus.PAYMENT_CONFIRMED) {
            throw new InvalidOrderStateException();
        }
        this.status = OrderStatus.PROCESSING;
    }
}
```

#### Day 5: Saga Implementation
```java
// Order Placement Saga
@Component
public class OrderPlacementSaga {
    
    @SagaOrchestrationStart
    public void startOrderPlacement(OrderPlacementCommand command) {
        // Step 1: Validate order
        validateOrder(command);
        
        // Step 2: Reserve inventory
        reserveInventory(command.getItems());
        
        // Step 3: Process payment
        processPayment(command.getPaymentDetails());
        
        // Step 4: Create order
        createOrder(command);
        
        // Step 5: Send confirmation
        sendConfirmation(command.getCustomerId());
    }
    
    @SagaOrchestrationEnd
    public void completeOrderPlacement(String sagaId) {
        log.info("Order placement saga completed: {}", sagaId);
    }
    
    @SagaCompensation
    public void compensateOrderPlacement(String sagaId, Exception error) {
        log.error("Order placement saga failed: {}", sagaId, error);
        // Implement compensation logic
    }
}
```

### Phase 3 Deliverables
- [ ] Inventory Service deployed
- [ ] Inventory reservation logic working
- [ ] Order Service deployed
- [ ] Order state machine implemented
- [ ] Saga orchestration integrated
- [ ] Distributed transaction testing complete

## Phase 4: Delivery & Notifications (Week 4)

### Goals
- Implement Delivery Management Service
- Implement Notification Service
- Integrate with existing services
- Set up monitoring

### Delivery Management Service

#### Day 1-2: Core Implementation
```java
// Delivery Tracking
@Entity
@Table(name = "deliveries")
public class Delivery {
    
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @Column(nullable = false)
    private Long orderId;
    
    @Column(nullable = false, unique = true)
    private String trackingNumber;
    
    @Enumerated(EnumType.STRING)
    private DeliveryStatus status = DeliveryStatus.PENDING;
    
    @Enumerated(EnumType.STRING)
    private CarrierType carrier;
    
    @Embedded
    private Address pickupAddress;
    
    @Embedded
    private Address deliveryAddress;
    
    @Column(nullable = false)
    private Instant estimatedDeliveryDate;
    
    @OneToMany(cascade = CascadeType.ALL)
    @JoinColumn(name = "delivery_id")
    private List<DeliveryEvent> events = new ArrayList<>();
    
    public void updateStatus(DeliveryStatus newStatus, String location, String notes) {
        this.status = newStatus;
        this.events.add(new DeliveryEvent(newStatus, location, notes, Instant.now()));
    }
}
```

#### Day 3-4: Notification Service
```java
// Notification Template
@Document(collection = "notification_templates")
@Data
public class NotificationTemplate {
    
    @Id
    private String id;
    
    private String name;
    private NotificationType type;
    private String subject;
    private String bodyTemplate;
    private Map<String, String> placeholders;
    private boolean active;
    
    public String render(Map<String, Object> context) {
        String rendered = bodyTemplate;
        for (Map.Entry<String, Object> entry : context.entrySet()) {
            String placeholder = "{{" + entry.getKey() + "}}";
            rendered = rendered.replace(placeholder, String.valueOf(entry.getValue()));
        }
        return rendered;
    }
}

// Notification Service
@Service
@Slf4j
public class NotificationService {
    
    private final NotificationTemplateRepository templateRepository;
    private final EmailSender emailSender;
    private final SmsSender smsSender;
    
    public void sendNotification(SendNotificationCommand command) {
        NotificationTemplate template = templateRepository
            .findByNameAndActive(command.getTemplateName(), true)
            .orElseThrow(() -> new TemplateNotFoundException(command.getTemplateName()));
            
        String content = template.render(command.getContext());
        
        switch (template.getType()) {
            case EMAIL:
                emailSender.send(command.getRecipient(), template.getSubject(), content);
                break;
            case SMS:
                smsSender.send(command.getRecipient(), content);
                break;
            case PUSH:
                // Push notification implementation
                break;
        }
    }
}
```

#### Day 5: Monitoring Setup
```yaml
# prometheus.yml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: 'spring-boot'
    metrics_path: '/actuator/prometheus'
    static_configs:
      - targets:
        - 'platform-coordination:8081'
        - 'product-catalog:8082'
        - 'inventory:8083'
        - 'order:8084'
        - 'delivery:8085'
        - 'notification:8086'
```

### Phase 4 Deliverables
- [ ] Delivery Service deployed
- [ ] Carrier integration configured
- [ ] Notification Service deployed
- [ ] Email/SMS providers integrated
- [ ] Monitoring dashboards created
- [ ] Alert rules configured

## Phase 5: Integration & Testing (Week 5)

### Goals
- Complete end-to-end testing
- Performance testing and optimization
- Security hardening
- Production readiness check

### Day 1-2: Integration Testing

#### End-to-End Test Scenarios
```java
@SpringBootTest
@AutoConfigureMockMvc
@TestMethodOrder(MethodOrderer.OrderAnnotation.class)
public class E2EOrderFlowTest {
    
    @Test
    @Order(1)
    void testCompleteOrderFlow() {
        // 1. Create product
        Product product = createTestProduct();
        
        // 2. Add inventory
        updateInventory(product.getId(), 100);
        
        // 3. Place order
        OrderResponse order = placeOrder(product.getId(), 2);
        
        // 4. Verify inventory reduced
        assertInventory(product.getId(), 98);
        
        // 5. Verify order status
        assertOrderStatus(order.getId(), "PROCESSING");
        
        // 6. Verify notification sent
        assertNotificationSent(order.getCustomerId(), "order_confirmation");
        
        // 7. Create delivery
        DeliveryResponse delivery = createDelivery(order.getId());
        
        // 8. Update delivery status
        updateDeliveryStatus(delivery.getId(), "IN_TRANSIT");
        
        // 9. Verify customer notified
        assertNotificationSent(order.getCustomerId(), "delivery_update");
    }
}
```

### Day 3: Performance Testing

#### Load Testing with K6
```javascript
import http from 'k6/http';
import { check, sleep } from 'k6';

export let options = {
  stages: [
    { duration: '5m', target: 100 },  // Ramp up
    { duration: '10m', target: 100 }, // Stay at 100 users
    { duration: '5m', target: 0 },    // Ramp down
  ],
  thresholds: {
    http_req_duration: ['p(95)<500'], // 95% of requests under 500ms
    http_req_failed: ['rate<0.1'],    // Error rate under 10%
  },
};

export default function() {
  // Search products
  let searchRes = http.get('http://api.firstviscount.com/api/v1/products?q=coffee');
  check(searchRes, {
    'search status is 200': (r) => r.status === 200,
    'search returns results': (r) => JSON.parse(r.body).items.length > 0,
  });
  
  sleep(1);
  
  // Place order
  let orderRes = http.post('http://api.firstviscount.com/api/v1/orders', 
    JSON.stringify({
      items: [{ productId: 1, quantity: 1 }],
      shippingAddress: { /* ... */ }
    }),
    { headers: { 'Content-Type': 'application/json' } }
  );
  
  check(orderRes, {
    'order status is 201': (r) => r.status === 201,
  });
  
  sleep(2);
}
```

### Day 4: Security Hardening

#### Security Checklist
```yaml
security_checklist:
  authentication:
    - [ ] JWT tokens implemented
    - [ ] Token expiration configured
    - [ ] Refresh token rotation
    
  authorization:
    - [ ] Role-based access control
    - [ ] API endpoint authorization
    - [ ] Service-to-service auth
    
  data_protection:
    - [ ] PII encryption at rest
    - [ ] TLS for all communication
    - [ ] Secrets in vault
    
  api_security:
    - [ ] Rate limiting enabled
    - [ ] Input validation
    - [ ] SQL injection prevention
    
  monitoring:
    - [ ] Security event logging
    - [ ] Anomaly detection
    - [ ] Audit trails
```

### Day 5: Production Readiness

#### Deployment Checklist
```markdown
## Pre-Production Checklist

### Code Quality
- [ ] All tests passing (>80% coverage)
- [ ] No critical SonarQube issues
- [ ] Performance benchmarks met
- [ ] Security scan passed

### Infrastructure
- [ ] Kubernetes manifests ready
- [ ] Resource limits configured
- [ ] Auto-scaling policies set
- [ ] Backup strategy implemented

### Monitoring
- [ ] Prometheus metrics exposed
- [ ] Grafana dashboards created
- [ ] Alert rules configured
- [ ] Log aggregation working

### Documentation
- [ ] API documentation complete
- [ ] Runbooks created
- [ ] Architecture diagrams updated
- [ ] Deployment guide written

### Operations
- [ ] Rollback procedure tested
- [ ] Disaster recovery plan
- [ ] On-call rotation set up
- [ ] SLAs defined
```

### Phase 5 Deliverables
- [ ] All integration tests passing
- [ ] Performance benchmarks achieved
- [ ] Security audit completed
- [ ] Production deployment ready
- [ ] Documentation finalized
- [ ] Team training completed

## Post-Implementation (Week 6+)

### Continuous Improvement
1. **Week 6**: Production deployment and monitoring
2. **Week 7**: Performance optimization based on real data
3. **Week 8**: Feature enhancements and bug fixes
4. **Ongoing**: Regular security updates and scaling

### Success Metrics
- **Availability**: 99.9% uptime
- **Performance**: <200ms p95 latency
- **Scalability**: Handle 10,000 concurrent users
- **Quality**: <0.1% error rate
- **Security**: Zero critical vulnerabilities

## Risk Mitigation

### Technical Risks
1. **Integration Complexity**
   - Mitigation: Comprehensive integration testing
   - Fallback: Circuit breakers and graceful degradation

2. **Performance Issues**
   - Mitigation: Load testing and optimization
   - Fallback: Horizontal scaling and caching

3. **Data Consistency**
   - Mitigation: Saga pattern and event sourcing
   - Fallback: Manual reconciliation tools

### Schedule Risks
1. **Delays in Phase 2/3**
   - Mitigation: Parallel development where possible
   - Fallback: Extend timeline or reduce scope

2. **Testing Bottlenecks**
   - Mitigation: Automated testing from day 1
   - Fallback: Additional testing resources

## Conclusion

This phased implementation approach ensures:
- Incremental value delivery
- Risk mitigation through staged rollout
- Comprehensive testing at each phase
- Team learning and adaptation
- Production-ready system in 5 weeks

Regular phase retrospectives will help adjust the plan based on learnings and maintain momentum throughout the implementation.

---
*Last Updated: January 2025*