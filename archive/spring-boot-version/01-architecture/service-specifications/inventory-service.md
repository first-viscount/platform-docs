# Inventory Service Specification

## Overview

The Inventory Service manages real-time inventory levels, stock reservations, and inventory movements across all warehouses. It ensures accurate stock tracking and prevents overselling through distributed locking mechanisms.

### Key Responsibilities
- Real-time inventory tracking across warehouses
- Stock reservation and release management
- Inventory movement tracking (receipts, adjustments, transfers)
- Low stock alerts and reorder point management
- Reservation timeout handling
- Inventory reconciliation and auditing

## API Specification

### Base URL
```
http://inventory-service:8083/api/v1
```

### Endpoints

#### Check Inventory Availability
```http
POST /inventory/check-availability
Content-Type: application/json
Authorization: Bearer {token}

{
  "items": [
    {
      "productId": "550e8400-e29b-41d4-a716-446655440000",
      "quantity": 2,
      "warehouseId": "wh-001" // optional, checks all warehouses if not specified
    },
    {
      "productId": "660e8400-e29b-41d4-a716-446655440000",
      "quantity": 1
    }
  ]
}

Response:
{
  "available": true,
  "items": [
    {
      "productId": "550e8400-e29b-41d4-a716-446655440000",
      "requestedQuantity": 2,
      "availableQuantity": 45,
      "warehouses": [
        {
          "warehouseId": "wh-001",
          "availableQuantity": 20
        },
        {
          "warehouseId": "wh-002",
          "availableQuantity": 25
        }
      ],
      "status": "AVAILABLE"
    },
    {
      "productId": "660e8400-e29b-41d4-a716-446655440000",
      "requestedQuantity": 1,
      "availableQuantity": 0,
      "status": "OUT_OF_STOCK"
    }
  ]
}
```

#### Reserve Inventory
```http
POST /inventory/reserve
Content-Type: application/json
Authorization: Bearer {token}

{
  "orderId": "ord-123e4567-e89b-12d3-a456-426614174000",
  "customerId": "cust-123e4567-e89b-12d3-a456-426614174000",
  "items": [
    {
      "productId": "550e8400-e29b-41d4-a716-446655440000",
      "quantity": 2,
      "warehouseId": "wh-001"
    }
  ],
  "timeoutMinutes": 15
}

Response:
{
  "reservationId": "res-770e8400-e29b-41d4-a716-446655440000",
  "orderId": "ord-123e4567-e89b-12d3-a456-426614174000",
  "status": "RESERVED",
  "items": [
    {
      "productId": "550e8400-e29b-41d4-a716-446655440000",
      "quantity": 2,
      "warehouseId": "wh-001",
      "reserved": true
    }
  ],
  "expiresAt": "2024-01-15T10:15:00Z",
  "createdAt": "2024-01-15T10:00:00Z"
}
```

#### Release Inventory
```http
POST /inventory/release
Content-Type: application/json
Authorization: Bearer {token}

{
  "reservationId": "res-770e8400-e29b-41d4-a716-446655440000",
  "reason": "Order cancelled by customer"
}

Response:
{
  "reservationId": "res-770e8400-e29b-41d4-a716-446655440000",
  "status": "RELEASED",
  "items": [
    {
      "productId": "550e8400-e29b-41d4-a716-446655440000",
      "quantity": 2,
      "warehouseId": "wh-001",
      "released": true
    }
  ],
  "releasedAt": "2024-01-15T10:05:00Z"
}
```

#### Get Inventory Levels
```http
GET /inventory/{productId}?warehouseId=wh-001
Authorization: Bearer {token}

Response:
{
  "productId": "550e8400-e29b-41d4-a716-446655440000",
  "totalQuantity": 45,
  "totalAvailable": 40,
  "totalReserved": 5,
  "warehouses": [
    {
      "warehouseId": "wh-001",
      "warehouseName": "Main Warehouse",
      "quantityOnHand": 20,
      "quantityReserved": 2,
      "quantityAvailable": 18,
      "reorderPoint": 10,
      "reorderQuantity": 50,
      "location": {
        "zone": "A",
        "aisle": "12",
        "shelf": "B3"
      }
    },
    {
      "warehouseId": "wh-002",
      "warehouseName": "Secondary Warehouse",
      "quantityOnHand": 25,
      "quantityReserved": 3,
      "quantityAvailable": 22
    }
  ],
  "movements": {
    "last24Hours": {
      "received": 0,
      "shipped": 5,
      "adjusted": 0
    }
  },
  "lastUpdated": "2024-01-15T09:45:00Z"
}
```

#### Update Inventory (Admin)
```http
PUT /inventory/adjust
Content-Type: application/json
Authorization: Bearer {admin-token}

{
  "adjustments": [
    {
      "productId": "550e8400-e29b-41d4-a716-446655440000",
      "warehouseId": "wh-001",
      "quantity": 10,
      "type": "RECEIPT",
      "reason": "New shipment received",
      "reference": "PO-12345"
    }
  ]
}

Response:
{
  "adjustmentId": "adj-880e8400-e29b-41d4-a716-446655440000",
  "status": "COMPLETED",
  "adjustments": [
    {
      "productId": "550e8400-e29b-41d4-a716-446655440000",
      "warehouseId": "wh-001",
      "previousQuantity": 20,
      "adjustedQuantity": 10,
      "newQuantity": 30,
      "type": "RECEIPT"
    }
  ],
  "adjustedAt": "2024-01-15T10:30:00Z",
  "adjustedBy": "admin-user-id"
}
```

#### Get Low Stock Items
```http
GET /inventory/low-stock?warehouseId=wh-001&threshold=0.2
Authorization: Bearer {token}

Response:
{
  "items": [
    {
      "productId": "990e8400-e29b-41d4-a716-446655440000",
      "productSku": "PROD-789",
      "productName": "Widget XL",
      "warehouseId": "wh-001",
      "currentStock": 5,
      "reorderPoint": 20,
      "percentageRemaining": 25.0,
      "daysOfSupply": 2,
      "status": "CRITICAL"
    },
    {
      "productId": "aa0e8400-e29b-41d4-a716-446655440000",
      "productSku": "PROD-456",
      "productName": "Gadget Pro",
      "warehouseId": "wh-001",
      "currentStock": 15,
      "reorderPoint": 50,
      "percentageRemaining": 30.0,
      "daysOfSupply": 5,
      "status": "LOW"
    }
  ],
  "totalItems": 12,
  "criticalItems": 3,
  "lowItems": 9
}
```

#### Transfer Inventory
```http
POST /inventory/transfer
Content-Type: application/json
Authorization: Bearer {admin-token}

{
  "items": [
    {
      "productId": "550e8400-e29b-41d4-a716-446655440000",
      "quantity": 10,
      "fromWarehouseId": "wh-001",
      "toWarehouseId": "wh-002"
    }
  ],
  "reason": "Balancing inventory levels",
  "expectedDelivery": "2024-01-17T10:00:00Z"
}

Response:
{
  "transferId": "tr-bb0e8400-e29b-41d4-a716-446655440000",
  "status": "IN_TRANSIT",
  "items": [
    {
      "productId": "550e8400-e29b-41d4-a716-446655440000",
      "quantity": 10,
      "fromWarehouse": {
        "id": "wh-001",
        "previousQuantity": 30,
        "newQuantity": 20
      },
      "toWarehouse": {
        "id": "wh-002",
        "expectedQuantity": 35
      }
    }
  ],
  "initiatedAt": "2024-01-15T11:00:00Z",
  "expectedDelivery": "2024-01-17T10:00:00Z"
}
```

## Event Contracts

### Published Events

#### InventoryReservedEvent
```json
{
  "eventId": "ev-110e8400-e29b-41d4-a716-446655440000",
  "reservationId": "res-770e8400-e29b-41d4-a716-446655440000",
  "orderId": "ord-123e4567-e89b-12d3-a456-426614174000",
  "items": [
    {
      "productId": "550e8400-e29b-41d4-a716-446655440000",
      "quantity": 2,
      "warehouseId": "wh-001"
    }
  ],
  "expiresAt": "2024-01-15T10:15:00Z",
  "timestamp": "2024-01-15T10:00:00Z"
}
```

#### InventoryReleasedEvent
```json
{
  "eventId": "ev-220e8400-e29b-41d4-a716-446655440000",
  "reservationId": "res-770e8400-e29b-41d4-a716-446655440000",
  "orderId": "ord-123e4567-e89b-12d3-a456-426614174000",
  "items": [
    {
      "productId": "550e8400-e29b-41d4-a716-446655440000",
      "quantity": 2,
      "warehouseId": "wh-001"
    }
  ],
  "reason": "Order cancelled",
  "timestamp": "2024-01-15T10:05:00Z"
}
```

#### LowStockAlertEvent
```json
{
  "eventId": "ev-330e8400-e29b-41d4-a716-446655440000",
  "productId": "990e8400-e29b-41d4-a716-446655440000",
  "warehouseId": "wh-001",
  "currentStock": 5,
  "reorderPoint": 20,
  "severity": "CRITICAL",
  "timestamp": "2024-01-15T10:30:00Z"
}
```

#### StockAdjustedEvent
```json
{
  "eventId": "ev-440e8400-e29b-41d4-a716-446655440000",
  "adjustmentId": "adj-880e8400-e29b-41d4-a716-446655440000",
  "productId": "550e8400-e29b-41d4-a716-446655440000",
  "warehouseId": "wh-001",
  "adjustmentType": "RECEIPT",
  "quantity": 10,
  "previousStock": 20,
  "newStock": 30,
  "reason": "New shipment received",
  "adjustedBy": "admin-user-id",
  "timestamp": "2024-01-15T10:30:00Z"
}
```

### Consumed Events

- `OrderCreatedEvent` - Triggers inventory reservation
- `OrderCancelledEvent` - Triggers inventory release
- `OrderCompletedEvent` - Confirms inventory allocation
- `ProductCreatedEvent` - Creates initial inventory records

## Data Model

### Database Schema

```sql
-- Inventory levels per warehouse
CREATE TABLE inventory (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    product_id UUID NOT NULL,
    warehouse_id UUID NOT NULL,
    quantity_on_hand INT NOT NULL DEFAULT 0,
    quantity_reserved INT NOT NULL DEFAULT 0,
    quantity_available INT GENERATED ALWAYS AS (quantity_on_hand - quantity_reserved) STORED,
    reorder_point INT,
    reorder_quantity INT,
    min_stock_level INT DEFAULT 0,
    max_stock_level INT,
    location_zone VARCHAR(10),
    location_aisle VARCHAR(10),
    location_shelf VARCHAR(10),
    last_counted_at TIMESTAMP,
    last_movement_at TIMESTAMP,
    version BIGINT NOT NULL DEFAULT 1, -- For optimistic locking
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    
    UNIQUE(product_id, warehouse_id),
    INDEX idx_product_id (product_id),
    INDEX idx_warehouse_id (warehouse_id),
    INDEX idx_quantity_available (quantity_available),
    INDEX idx_reorder_point (reorder_point, quantity_available),
    CHECK (quantity_on_hand >= 0),
    CHECK (quantity_reserved >= 0),
    CHECK (quantity_reserved <= quantity_on_hand)
);

-- Inventory reservations
CREATE TABLE inventory_reservations (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    reservation_number VARCHAR(50) UNIQUE NOT NULL,
    order_id UUID NOT NULL,
    customer_id UUID,
    status VARCHAR(50) NOT NULL DEFAULT 'ACTIVE',
    expires_at TIMESTAMP NOT NULL,
    total_items INT NOT NULL,
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    released_at TIMESTAMP,
    completed_at TIMESTAMP,
    
    INDEX idx_order_id (order_id),
    INDEX idx_status_expires (status, expires_at),
    INDEX idx_customer_id (customer_id),
    INDEX idx_created_at (created_at)
);

-- Reservation line items
CREATE TABLE reservation_items (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    reservation_id UUID NOT NULL REFERENCES inventory_reservations(id),
    product_id UUID NOT NULL,
    warehouse_id UUID NOT NULL,
    quantity INT NOT NULL,
    unit_price DECIMAL(10,2),
    
    INDEX idx_reservation_id (reservation_id),
    INDEX idx_product_warehouse (product_id, warehouse_id)
);

-- Inventory movements (audit trail)
CREATE TABLE inventory_movements (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    movement_number VARCHAR(50) UNIQUE NOT NULL,
    product_id UUID NOT NULL,
    warehouse_id UUID NOT NULL,
    movement_type VARCHAR(50) NOT NULL, -- RECEIPT, SHIPMENT, ADJUSTMENT, TRANSFER, RETURN
    quantity INT NOT NULL,
    quantity_before INT NOT NULL,
    quantity_after INT NOT NULL,
    reference_type VARCHAR(50),
    reference_id UUID,
    reason VARCHAR(500),
    cost_per_unit DECIMAL(10,2),
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    created_by UUID,
    
    INDEX idx_product_id (product_id),
    INDEX idx_warehouse_id (warehouse_id),
    INDEX idx_movement_type (movement_type),
    INDEX idx_created_at (created_at),
    INDEX idx_reference (reference_type, reference_id)
);

-- Inventory transfers
CREATE TABLE inventory_transfers (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    transfer_number VARCHAR(50) UNIQUE NOT NULL,
    from_warehouse_id UUID NOT NULL,
    to_warehouse_id UUID NOT NULL,
    status VARCHAR(50) NOT NULL DEFAULT 'PENDING',
    total_items INT NOT NULL,
    total_quantity INT NOT NULL,
    initiated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    shipped_at TIMESTAMP,
    received_at TIMESTAMP,
    expected_delivery TIMESTAMP,
    initiated_by UUID,
    
    INDEX idx_status (status),
    INDEX idx_from_warehouse (from_warehouse_id),
    INDEX idx_to_warehouse (to_warehouse_id)
);

-- Transfer line items
CREATE TABLE transfer_items (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    transfer_id UUID NOT NULL REFERENCES inventory_transfers(id),
    product_id UUID NOT NULL,
    quantity_shipped INT NOT NULL,
    quantity_received INT,
    
    INDEX idx_transfer_id (transfer_id)
);

-- Warehouses
CREATE TABLE warehouses (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    code VARCHAR(20) UNIQUE NOT NULL,
    name VARCHAR(100) NOT NULL,
    type VARCHAR(50) NOT NULL, -- MAIN, SECONDARY, DROPSHIP
    address JSONB NOT NULL,
    contact_info JSONB,
    is_active BOOLEAN DEFAULT true,
    capabilities JSONB, -- shipping methods, storage types, etc.
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    
    INDEX idx_code (code),
    INDEX idx_is_active (is_active)
);
```

### Entity Models

```java
@Entity
@Table(name = "inventory")
public class Inventory {
    @Id
    @GeneratedValue
    private UUID id;
    
    @Column(name = "product_id", nullable = false)
    private UUID productId;
    
    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "warehouse_id", nullable = false)
    private Warehouse warehouse;
    
    @Column(name = "quantity_on_hand", nullable = false)
    private Integer quantityOnHand = 0;
    
    @Column(name = "quantity_reserved", nullable = false)
    private Integer quantityReserved = 0;
    
    @Column(name = "quantity_available", insertable = false, updatable = false)
    private Integer quantityAvailable;
    
    @Column(name = "reorder_point")
    private Integer reorderPoint;
    
    @Column(name = "reorder_quantity")
    private Integer reorderQuantity;
    
    @Embedded
    private Location location;
    
    @Column(name = "last_counted_at")
    private Instant lastCountedAt;
    
    @Column(name = "last_movement_at")
    private Instant lastMovementAt;
    
    @Version
    private Long version;
    
    @CreatedDate
    private Instant createdAt;
    
    @LastModifiedDate
    private Instant updatedAt;
    
    // Business methods
    public boolean canReserve(int quantity) {
        return this.quantityAvailable >= quantity;
    }
    
    public void reserve(int quantity) {
        if (!canReserve(quantity)) {
            throw new InsufficientInventoryException(productId, quantity, quantityAvailable);
        }
        this.quantityReserved += quantity;
    }
    
    public void release(int quantity) {
        if (this.quantityReserved < quantity) {
            throw new InvalidInventoryOperationException("Cannot release more than reserved");
        }
        this.quantityReserved -= quantity;
    }
    
    public void adjust(int quantity, MovementType type) {
        switch (type) {
            case RECEIPT:
                this.quantityOnHand += quantity;
                break;
            case SHIPMENT:
                if (this.quantityOnHand < quantity) {
                    throw new InsufficientInventoryException(productId, quantity, quantityOnHand);
                }
                this.quantityOnHand -= quantity;
                break;
            case ADJUSTMENT:
                this.quantityOnHand += quantity; // Can be negative for write-offs
                break;
        }
        this.lastMovementAt = Instant.now();
    }
}

@Embeddable
public class Location {
    private String zone;
    private String aisle;
    private String shelf;
    
    public String getFullLocation() {
        return String.format("%s-%s-%s", zone, aisle, shelf);
    }
}
```

## Configuration

### Application Properties

```yaml
spring:
  application:
    name: inventory-service
  
  datasource:
    url: jdbc:postgresql://${DB_HOST:localhost}:5432/inventory_db
    username: ${DB_USER:inventory_user}
    password: ${DB_PASSWORD:password}
    hikari:
      maximum-pool-size: 30
      minimum-idle: 10
      connection-timeout: 30000
  
  jpa:
    hibernate:
      ddl-auto: validate
    properties:
      hibernate:
        dialect: org.hibernate.dialect.PostgreSQLDialect
        jdbc:
          batch_size: 25
        order_inserts: true
  
  kafka:
    bootstrap-servers: ${KAFKA_BROKERS:localhost:9092}
    consumer:
      group-id: inventory-service
      auto-offset-reset: earliest
      enable-auto-commit: false
    producer:
      acks: all
      retries: 3

inventory:
  reservation:
    default-timeout-minutes: 15
    max-timeout-minutes: 60
    cleanup-interval-minutes: 5
  
  low-stock:
    check-interval-minutes: 30
    default-threshold-percentage: 20
    critical-threshold-percentage: 10
  
  movement:
    batch-size: 100
    retention-days: 365

distributed-lock:
  enabled: true
  timeout-seconds: 10
  retry-attempts: 3
```

### Environment Variables

```bash
# Database
DB_HOST=postgres
DB_USER=inventory_user
DB_PASSWORD=secure_password

# Kafka
KAFKA_BROKERS=kafka1:9092,kafka2:9092,kafka3:9092

# Services
PRODUCT_SERVICE_URL=http://product-catalog:8082
NOTIFICATION_SERVICE_URL=http://notification-service:8086

# Redis (for distributed locking)
REDIS_HOST=redis
REDIS_PORT=6379

# Monitoring
PROMETHEUS_ENDPOINT=http://prometheus:9090
```

## Dependencies

### External Services
- **Product Catalog Service** - Product information
- **Notification Service** - Low stock alerts
- **Order Service** - Order validation

### Libraries
```xml
<dependencies>
    <!-- Spring Boot -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-jpa</artifactId>
    </dependency>
    
    <!-- PostgreSQL -->
    <dependency>
        <groupId>org.postgresql</groupId>
        <artifactId>postgresql</artifactId>
    </dependency>
    
    <!-- Distributed Locking -->
    <dependency>
        <groupId>org.redisson</groupId>
        <artifactId>redisson-spring-boot-starter</artifactId>
        <version>3.25.0</version>
    </dependency>
    
    <!-- Kafka -->
    <dependency>
        <groupId>org.springframework.kafka</groupId>
        <artifactId>spring-kafka</artifactId>
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
void shouldReserveInventoryWithOptimisticLocking() {
    // Given
    Inventory inventory = createInventory(100, 10); // 100 on hand, 10 reserved
    
    // When
    inventory.reserve(20);
    
    // Then
    assertThat(inventory.getQuantityReserved()).isEqualTo(30);
    assertThat(inventory.getQuantityAvailable()).isEqualTo(70);
}

@Test
void shouldFailReservationWhenInsufficientStock() {
    // Given
    Inventory inventory = createInventory(100, 90); // Only 10 available
    
    // When/Then
    assertThrows(InsufficientInventoryException.class, () -> {
        inventory.reserve(20);
    });
}
```

### Integration Tests
```java
@SpringBootTest
@Transactional
class InventoryServiceIntegrationTest {
    
    @Test
    void shouldHandleConcurrentReservations() throws InterruptedException {
        // Given
        UUID productId = UUID.randomUUID();
        inventoryRepository.save(createInventory(productId, 100, 0));
        
        // When - Simulate concurrent reservations
        CountDownLatch latch = new CountDownLatch(10);
        AtomicInteger successCount = new AtomicInteger(0);
        
        for (int i = 0; i < 10; i++) {
            executor.submit(() -> {
                try {
                    inventoryService.reserveInventory(productId, 15);
                    successCount.incrementAndGet();
                } catch (Exception e) {
                    // Expected for some threads
                } finally {
                    latch.countDown();
                }
            });
        }
        
        latch.await();
        
        // Then - Only 6 reservations should succeed (6 * 15 = 90, leaving 10)
        assertThat(successCount.get()).isEqualTo(6);
        
        Inventory finalInventory = inventoryRepository.findByProductIdAndWarehouseId(productId, warehouseId);
        assertThat(finalInventory.getQuantityReserved()).isEqualTo(90);
    }
}
```

## Deployment

### Kubernetes Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: inventory-service
  namespace: ecommerce
spec:
  replicas: 3
  selector:
    matchLabels:
      app: inventory-service
  template:
    metadata:
      labels:
        app: inventory-service
    spec:
      containers:
      - name: inventory
        image: firstviscount/inventory-service:latest
        ports:
        - containerPort: 8083
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
            port: 8083
          initialDelaySeconds: 30
        readinessProbe:
          httpGet:
            path: /actuator/health/readiness
            port: 8083
          initialDelaySeconds: 20
```

## Monitoring

### Health Checks

```java
@Component
public class InventoryHealthIndicator implements HealthIndicator {
    
    @Override
    public Health health() {
        try {
            // Check for stuck reservations
            long stuckReservations = reservationRepository
                .countByStatusAndExpiresAtBefore("ACTIVE", Instant.now());
                
            // Check for critical stock levels
            long criticalItems = inventoryRepository
                .countCriticalStockItems(0.1); // 10% threshold
                
            if (stuckReservations > 100) {
                return Health.down()
                    .withDetail("stuck_reservations", stuckReservations)
                    .build();
            }
            
            return Health.up()
                .withDetail("critical_stock_items", criticalItems)
                .withDetail("active_reservations", getActiveReservations())
                .build();
                
        } catch (Exception e) {
            return Health.down().withException(e).build();
        }
    }
}
```

### Metrics

Key metrics exposed:
- `inventory.reservation.created` - Reservations created
- `inventory.reservation.released` - Reservations released
- `inventory.movement.processed` - Inventory movements
- `inventory.stock.level` - Current stock levels by product
- `inventory.low_stock.alerts` - Low stock alerts sent

### Alerts

```yaml
groups:
  - name: inventory
    rules:
      - alert: HighReservationFailureRate
        expr: rate(inventory_reservation_failed_total[5m]) > 0.1
        for: 5m
        annotations:
          summary: "High reservation failure rate"
          
      - alert: CriticalStockLevels
        expr: inventory_critical_stock_items > 10
        for: 15m
        annotations:
          summary: "Multiple items at critical stock levels"
          
      - alert: StuckReservations
        expr: inventory_stuck_reservations > 50
        for: 10m
        annotations:
          summary: "Too many stuck reservations"
```

## Best Practices

1. **Optimistic Locking** - Use version field to handle concurrent updates
2. **Distributed Locking** - Use Redis/Redisson for critical operations
3. **Reservation Timeout** - Always set expiration on reservations
4. **Audit Trail** - Track all inventory movements
5. **Batch Operations** - Process movements in batches for performance
6. **Regular Reconciliation** - Schedule periodic inventory counts