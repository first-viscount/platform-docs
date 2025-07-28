# Service Specifications

This directory contains detailed specifications for each microservice in the First Viscount platform.

## Services Overview

| Service | Port | Purpose | Database |
|---------|------|---------|----------|
| [Platform Coordination](./platform-coordination-service.md) | 8081 | Workflow orchestration, saga management | PostgreSQL |
| [Product Catalog](./product-catalog-service.md) | 8082 | Product management, search, pricing | PostgreSQL + Redis |
| [Inventory Management](./inventory-service.md) | 8083 | Stock tracking, reservations | PostgreSQL |
| [Order Management](./order-service.md) | 8084 | Order lifecycle, payment processing | PostgreSQL |
| [Delivery Management](./delivery-service.md) | 8085 | Shipping, tracking, carrier integration | PostgreSQL |
| [Notifications](./notification-service.md) | 8086 | Email/SMS notifications, templates | MongoDB |

## Service Documentation Structure

Each service specification includes:

1. **Overview** - Service purpose and responsibilities
2. **API Specification** - RESTful endpoints and contracts
3. **Event Contracts** - Kafka events published and consumed
4. **Data Model** - Database schema and entities
5. **Configuration** - Environment variables and settings
6. **Dependencies** - External services and libraries
7. **Testing** - Testing approach and strategies
8. **Deployment** - Deployment configuration and requirements
9. **Monitoring** - Metrics, health checks, and alerts

## Communication Matrix

| Service | Calls | Called By | Events Published | Events Consumed |
|---------|-------|-----------|------------------|-----------------|
| Platform Coordination | All services | API Gateway | WorkflowStarted, SagaCompleted | All events |
| Product Catalog | - | API Gateway, Order Service | ProductUpdated, PriceChanged | - |
| Inventory | - | Platform Coordination | InventoryReserved, StockUpdated | OrderCreated |
| Order Management | Product Catalog | API Gateway | OrderCreated, OrderCompleted | PaymentProcessed |
| Delivery | - | Platform Coordination | DeliveryScheduled, Delivered | OrderPaid |
| Notifications | - | Platform Coordination | NotificationSent | All business events |

## Service Development Guidelines

1. Each service must be independently deployable
2. Services communicate asynchronously via Kafka when possible
3. Synchronous calls should be resilient (circuit breakers, retries)
4. Each service owns its data - no shared databases
5. Services should be stateless for horizontal scaling
6. All services must implement health checks and metrics endpoints

## Quick Links

- [Microservices Design](../microservices-design.md)
- [Messaging Architecture](../messaging-architecture.md)
- [API Design Patterns](../../02-implementation/api-design-patterns.md)