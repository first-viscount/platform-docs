# Hybrid Development Strategy: k3s and Docker Compose

This document outlines when and how to use k3s versus Docker Compose for developing the First Viscount microservices platform.

## Overview

Based on practical experience and honest assessment, a hybrid approach using both k3s and Docker Compose provides the best developer experience while maintaining production parity.

### Key Principle
> "Use Docker Compose for speed, k3s for accuracy"

## When to Use Each Tool

### Use Docker Compose When:

1. **Single Service Development**
   - Working on one microservice in isolation
   - Rapid prototyping of new features
   - Unit testing with minimal dependencies
   - Database schema development

2. **Quick Debugging**
   - Need immediate feedback loops
   - Testing specific API endpoints
   - Debugging business logic
   - Local IDE debugging sessions

3. **Limited Resources**
   - Working on a machine with < 16GB RAM
   - Battery-conscious development on laptops
   - Quick demos or presentations

### Use k3s When:

1. **Integration Testing**
   - Testing service-to-service communication
   - Validating event flows through Kafka
   - Testing distributed transactions
   - Debugging network issues

2. **Production Parity Required**
   - Testing Kubernetes-specific features (probes, configmaps)
   - Validating resource limits and requests
   - Testing rolling deployments
   - Debugging pod lifecycle issues

3. **Full System Testing**
   - End-to-end workflow validation
   - Performance testing
   - Chaos engineering experiments
   - Pre-production validation

## Development Workflows

### Workflow 1: Single Service Development

```bash
# 1. Start only required infrastructure
docker-compose up -d postgres redis

# 2. Run your service locally with hot reload
cd services/product-catalog
go run cmd/main.go

# 3. Test with direct API calls
curl http://localhost:8082/api/v1/products
```

### Workflow 2: Service Integration Development

```bash
# 1. Deploy infrastructure to k3s
kubectl apply -f k8s/infrastructure/

# 2. Deploy dependent services
kubectl apply -f k8s/services/inventory/
kubectl apply -f k8s/services/order/

# 3. Run your service locally with k3s services
export INVENTORY_SERVICE_URL=http://localhost:30083
export ORDER_SERVICE_URL=http://localhost:30084
go run cmd/main.go
```

### Workflow 3: Full System Testing

```bash
# 1. Deploy everything to k3s
./scripts/k3s-deploy-all.sh

# 2. Run integration tests
go test ./tests/integration/... -tags=k3s

# 3. Monitor with k9s
k9s -n firstviscount-dev
```

## Configuration Management

### Environment-Aware Configuration

```go
// pkg/config/config.go
package config

import (
    "os"
)

type Environment string

const (
    EnvLocal  Environment = "local"
    EnvDocker Environment = "docker"
    EnvK3s    Environment = "k3s"
    EnvProd   Environment = "production"
)

func GetEnvironment() Environment {
    env := os.Getenv("ENVIRONMENT")
    switch env {
    case "docker":
        return EnvDocker
    case "k3s":
        return EnvK3s
    case "production":
        return EnvProd
    default:
        return EnvLocal
    }
}

func GetServiceURL(service string) string {
    env := GetEnvironment()
    
    switch env {
    case EnvLocal:
        return getLocalServiceURL(service)
    case EnvDocker:
        return getDockerServiceURL(service)
    case EnvK3s:
        return getK3sServiceURL(service)
    default:
        return getProductionServiceURL(service)
    }
}
```

### Unified Configuration Files

```yaml
# config/base.yaml - Shared configuration
app:
  name: ${SERVICE_NAME}
  version: ${VERSION}

logging:
  level: ${LOG_LEVEL:info}
  format: json

metrics:
  enabled: true
  port: 9090

---
# config/docker.yaml - Docker Compose overrides
database:
  host: postgres
  port: 5432

kafka:
  brokers:
    - kafka:9092

---
# config/k3s.yaml - k3s overrides  
database:
  host: postgres.infrastructure.svc.cluster.local
  port: 5432

kafka:
  brokers:
    - kafka.infrastructure.svc.cluster.local:9092
```

## Gradual Migration Path

### Phase 1: Docker Compose Start (Week 1)
```yaml
# docker-compose.yml - Start simple
version: '3.8'
services:
  postgres:
    image: postgres:15
    environment:
      POSTGRES_PASSWORD: dev
      
  product-catalog:
    build: ./services/product-catalog
    ports:
      - "8082:8082"
    environment:
      DATABASE_URL: postgres://postgres:dev@postgres:5432/products
```

### Phase 2: Add k3s Manifests (Week 2)
```yaml
# k8s/product-catalog/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: product-catalog
spec:
  replicas: 1
  template:
    spec:
      containers:
      - name: product-catalog
        image: localhost:30500/product-catalog:dev
        env:
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: db-credentials
              key: url
```

### Phase 3: Parallel Development (Week 3+)
- Maintain both configurations
- Use Docker Compose for development
- Use k3s for integration testing
- Gradually move more testing to k3s

## Common Patterns

### 1. Database Migrations

```bash
# Docker Compose approach
docker-compose run --rm product-catalog migrate up

# k3s approach
kubectl run migration --rm -it --restart=Never \
  --image=localhost:30500/product-catalog:dev \
  -- migrate up
```

### 2. Debugging Services

```bash
# Docker Compose - Direct access
docker-compose exec product-catalog sh
dlv attach $(pgrep product-catalog)

# k3s - Port forwarding required
kubectl port-forward pod/product-catalog-xxx 2345:2345
dlv connect localhost:2345
```

### 3. Log Viewing

```bash
# Docker Compose
docker-compose logs -f product-catalog

# k3s
kubectl logs -f deployment/product-catalog
# or better with stern
stern product-catalog
```

## Challenges and Solutions

### Challenge 1: Configuration Drift

**Problem**: Docker Compose and k3s configurations diverge over time

**Solution**: 
- Single source of truth: Generate both from templates
- Regular sync meetings to review differences
- Automated tests to verify parity

```bash
# scripts/sync-configs.sh
#!/bin/bash
# Generate Docker Compose from k8s manifests
kompose convert -f k8s/ -o docker-compose.generated.yml

# Compare with existing
diff docker-compose.yml docker-compose.generated.yml
```

### Challenge 2: Resource Constraints

**Problem**: k3s uses too much memory for daily development

**Solution**: Selective service deployment

```bash
# Start minimal k3s
k3s server --disable traefik --disable metrics-server

# Deploy only what you need
kubectl apply -f k8s/core-services.yaml  # Just databases
kubectl apply -f k8s/my-service.yaml     # Service under development
```

### Challenge 3: Network Complexity

**Problem**: Service discovery works differently

**Solution**: Abstraction layer

```go
// Use environment variables consistently
serviceURL := os.Getenv("ORDER_SERVICE_URL")
if serviceURL == "" {
    serviceURL = defaultURLForEnvironment()
}
```

## Best Practices

### 1. Start Simple
- Begin with Docker Compose
- Add k3s when you need it
- Don't over-engineer early

### 2. Document Environment Differences
```markdown
# Service README.md
## Environment Differences
- **Docker Compose**: Uses host networking, no SSL
- **k3s**: Cluster networking, mTLS enabled
- **Production**: External load balancer, full SSL
```

### 3. Automated Environment Setup
```bash
# Makefile
.PHONY: dev-docker dev-k3s

dev-docker:
	docker-compose up -d
	@echo "Services available at:"
	@echo "  Product Catalog: http://localhost:8082"
	@echo "  Order Service: http://localhost:8084"

dev-k3s:
	kubectl apply -f k8s/dev/
	@echo "Waiting for pods..."
	kubectl wait --for=condition=ready pod -l app -n firstviscount-dev
	@echo "Services available via kubectl port-forward"
```

### 4. Health Checks for Both Environments

```go
// Kubernetes-style health checks that work everywhere
func healthCheck() http.HandlerFunc {
    return func(w http.ResponseWriter, r *http.Request) {
        checks := []struct {
            name string
            fn   func() error
        }{
            {"database", checkDatabase},
            {"kafka", checkKafka},
            {"dependencies", checkDependencies},
        }
        
        failed := false
        results := make(map[string]string)
        
        for _, check := range checks {
            if err := check.fn(); err != nil {
                results[check.name] = "unhealthy: " + err.Error()
                failed = true
            } else {
                results[check.name] = "healthy"
            }
        }
        
        if failed {
            w.WriteHeader(http.StatusServiceUnavailable)
        }
        
        json.NewEncoder(w).Encode(results)
    }
}
```

## Decision Matrix

| Scenario | Docker Compose | k3s | Notes |
|----------|---------------|-----|-------|
| Learning the codebase | ✅ | ❌ | Keep it simple |
| Adding new API endpoint | ✅ | ❌ | Fast feedback |
| Testing Kafka integration | ❌ | ✅ | Network complexity |
| Debugging memory leaks | ❌ | ✅ | Real resource limits |
| Testing failover | ❌ | ✅ | Pod restarts |
| Daily development | ✅ | ⚠️ | Use k3s for integration |
| CI/CD pipeline | ⚠️ | ✅ | Match production |
| Demo to stakeholders | ✅ | ✅ | Depends on focus |

## Migration Checklist

When moving from Docker Compose to k3s:

- [ ] All environment variables documented
- [ ] Health checks implemented
- [ ] Graceful shutdown handled
- [ ] Resource limits defined
- [ ] Persistent volumes configured
- [ ] Service dependencies mapped
- [ ] Network policies defined (if needed)
- [ ] Secrets management in place
- [ ] Monitoring configured
- [ ] Logging aggregation working

## Conclusion

The hybrid approach acknowledges that different development tasks have different requirements. By using the right tool for each job, developers can maintain high productivity while ensuring production readiness.

Remember: **Start with Docker Compose, graduate to k3s, deploy to production Kubernetes.**

---
*Last Updated: January 2025*