# Architecture Decisions Log

For current technology versions and compatibility information, see [technology-stack.md](architecture/technology-stack.md).

## Decision Template
```
Decision: [Title]
Date: [YYYY-MM-DD]
Status: [Proposed | Accepted | Superseded]
Context: [Why this decision needed to be made]
Decision: [What we decided]
Consequences: [What happens as a result]
```

---

## ADR-001: Python FastAPI over Java Spring Boot

**Date**: 2025-07-28  
**Status**: Accepted

### Context
Need to choose primary development language and framework for microservices. Considering learning goals, development velocity, and industry relevance.

### Decision
Use Python with FastAPI framework for all microservices.

### Rationale
- **Development Velocity**: 2-3x faster development than Spring Boot
- **Modern Async**: Built on ASGI/asyncio from ground up
- **Type Safety**: Pydantic provides excellent runtime validation
- **Container Size**: 50-100MB images vs 200-300MB for Spring Boot
- **Learning Curve**: Productive in days vs weeks
- **Performance**: Comparable to Spring Boot for I/O bound operations

### Consequences
- ✅ Faster iteration and learning cycles
- ✅ Modern Python ecosystem (ML, data science integration potential)
- ✅ Excellent documentation and community
- ❌ Fewer enterprise-specific libraries than Java
- ❌ Less mature distributed tracing ecosystem
- ❌ Team hiring might be more challenging in enterprise settings

---

## ADR-002: Redpanda over Apache Kafka

**Date**: 2025-07-28  
**Status**: Accepted

### Context
Need event streaming platform for inter-service communication. Want Kafka-compatible learning but with operational simplicity.

### Decision
Use Redpanda as the event streaming platform.

### Rationale
- **Kafka API Compatible**: Skills directly transferable
- **No Zookeeper**: Simpler architecture, fewer components
- **Performance**: Often faster than Kafka for our scale
- **Resource Usage**: 50% less memory and CPU
- **Developer Experience**: Single binary, easier configuration

### Consequences
- ✅ Authentic Kafka patterns and APIs
- ✅ Lower operational overhead
- ✅ Faster local development setup
- ✅ C++ implementation means better performance
- ❌ Smaller community than Kafka
- ❌ Fewer third-party integrations
- ❌ Less battle-tested at massive scale

### Migration Path
If we need to migrate to Kafka:
1. API compatibility means code changes minimal
2. Consumer group concepts identical
3. Partition strategies transfer directly
4. Only operational changes needed

---

## ADR-003: PostgreSQL with Database-per-Service

**Date**: 2025-07-28  
**Status**: Accepted

### Context
Need to decide on database strategy that enforces microservice boundaries while providing ACID guarantees.

### Decision
Each service gets its own PostgreSQL instance. No shared schemas, no cross-database joins.

### Rationale
- **Service Independence**: True microservice isolation
- **ACID Guarantees**: Critical for e-commerce
- **JSON Support**: Flexibility for varying schemas
- **Maturity**: Battle-tested, excellent tooling
- **Performance**: Excellent for our scale

### Consequences
- ✅ Enforced service boundaries
- ✅ Independent scaling and optimization
- ✅ Technology flexibility per service
- ❌ No cross-service joins (must use APIs)
- ❌ Data duplication across services
- ❌ Complex distributed transactions

---

## ADR-004: Multi-Repository Structure

**Date**: 2025-07-28  
**Status**: Accepted

### Context
Choosing between monorepo and multi-repo for learning maximum microservice complexity.

### Decision
Use separate repository for each microservice.

### Rationale
- **Learning Goal**: Experience real coordination pain
- **True Independence**: Enforced service boundaries
- **Real-World Practice**: Most enterprises use multi-repo
- **Team Autonomy**: Services truly independent

### Consequences
- ✅ Enforced service independence
- ✅ Clear ownership boundaries
- ✅ Independent deployment pipelines
- ❌ Cross-service changes require coordination
- ❌ Dependency management complexity
- ❌ Need robust contract testing

---

## ADR-005: Traefik as Service Mesh

**Date**: 2025-07-28  
**Status**: Accepted

### Context
Need service mesh capabilities without overwhelming complexity of Istio.

### Decision
Use Traefik as our service mesh and API gateway.

### Rationale
- **Simplicity**: Easier than Istio/Linkerd
- **Features**: Sufficient for learning (routing, load balancing, circuit breaking)
- **Performance**: Lower overhead than Istio
- **Kubernetes Native**: Excellent K8s integration
- **Learning Curve**: Can be productive quickly

### Consequences
- ✅ Service discovery out of the box
- ✅ Automatic SSL/TLS
- ✅ Built-in monitoring
- ✅ Good documentation
- ❌ Fewer advanced features than Istio
- ❌ Less enterprise adoption than Istio
- ❌ Limited service-to-service features

---

## ADR-006: JWT + OAuth2 for Authentication

**Date**: 2025-07-28  
**Status**: Accepted

### Context
Need authentication strategy that supports both user-facing and service-to-service auth.

### Decision
- User Authentication: JWT tokens with OAuth2 flow
- Service-to-Service: Service accounts with JWT
- Consider mTLS for additional security later

### Rationale
- **Industry Standard**: Widely understood patterns
- **Stateless**: Perfect for microservices
- **Flexible**: Supports various auth flows
- **Libraries**: Excellent FastAPI integration

### Consequences
- ✅ Stateless authentication
- ✅ Standard implementation patterns
- ✅ Good library support
- ❌ Token management complexity
- ❌ Need centralized auth service
- ❌ Revocation challenges

---

## ADR-007: Docker Compose to K3s Migration Path

**Date**: 2025-07-28  
**Status**: Accepted

### Context
Need to balance initial development simplicity with production-like environment learning.

### Decision
Start with Docker Compose, migrate to K3s after 2-3 services are stable.

### Rationale
- **Progressive Complexity**: Learn basics first
- **Fast Iteration**: Docker Compose quicker for development
- **Real K8s**: K3s provides authentic Kubernetes experience
- **Resource Efficient**: K3s lighter than full K8s

### Consequences
- ✅ Gentle learning curve
- ✅ Fast initial development
- ✅ Real Kubernetes patterns later
- ❌ Need to maintain two deployment methods
- ❌ Migration effort required
- ❌ Some rework of configurations

---

## Future Decisions to Document

- [ ] API versioning strategy (URL vs Header)
- [ ] Event schema evolution approach
- [ ] Monitoring and observability stack
- [ ] CI/CD platform choice
- [ ] Secret management approach
- [ ] Testing strategy (unit/integration/e2e split)