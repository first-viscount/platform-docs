# Data Architecture

## Overview

This document outlines the data architecture strategy for the First Viscount e-commerce platform, including database design patterns, data consistency approaches, and storage solutions for each microservice.

## Core Principles

### 1. Database per Service
- Each microservice owns its data
- No direct database access between services
- Data sharing through APIs or events
- Service autonomy and independent scaling

### 2. Polyglot Persistence
- Choose the right database for each use case
- PostgreSQL for transactional data
- MongoDB for document storage
- Redis for caching and sessions

### 3. Event Sourcing (Selective)
- Applied to critical business domains
- Complete audit trail
- Time-travel capabilities
- Event replay for debugging

## Database Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    Service Databases                         │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Platform Coordination          Product Catalog              │
│  ┌──────────────────┐         ┌──────────────────┐         │
│  │   PostgreSQL     │         │   PostgreSQL     │         │
│  │  - Workflows     │         │  - Products      │         │
│  │  - Saga State    │         │  - Categories    │         │
│  │  - Events        │         │  - Prices        │         │
│  └──────────────────┘         └──────────────────┘         │
│          │                             │                     │
│          │                        ┌────┴────┐                │
│          │                        │  Redis  │                │
│          │                        │ (Cache) │                │
│          │                        └─────────┘                │
│                                                              │
│  Inventory Service             Order Management              │
│  ┌──────────────────┐         ┌──────────────────┐         │
│  │   PostgreSQL     │         │   PostgreSQL     │         │
│  │  - Stock Levels  │         │  - Orders        │         │
│  │  - Reservations  │         │  - Payments      │         │
│  │  - Movements     │         │  - Order Items   │         │
│  └──────────────────┘         └──────────────────┘         │
│                                                              │
│  Delivery Service              Notification Service          │
│  ┌──────────────────┐         ┌──────────────────┐         │
│  │   PostgreSQL     │         │    MongoDB       │         │
│  │  - Shipments     │         │  - Templates     │         │
│  │  - Tracking      │         │  - Send History  │         │
│  │  - Carriers      │         │  - Preferences   │         │
│  └──────────────────┘         └──────────────────┘         │
└──────────────────────────────────────────────────────────────┘
```

## Service-Specific Data Models

### 1. Platform Coordination Service

**Database**: PostgreSQL

```sql
-- Workflow instances
CREATE TABLE workflows (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    workflow_type VARCHAR(100) NOT NULL,
    status VARCHAR(50) NOT NULL,
    current_step VARCHAR(200),
    context JSONB NOT NULL,
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    completed_at TIMESTAMP,
    
    INDEX idx_status (status),
    INDEX idx_created_at (created_at),
    INDEX idx_workflow_type (workflow_type)
);

-- Saga instances
CREATE TABLE sagas (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    saga_type VARCHAR(100) NOT NULL,
    associated_id VARCHAR(200) NOT NULL,
    state JSONB NOT NULL,
    version INT NOT NULL DEFAULT 1,
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    
    INDEX idx_associated_id (associated_id),
    INDEX idx_saga_type (saga_type)
);

-- Workflow events (Event Sourcing)
CREATE TABLE workflow_events (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    workflow_id UUID NOT NULL REFERENCES workflows(id),
    event_type VARCHAR(100) NOT NULL,
    payload JSONB NOT NULL,
    timestamp TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    
    INDEX idx_workflow_id (workflow_id),
    INDEX idx_timestamp (timestamp)
);
```

### 2. Product Catalog Service

**Database**: PostgreSQL + Redis Cache

```sql
-- Products
CREATE TABLE products (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    sku VARCHAR(100) UNIQUE NOT NULL,
    name VARCHAR(255) NOT NULL,
    description TEXT,
    category_id UUID REFERENCES categories(id),
    brand VARCHAR(100),
    status VARCHAR(50) NOT NULL DEFAULT 'ACTIVE',
    attributes JSONB,
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    
    INDEX idx_sku (sku),
    INDEX idx_category_id (category_id),
    INDEX idx_status (status),
    INDEX idx_brand (brand)
);

-- Categories
CREATE TABLE categories (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    parent_id UUID REFERENCES categories(id),
    name VARCHAR(100) NOT NULL,
    slug VARCHAR(100) UNIQUE NOT NULL,
    path LTREE NOT NULL,
    attributes_schema JSONB,
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    
    INDEX idx_path ON categories USING GIST (path),
    INDEX idx_parent_id (parent_id)
);

-- Prices
CREATE TABLE prices (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    product_id UUID NOT NULL REFERENCES products(id),
    price DECIMAL(10,2) NOT NULL,
    currency VARCHAR(3) NOT NULL DEFAULT 'USD',
    effective_from TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    effective_to TIMESTAMP,
    price_type VARCHAR(50) NOT NULL DEFAULT 'REGULAR',
    
    INDEX idx_product_id (product_id),
    INDEX idx_effective_dates (effective_from, effective_to)
);

-- Full-text search
CREATE INDEX idx_product_search ON products 
USING GIN (to_tsvector('english', name || ' ' || COALESCE(description, '')));
```

### 3. Inventory Service

**Database**: PostgreSQL with Row-Level Locking

```sql
-- Inventory levels
CREATE TABLE inventory (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    product_id UUID NOT NULL,
    warehouse_id UUID NOT NULL,
    quantity_on_hand INT NOT NULL DEFAULT 0,
    quantity_reserved INT NOT NULL DEFAULT 0,
    quantity_available INT GENERATED ALWAYS AS (quantity_on_hand - quantity_reserved) STORED,
    reorder_point INT,
    reorder_quantity INT,
    last_updated TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    version INT NOT NULL DEFAULT 1, -- For optimistic locking
    
    UNIQUE(product_id, warehouse_id),
    INDEX idx_product_id (product_id),
    INDEX idx_quantity_available (quantity_available),
    CHECK (quantity_on_hand >= 0),
    CHECK (quantity_reserved >= 0),
    CHECK (quantity_reserved <= quantity_on_hand)
);

-- Inventory reservations
CREATE TABLE inventory_reservations (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    order_id UUID NOT NULL,
    product_id UUID NOT NULL,
    warehouse_id UUID NOT NULL,
    quantity INT NOT NULL,
    status VARCHAR(50) NOT NULL DEFAULT 'ACTIVE',
    expires_at TIMESTAMP NOT NULL,
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    released_at TIMESTAMP,
    
    INDEX idx_order_id (order_id),
    INDEX idx_status_expires (status, expires_at),
    INDEX idx_product_warehouse (product_id, warehouse_id)
);

-- Inventory movements (Event Sourcing)
CREATE TABLE inventory_movements (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    product_id UUID NOT NULL,
    warehouse_id UUID NOT NULL,
    movement_type VARCHAR(50) NOT NULL,
    quantity INT NOT NULL,
    reference_type VARCHAR(50),
    reference_id UUID,
    reason VARCHAR(255),
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    created_by UUID,
    
    INDEX idx_product_id (product_id),
    INDEX idx_created_at (created_at),
    INDEX idx_reference (reference_type, reference_id)
);
```

### 4. Order Management Service

**Database**: PostgreSQL

```sql
-- Orders
CREATE TABLE orders (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    order_number VARCHAR(50) UNIQUE NOT NULL,
    customer_id UUID NOT NULL,
    status VARCHAR(50) NOT NULL DEFAULT 'PENDING',
    total_amount DECIMAL(10,2) NOT NULL,
    currency VARCHAR(3) NOT NULL DEFAULT 'USD',
    shipping_address JSONB NOT NULL,
    billing_address JSONB NOT NULL,
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    completed_at TIMESTAMP,
    
    INDEX idx_customer_id (customer_id),
    INDEX idx_status (status),
    INDEX idx_created_at (created_at),
    INDEX idx_order_number (order_number)
);

-- Order items
CREATE TABLE order_items (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    order_id UUID NOT NULL REFERENCES orders(id),
    product_id UUID NOT NULL,
    quantity INT NOT NULL,
    unit_price DECIMAL(10,2) NOT NULL,
    total_price DECIMAL(10,2) NOT NULL,
    discount_amount DECIMAL(10,2) DEFAULT 0,
    tax_amount DECIMAL(10,2) DEFAULT 0,
    
    INDEX idx_order_id (order_id),
    INDEX idx_product_id (product_id)
);

-- Payments
CREATE TABLE payments (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    order_id UUID NOT NULL REFERENCES orders(id),
    payment_method VARCHAR(50) NOT NULL,
    amount DECIMAL(10,2) NOT NULL,
    currency VARCHAR(3) NOT NULL DEFAULT 'USD',
    status VARCHAR(50) NOT NULL DEFAULT 'PENDING',
    gateway_transaction_id VARCHAR(255),
    gateway_response JSONB,
    processed_at TIMESTAMP,
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    
    INDEX idx_order_id (order_id),
    INDEX idx_status (status),
    INDEX idx_gateway_transaction_id (gateway_transaction_id)
);

-- Order events (Event Sourcing)
CREATE TABLE order_events (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    order_id UUID NOT NULL REFERENCES orders(id),
    event_type VARCHAR(100) NOT NULL,
    payload JSONB NOT NULL,
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    created_by UUID,
    
    INDEX idx_order_id (order_id),
    INDEX idx_created_at (created_at)
);
```

### 5. Delivery Service

**Database**: PostgreSQL

```sql
-- Shipments
CREATE TABLE shipments (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    order_id UUID NOT NULL,
    carrier VARCHAR(50) NOT NULL,
    tracking_number VARCHAR(100) UNIQUE,
    status VARCHAR(50) NOT NULL DEFAULT 'PENDING',
    shipping_method VARCHAR(50) NOT NULL,
    estimated_delivery DATE,
    actual_delivery TIMESTAMP,
    shipping_address JSONB NOT NULL,
    package_details JSONB,
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    shipped_at TIMESTAMP,
    
    INDEX idx_order_id (order_id),
    INDEX idx_tracking_number (tracking_number),
    INDEX idx_status (status),
    INDEX idx_carrier (carrier)
);

-- Tracking events
CREATE TABLE tracking_events (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    shipment_id UUID NOT NULL REFERENCES shipments(id),
    event_type VARCHAR(100) NOT NULL,
    location VARCHAR(255),
    description TEXT,
    carrier_status VARCHAR(100),
    occurred_at TIMESTAMP NOT NULL,
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    
    INDEX idx_shipment_id (shipment_id),
    INDEX idx_occurred_at (occurred_at)
);

-- Carrier configurations
CREATE TABLE carrier_configs (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    carrier VARCHAR(50) NOT NULL UNIQUE,
    api_endpoint VARCHAR(255) NOT NULL,
    credentials JSONB NOT NULL, -- Encrypted
    settings JSONB,
    active BOOLEAN NOT NULL DEFAULT true,
    
    INDEX idx_carrier (carrier),
    INDEX idx_active (active)
);
```

### 6. Notification Service

**Database**: MongoDB

```javascript
// Templates collection
{
  _id: ObjectId(),
  name: "order_confirmation",
  channel: "email",
  subject: "Order {{order.number}} Confirmed",
  body: "<html>...</html>",
  variables: ["order", "customer", "items"],
  active: true,
  version: 1,
  created_at: ISODate(),
  updated_at: ISODate()
}

// Notifications collection
{
  _id: ObjectId(),
  recipient: {
    type: "email",
    address: "customer@example.com",
    customer_id: "uuid"
  },
  template_id: ObjectId(),
  context: {
    order: { /* order data */ },
    customer: { /* customer data */ }
  },
  status: "sent",
  attempts: 1,
  sent_at: ISODate(),
  created_at: ISODate(),
  metadata: {
    order_id: "uuid",
    event_type: "order_created"
  }
}

// Preferences collection
{
  _id: ObjectId(),
  customer_id: "uuid",
  channels: {
    email: {
      enabled: true,
      address: "customer@example.com",
      verified: true
    },
    sms: {
      enabled: false,
      number: "+1234567890",
      verified: false
    }
  },
  subscriptions: {
    order_updates: true,
    promotions: false,
    inventory_alerts: true
  },
  timezone: "America/New_York",
  language: "en",
  updated_at: ISODate()
}
```

## Data Consistency Patterns

### 1. Eventual Consistency

Services maintain eventual consistency through events:

```java
@Service
@Transactional
public class OrderService {
    
    public Order createOrder(CreateOrderRequest request) {
        // 1. Create order in local database
        Order order = orderRepository.save(buildOrder(request));
        
        // 2. Publish event for other services
        eventPublisher.publish(OrderCreatedEvent.builder()
            .orderId(order.getId())
            .customerId(order.getCustomerId())
            .items(order.getItems())
            .build());
        
        // 3. Return immediately (eventual consistency)
        return order;
    }
}
```

### 2. Saga Pattern for Distributed Transactions

```java
@Component
public class OrderFulfillmentSaga {
    
    @Autowired
    private TransactionManager transactionManager;
    
    public void executeOrderFulfillment(Order order) {
        SagaTransaction transaction = transactionManager.begin();
        
        try {
            // Step 1: Reserve inventory
            transaction.addStep(
                () -> inventoryService.reserveItems(order.getItems()),
                () -> inventoryService.releaseItems(order.getItems())
            );
            
            // Step 2: Process payment
            transaction.addStep(
                () -> paymentService.processPayment(order),
                () -> paymentService.refundPayment(order)
            );
            
            // Step 3: Schedule delivery
            transaction.addStep(
                () -> deliveryService.scheduleDelivery(order),
                () -> deliveryService.cancelDelivery(order)
            );
            
            transaction.commit();
            
        } catch (Exception e) {
            transaction.rollback();
            throw new OrderFulfillmentException("Failed to fulfill order", e);
        }
    }
}
```

### 3. Read Model Synchronization

```java
@Component
public class ProductReadModelUpdater {
    
    @EventHandler
    public void on(ProductUpdatedEvent event) {
        // Update read model in cache
        ProductView view = ProductView.builder()
            .id(event.getProductId())
            .name(event.getName())
            .price(event.getPrice())
            .availableStock(getAvailableStock(event.getProductId()))
            .build();
            
        redisTemplate.opsForValue().set(
            "product:" + event.getProductId(),
            view,
            Duration.ofHours(1)
        );
    }
}
```

## Caching Strategy

### 1. Cache-Aside Pattern

```java
@Service
public class ProductService {
    
    @Autowired
    private RedisTemplate<String, Product> redisTemplate;
    
    public Product getProduct(UUID productId) {
        String key = "product:" + productId;
        
        // Try cache first
        Product cached = redisTemplate.opsForValue().get(key);
        if (cached != null) {
            return cached;
        }
        
        // Load from database
        Product product = productRepository.findById(productId)
            .orElseThrow(() -> new ProductNotFoundException(productId));
            
        // Update cache
        redisTemplate.opsForValue().set(key, product, Duration.ofHours(1));
        
        return product;
    }
    
    @CacheEvict(value = "products", key = "#product.id")
    public Product updateProduct(Product product) {
        return productRepository.save(product);
    }
}
```

### 2. Cache Configuration

```yaml
spring:
  redis:
    host: localhost
    port: 6379
    timeout: 2000ms
    lettuce:
      pool:
        max-active: 8
        max-idle: 8
        min-idle: 0
  cache:
    type: redis
    redis:
      time-to-live: 3600000 # 1 hour
      key-prefix: "firstviscount:"
      use-key-prefix: true
      cache-null-values: false
```

## Database Migration Strategy

### 1. Flyway Configuration

```yaml
spring:
  flyway:
    enabled: true
    locations: classpath:db/migration
    baseline-on-migrate: true
    validate-on-migrate: true
```

### 2. Migration Scripts

```sql
-- V1__create_products_table.sql
CREATE TABLE products (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    sku VARCHAR(100) UNIQUE NOT NULL,
    name VARCHAR(255) NOT NULL,
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);

-- V2__add_product_attributes.sql
ALTER TABLE products 
ADD COLUMN attributes JSONB,
ADD COLUMN updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP;

CREATE INDEX idx_product_attributes ON products USING GIN (attributes);
```

## Data Security

### 1. Encryption at Rest

```java
@Configuration
public class EncryptionConfig {
    
    @Bean
    public AttributeEncryptor attributeEncryptor() {
        return new AttributeEncryptor() {
            @Override
            public String encrypt(String attribute) {
                return AES.encrypt(attribute, getEncryptionKey());
            }
            
            @Override
            public String decrypt(String dbData) {
                return AES.decrypt(dbData, getEncryptionKey());
            }
        };
    }
}

@Entity
public class Payment {
    @Id
    private UUID id;
    
    @Convert(converter = EncryptedStringConverter.class)
    private String cardNumber;
    
    @Convert(converter = EncryptedStringConverter.class)
    private String cvv;
}
```

### 2. Data Masking

```java
@Component
public class DataMasker {
    
    public OrderDTO maskSensitiveData(Order order) {
        return OrderDTO.builder()
            .id(order.getId())
            .customerEmail(maskEmail(order.getCustomerEmail()))
            .phoneNumber(maskPhone(order.getPhoneNumber()))
            .lastFourDigits(getLastFour(order.getPaymentMethod()))
            .build();
    }
    
    private String maskEmail(String email) {
        int atIndex = email.indexOf('@');
        if (atIndex <= 2) return "***@" + email.substring(atIndex + 1);
        return email.substring(0, 2) + "***@" + email.substring(atIndex + 1);
    }
}
```

## Performance Optimization

### 1. Database Indexing Strategy

```sql
-- Composite indexes for common queries
CREATE INDEX idx_orders_customer_status ON orders(customer_id, status);
CREATE INDEX idx_orders_created_status ON orders(created_at DESC, status);

-- Partial indexes for performance
CREATE INDEX idx_active_products ON products(sku) WHERE status = 'ACTIVE';
CREATE INDEX idx_pending_shipments ON shipments(created_at) WHERE status = 'PENDING';

-- JSON indexes
CREATE INDEX idx_order_metadata ON orders USING GIN (metadata);
```

### 2. Query Optimization

```java
@Repository
public interface OrderRepository extends JpaRepository<Order, UUID> {
    
    @Query("""
        SELECT o FROM Order o 
        JOIN FETCH o.items 
        WHERE o.customerId = :customerId 
        AND o.status IN :statuses
        ORDER BY o.createdAt DESC
        """)
    @QueryHints(@QueryHint(name = HINT_FETCH_SIZE, value = "25"))
    Page<Order> findByCustomerIdAndStatuses(
        @Param("customerId") UUID customerId,
        @Param("statuses") List<OrderStatus> statuses,
        Pageable pageable
    );
}
```

## Monitoring & Maintenance

### 1. Database Metrics

```java
@Component
public class DatabaseMetricsCollector {
    
    @Scheduled(fixedDelay = 60000)
    public void collectMetrics() {
        // Connection pool metrics
        HikariDataSource dataSource = (HikariDataSource) this.dataSource;
        meterRegistry.gauge("db.connections.active", dataSource.getHikariPoolMXBean().getActiveConnections());
        meterRegistry.gauge("db.connections.idle", dataSource.getHikariPoolMXBean().getIdleConnections());
        
        // Query performance
        jdbcTemplate.query(
            "SELECT schemaname, tablename, n_live_tup, n_dead_tup FROM pg_stat_user_tables",
            (rs) -> {
                meterRegistry.gauge("db.table.rows",
                    Tags.of("schema", rs.getString("schemaname"), 
                           "table", rs.getString("tablename")),
                    rs.getLong("n_live_tup")
                );
            }
        );
    }
}
```

### 2. Backup Strategy

```yaml
# Automated backup configuration
backup:
  postgresql:
    schedule: "0 2 * * *" # 2 AM daily
    retention: 30 # days
    type: "incremental"
    
  mongodb:
    schedule: "0 3 * * *" # 3 AM daily
    retention: 7 # days
    type: "full"
```

## Future Considerations

### 1. Scaling Strategies
- Read replicas for high-traffic services
- Sharding for large datasets
- Partitioning for time-series data

### 2. Advanced Patterns
- CQRS with separate read/write databases
- Event store for complete audit trails
- Graph database for recommendation engine

## References

- [Microservices Design](./microservices-design.md)
- [Service Specifications](./service-specifications/)
- [Testing Strategies](../03-testing/)