# Platform Constraints

## Technical Constraints

### Hard Requirements
- **Python 3.11+** - Latest stable Python for modern async features
- **PostgreSQL 15+** - One database instance per service
- **Docker 24+** - Container runtime requirement
- **Linux/macOS** - Development environment (Windows via WSL2)
- **16GB RAM minimum** - For running all services locally
- **Multi-repo structure** - Each service in separate repository

### Architectural Constraints
- **No shared databases** - Each service owns its data completely
- **Event-driven communication** - No synchronous service-to-service calls
- **API versioning required** - All endpoints must include version
- **Backward compatibility** - No breaking changes without migration path
- **Stateless services** - All state in databases or event streams

### Security Constraints
- **JWT tokens required** - All API calls must be authenticated
- **Service-to-service auth** - Mutual TLS or service tokens
- **No secrets in code** - Environment variables or secret management
- **HTTPS only** - Even in development environment
- **Rate limiting** - All public endpoints must have limits

## Business Constraints

### Performance Requirements
- **Response time**: < 500ms for 95th percentile
- **Throughput**: Support 1000 concurrent users minimum
- **Availability**: 99.9% uptime for critical services
- **Data consistency**: Eventual consistency acceptable (< 5 seconds)

### Scalability Requirements
- **Horizontal scaling** - All services must scale horizontally
- **No single points of failure** - Every component redundant
- **Database connection pooling** - Maximum 100 connections per service
- **Event stream partitioning** - Support for parallel processing

### Operational Constraints
- **Logging**: Structured JSON logs only
- **Monitoring**: Prometheus metrics required
- **Tracing**: OpenTelemetry support mandatory
- **Health checks**: Liveness and readiness probes
- **Graceful shutdown**: 30-second shutdown window

## Development Constraints

### Code Quality
- **Test coverage**: Minimum 80% for business logic
- **Type hints**: Required for all functions
- **Linting**: Black + Ruff must pass
- **Documentation**: OpenAPI specs for all endpoints
- **Code review**: All changes require approval

### Repository Standards
- **Branching**: GitFlow with feature branches
- **Commit messages**: Conventional commits format
- **CI/CD**: All tests must pass before merge
- **Dependencies**: Lock files required
- **Versioning**: Semantic versioning for services

### Multi-Repo Coordination
- **API contracts**: Must be versioned and published
- **Breaking changes**: Require deprecation period
- **Dependency updates**: Automated with Dependabot
- **Cross-repo testing**: Contract tests required
- **Release coordination**: Backward compatibility mandatory

## Resource Constraints

### Development Environment
- **Local resources**: Must run on 16GB RAM machine
- **Container limits**: 512MB RAM per service container
- **Database size**: 10GB maximum per service
- **Network bandwidth**: Assume residential internet
- **Storage**: 100GB available disk space

### Production Estimates
- **Cloud budget**: Assume $200/month limit
- **Compute**: t3.small instances or equivalent
- **Storage**: 100GB total across all services
- **Network**: 1TB transfer per month
- **Monitoring**: Basic tier only

## Learning Constraints

### Complexity Requirements
- **Must implement**: Distributed transactions (Saga pattern)
- **Must experience**: Network partitions and failures
- **Must handle**: Data consistency challenges
- **Must debug**: Distributed system issues
- **Must measure**: Real performance metrics

### Anti-Patterns to Avoid
- **No distributed monolith** - Services must be truly independent
- **No chatty services** - Batch operations where possible
- **No god services** - Keep services focused
- **No shared libraries** - Only shared contracts
- **No synchronous chains** - Maximum 2-hop synchronous calls

## Timeline Constraints

### Delivery Phases
- **4 weeks per service** - Complete implementation cycle
- **1 week integration** - Between each service
- **No shortcuts** - Full implementation required
- **Sequential delivery** - One service at a time
- **Documentation included** - Part of definition of done

### Learning Checkpoints
- **Weekly retrospectives** - Document challenges faced
- **Pattern implementation** - One new pattern per service
- **Failure scenarios** - Intentional chaos testing
- **Performance testing** - Before moving to next service
- **Architecture review** - After each service completion

---
*These constraints are designed to create realistic challenges while maintaining feasibility for a learning project.*