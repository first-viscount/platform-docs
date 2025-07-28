# Service Dependencies Matrix

## Overview

This document maps all dependencies between services, identifying potential bottlenecks and coordination requirements. Dependencies are categorized as:
- **Runtime**: Service requires another to function
- **Data**: Service needs data from another
- **Event**: Service reacts to events from another
- **Optional**: Service can function without, but degraded

## Dependency Matrix

| Service | Depends On | Dependency Type | Description | Can Start Without? |
|---------|------------|-----------------|-------------|-------------------|
| **Platform Coordination** | None | - | Foundation service | ✅ Yes |
| **Product Catalog** | Platform Coordination | Runtime | Service discovery | ⚠️ Degraded |
| | Inventory | Event | Stock status updates | ✅ Yes |
| **Inventory Management** | Platform Coordination | Runtime | Service discovery | ⚠️ Degraded |
| | Order Management | Event | Order events | ✅ Yes |
| **Order Management** | Platform Coordination | Runtime | Service discovery | ⚠️ Degraded |
| | Product Catalog | Data | Product validation | ❌ No |
| | Inventory | Data | Stock checking | ❌ No |
| **Delivery Management** | Platform Coordination | Runtime | Service discovery | ⚠️ Degraded |
| | Order Management | Event | Order completion | ✅ Yes |
| **Notifications** | Platform Coordination | Runtime | Service discovery | ⚠️ Degraded |
| | All Services | Event | Status updates | ✅ Yes |

## Service Startup Order

### Minimal System (Phase 1)
```
1. Platform Coordination (no dependencies)
2. Product Catalog (can start degraded)
3. Inventory Management (can start degraded)
4. Order Management (requires 2 & 3)
```

### Full System (Phase 2)
```
1. Platform Coordination
2. Product Catalog + Inventory (parallel)
3. Order Management
4. Delivery + Notifications (parallel)
```

## API Dependencies

### Product Catalog Service
**Exposes:**
- `GET /products` - No dependencies
- `GET /products/{id}` - No dependencies
- `POST /products` - No dependencies

**Consumes:**
- None (during startup)
- `InventoryUpdated` events (runtime)

### Inventory Management Service
**Exposes:**
- `GET /inventory/{product_id}` - No dependencies
- `POST /inventory/reserve` - No dependencies
- `POST /inventory/adjust` - No dependencies

**Consumes:**
- `OrderCreated` - From Order Service
- `OrderCancelled` - From Order Service

### Order Management Service
**Exposes:**
- `POST /orders` - Requires Product & Inventory
- `GET /orders/{id}` - No dependencies
- `POST /orders/{id}/cancel` - No dependencies

**Depends on:**
- `GET /products/{id}` - Product validation
- `POST /inventory/reserve` - Stock reservation
- `POST /inventory/release` - On cancellation

## Event Dependencies

### Event Flow Diagram
```
OrderCreated (Order Service)
    → Inventory Service (Reserve stock)
    → Notification Service (Send confirmation)
    
InventoryReserved (Inventory Service)
    → Order Service (Update status)
    
ProductUpdated (Product Service)
    → Inventory Service (Update thresholds)
    → Order Service (Cache invalidation)
```

### Critical Event Chains
1. **Order Fulfillment**
   - OrderCreated → InventoryReserved → PaymentProcessed → OrderConfirmed
   - Failure at any step triggers compensation

2. **Stock Management**
   - InventoryAdjusted → ProductUpdated → CatalogRefreshed
   - Must maintain consistency across services

## Data Dependencies

### Shared Domain Concepts
| Concept | Owner | Consumers | Sync Method |
|---------|-------|-----------|-------------|
| Product ID | Product Catalog | All services | API lookup |
| Customer ID | Future Auth Service | Order, Notifications | JWT token |
| Order ID | Order Management | Inventory, Delivery | Events |
| SKU | Product Catalog | Inventory, Orders | API + Cache |

### Data Consistency Requirements
- **Product Data**: Eventually consistent (5 second window)
- **Inventory Levels**: Strongly consistent (immediate)
- **Order Status**: Eventually consistent (1 second)
- **Prices**: Eventually consistent (cache TTL: 60s)

## Configuration Dependencies

### Environment Variables
Each service requires:
```bash
# Platform discovery
PLATFORM_COORDINATION_URL=http://platform:8081

# Message broker
REDPANDA_BROKER=redpanda:9092

# Database (unique per service)
DATABASE_URL=postgresql://user:pass@postgres:5432/service_db

# Tracing
JAEGER_ENDPOINT=http://jaeger:14268
```

### Service Discovery
Services register on startup:
```python
# Required for all services except Platform Coordination
async def register_service(name: str, port: int):
    await platform_client.register({
        "name": name,
        "port": port,
        "health": "/health"
    })
```

## Failure Scenarios

### Platform Coordination Failure
- **Impact**: No new service registration
- **Mitigation**: Services use cached discovery data
- **Recovery**: Auto-reconnect with exponential backoff

### Product Catalog Failure
- **Impact**: Orders cannot be created
- **Mitigation**: Order service caches product data
- **Recovery**: Queue orders for processing

### Inventory Service Failure
- **Impact**: Orders cannot reserve stock
- **Mitigation**: Pessimistic inventory assumptions
- **Recovery**: Reconciliation on service return

### Message Broker Failure
- **Impact**: No event propagation
- **Mitigation**: Local event queue
- **Recovery**: Replay events from last checkpoint

## Testing Dependencies

### Integration Test Order
1. Start Platform Coordination
2. Start Redpanda
3. Start target service
4. Start mock dependencies
5. Run test suite

### Contract Testing
Each service must:
- Publish API contracts
- Verify consumed API contracts
- Test event schema compatibility
- Validate error responses

## Deployment Dependencies

### Rolling Deployment Order
1. Platform Coordination (if changed)
2. Services with no downstream dependencies
3. Services with backward-compatible changes
4. Services with breaking changes (with feature flags)

### Database Migration Order
1. Backward-compatible schema changes
2. Service deployment
3. Data migration (if needed)
4. Cleanup old schema (after verification)

## Monitoring Dependencies

### Health Check Chain
```
Load Balancer → Traefik → Service → Dependencies
                                       ↓
                              Database + Cache + Platform
```

### Alert Dependencies
- Service down → Check Platform Coordination first
- Order failures → Check Product + Inventory
- Event lag → Check Redpanda + consumers

---

*Understanding these dependencies is crucial for debugging, deployment, and maintaining system reliability.*