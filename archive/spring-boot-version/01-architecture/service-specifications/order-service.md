# Order Management Service Specification

## Overview

The Order Management Service handles the complete order lifecycle from creation through fulfillment. It manages order state transitions, payment processing integration, and coordinates with other services to ensure successful order completion.

### Key Responsibilities
- Order creation and validation
- Order state management and transitions
- Payment processing orchestration
- Order history and tracking
- Refund and cancellation handling
- Order analytics and reporting

## API Specification

### Base URL
```
http://order-service:8084/api/v1
```

### Endpoints

#### Create Order
```http
POST /orders
Content-Type: application/json
Authorization: Bearer {token}

{
  "customerId": "123e4567-e89b-12d3-a456-426614174000",
  "items": [
    {
      "productId": "550e8400-e29b-41d4-a716-446655440000",
      "quantity": 2,
      "unitPrice": 599.99
    },
    {
      "productId": "660e8400-e29b-41d4-a716-446655440000",
      "quantity": 1,
      "unitPrice": 299.99
    }
  ],
  "shippingAddress": {
    "name": "John Doe",
    "street": "123 Main St",
    "city": "New York",
    "state": "NY",
    "zipCode": "10001",
    "country": "US",
    "phone": "+1234567890"
  },
  "billingAddress": {
    "sameAsShipping": true
  },
  "paymentMethod": {
    "type": "CREDIT_CARD",
    "token": "tok_visa_4242",
    "lastFourDigits": "4242"
  },
  "shippingMethod": "STANDARD",
  "promotionCodes": ["SAVE10"]
}

Response:
{
  "orderId": "ord-770e8400-e29b-41d4-a716-446655440000",
  "orderNumber": "ORD-2024-001234",
  "status": "PENDING_PAYMENT",
  "customerId": "123e4567-e89b-12d3-a456-426614174000",
  "items": [
    {
      "orderItemId": "item-880e8400-e29b-41d4-a716-446655440000",
      "productId": "550e8400-e29b-41d4-a716-446655440000",
      "productName": "Premium Laptop Pro 15",
      "quantity": 2,
      "unitPrice": 599.99,
      "subtotal": 1199.98
    }
  ],
  "pricing": {
    "subtotal": 1499.97,
    "shippingCost": 9.99,
    "tax": 120.00,
    "discount": 149.99,
    "total": 1479.97
  },
  "estimatedDelivery": "2024-01-22",
  "createdAt": "2024-01-15T10:00:00Z"
}
```

#### Get Order by ID
```http
GET /orders/{orderId}
Authorization: Bearer {token}

Response:
{
  "orderId": "ord-770e8400-e29b-41d4-a716-446655440000",
  "orderNumber": "ORD-2024-001234",
  "status": "PROCESSING",
  "statusHistory": [
    {
      "status": "CREATED",
      "timestamp": "2024-01-15T10:00:00Z"
    },
    {
      "status": "PENDING_PAYMENT",
      "timestamp": "2024-01-15T10:00:01Z"
    },
    {
      "status": "PAID",
      "timestamp": "2024-01-15T10:00:30Z"
    },
    {
      "status": "PROCESSING",
      "timestamp": "2024-01-15T10:01:00Z",
      "note": "Order sent to fulfillment center"
    }
  ],
  "customer": {
    "customerId": "123e4567-e89b-12d3-a456-426614174000",
    "email": "john.doe@example.com",
    "name": "John Doe"
  },
  "items": [...],
  "payment": {
    "method": "CREDIT_CARD",
    "status": "CAPTURED",
    "transactionId": "ch_1234567890",
    "amount": 1479.97,
    "processedAt": "2024-01-15T10:00:30Z"
  },
  "shipping": {
    "method": "STANDARD",
    "carrier": "FEDEX",
    "trackingNumber": "1234567890",
    "estimatedDelivery": "2024-01-22",
    "address": {...}
  },
  "createdAt": "2024-01-15T10:00:00Z",
  "updatedAt": "2024-01-15T10:01:00Z"
}
```

#### List Customer Orders
```http
GET /orders/customer/{customerId}?status=COMPLETED&page=0&size=20&sort=createdAt,desc
Authorization: Bearer {token}

Response:
{
  "content": [
    {
      "orderId": "ord-770e8400-e29b-41d4-a716-446655440000",
      "orderNumber": "ORD-2024-001234",
      "status": "COMPLETED",
      "itemCount": 3,
      "total": 1479.97,
      "createdAt": "2024-01-15T10:00:00Z",
      "completedAt": "2024-01-22T14:30:00Z"
    }
  ],
  "totalElements": 45,
  "totalPages": 3,
  "number": 0
}
```

#### Update Order Status (Internal/Admin)
```http
PUT /orders/{orderId}/status
Content-Type: application/json
Authorization: Bearer {admin-token}

{
  "status": "SHIPPED",
  "reason": "Order shipped from warehouse",
  "metadata": {
    "trackingNumber": "1234567890",
    "carrier": "FEDEX"
  }
}

Response:
{
  "orderId": "ord-770e8400-e29b-41d4-a716-446655440000",
  "previousStatus": "PROCESSING",
  "newStatus": "SHIPPED",
  "updatedAt": "2024-01-16T09:00:00Z"
}
```

#### Cancel Order
```http
POST /orders/{orderId}/cancel
Content-Type: application/json
Authorization: Bearer {token}

{
  "reason": "Customer requested cancellation",
  "refundMethod": "ORIGINAL_PAYMENT_METHOD"
}

Response:
{
  "orderId": "ord-770e8400-e29b-41d4-a716-446655440000",
  "status": "CANCELLING",
  "cancellationId": "cancel-990e8400-e29b-41d4-a716-446655440000",
  "refund": {
    "amount": 1479.97,
    "status": "PENDING",
    "estimatedDate": "2024-01-18"
  },
  "message": "Your order is being cancelled. Refund will be processed within 3-5 business days."
}
```

#### Process Refund
```http
POST /orders/{orderId}/refund
Content-Type: application/json
Authorization: Bearer {admin-token}

{
  "items": [
    {
      "orderItemId": "item-880e8400-e29b-41d4-a716-446655440000",
      "quantity": 1,
      "reason": "DEFECTIVE"
    }
  ],
  "refundShipping": false,
  "customerNote": "Item was defective upon arrival"
}

Response:
{
  "refundId": "ref-aa0e8400-e29b-41d4-a716-446655440000",
  "orderId": "ord-770e8400-e29b-41d4-a716-446655440000",
  "amount": 599.99,
  "status": "PROCESSING",
  "items": [
    {
      "orderItemId": "item-880e8400-e29b-41d4-a716-446655440000",
      "productName": "Premium Laptop Pro 15",
      "quantity": 1,
      "amount": 599.99
    }
  ],
  "processedAt": "2024-01-25T10:00:00Z"
}
```

#### Search Orders (Admin)
```http
POST /orders/search
Content-Type: application/json
Authorization: Bearer {admin-token}

{
  "filters": {
    "status": ["PROCESSING", "SHIPPED"],
    "dateRange": {
      "from": "2024-01-01",
      "to": "2024-01-31"
    },
    "totalRange": {
      "min": 100,
      "max": 1000
    },
    "customerEmail": "john*"
  },
  "page": 0,
  "size": 50,
  "sort": {
    "field": "createdAt",
    "direction": "DESC"
  }
}

Response:
{
  "results": [...],
  "totalResults": 234,
  "aggregations": {
    "totalRevenue": 125430.50,
    "averageOrderValue": 536.00,
    "statusBreakdown": {
      "PROCESSING": 45,
      "SHIPPED": 189
    }
  }
}
```

## Event Contracts

### Published Events

#### OrderCreatedEvent
```json
{
  "eventId": "ev-110e8400-e29b-41d4-a716-446655440000",
  "orderId": "ord-770e8400-e29b-41d4-a716-446655440000",
  "orderNumber": "ORD-2024-001234",
  "customerId": "123e4567-e89b-12d3-a456-426614174000",
  "items": [
    {
      "productId": "550e8400-e29b-41d4-a716-446655440000",
      "quantity": 2,
      "unitPrice": 599.99
    }
  ],
  "total": 1479.97,
  "shippingAddress": {...},
  "timestamp": "2024-01-15T10:00:00Z"
}
```

#### OrderPaidEvent
```json
{
  "eventId": "ev-220e8400-e29b-41d4-a716-446655440000",
  "orderId": "ord-770e8400-e29b-41d4-a716-446655440000",
  "paymentId": "pay-330e8400-e29b-41d4-a716-446655440000",
  "amount": 1479.97,
  "paymentMethod": "CREDIT_CARD",
  "transactionId": "ch_1234567890",
  "timestamp": "2024-01-15T10:00:30Z"
}
```

#### OrderCancelledEvent
```json
{
  "eventId": "ev-440e8400-e29b-41d4-a716-446655440000",
  "orderId": "ord-770e8400-e29b-41d4-a716-446655440000",
  "customerId": "123e4567-e89b-12d3-a456-426614174000",
  "reason": "Customer requested cancellation",
  "refundAmount": 1479.97,
  "items": [...],
  "timestamp": "2024-01-15T11:00:00Z"
}
```

#### OrderShippedEvent
```json
{
  "eventId": "ev-550e8400-e29b-41d4-a716-446655440000",
  "orderId": "ord-770e8400-e29b-41d4-a716-446655440000",
  "trackingNumber": "1234567890",
  "carrier": "FEDEX",
  "estimatedDelivery": "2024-01-22",
  "timestamp": "2024-01-16T09:00:00Z"
}
```

#### OrderCompletedEvent
```json
{
  "eventId": "ev-660e8400-e29b-41d4-a716-446655440000",
  "orderId": "ord-770e8400-e29b-41d4-a716-446655440000",
  "completedAt": "2024-01-22T14:30:00Z",
  "fulfillmentDays": 7,
  "customerSatisfaction": null,
  "timestamp": "2024-01-22T14:30:00Z"
}
```

### Consumed Events

- `InventoryReservedEvent` - Updates order status to payment processing
- `PaymentProcessedEvent` - Updates order status to processing
- `ShipmentCreatedEvent` - Updates order with tracking information
- `DeliveryCompletedEvent` - Marks order as completed

## Data Model

### Database Schema

```sql
-- Orders table
CREATE TABLE orders (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    order_number VARCHAR(50) UNIQUE NOT NULL,
    customer_id UUID NOT NULL,
    status VARCHAR(50) NOT NULL DEFAULT 'CREATED',
    subtotal DECIMAL(10,2) NOT NULL,
    shipping_cost DECIMAL(10,2) NOT NULL DEFAULT 0,
    tax_amount DECIMAL(10,2) NOT NULL DEFAULT 0,
    discount_amount DECIMAL(10,2) NOT NULL DEFAULT 0,
    total_amount DECIMAL(10,2) NOT NULL,
    currency VARCHAR(3) NOT NULL DEFAULT 'USD',
    shipping_method VARCHAR(50) NOT NULL,
    payment_method VARCHAR(50),
    notes TEXT,
    metadata JSONB,
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    paid_at TIMESTAMP,
    shipped_at TIMESTAMP,
    delivered_at TIMESTAMP,
    cancelled_at TIMESTAMP,
    
    INDEX idx_order_number (order_number),
    INDEX idx_customer_id (customer_id),
    INDEX idx_status (status),
    INDEX idx_created_at (created_at),
    INDEX idx_total_amount (total_amount)
);

-- Order items
CREATE TABLE order_items (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    order_id UUID NOT NULL REFERENCES orders(id),
    product_id UUID NOT NULL,
    product_sku VARCHAR(100) NOT NULL,
    product_name VARCHAR(255) NOT NULL,
    quantity INT NOT NULL,
    unit_price DECIMAL(10,2) NOT NULL,
    subtotal DECIMAL(10,2) NOT NULL,
    discount_amount DECIMAL(10,2) DEFAULT 0,
    tax_amount DECIMAL(10,2) DEFAULT 0,
    total_amount DECIMAL(10,2) NOT NULL,
    metadata JSONB,
    
    INDEX idx_order_id (order_id),
    INDEX idx_product_id (product_id)
);

-- Order status history
CREATE TABLE order_status_history (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    order_id UUID NOT NULL REFERENCES orders(id),
    status VARCHAR(50) NOT NULL,
    previous_status VARCHAR(50),
    reason VARCHAR(500),
    metadata JSONB,
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    created_by UUID,
    
    INDEX idx_order_id (order_id),
    INDEX idx_created_at (created_at)
);

-- Addresses
CREATE TABLE order_addresses (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    order_id UUID NOT NULL REFERENCES orders(id),
    type VARCHAR(20) NOT NULL, -- SHIPPING, BILLING
    name VARCHAR(100) NOT NULL,
    company VARCHAR(100),
    street_line1 VARCHAR(255) NOT NULL,
    street_line2 VARCHAR(255),
    city VARCHAR(100) NOT NULL,
    state VARCHAR(50) NOT NULL,
    zip_code VARCHAR(20) NOT NULL,
    country VARCHAR(2) NOT NULL,
    phone VARCHAR(50),
    email VARCHAR(255),
    
    INDEX idx_order_id (order_id),
    UNIQUE(order_id, type)
);

-- Payments
CREATE TABLE order_payments (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    order_id UUID NOT NULL REFERENCES orders(id),
    payment_method VARCHAR(50) NOT NULL,
    amount DECIMAL(10,2) NOT NULL,
    currency VARCHAR(3) NOT NULL DEFAULT 'USD',
    status VARCHAR(50) NOT NULL DEFAULT 'PENDING',
    gateway VARCHAR(50) NOT NULL,
    gateway_transaction_id VARCHAR(255),
    gateway_response JSONB,
    card_last_four VARCHAR(4),
    card_brand VARCHAR(50),
    failure_reason VARCHAR(500),
    processed_at TIMESTAMP,
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    
    INDEX idx_order_id (order_id),
    INDEX idx_status (status),
    INDEX idx_gateway_transaction_id (gateway_transaction_id)
);

-- Refunds
CREATE TABLE order_refunds (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    order_id UUID NOT NULL REFERENCES orders(id),
    payment_id UUID REFERENCES order_payments(id),
    amount DECIMAL(10,2) NOT NULL,
    currency VARCHAR(3) NOT NULL DEFAULT 'USD',
    status VARCHAR(50) NOT NULL DEFAULT 'PENDING',
    reason VARCHAR(500) NOT NULL,
    gateway_refund_id VARCHAR(255),
    processed_at TIMESTAMP,
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    created_by UUID,
    
    INDEX idx_order_id (order_id),
    INDEX idx_status (status)
);

-- Refund items
CREATE TABLE refund_items (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    refund_id UUID NOT NULL REFERENCES order_refunds(id),
    order_item_id UUID NOT NULL REFERENCES order_items(id),
    quantity INT NOT NULL,
    amount DECIMAL(10,2) NOT NULL,
    reason VARCHAR(100) NOT NULL,
    
    INDEX idx_refund_id (refund_id)
);

-- Promotions applied
CREATE TABLE order_promotions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    order_id UUID NOT NULL REFERENCES orders(id),
    promotion_code VARCHAR(50) NOT NULL,
    promotion_type VARCHAR(50) NOT NULL,
    discount_amount DECIMAL(10,2) NOT NULL,
    metadata JSONB,
    
    INDEX idx_order_id (order_id)
);
```

### Entity Models

```java
@Entity
@Table(name = "orders")
@EntityListeners(AuditingEntityListener.class)
public class Order {
    @Id
    @GeneratedValue
    private UUID id;
    
    @Column(name = "order_number", unique = true, nullable = false)
    private String orderNumber;
    
    @Column(name = "customer_id", nullable = false)
    private UUID customerId;
    
    @Enumerated(EnumType.STRING)
    private OrderStatus status = OrderStatus.CREATED;
    
    @OneToMany(mappedBy = "order", cascade = CascadeType.ALL, fetch = FetchType.LAZY)
    private List<OrderItem> items = new ArrayList<>();
    
    @Embedded
    private OrderPricing pricing;
    
    @OneToMany(mappedBy = "order", cascade = CascadeType.ALL)
    private List<OrderAddress> addresses = new ArrayList<>();
    
    @OneToMany(mappedBy = "order", cascade = CascadeType.ALL)
    private List<OrderPayment> payments = new ArrayList<>();
    
    @OneToMany(mappedBy = "order")
    @OrderBy("createdAt ASC")
    private List<OrderStatusHistory> statusHistory = new ArrayList<>();
    
    @Column(name = "shipping_method", nullable = false)
    @Enumerated(EnumType.STRING)
    private ShippingMethod shippingMethod;
    
    @Type(type = "jsonb")
    @Column(columnDefinition = "jsonb")
    private Map<String, Object> metadata;
    
    @CreatedDate
    private Instant createdAt;
    
    @LastModifiedDate
    private Instant updatedAt;
    
    private Instant paidAt;
    private Instant shippedAt;
    private Instant deliveredAt;
    private Instant cancelledAt;
    
    // Business methods
    public void addItem(Product product, int quantity, Money unitPrice) {
        OrderItem item = OrderItem.builder()
            .order(this)
            .productId(product.getId())
            .productSku(product.getSku())
            .productName(product.getName())
            .quantity(quantity)
            .unitPrice(unitPrice)
            .build();
        
        this.items.add(item);
        recalculateTotals();
    }
    
    public void transitionTo(OrderStatus newStatus, String reason) {
        if (!canTransitionTo(newStatus)) {
            throw new InvalidOrderStateTransitionException(this.status, newStatus);
        }
        
        OrderStatusHistory history = OrderStatusHistory.builder()
            .order(this)
            .previousStatus(this.status)
            .status(newStatus)
            .reason(reason)
            .build();
            
        this.statusHistory.add(history);
        this.status = newStatus;
        
        // Update timestamps
        switch (newStatus) {
            case PAID:
                this.paidAt = Instant.now();
                break;
            case SHIPPED:
                this.shippedAt = Instant.now();
                break;
            case DELIVERED:
                this.deliveredAt = Instant.now();
                break;
            case CANCELLED:
                this.cancelledAt = Instant.now();
                break;
        }
    }
    
    private boolean canTransitionTo(OrderStatus newStatus) {
        return this.status.canTransitionTo(newStatus);
    }
}

@Embeddable
public class OrderPricing {
    @Column(nullable = false)
    private BigDecimal subtotal = BigDecimal.ZERO;
    
    @Column(name = "shipping_cost", nullable = false)
    private BigDecimal shippingCost = BigDecimal.ZERO;
    
    @Column(name = "tax_amount", nullable = false)
    private BigDecimal taxAmount = BigDecimal.ZERO;
    
    @Column(name = "discount_amount", nullable = false)
    private BigDecimal discountAmount = BigDecimal.ZERO;
    
    @Column(name = "total_amount", nullable = false)
    private BigDecimal totalAmount = BigDecimal.ZERO;
    
    @Column(nullable = false)
    private String currency = "USD";
    
    public void calculate(List<OrderItem> items, List<Promotion> promotions) {
        this.subtotal = items.stream()
            .map(OrderItem::getSubtotal)
            .reduce(BigDecimal.ZERO, BigDecimal::add);
            
        this.discountAmount = promotions.stream()
            .map(Promotion::calculateDiscount)
            .reduce(BigDecimal.ZERO, BigDecimal::add);
            
        this.taxAmount = calculateTax(subtotal.subtract(discountAmount));
        
        this.totalAmount = subtotal
            .add(shippingCost)
            .add(taxAmount)
            .subtract(discountAmount);
    }
}

public enum OrderStatus {
    CREATED,
    PENDING_PAYMENT,
    PAID,
    PROCESSING,
    SHIPPED,
    DELIVERED,
    COMPLETED,
    CANCELLED,
    REFUNDED;
    
    public boolean canTransitionTo(OrderStatus newStatus) {
        switch (this) {
            case CREATED:
                return newStatus == PENDING_PAYMENT || newStatus == CANCELLED;
            case PENDING_PAYMENT:
                return newStatus == PAID || newStatus == CANCELLED;
            case PAID:
                return newStatus == PROCESSING || newStatus == CANCELLED || newStatus == REFUNDED;
            case PROCESSING:
                return newStatus == SHIPPED || newStatus == CANCELLED;
            case SHIPPED:
                return newStatus == DELIVERED || newStatus == CANCELLED;
            case DELIVERED:
                return newStatus == COMPLETED || newStatus == REFUNDED;
            case COMPLETED:
                return newStatus == REFUNDED;
            default:
                return false;
        }
    }
}
```

## Configuration

### Application Properties

```yaml
spring:
  application:
    name: order-service
  
  datasource:
    url: jdbc:postgresql://${DB_HOST:localhost}:5432/order_db
    username: ${DB_USER:order_user}
    password: ${DB_PASSWORD:password}
    hikari:
      maximum-pool-size: 25
      minimum-idle: 5
  
  kafka:
    bootstrap-servers: ${KAFKA_BROKERS:localhost:9092}
    consumer:
      group-id: order-service
      auto-offset-reset: earliest
    producer:
      acks: all
      retries: 3

order:
  number:
    prefix: "ORD"
    pattern: "${prefix}-${year}-${sequence}"
    
  payment:
    timeout-minutes: 30
    retry-attempts: 3
    
  cancellation:
    allowed-statuses: [CREATED, PENDING_PAYMENT, PAID, PROCESSING]
    auto-refund: true
    
  validation:
    min-order-amount: 10.00
    max-order-amount: 10000.00
    max-items-per-order: 100

payment:
  gateways:
    stripe:
      api-key: ${STRIPE_API_KEY}
      webhook-secret: ${STRIPE_WEBHOOK_SECRET}
    paypal:
      client-id: ${PAYPAL_CLIENT_ID}
      client-secret: ${PAYPAL_CLIENT_SECRET}
```

### Environment Variables

```bash
# Database
DB_HOST=postgres
DB_USER=order_user
DB_PASSWORD=secure_password

# Kafka
KAFKA_BROKERS=kafka1:9092,kafka2:9092,kafka3:9092

# Services
INVENTORY_SERVICE_URL=http://inventory-service:8083
PRODUCT_SERVICE_URL=http://product-catalog:8082
PAYMENT_SERVICE_URL=http://payment-service:8085
CUSTOMER_SERVICE_URL=http://customer-service:8086

# Payment Gateways
STRIPE_API_KEY=sk_live_xxx
STRIPE_WEBHOOK_SECRET=whsec_xxx
PAYPAL_CLIENT_ID=xxx
PAYPAL_CLIENT_SECRET=xxx

# Security
JWT_SECRET=your-secret-key
```

## Dependencies

### External Services
- **Product Catalog Service** - Product information and pricing
- **Inventory Service** - Stock availability and reservation
- **Payment Service** - Payment processing
- **Customer Service** - Customer information
- **Notification Service** - Order notifications

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
    
    <!-- State Machine -->
    <dependency>
        <groupId>org.springframework.statemachine</groupId>
        <artifactId>spring-statemachine-core</artifactId>
        <version>3.2.0</version>
    </dependency>
    
    <!-- PostgreSQL -->
    <dependency>
        <groupId>org.postgresql</groupId>
        <artifactId>postgresql</artifactId>
    </dependency>
    
    <!-- Kafka -->
    <dependency>
        <groupId>org.springframework.kafka</groupId>
        <artifactId>spring-kafka</artifactId>
    </dependency>
    
    <!-- Payment SDKs -->
    <dependency>
        <groupId>com.stripe</groupId>
        <artifactId>stripe-java</artifactId>
        <version>22.0.0</version>
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
void shouldCreateOrderSuccessfully() {
    // Given
    CreateOrderRequest request = createValidOrderRequest();
    when(inventoryService.checkAvailability(any())).thenReturn(true);
    when(productService.getProducts(any())).thenReturn(mockProducts());
    
    // When
    Order order = orderService.createOrder(request);
    
    // Then
    assertThat(order.getStatus()).isEqualTo(OrderStatus.PENDING_PAYMENT);
    assertThat(order.getItems()).hasSize(2);
    assertThat(order.getPricing().getTotalAmount()).isEqualTo(new BigDecimal("1479.97"));
    verify(eventPublisher).publish(any(OrderCreatedEvent.class));
}

@Test
void shouldTransitionOrderStatusCorrectly() {
    // Given
    Order order = createOrder(OrderStatus.PENDING_PAYMENT);
    
    // When
    order.transitionTo(OrderStatus.PAID, "Payment successful");
    
    // Then
    assertThat(order.getStatus()).isEqualTo(OrderStatus.PAID);
    assertThat(order.getPaidAt()).isNotNull();
    assertThat(order.getStatusHistory()).hasSize(1);
}
```

### Integration Tests
```java
@SpringBootTest
@AutoConfigureMockMvc
class OrderControllerIntegrationTest {
    
    @Test
    void shouldProcessCompleteOrderFlow() throws Exception {
        // Create order
        String orderResponse = mockMvc.perform(post("/api/v1/orders")
                .contentType(MediaType.APPLICATION_JSON)
                .content(readJson("create-order-request.json")))
                .andExpect(status().isCreated())
                .andReturn()
                .getResponse()
                .getContentAsString();
                
        String orderId = JsonPath.read(orderResponse, "$.orderId");
        
        // Simulate payment
        mockMvc.perform(post("/api/v1/orders/" + orderId + "/payment")
                .contentType(MediaType.APPLICATION_JSON)
                .content("{\"paymentToken\": \"tok_visa\"}"))
                .andExpect(status().isOk());
                
        // Verify order status
        mockMvc.perform(get("/api/v1/orders/" + orderId))
                .andExpect(status().isOk())
                .andExpect(jsonPath("$.status").value("PROCESSING"));
    }
}
```

## Deployment

### Kubernetes Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service
  namespace: ecommerce
spec:
  replicas: 3
  selector:
    matchLabels:
      app: order-service
  template:
    metadata:
      labels:
        app: order-service
    spec:
      containers:
      - name: order
        image: firstviscount/order-service:latest
        ports:
        - containerPort: 8084
        env:
        - name: SPRING_PROFILES_ACTIVE
          value: "production"
        - name: DB_HOST
          valueFrom:
            secretKeyRef:
              name: db-secrets
              key: host
        - name: STRIPE_API_KEY
          valueFrom:
            secretKeyRef:
              name: payment-secrets
              key: stripe-api-key
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
            port: 8084
          initialDelaySeconds: 30
        readinessProbe:
          httpGet:
            path: /actuator/health/readiness
            port: 8084
          initialDelaySeconds: 20
```

## Monitoring

### Health Checks

```java
@Component
public class OrderServiceHealthIndicator implements HealthIndicator {
    
    @Override
    public Health health() {
        try {
            // Check pending payments
            long pendingPayments = orderRepository
                .countByStatusAndCreatedAtBefore(
                    OrderStatus.PENDING_PAYMENT, 
                    Instant.now().minus(30, ChronoUnit.MINUTES)
                );
                
            // Check payment gateway connectivity
            boolean stripeHealthy = checkStripeHealth();
            
            if (pendingPayments > 100) {
                return Health.down()
                    .withDetail("stuck_payments", pendingPayments)
                    .build();
            }
            
            return Health.up()
                .withDetail("pending_orders", getPendingOrderCount())
                .withDetail("stripe_gateway", stripeHealthy ? "UP" : "DOWN")
                .build();
                
        } catch (Exception e) {
            return Health.down().withException(e).build();
        }
    }
}
```

### Metrics

Key metrics exposed:
- `orders.created.total` - Total orders created
- `orders.completed.total` - Total orders completed
- `orders.cancelled.total` - Total orders cancelled
- `orders.value.total` - Total order value
- `orders.processing.duration` - Order processing time
- `payment.success.rate` - Payment success rate

### Alerts

```yaml
groups:
  - name: order-service
    rules:
      - alert: HighOrderFailureRate
        expr: rate(orders_failed_total[5m]) > 0.05
        for: 5m
        annotations:
          summary: "High order failure rate"
          
      - alert: PaymentProcessingDelays
        expr: orders_pending_payment_count > 50
        for: 10m
        annotations:
          summary: "Many orders stuck in payment processing"
          
      - alert: HighOrderValue
        expr: order_value_amount > 5000
        annotations:
          summary: "High value order placed: ${{ $value }}"
```

## Best Practices

1. **Idempotency** - Ensure order creation is idempotent
2. **State Management** - Use state machine for order transitions
3. **Payment Security** - Never store sensitive payment data
4. **Event Sourcing** - Track all order state changes
5. **Validation** - Validate all order data thoroughly
6. **Monitoring** - Track key business metrics