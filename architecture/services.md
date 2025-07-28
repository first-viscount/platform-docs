# Microservices Architecture

## Service Overview

The First Viscount platform consists of 6 microservices, each with clear boundaries and specific responsibilities. Services communicate exclusively through events (Redpanda) and REST APIs.

## Initial Services (Phase 1)

### 1. Platform Coordination Service
**Port**: 8081  
**Database**: PostgreSQL (platform_coordination_db)  
**Purpose**: Service discovery, health monitoring, and distributed coordination

**Responsibilities**:
- Service registry and discovery
- Health check aggregation
- Configuration management
- Distributed locking (if needed)
- Platform-wide metrics collection

**Key APIs**:
- `POST /services/register` - Register a service
- `GET /services` - List all services
- `GET /services/{id}/health` - Get service health
- `GET /config/{service}` - Get service configuration

**Events Published**:
- `ServiceRegistered`
- `ServiceDeregistered` 
- `ServiceHealthChanged`
- `ConfigurationUpdated`

**Events Consumed**:
- None (source of truth for platform state)

### 2. Product Catalog Service
**Port**: 8082  
**Database**: PostgreSQL (product_catalog_db)  
**Purpose**: Manage product information, categories, and search

**Responsibilities**:
- Product CRUD operations
- Category management
- Product search and filtering
- Price management
- Product attributes and variants

**Key APIs**:
- `GET /products` - List products with filtering
- `GET /products/{id}` - Get product details
- `POST /products` - Create product (admin)
- `PUT /products/{id}` - Update product (admin)
- `GET /products/search` - Full-text search

**Events Published**:
- `ProductCreated`
- `ProductUpdated`
- `ProductDeleted`
- `PriceChanged`
- `ProductOutOfStock`

**Events Consumed**:
- `InventoryUpdated` (to update stock status)

### 3. Inventory Management Service
**Port**: 8083  
**Database**: PostgreSQL (inventory_db)  
**Purpose**: Track inventory levels and manage stock

**Responsibilities**:
- Real-time inventory tracking
- Stock reservations
- Inventory adjustments
- Low stock alerts
- Multi-location inventory

**Key APIs**:
- `GET /inventory/{product_id}` - Get current stock
- `POST /inventory/reserve` - Reserve inventory
- `POST /inventory/release` - Release reservation
- `POST /inventory/adjust` - Adjust stock levels
- `GET /inventory/low-stock` - Get low stock items

**Events Published**:
- `InventoryReserved`
- `InventoryReleased`
- `InventoryAdjusted`
- `LowStockAlert`
- `InventoryUpdated`

**Events Consumed**:
- `OrderCreated` (to reserve inventory)
- `OrderCancelled` (to release inventory)
- `OrderCompleted` (to deduct inventory)

### 4. Order Management Service
**Port**: 8084  
**Database**: PostgreSQL (order_db)  
**Purpose**: Handle order lifecycle from creation to completion

**Responsibilities**:
- Order creation and validation
- Order state management
- Payment processing integration
- Order history
- Refund and cancellation handling

**Key APIs**:
- `POST /orders` - Create new order
- `GET /orders/{id}` - Get order details
- `GET /orders` - List orders (with filters)
- `POST /orders/{id}/cancel` - Cancel order
- `POST /orders/{id}/refund` - Process refund

**Events Published**:
- `OrderCreated`
- `OrderValidated`
- `OrderPaymentProcessed`
- `OrderCancelled`
- `OrderCompleted`
- `OrderRefunded`

**Events Consumed**:
- `InventoryReserved` (to confirm order)
- `PaymentProcessed` (future)
- `ShipmentCreated` (future)

## Future Services (Phase 2)

### 5. Delivery Management Service
**Port**: 8085  
**Database**: PostgreSQL (delivery_db)  
**Purpose**: Manage shipping and delivery

**Planned Responsibilities**:
- Shipping label generation
- Carrier integration
- Tracking updates
- Delivery confirmation
- Returns management

### 6. Notifications Service
**Port**: 8086  
**Database**: PostgreSQL (notifications_db)  
**Purpose**: Handle all customer communications

**Planned Responsibilities**:
- Email notifications
- SMS notifications
- Push notifications
- Template management
- Notification preferences

## Service Communication Patterns

### Synchronous (REST)
- Used for queries and immediate responses
- Service discovery via Platform Coordination
- Circuit breakers on all calls
- Maximum 2-hop chains

### Asynchronous (Events)
- Primary communication method
- All state changes published as events
- Event sourcing for audit trail
- Eventual consistency embraced

## Service Boundaries

### Data Ownership
- Each service owns its database completely
- No shared tables or schemas
- No direct database access between services
- Data duplication is acceptable for autonomy

### API Contracts
- All APIs versioned (v1, v2, etc.)
- OpenAPI specifications required
- Breaking changes require new version
- Deprecation period of 30 days minimum

### Event Contracts
- Schema registry for all events
- Backward compatibility required
- Event versioning strategy
- Protobuf or Avro for schema evolution

## Development Principles

### Service Independence
- Services must start without dependencies
- Graceful degradation if dependencies unavailable
- No shared libraries (only contracts)
- Independent deployment pipelines

### Testing Strategy
- Unit tests within service
- Contract tests between services
- End-to-end tests for critical paths
- Chaos testing for resilience

### Monitoring & Observability
- Structured logging (JSON)
- Distributed tracing (OpenTelemetry)
- Custom metrics (Prometheus)
- Health checks (liveness & readiness)

## Anti-Patterns to Avoid

1. **Distributed Monolith**: Services calling each other synchronously in chains
2. **Shared Database**: Multiple services accessing same tables
3. **Chatty Services**: Too many fine-grained calls
4. **Missing Service Boundaries**: Unclear ownership
5. **Synchronous Everything**: Not embracing eventual consistency

---

*This architecture promotes true service independence while maintaining system cohesion through well-defined contracts and events.*