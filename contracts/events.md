# Event Contracts

This document defines the event contracts for all services in the First Viscount e-commerce platform. Events are the primary mechanism for service communication and data consistency across the distributed system.

## Overview

Events in our platform follow a consistent structure and naming convention to ensure reliable service communication. During Phase 1 MVP, events are logged locally within each service. Phase 2 will implement an event bus (Apache Kafka) for actual event publishing and consumption.

### Event Structure

All events share a common base structure:

```json
{
  "event_id": "550e8400-e29b-41d4-a716-446655440000",
  "correlation_id": "550e8400-e29b-41d4-a716-446655440001", 
  "timestamp": "2024-01-15T10:30:00Z",
  "source_service": "service-name",
  "event_version": "1.0",
  "event_type": "domain.action",
  "data": { /* event-specific payload */ }
}
```

### Naming Conventions

- **Event Types**: Use `domain.action` format (e.g., `product.created`, `inventory.reserved`)
- **Field Names**: Use snake_case for consistency
- **Service Names**: Use kebab-case (e.g., `product-catalog-service`)
- **Timestamps**: Always UTC in ISO 8601 format

### Event Versioning Strategy

- Start with version `1.0` for all events
- Increment minor version (1.1, 1.2) for backward-compatible changes
- Increment major version (2.0) for breaking changes
- Maintain compatibility for at least one major version

## Product Catalog Service Events

The Product Catalog Service publishes events related to product lifecycle management.

### ProductCreatedEvent

**Purpose**: Published when a new product is successfully created in the catalog.

**When Published**: After successful product creation and database persistence.

**Future Consumers**: Inventory Management Service (for stock initialization), Order Management Service (for order validation), Search Service (for indexing).

**Event Structure**:
```json
{
  "event_id": "550e8400-e29b-41d4-a716-446655440000",
  "correlation_id": "550e8400-e29b-41d4-a716-446655440001",
  "timestamp": "2024-01-15T10:30:00Z",
  "source_service": "product-catalog-service",
  "event_version": "1.0",
  "event_type": "product.created",
  "data": {
    "product_id": "prod_123456789",
    "name": "Premium Coffee Beans",
    "description": "High-quality arabica coffee beans from Guatemala",
    "category_id": "cat_beverages",
    "price": "24.99",
    "sku": "COFFEE-PREM-001",
    "is_active": true,
    "attributes": {
      "weight": "1kg",
      "origin": "Guatemala",
      "roast_level": "medium"
    }
  }
}
```

### ProductUpdatedEvent

**Purpose**: Published when product information is modified.

**When Published**: After successful product update and database persistence.

**Future Consumers**: Inventory Management Service (for SKU changes), Order Management Service (for active status changes), Search Service (for re-indexing).

**Event Structure**:
```json
{
  "event_id": "550e8400-e29b-41d4-a716-446655440002",
  "correlation_id": "550e8400-e29b-41d4-a716-446655440001",
  "timestamp": "2024-01-15T10:35:00Z", 
  "source_service": "product-catalog-service",
  "event_version": "1.0",
  "event_type": "product.updated",
  "data": {
    "product_id": "prod_123456789",
    "name": "Premium Coffee Beans",
    "description": "Premium arabica coffee beans from Guatemala - now organic certified",
    "category_id": "cat_beverages",
    "price": "24.99",
    "sku": "COFFEE-PREM-001",
    "is_active": true,
    "attributes": {
      "weight": "1kg",
      "origin": "Guatemala", 
      "roast_level": "medium",
      "organic": true
    },
    "changes": [
      {
        "field": "description",
        "old_value": "High-quality arabica coffee beans from Guatemala",
        "new_value": "Premium arabica coffee beans from Guatemala - now organic certified"
      },
      {
        "field": "attributes.organic", 
        "old_value": null,
        "new_value": true
      }
    ]
  }
}
```

### ProductDeletedEvent

**Purpose**: Published when a product is deleted from the catalog.

**When Published**: After successful product deletion from database.

**Future Consumers**: Inventory Management Service (for stock cleanup), Order Management Service (for order validation), Search Service (for de-indexing).

**Event Structure**:
```json
{
  "event_id": "550e8400-e29b-41d4-a716-446655440003",
  "correlation_id": "550e8400-e29b-41d4-a716-446655440001",
  "timestamp": "2024-01-15T10:40:00Z",
  "source_service": "product-catalog-service", 
  "event_version": "1.0",
  "event_type": "product.deleted",
  "data": {
    "product_id": "prod_123456789",
    "name": "Premium Coffee Beans",
    "reason": "discontinued"
  }
}
```

### PriceChangedEvent

**Purpose**: Published when a product's price is modified.

**When Published**: After successful price update in database.

**Future Consumers**: Order Management Service (for pricing validation), Analytics Service (for price tracking), Notification Service (for customer alerts).

**Event Structure**:
```json
{
  "event_id": "550e8400-e29b-41d4-a716-446655440004",
  "correlation_id": "550e8400-e29b-41d4-a716-446655440001", 
  "timestamp": "2024-01-15T10:45:00Z",
  "source_service": "product-catalog-service",
  "event_version": "1.0",
  "event_type": "product.price_changed",
  "data": {
    "product_id": "prod_123456789",
    "old_price": "24.99", 
    "new_price": "22.99",
    "effective_date": "2024-01-16T00:00:00Z"
  }
}
```

### ProductOutOfStockEvent

**Purpose**: Published when a product's inventory reaches zero.

**When Published**: Triggered by inventory service communication (Phase 2).

**Future Consumers**: Order Management Service (for order validation), Notification Service (for alerts), Analytics Service (for reporting).

**Event Structure**:
```json
{
  "event_id": "550e8400-e29b-41d4-a716-446655440005",
  "correlation_id": "550e8400-e29b-41d4-a716-446655440001",
  "timestamp": "2024-01-15T10:50:00Z",
  "source_service": "product-catalog-service",
  "event_version": "1.0", 
  "event_type": "product.out_of_stock",
  "data": {
    "product_id": "prod_123456789",
    "last_stock_level": 0,
    "location_id": "warehouse_main"
  }
}
```

## Inventory Management Service Events

The Inventory Management Service publishes events related to stock levels, reservations, and inventory operations.

### InventoryReservedEvent

**Purpose**: Published when inventory is reserved for an order.

**When Published**: After successful inventory reservation in database.

**Future Consumers**: Order Management Service (for order processing), Analytics Service (for demand tracking).

**Event Structure**:
```json
{
  "event_id": "550e8400-e29b-41d4-a716-446655440006", 
  "correlation_id": "order_550e8400-e29b-41d4-a716-446655440007",
  "timestamp": "2024-01-15T11:00:00Z",
  "source_service": "inventory-management-service",
  "version": "1.0",
  "event_type": "InventoryReserved",
  "data": {
    "reservation_id": "res_123456789",
    "product_id": "prod_123456789",
    "quantity": 2,
    "location_id": "warehouse_main",
    "order_id": "order_987654321",
    "expires_at": "2024-01-15T11:15:00Z"
  }
}
```

### InventoryReleasedEvent

**Purpose**: Published when a reserved inventory is released (order cancelled, expired reservation).

**When Published**: After successful inventory release in database.

**Future Consumers**: Order Management Service (for order status updates), Analytics Service (for cancellation tracking).

**Event Structure**:
```json
{
  "event_id": "550e8400-e29b-41d4-a716-446655440008",
  "correlation_id": "order_550e8400-e29b-41d4-a716-446655440007",
  "timestamp": "2024-01-15T11:10:00Z",
  "source_service": "inventory-management-service", 
  "version": "1.0",
  "event_type": "InventoryReleased",
  "data": {
    "reservation_id": "res_123456789",
    "product_id": "prod_123456789", 
    "quantity": 2,
    "location_id": "warehouse_main",
    "reason": "order_cancelled"
  }
}
```

### InventoryAdjustedEvent

**Purpose**: Published when inventory levels are manually adjusted.

**When Published**: After successful inventory adjustment in database.

**Future Consumers**: Analytics Service (for adjustment tracking), Audit Service (for inventory audit trail).

**Event Structure**:
```json
{
  "event_id": "550e8400-e29b-41d4-a716-446655440009",
  "correlation_id": "adjustment_550e8400-e29b-41d4-a716-446655440010",
  "timestamp": "2024-01-15T11:20:00Z",
  "source_service": "inventory-management-service",
  "version": "1.0", 
  "event_type": "InventoryAdjusted",
  "data": {
    "product_id": "prod_123456789",
    "location_id": "warehouse_main",
    "old_quantity": 100,
    "new_quantity": 95,
    "adjustment_type": "damage",
    "reason": "5 units damaged during inspection",
    "reference_number": "ADJ-2024-001"
  }
}
```

### LowStockAlertEvent

**Purpose**: Published when inventory falls below defined thresholds.

**When Published**: When stock level crosses threshold during any inventory operation.

**Future Consumers**: Notification Service (for alerts), Procurement Service (for reorder triggers), Analytics Service (for reporting).

**Event Structure**:
```json
{
  "event_id": "550e8400-e29b-41d4-a716-446655440011",
  "correlation_id": "monitoring_550e8400-e29b-41d4-a716-446655440012", 
  "timestamp": "2024-01-15T11:25:00Z",
  "source_service": "inventory-management-service",
  "version": "1.0",
  "event_type": "LowStockAlert",
  "data": {
    "product_id": "prod_123456789",
    "location_id": "warehouse_main",
    "current_quantity": 8,
    "threshold": 10,
    "alert_level": "warning"
  }
}
```

### InventoryUpdatedEvent

**Purpose**: Published for general inventory updates not covered by specific events.

**When Published**: After any significant inventory change (restocking, manual updates).

**Future Consumers**: Analytics Service (for general tracking), Product Catalog Service (for stock status updates).

**Event Structure**:
```json
{
  "event_id": "550e8400-e29b-41d4-a716-446655440013",
  "correlation_id": "update_550e8400-e29b-41d4-a716-446655440014",
  "timestamp": "2024-01-15T11:30:00Z", 
  "source_service": "inventory-management-service",
  "version": "1.0",
  "event_type": "InventoryUpdated",
  "data": {
    "product_id": "prod_123456789",
    "location_id": "warehouse_main",
    "quantity": 150,
    "reserved_quantity": 5,
    "available_quantity": 145,
    "update_type": "restock",
    "source_event": "purchase_order_received"
  }
}
```

## Order Management Service Events (Future)

The Order Management Service will publish events related to order lifecycle. These events are planned for implementation in Phase 2.

### OrderCreatedEvent

**Purpose**: Published when a new order is created.

**When Published**: After successful order creation and initial validation.

**Future Consumers**: Inventory Management Service (for reservation), Payment Service (for payment processing), Notification Service (for customer updates).

**Event Structure**:
```json
{
  "event_id": "550e8400-e29b-41d4-a716-446655440015",
  "correlation_id": "order_550e8400-e29b-41d4-a716-446655440016",
  "timestamp": "2024-01-15T12:00:00Z",
  "source_service": "order-management-service",
  "event_version": "1.0",
  "event_type": "order.created",
  "data": {
    "order_id": "order_987654321",
    "customer_id": "cust_123456789", 
    "total_amount": "47.98",
    "currency": "USD",
    "status": "pending",
    "items": [
      {
        "product_id": "prod_123456789",
        "quantity": 2,
        "unit_price": "22.99",
        "total_price": "45.98"
      }
    ],
    "shipping_address": {
      "street": "123 Main St",
      "city": "Anytown",
      "state": "CA",
      "zip": "12345",
      "country": "US"
    },
    "billing_address": {
      "street": "123 Main St", 
      "city": "Anytown",
      "state": "CA",
      "zip": "12345",
      "country": "US"
    }
  }
}
```

### OrderValidatedEvent

**Purpose**: Published when an order passes all validation checks.

**When Published**: After inventory verification, pricing validation, and customer verification.

**Future Consumers**: Payment Service (to proceed with payment), Fulfillment Service (for preparation), Notification Service (for updates).

**Event Structure**:
```json
{
  "event_id": "550e8400-e29b-41d4-a716-446655440017",
  "correlation_id": "order_550e8400-e29b-41d4-a716-446655440016",
  "timestamp": "2024-01-15T12:05:00Z",
  "source_service": "order-management-service",
  "event_version": "1.0",
  "event_type": "order.validated", 
  "data": {
    "order_id": "order_987654321",
    "validation_results": {
      "inventory_check": "passed",
      "pricing_check": "passed", 
      "customer_check": "passed"
    },
    "reservations": [
      {
        "product_id": "prod_123456789",
        "reservation_id": "res_123456789",
        "quantity": 2
      }
    ]
  }
}
```

### OrderCompletedEvent

**Purpose**: Published when an order is fully completed (paid and fulfilled).

**When Published**: After payment confirmation and fulfillment completion.

**Future Consumers**: Analytics Service (for sales tracking), Customer Service (for history), Inventory Service (for final stock adjustment).

**Event Structure**:
```json
{
  "event_id": "550e8400-e29b-41d4-a716-446655440018", 
  "correlation_id": "order_550e8400-e29b-41d4-a716-446655440016",
  "timestamp": "2024-01-15T12:45:00Z",
  "source_service": "order-management-service",
  "event_version": "1.0",
  "event_type": "order.completed",
  "data": {
    "order_id": "order_987654321",
    "completion_time": "2024-01-15T12:45:00Z",
    "payment_id": "pay_123456789",
    "fulfillment_id": "ful_123456789",
    "final_amount": "47.98"
  }
}
```

### OrderCancelledEvent

**Purpose**: Published when an order is cancelled.

**When Published**: After order cancellation processing and inventory release.

**Future Consumers**: Inventory Service (for reservation release), Payment Service (for refund processing), Notification Service (for customer updates).

**Event Structure**:
```json
{
  "event_id": "550e8400-e29b-41d4-a716-446655440019",
  "correlation_id": "order_550e8400-e29b-41d4-a716-446655440016",
  "timestamp": "2024-01-15T12:30:00Z",
  "source_service": "order-management-service",
  "event_version": "1.0", 
  "event_type": "order.cancelled",
  "data": {
    "order_id": "order_987654321",
    "cancellation_reason": "customer_request",
    "cancelled_by": "customer",
    "cancelled_at": "2024-01-15T12:30:00Z",
    "refund_required": true,
    "refund_amount": "47.98"
  }
}
```

### OrderRefundedEvent

**Purpose**: Published when an order refund is processed.

**When Published**: After successful refund processing.

**Future Consumers**: Analytics Service (for refund tracking), Customer Service (for customer history), Accounting Service (for financial records).

**Event Structure**:
```json
{
  "event_id": "550e8400-e29b-41d4-a716-446655440020",
  "correlation_id": "order_550e8400-e29b-41d4-a716-446655440016",
  "timestamp": "2024-01-15T12:50:00Z",
  "source_service": "order-management-service",
  "event_version": "1.0",
  "event_type": "order.refunded",
  "data": {
    "order_id": "order_987654321",
    "refund_id": "ref_123456789",
    "refund_amount": "47.98",
    "refund_reason": "order_cancelled", 
    "refund_method": "original_payment_method",
    "processed_at": "2024-01-15T12:50:00Z"
  }
}
```

## Event Handling Guidelines

### Service Implementation

Each service should:

1. **Event Publishing**:
   - Log events to structured logs (Phase 1)
   - Validate event data before publishing
   - Include correlation IDs for request tracing
   - Use consistent event naming conventions

2. **Event Consumption** (Phase 2):
   - Implement idempotent event handlers
   - Validate incoming event structure
   - Handle event ordering and duplicate detection
   - Implement proper error handling and retry logic

3. **Error Handling**:
   - Log failed event processing
   - Implement dead letter queues for failed events
   - Monitor event processing latency and failures
   - Provide manual intervention capabilities

### Event Ordering

- Events within the same correlation_id should be processed in timestamp order
- Services should handle out-of-order events gracefully
- Critical ordering dependencies should be documented in event contracts

### Event Schema Evolution

- New fields can be added without version changes (backward compatible)
- Existing fields cannot be removed or changed without major version increment
- Deprecated fields should be marked and maintained for one major version
- Include schema migration strategy in breaking changes

## Future Event Bus Implementation

**Phase 2 Implementation Notes**:

- **Technology**: Apache Kafka for high-throughput, reliable event streaming
- **Topics**: One topic per event type for better scaling and isolation
- **Partitioning**: Partition by correlation_id or entity_id for ordering
- **Retention**: Configure appropriate retention policies per event type
- **Monitoring**: Implement comprehensive monitoring for event lag, failures, and throughput

**Topic Naming Convention**:
```
{service-name}.{domain}.{action}.{version}
```

Examples:
- `product-catalog.product.created.v1`
- `inventory-management.inventory.reserved.v1`
- `order-management.order.completed.v1`

## Monitoring and Observability

### Key Metrics to Track

1. **Event Volume**: Events published/consumed per service
2. **Event Latency**: Time from event creation to consumption
3. **Event Failures**: Failed event processing and reasons
4. **Event Lag**: Delay in event processing per consumer
5. **Schema Validation**: Failed event validations

### Correlation Tracking

- Use correlation_id for tracing events across service boundaries
- Implement distributed tracing for complex event flows
- Maintain event lineage for debugging and audit purposes

---

*This document will be updated as services are implemented and event contracts evolve. All changes should be communicated to affected service teams and documented in the change log.*