# Architecture Documentation

This section contains all architectural documentation for the First Viscount e-commerce platform.

## 📑 Contents

### Core Architecture
- **[System Overview](./system-overview.md)** - High-level architecture and design principles
- **[Microservices Design](./microservices-design.md)** - Detailed microservices architecture and patterns

### Technical Architecture
- **[Messaging Architecture](./messaging-architecture.md)** - Apache Kafka event-driven design
- **[Data Architecture](./data-architecture.md)** - Database design and data management patterns
- **[Security Architecture](./security-architecture.md)** - Authentication, authorization, and security patterns

### Service Specifications
Detailed specifications for each microservice:
- [Platform Coordination Service](./service-specifications/platform-coordination-service.md)
- [Product Catalog Service](./service-specifications/product-catalog-service.md)
- [Inventory Management Service](./service-specifications/inventory-service.md)
- [Order Management Service](./service-specifications/order-service.md)
- [Delivery Management Service](./service-specifications/delivery-service.md)
- [Notifications Service](./service-specifications/notification-service.md)

## 🎯 Key Architectural Principles

1. **Service Autonomy** - Each service owns its data and business logic
2. **Event-Driven** - Asynchronous communication via Kafka
3. **API-First** - All services expose RESTful APIs
4. **Cloud-Native** - Designed for containerized deployment
5. **Observable** - Built-in monitoring and tracing

## 🔗 Quick Links

- [Implementation Phases](../02-implementation/implementation-phases.md)
- [API Reference](../06-api-reference/)
- [Architecture Decisions](../07-decisions/)