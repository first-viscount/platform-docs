# First Viscount Platform Documentation

## 🎯 Purpose

This documentation guides the development of a microservices-based e-commerce platform designed for learning distributed systems patterns while building production-grade software.

## 📚 Documentation Structure

### Planning & Strategy
- **[Charter](./charter.md)** - Project vision, goals, and commitment
- **[Constraints](./constraints.md)** - Technical and business boundaries
- **[Decisions](./decisions.md)** - Architecture decisions and rationale

### Goals
- **[Learning Goals](./goals/learning-goals.md)** - Distributed systems mastery objectives
- **[Business Goals](./goals/business-goals.md)** - Platform capabilities and metrics

### Architecture
- **[Services](./architecture/services.md)** - Microservice specifications and boundaries
- **[Messaging](./architecture/messaging.md)** - Event-driven architecture with Redpanda
- **[Service Mesh](./architecture/service-mesh.md)** - Traefik configuration and patterns

### Planning
- **[Sprint Plan](./planning/sprint-plan.md)** - 20-week development roadmap
- **[Dependencies](./planning/dependencies.md)** - Service interaction matrix
- **[Risk Register](./planning/risk-register.md)** - Identified risks and mitigations

### Experiments
- **[Chaos Testing](./experiments/chaos-testing.md)** - Breaking things to learn
- **[Migration Paths](./experiments/migration-paths.md)** - Future technology transitions

### Archive
- **[Spring Boot Version](./archive/spring-boot-version/)** - Original Java-based documentation

## 🚀 Quick Start

### For New Developers
1. Read the [Charter](./charter.md) to understand project vision
2. Review [Services](./architecture/services.md) for system overview
3. Check [Sprint Plan](./planning/sprint-plan.md) for current phase
4. Set up development environment (see below)

### For Architecture Review
1. Start with [Decisions](./decisions.md) for technology choices
2. Review [Architecture](./architecture/) for system design
3. Check [Risk Register](./planning/risk-register.md) for concerns

## 🛠 Technology Stack

- **Language**: Python 3.11+ with FastAPI
- **Message Broker**: Redpanda (Kafka-compatible)
- **Databases**: PostgreSQL (one per service)
- **Service Mesh**: Traefik
- **Containers**: Docker & Docker Compose
- **Orchestration**: K3s (future)

## 📋 Development Approach

1. **One Service at a Time** - Complete each service before starting the next
2. **Multi-Repository** - Each service in its own repository
3. **Event-Driven** - Services communicate through events
4. **API-First** - Define contracts before implementation
5. **Test Everything** - 80%+ coverage for business logic

## 🔄 Current Status

### Phase 1 Services (In Planning)
- [ ] Platform Coordination Service
- [ ] Product Catalog Service
- [ ] Inventory Management Service
- [ ] Order Management Service

### Phase 2 Services (Future)
- [ ] Delivery Management Service
- [ ] Notifications Service

## 🏗 Repository Structure

```
github.com/first-viscount/
├── platform-docs                    # This repository
├── platform-coordination-service    # Service discovery & health
├── product-catalog-service         # Product management
├── inventory-service              # Stock tracking
├── order-service                  # Order processing
└── shared-contracts               # API & event schemas
```

## 🔧 Local Development

### Prerequisites
- Docker Desktop
- Python 3.11+
- 16GB RAM minimum
- Git

### Quick Start
```bash
# Clone platform docs
git clone https://github.com/first-viscount/platform-docs.git

# Start infrastructure
docker-compose up -d redpanda postgres traefik

# Clone and run first service
git clone https://github.com/first-viscount/platform-coordination-service.git
cd platform-coordination-service
docker-compose up
```

## 📈 Learning Progress

Track your learning journey:
- [ ] Set up development environment
- [ ] Deploy first service
- [ ] Implement service discovery
- [ ] Publish first event
- [ ] Handle first distributed transaction
- [ ] Survive first chaos test
- [ ] Debug distributed issue
- [ ] Deploy to Kubernetes

## 🤝 Contributing

1. Follow [Sprint Plan](./planning/sprint-plan.md) for current priorities
2. Check [Dependencies](./planning/dependencies.md) before making changes
3. Update documentation as you build
4. Add chaos tests for new features

## ⚠️ Important Notes

- This is a learning project that intentionally chooses complexity
- We're building microservices to experience the challenges
- Documentation reflects planning, not existing implementation
- Expect 5-10x more complexity than a monolith

## 📞 Contact
- Email: dwdrake90@gmail.com