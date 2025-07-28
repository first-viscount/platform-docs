# Delivery Management Service Specification

## Overview

The Delivery Management Service handles all aspects of order fulfillment including shipping label generation, carrier integration, package tracking, and delivery confirmation. It supports multiple shipping carriers and provides real-time tracking updates.

### Key Responsibilities
- Shipping label generation and management
- Multi-carrier integration (FedEx, UPS, USPS, DHL)
- Real-time tracking updates
- Delivery confirmation and proof of delivery
- Returns and reverse logistics
- Shipping cost calculation and optimization

## API Specification

### Base URL
```
http://delivery-service:8085/api/v1
```

### Endpoints

#### Create Shipment
```http
POST /shipments
Content-Type: application/json
Authorization: Bearer {token}

{
  "orderId": "ord-770e8400-e29b-41d4-a716-446655440000",
  "items": [
    {
      "productId": "550e8400-e29b-41d4-a716-446655440000",
      "quantity": 2,
      "weight": 3.5,
      "dimensions": {
        "length": 15,
        "width": 10,
        "height": 5,
        "unit": "INCH"
      }
    }
  ],
  "origin": {
    "warehouseId": "wh-001",
    "address": {
      "name": "Main Warehouse",
      "street": "100 Warehouse Way",
      "city": "Dallas",
      "state": "TX",
      "zipCode": "75201",
      "country": "US"
    }
  },
  "destination": {
    "name": "John Doe",
    "street": "123 Main St",
    "city": "New York",
    "state": "NY",
    "zipCode": "10001",
    "country": "US",
    "phone": "+1234567890",
    "email": "john.doe@example.com"
  },
  "shippingMethod": "GROUND",
  "insurance": {
    "required": true,
    "value": 1199.98
  },
  "signatureRequired": true
}

Response:
{
  "shipmentId": "ship-880e8400-e29b-41d4-a716-446655440000",
  "orderId": "ord-770e8400-e29b-41d4-a716-446655440000",
  "carrier": "FEDEX",
  "trackingNumber": "1234567890123",
  "status": "LABEL_CREATED",
  "shippingCost": {
    "base": 12.99,
    "insurance": 5.00,
    "signature": 3.00,
    "total": 20.99
  },
  "estimatedDelivery": {
    "earliest": "2024-01-18",
    "latest": "2024-01-20"
  },
  "labels": [
    {
      "type": "SHIPPING_LABEL",
      "format": "PDF",
      "url": "https://api.firstviscount.com/labels/ship-880e8400.pdf"
    }
  ],
  "createdAt": "2024-01-15T10:00:00Z"
}
```

#### Get Shipment Details
```http
GET /shipments/{shipmentId}
Authorization: Bearer {token}

Response:
{
  "shipmentId": "ship-880e8400-e29b-41d4-a716-446655440000",
  "orderId": "ord-770e8400-e29b-41d4-a716-446655440000",
  "carrier": "FEDEX",
  "service": "FedEx Ground",
  "trackingNumber": "1234567890123",
  "status": "IN_TRANSIT",
  "statusHistory": [
    {
      "status": "LABEL_CREATED",
      "timestamp": "2024-01-15T10:00:00Z",
      "location": "Dallas, TX"
    },
    {
      "status": "PICKED_UP",
      "timestamp": "2024-01-15T14:00:00Z",
      "location": "Dallas, TX"
    },
    {
      "status": "IN_TRANSIT",
      "timestamp": "2024-01-16T08:00:00Z",
      "location": "Memphis, TN",
      "description": "Package departed FedEx hub"
    }
  ],
  "currentLocation": {
    "city": "Memphis",
    "state": "TN",
    "timestamp": "2024-01-16T08:00:00Z"
  },
  "estimatedDelivery": "2024-01-18",
  "actualDelivery": null,
  "proofOfDelivery": null
}
```

#### Track Shipment
```http
GET /shipments/track/{trackingNumber}
Authorization: Bearer {token}

Response:
{
  "trackingNumber": "1234567890123",
  "carrier": "FEDEX",
  "status": "OUT_FOR_DELIVERY",
  "currentLocation": {
    "description": "On FedEx vehicle for delivery",
    "city": "New York",
    "state": "NY",
    "timestamp": "2024-01-18T08:30:00Z"
  },
  "estimatedDelivery": {
    "date": "2024-01-18",
    "timeWindow": {
      "start": "09:00",
      "end": "17:00"
    }
  },
  "events": [
    {
      "timestamp": "2024-01-18T08:30:00Z",
      "description": "Out for delivery",
      "location": "New York, NY",
      "status": "OUT_FOR_DELIVERY"
    },
    {
      "timestamp": "2024-01-18T05:00:00Z",
      "description": "At local FedEx facility",
      "location": "New York, NY",
      "status": "AT_FACILITY"
    }
  ],
  "deliveryAttempts": 0
}
```

#### Calculate Shipping Rates
```http
POST /shipping/calculate
Content-Type: application/json
Authorization: Bearer {token}

{
  "origin": {
    "zipCode": "75201",
    "country": "US"
  },
  "destination": {
    "zipCode": "10001",
    "country": "US"
  },
  "packages": [
    {
      "weight": 5.0,
      "dimensions": {
        "length": 12,
        "width": 8,
        "height": 6,
        "unit": "INCH"
      }
    }
  ],
  "services": ["GROUND", "EXPRESS", "OVERNIGHT"],
  "includeInsurance": true,
  "insuredValue": 599.99
}

Response:
{
  "rates": [
    {
      "carrier": "FEDEX",
      "service": "FedEx Ground",
      "serviceCode": "FEDEX_GROUND",
      "cost": 12.99,
      "currency": "USD",
      "estimatedDays": 3,
      "estimatedDelivery": "2024-01-18"
    },
    {
      "carrier": "FEDEX",
      "service": "FedEx 2Day",
      "serviceCode": "FEDEX_2_DAY",
      "cost": 24.99,
      "currency": "USD",
      "estimatedDays": 2,
      "estimatedDelivery": "2024-01-17"
    },
    {
      "carrier": "UPS",
      "service": "UPS Ground",
      "serviceCode": "UPS_GROUND",
      "cost": 13.49,
      "currency": "USD",
      "estimatedDays": 3,
      "estimatedDelivery": "2024-01-18"
    }
  ]
}
```

#### Confirm Delivery
```http
POST /shipments/{shipmentId}/confirm
Content-Type: application/json
Authorization: Bearer {token}

{
  "deliveredAt": "2024-01-18T14:30:00Z",
  "signedBy": "J. DOE",
  "deliveryLocation": "FRONT_DOOR",
  "proofOfDelivery": {
    "imageUrl": "https://cdn.firstviscount.com/pod/ship-880e8400.jpg",
    "signatureUrl": "https://cdn.firstviscount.com/signatures/ship-880e8400.png"
  },
  "notes": "Left with building concierge"
}

Response:
{
  "shipmentId": "ship-880e8400-e29b-41d4-a716-446655440000",
  "status": "DELIVERED",
  "deliveredAt": "2024-01-18T14:30:00Z",
  "confirmationNumber": "DEL-2024011814300123"
}
```

#### Create Return Shipment
```http
POST /returns
Content-Type: application/json
Authorization: Bearer {token}

{
  "originalShipmentId": "ship-880e8400-e29b-41d4-a716-446655440000",
  "orderId": "ord-770e8400-e29b-41d4-a716-446655440000",
  "reason": "DEFECTIVE",
  "items": [
    {
      "productId": "550e8400-e29b-41d4-a716-446655440000",
      "quantity": 1
    }
  ],
  "pickupRequired": true,
  "pickupDate": "2024-01-20",
  "pickupTimeWindow": {
    "start": "09:00",
    "end": "17:00"
  }
}

Response:
{
  "returnId": "ret-990e8400-e29b-41d4-a716-446655440000",
  "returnNumber": "RMA-2024-001234",
  "shipmentId": "ship-aa0e8400-e29b-41d4-a716-446655440000",
  "carrier": "FEDEX",
  "trackingNumber": "9876543210987",
  "status": "LABEL_CREATED",
  "labels": [
    {
      "type": "RETURN_LABEL",
      "format": "PDF",
      "url": "https://api.firstviscount.com/labels/ret-990e8400.pdf"
    }
  ],
  "pickupConfirmation": {
    "scheduled": true,
    "date": "2024-01-20",
    "confirmationNumber": "PICKUP-123456"
  },
  "createdAt": "2024-01-19T10:00:00Z"
}
```

#### Get Delivery Metrics
```http
GET /metrics/deliveries?startDate=2024-01-01&endDate=2024-01-31
Authorization: Bearer {admin-token}

Response:
{
  "period": {
    "start": "2024-01-01",
    "end": "2024-01-31"
  },
  "summary": {
    "totalShipments": 12543,
    "delivered": 11234,
    "inTransit": 1098,
    "exceptions": 211
  },
  "performance": {
    "onTimeDeliveryRate": 94.5,
    "averageDeliveryDays": 2.8,
    "deliveryAttempts": {
      "firstAttempt": 10456,
      "secondAttempt": 698,
      "thirdAttempt": 80
    }
  },
  "carrierBreakdown": {
    "FEDEX": {
      "shipments": 7234,
      "onTimeRate": 95.2,
      "averageCost": 14.56
    },
    "UPS": {
      "shipments": 4521,
      "onTimeRate": 93.8,
      "averageCost": 13.89
    }
  },
  "costs": {
    "total": 182456.78,
    "average": 14.54,
    "byService": {
      "GROUND": 123456.78,
      "EXPRESS": 45678.90,
      "OVERNIGHT": 13321.10
    }
  }
}
```

## Event Contracts

### Published Events

#### ShipmentCreatedEvent
```json
{
  "eventId": "ev-110e8400-e29b-41d4-a716-446655440000",
  "shipmentId": "ship-880e8400-e29b-41d4-a716-446655440000",
  "orderId": "ord-770e8400-e29b-41d4-a716-446655440000",
  "carrier": "FEDEX",
  "trackingNumber": "1234567890123",
  "estimatedDelivery": "2024-01-18",
  "cost": 20.99,
  "timestamp": "2024-01-15T10:00:00Z"
}
```

#### ShipmentStatusUpdatedEvent
```json
{
  "eventId": "ev-220e8400-e29b-41d4-a716-446655440000",
  "shipmentId": "ship-880e8400-e29b-41d4-a716-446655440000",
  "trackingNumber": "1234567890123",
  "previousStatus": "PICKED_UP",
  "newStatus": "IN_TRANSIT",
  "location": {
    "city": "Memphis",
    "state": "TN",
    "description": "FedEx hub"
  },
  "timestamp": "2024-01-16T08:00:00Z"
}
```

#### DeliveryCompletedEvent
```json
{
  "eventId": "ev-330e8400-e29b-41d4-a716-446655440000",
  "shipmentId": "ship-880e8400-e29b-41d4-a716-446655440000",
  "orderId": "ord-770e8400-e29b-41d4-a716-446655440000",
  "trackingNumber": "1234567890123",
  "deliveredAt": "2024-01-18T14:30:00Z",
  "signedBy": "J. DOE",
  "proofOfDelivery": {
    "available": true,
    "imageUrl": "https://cdn.firstviscount.com/pod/ship-880e8400.jpg"
  },
  "timestamp": "2024-01-18T14:30:00Z"
}
```

#### DeliveryExceptionEvent
```json
{
  "eventId": "ev-440e8400-e29b-41d4-a716-446655440000",
  "shipmentId": "ship-880e8400-e29b-41d4-a716-446655440000",
  "trackingNumber": "1234567890123",
  "exceptionType": "DELIVERY_ATTEMPTED",
  "reason": "Customer not available",
  "nextAttemptDate": "2024-01-19",
  "timestamp": "2024-01-18T15:00:00Z"
}
```

### Consumed Events

- `OrderPaidEvent` - Triggers shipment creation
- `OrderCancelledEvent` - Cancels pending shipments
- `ReturnApprovedEvent` - Creates return shipping labels

## Data Model

### Database Schema

```sql
-- Shipments table
CREATE TABLE shipments (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    shipment_number VARCHAR(50) UNIQUE NOT NULL,
    order_id UUID NOT NULL,
    warehouse_id UUID NOT NULL,
    carrier VARCHAR(50) NOT NULL,
    service VARCHAR(100) NOT NULL,
    tracking_number VARCHAR(100) UNIQUE,
    status VARCHAR(50) NOT NULL DEFAULT 'CREATED',
    shipping_cost DECIMAL(10,2),
    insurance_cost DECIMAL(10,2),
    additional_fees DECIMAL(10,2),
    total_cost DECIMAL(10,2),
    weight DECIMAL(10,2),
    estimated_delivery DATE,
    actual_delivery TIMESTAMP,
    metadata JSONB,
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    shipped_at TIMESTAMP,
    delivered_at TIMESTAMP,
    cancelled_at TIMESTAMP,
    
    INDEX idx_order_id (order_id),
    INDEX idx_tracking_number (tracking_number),
    INDEX idx_status (status),
    INDEX idx_carrier (carrier),
    INDEX idx_created_at (created_at)
);

-- Shipment addresses
CREATE TABLE shipment_addresses (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    shipment_id UUID NOT NULL REFERENCES shipments(id),
    type VARCHAR(20) NOT NULL, -- ORIGIN, DESTINATION
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
    is_residential BOOLEAN DEFAULT true,
    
    INDEX idx_shipment_id (shipment_id),
    UNIQUE(shipment_id, type)
);

-- Packages
CREATE TABLE packages (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    shipment_id UUID NOT NULL REFERENCES shipments(id),
    package_number INT NOT NULL,
    weight DECIMAL(10,2) NOT NULL,
    length DECIMAL(10,2),
    width DECIMAL(10,2),
    height DECIMAL(10,2),
    dimension_unit VARCHAR(10) DEFAULT 'INCH',
    package_type VARCHAR(50),
    tracking_number VARCHAR(100),
    
    INDEX idx_shipment_id (shipment_id),
    UNIQUE(shipment_id, package_number)
);

-- Package items
CREATE TABLE package_items (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    package_id UUID NOT NULL REFERENCES packages(id),
    product_id UUID NOT NULL,
    quantity INT NOT NULL,
    
    INDEX idx_package_id (package_id)
);

-- Tracking events
CREATE TABLE tracking_events (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    shipment_id UUID NOT NULL REFERENCES shipments(id),
    carrier_event_id VARCHAR(100),
    event_type VARCHAR(100) NOT NULL,
    status VARCHAR(50) NOT NULL,
    location_city VARCHAR(100),
    location_state VARCHAR(50),
    location_zip VARCHAR(20),
    location_country VARCHAR(2),
    description TEXT,
    carrier_timestamp TIMESTAMP,
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    
    INDEX idx_shipment_id (shipment_id),
    INDEX idx_carrier_timestamp (carrier_timestamp),
    UNIQUE(shipment_id, carrier_event_id)
);

-- Delivery confirmations
CREATE TABLE delivery_confirmations (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    shipment_id UUID NOT NULL REFERENCES shipments(id),
    delivered_at TIMESTAMP NOT NULL,
    signed_by VARCHAR(100),
    delivery_location VARCHAR(100),
    proof_image_url VARCHAR(500),
    signature_image_url VARCHAR(500),
    notes TEXT,
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    
    INDEX idx_shipment_id (shipment_id),
    UNIQUE(shipment_id)
);

-- Returns
CREATE TABLE returns (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    return_number VARCHAR(50) UNIQUE NOT NULL,
    original_shipment_id UUID REFERENCES shipments(id),
    return_shipment_id UUID REFERENCES shipments(id),
    order_id UUID NOT NULL,
    reason VARCHAR(100) NOT NULL,
    status VARCHAR(50) NOT NULL DEFAULT 'INITIATED',
    pickup_scheduled BOOLEAN DEFAULT false,
    pickup_date DATE,
    pickup_confirmation VARCHAR(100),
    refund_amount DECIMAL(10,2),
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    received_at TIMESTAMP,
    processed_at TIMESTAMP,
    
    INDEX idx_order_id (order_id),
    INDEX idx_status (status),
    INDEX idx_created_at (created_at)
);

-- Carrier configurations
CREATE TABLE carrier_configs (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    carrier VARCHAR(50) NOT NULL UNIQUE,
    api_endpoint VARCHAR(500) NOT NULL,
    account_number VARCHAR(100),
    meter_number VARCHAR(100),
    credentials JSONB NOT NULL, -- Encrypted
    settings JSONB,
    supported_services JSONB,
    is_active BOOLEAN DEFAULT true,
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    
    INDEX idx_carrier (carrier),
    INDEX idx_is_active (is_active)
);

-- Shipping zones
CREATE TABLE shipping_zones (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(100) NOT NULL,
    countries TEXT[] NOT NULL,
    states TEXT[],
    zip_patterns TEXT[],
    shipping_methods JSONB,
    restrictions JSONB,
    is_active BOOLEAN DEFAULT true,
    
    INDEX idx_is_active (is_active)
);
```

### Entity Models

```java
@Entity
@Table(name = "shipments")
@EntityListeners(AuditingEntityListener.class)
public class Shipment {
    @Id
    @GeneratedValue
    private UUID id;
    
    @Column(name = "shipment_number", unique = true, nullable = false)
    private String shipmentNumber;
    
    @Column(name = "order_id", nullable = false)
    private UUID orderId;
    
    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "warehouse_id", nullable = false)
    private Warehouse warehouse;
    
    @Enumerated(EnumType.STRING)
    private Carrier carrier;
    
    @Column(nullable = false)
    private String service;
    
    @Column(name = "tracking_number", unique = true)
    private String trackingNumber;
    
    @Enumerated(EnumType.STRING)
    private ShipmentStatus status = ShipmentStatus.CREATED;
    
    @OneToMany(mappedBy = "shipment", cascade = CascadeType.ALL)
    private List<Package> packages = new ArrayList<>();
    
    @OneToMany(mappedBy = "shipment", cascade = CascadeType.ALL)
    private List<ShipmentAddress> addresses = new ArrayList<>();
    
    @OneToMany(mappedBy = "shipment")
    @OrderBy("carrierTimestamp DESC")
    private List<TrackingEvent> trackingEvents = new ArrayList<>();
    
    @Embedded
    private ShippingCost cost;
    
    @Column(name = "estimated_delivery")
    private LocalDate estimatedDelivery;
    
    @Column(name = "actual_delivery")
    private Instant actualDelivery;
    
    @Type(type = "jsonb")
    @Column(columnDefinition = "jsonb")
    private Map<String, Object> metadata;
    
    @CreatedDate
    private Instant createdAt;
    
    private Instant shippedAt;
    private Instant deliveredAt;
    private Instant cancelledAt;
    
    // Business methods
    public void ship() {
        if (this.status != ShipmentStatus.LABEL_CREATED) {
            throw new InvalidShipmentStateException("Cannot ship from status: " + this.status);
        }
        this.status = ShipmentStatus.SHIPPED;
        this.shippedAt = Instant.now();
    }
    
    public void updateTracking(TrackingEvent event) {
        this.trackingEvents.add(event);
        this.status = mapTrackingStatusToShipmentStatus(event.getStatus());
        
        if (event.getStatus() == TrackingStatus.DELIVERED) {
            this.deliveredAt = event.getCarrierTimestamp();
            this.actualDelivery = event.getCarrierTimestamp();
        }
    }
    
    public ShipmentAddress getOriginAddress() {
        return addresses.stream()
            .filter(a -> a.getType() == AddressType.ORIGIN)
            .findFirst()
            .orElseThrow();
    }
    
    public ShipmentAddress getDestinationAddress() {
        return addresses.stream()
            .filter(a -> a.getType() == AddressType.DESTINATION)
            .findFirst()
            .orElseThrow();
    }
}

@Embeddable
public class ShippingCost {
    @Column(name = "shipping_cost")
    private BigDecimal shippingCost;
    
    @Column(name = "insurance_cost")
    private BigDecimal insuranceCost;
    
    @Column(name = "additional_fees")
    private BigDecimal additionalFees;
    
    @Column(name = "total_cost")
    private BigDecimal totalCost;
    
    public void calculate() {
        this.totalCost = Stream.of(shippingCost, insuranceCost, additionalFees)
            .filter(Objects::nonNull)
            .reduce(BigDecimal.ZERO, BigDecimal::add);
    }
}

public enum ShipmentStatus {
    CREATED,
    LABEL_CREATED,
    PICKED_UP,
    SHIPPED,
    IN_TRANSIT,
    OUT_FOR_DELIVERY,
    DELIVERED,
    EXCEPTION,
    RETURNED,
    CANCELLED
}

public enum Carrier {
    FEDEX("FedEx"),
    UPS("UPS"),
    USPS("USPS"),
    DHL("DHL");
    
    private final String displayName;
    
    Carrier(String displayName) {
        this.displayName = displayName;
    }
}
```

## Configuration

### Application Properties

```yaml
spring:
  application:
    name: delivery-service
  
  datasource:
    url: jdbc:postgresql://${DB_HOST:localhost}:5432/delivery_db
    username: ${DB_USER:delivery_user}
    password: ${DB_PASSWORD:password}
    hikari:
      maximum-pool-size: 20
      minimum-idle: 5
  
  kafka:
    bootstrap-servers: ${KAFKA_BROKERS:localhost:9092}
    consumer:
      group-id: delivery-service
      auto-offset-reset: earliest
    producer:
      acks: all

delivery:
  carriers:
    fedex:
      enabled: true
      api-url: https://apis.fedex.com
      account-number: ${FEDEX_ACCOUNT}
      meter-number: ${FEDEX_METER}
      key: ${FEDEX_KEY}
      password: ${FEDEX_PASSWORD}
      
    ups:
      enabled: true
      api-url: https://onlinetools.ups.com/api
      account-number: ${UPS_ACCOUNT}
      user-id: ${UPS_USER_ID}
      password: ${UPS_PASSWORD}
      access-key: ${UPS_ACCESS_KEY}
      
    usps:
      enabled: true
      api-url: https://secure.shippingapis.com
      user-id: ${USPS_USER_ID}
      
  tracking:
    update-interval-minutes: 30
    max-age-days: 90
    
  label:
    storage: S3
    bucket: ${LABEL_BUCKET:shipping-labels}
    expiry-days: 30
    
  returns:
    default-window-days: 30
    pickup-enabled: true
```

### Environment Variables

```bash
# Database
DB_HOST=postgres
DB_USER=delivery_user
DB_PASSWORD=secure_password

# Kafka
KAFKA_BROKERS=kafka1:9092,kafka2:9092,kafka3:9092

# Carrier Credentials
FEDEX_ACCOUNT=123456789
FEDEX_METER=987654321
FEDEX_KEY=xxx
FEDEX_PASSWORD=xxx

UPS_ACCOUNT=ABC123
UPS_USER_ID=xxx
UPS_PASSWORD=xxx
UPS_ACCESS_KEY=xxx

USPS_USER_ID=xxx

# AWS S3 for label storage
AWS_ACCESS_KEY_ID=xxx
AWS_SECRET_ACCESS_KEY=xxx
AWS_REGION=us-east-1
LABEL_BUCKET=firstviscount-shipping-labels

# Services
ORDER_SERVICE_URL=http://order-service:8084
INVENTORY_SERVICE_URL=http://inventory-service:8083
```

## Dependencies

### External Services
- **Order Service** - Order information
- **Inventory Service** - Warehouse locations
- **Notification Service** - Shipping notifications
- **Carrier APIs** - FedEx, UPS, USPS, DHL

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
    
    <!-- Kafka -->
    <dependency>
        <groupId>org.springframework.kafka</groupId>
        <artifactId>spring-kafka</artifactId>
    </dependency>
    
    <!-- AWS SDK for S3 -->
    <dependency>
        <groupId>software.amazon.awssdk</groupId>
        <artifactId>s3</artifactId>
        <version>2.20.0</version>
    </dependency>
    
    <!-- PDF Generation -->
    <dependency>
        <groupId>com.itextpdf</groupId>
        <artifactId>itext7-core</artifactId>
        <version>7.2.5</version>
    </dependency>
    
    <!-- Carrier SDKs -->
    <dependency>
        <groupId>com.fedex</groupId>
        <artifactId>fedex-java-sdk</artifactId>
        <version>4.0.0</version>
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
void shouldCalculateShippingCost() {
    // Given
    ShippingRequest request = ShippingRequest.builder()
        .origin(createWarehouseAddress())
        .destination(createCustomerAddress())
        .packages(List.of(createPackage(5.0, 12, 8, 6)))
        .service(ShippingService.GROUND)
        .build();
        
    when(fedexClient.calculateRate(any())).thenReturn(
        Rate.builder().amount(12.99).build()
    );
    
    // When
    ShippingRate rate = shippingService.calculateRate(request);
    
    // Then
    assertThat(rate.getAmount()).isEqualTo(new BigDecimal("12.99"));
    assertThat(rate.getEstimatedDays()).isEqualTo(3);
}

@Test
void shouldUpdateTrackingStatus() {
    // Given
    Shipment shipment = createShipment(ShipmentStatus.IN_TRANSIT);
    TrackingUpdate update = TrackingUpdate.builder()
        .status("OUT_FOR_DELIVERY")
        .location("New York, NY")
        .timestamp(Instant.now())
        .build();
    
    // When
    shipment.updateTracking(update);
    
    // Then
    assertThat(shipment.getStatus()).isEqualTo(ShipmentStatus.OUT_FOR_DELIVERY);
    assertThat(shipment.getTrackingEvents()).hasSize(1);
}
```

### Integration Tests
```java
@SpringBootTest
@AutoConfigureMockMvc
class DeliveryControllerIntegrationTest {
    
    @Test
    void shouldCreateShipmentEndToEnd() throws Exception {
        // Create shipment
        String response = mockMvc.perform(post("/api/v1/shipments")
                .contentType(MediaType.APPLICATION_JSON)
                .content(readJson("create-shipment-request.json")))
                .andExpect(status().isCreated())
                .andReturn()
                .getResponse()
                .getContentAsString();
                
        String shipmentId = JsonPath.read(response, "$.shipmentId");
        String trackingNumber = JsonPath.read(response, "$.trackingNumber");
        
        // Verify label generated
        assertThat(JsonPath.read(response, "$.labels[0].url")).isNotNull();
        
        // Track shipment
        mockMvc.perform(get("/api/v1/shipments/track/" + trackingNumber))
                .andExpect(status().isOk())
                .andExpect(jsonPath("$.status").value("LABEL_CREATED"));
    }
}
```

## Deployment

### Kubernetes Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: delivery-service
  namespace: ecommerce
spec:
  replicas: 2
  selector:
    matchLabels:
      app: delivery-service
  template:
    metadata:
      labels:
        app: delivery-service
    spec:
      containers:
      - name: delivery
        image: firstviscount/delivery-service:latest
        ports:
        - containerPort: 8085
        env:
        - name: SPRING_PROFILES_ACTIVE
          value: "production"
        - name: DB_HOST
          valueFrom:
            secretKeyRef:
              name: db-secrets
              key: host
        - name: FEDEX_KEY
          valueFrom:
            secretKeyRef:
              name: carrier-secrets
              key: fedex-key
        resources:
          requests:
            memory: "512Mi"
            cpu: "250m"
          limits:
            memory: "1Gi"
            cpu: "500m"
        livenessProbe:
          httpGet:
            path: /actuator/health/liveness
            port: 8085
          initialDelaySeconds: 30
        readinessProbe:
          httpGet:
            path: /actuator/health/readiness
            port: 8085
          initialDelaySeconds: 20
```

## Monitoring

### Health Checks

```java
@Component
public class CarrierHealthIndicator implements HealthIndicator {
    
    @Override
    public Health health() {
        Map<String, Object> details = new HashMap<>();
        
        // Check each carrier API
        for (Carrier carrier : Carrier.values()) {
            try {
                boolean healthy = carrierClients.get(carrier).healthCheck();
                details.put(carrier.name().toLowerCase(), healthy ? "UP" : "DOWN");
            } catch (Exception e) {
                details.put(carrier.name().toLowerCase(), "DOWN");
            }
        }
        
        // Check label storage
        try {
            s3Client.headBucket(HeadBucketRequest.builder()
                .bucket(labelBucket)
                .build());
            details.put("label_storage", "UP");
        } catch (Exception e) {
            details.put("label_storage", "DOWN");
        }
        
        boolean allHealthy = details.values().stream()
            .allMatch(status -> "UP".equals(status));
            
        return allHealthy ? Health.up().withDetails(details).build()
                         : Health.down().withDetails(details).build();
    }
}
```

### Metrics

Key metrics exposed:
- `shipments.created.total` - Total shipments created
- `shipments.delivered.total` - Total deliveries completed
- `delivery.time.days` - Delivery time histogram
- `shipping.cost.amount` - Shipping costs by carrier
- `tracking.updates.processed` - Tracking updates processed
- `delivery.exceptions.total` - Delivery exceptions by type

### Alerts

```yaml
groups:
  - name: delivery-service
    rules:
      - alert: HighDeliveryExceptionRate
        expr: rate(delivery_exceptions_total[1h]) > 0.05
        for: 15m
        annotations:
          summary: "High rate of delivery exceptions"
          
      - alert: CarrierAPIDown
        expr: carrier_api_health == 0
        for: 5m
        annotations:
          summary: "Carrier API {{ $labels.carrier }} is down"
          
      - alert: LateDeliveries
        expr: delivery_late_count > 100
        for: 30m
        annotations:
          summary: "High number of late deliveries"
```

## Best Practices

1. **Carrier Abstraction** - Abstract carrier-specific logic
2. **Retry Logic** - Implement retry for carrier API calls
3. **Label Storage** - Store labels securely with expiration
4. **Tracking Updates** - Process updates asynchronously
5. **Cost Optimization** - Compare rates across carriers
6. **Exception Handling** - Handle delivery exceptions gracefully