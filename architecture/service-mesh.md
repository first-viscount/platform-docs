# Service Mesh with Traefik

## Overview

Traefik serves as our service mesh, providing intelligent routing, load balancing, and observability for our microservices architecture. This document outlines our Traefik configuration and service mesh patterns.

## Why Traefik?

- **Simplicity**: Easier to operate than Istio/Linkerd
- **Native Integration**: Excellent Docker and Kubernetes support
- **Dynamic Configuration**: Auto-discovery of services
- **Performance**: Lower overhead than traditional service meshes
- **Developer Friendly**: Great dashboard and debugging tools

## Architecture

```
Internet → Traefik Edge Router → Service Discovery → Microservices
                ↓
          Load Balancer
                ↓
          Middleware
          - Auth
          - Rate Limiting
          - Circuit Breaker
```

## Docker Compose Configuration

```yaml
# docker-compose.yml
version: '3.8'

services:
  traefik:
    image: traefik:v3.0
    command:
      # API and Dashboard
      - "--api.dashboard=true"
      - "--api.insecure=true"
      
      # Docker provider
      - "--providers.docker=true"
      - "--providers.docker.exposedbydefault=false"
      
      # Entrypoints
      - "--entrypoints.web.address=:80"
      - "--entrypoints.websecure.address=:443"
      
      # Logs
      - "--accesslog=true"
      - "--log.level=INFO"
      
      # Metrics
      - "--metrics.prometheus=true"
      - "--metrics.prometheus.buckets=0.1,0.3,1.2,5.0"
    ports:
      - "80:80"
      - "443:443"
      - "8080:8080"  # Dashboard
    volumes:
      - "/var/run/docker.sock:/var/run/docker.sock:ro"
    networks:
      - platform-network

  # Example service configuration
  product-service:
    build: ./product-service
    networks:
      - platform-network
    labels:
      # Enable Traefik
      - "traefik.enable=true"
      
      # Router configuration
      - "traefik.http.routers.product-service.rule=Host(`api.firstviscount.local`) && PathPrefix(`/api/v1/products`)"
      - "traefik.http.routers.product-service.entrypoints=web"
      
      # Service configuration
      - "traefik.http.services.product-service.loadbalancer.server.port=8082"
      
      # Middleware
      - "traefik.http.routers.product-service.middlewares=auth,rate-limit,circuit-breaker"

networks:
  platform-network:
    driver: bridge
```

## Service Configuration

### Service Registration
Each service self-registers with Traefik using Docker labels:

```yaml
# Service docker-compose configuration
labels:
  # Basic routing
  - "traefik.enable=true"
  - "traefik.http.routers.${SERVICE_NAME}.rule=PathPrefix(`/api/v1/${SERVICE_PATH}`)"
  
  # Load balancing
  - "traefik.http.services.${SERVICE_NAME}.loadbalancer.server.port=${SERVICE_PORT}"
  - "traefik.http.services.${SERVICE_NAME}.loadbalancer.sticky.cookie=true"
  
  # Health check
  - "traefik.http.services.${SERVICE_NAME}.loadbalancer.healthcheck.path=/health"
  - "traefik.http.services.${SERVICE_NAME}.loadbalancer.healthcheck.interval=10s"
```

### Middleware Configuration

#### Authentication Middleware
```yaml
# JWT Authentication
labels:
  - "traefik.http.middlewares.auth.forwardauth.address=http://auth-service:8080/verify"
  - "traefik.http.middlewares.auth.forwardauth.authResponseHeaders=X-User-Id,X-User-Role"
```

#### Rate Limiting
```yaml
# Rate limit configuration
labels:
  - "traefik.http.middlewares.rate-limit.ratelimit.average=100"
  - "traefik.http.middlewares.rate-limit.ratelimit.burst=50"
  - "traefik.http.middlewares.rate-limit.ratelimit.period=1m"
```

#### Circuit Breaker
```yaml
# Circuit breaker configuration
labels:
  - "traefik.http.middlewares.circuit-breaker.circuitbreaker.expression=ResponseCodeRatio(500, 600, 0, 600) > 0.30"
  - "traefik.http.middlewares.circuit-breaker.circuitbreaker.checkperiod=10s"
  - "traefik.http.middlewares.circuit-breaker.circuitbreaker.fallbackduration=10s"
```

## Routing Patterns

### API Gateway Pattern
All external requests route through Traefik:
```
External Client → Traefik → Service
```

### Service-to-Service Communication
Internal services communicate directly:
```
Service A → Service B (via internal network)
```

### Path-Based Routing
```yaml
# Routes configuration
/api/v1/products → product-service
/api/v1/orders → order-service
/api/v1/inventory → inventory-service
/api/v1/platform → platform-coordination-service
```

## Load Balancing

### Strategies Available
- **Round Robin** (default): Distribute evenly
- **Weighted Round Robin**: Assign weights to instances
- **Sticky Sessions**: Route same client to same instance
- **Least Connections**: Route to instance with fewest connections

### Configuration Example
```yaml
# Weighted load balancing
labels:
  - "traefik.http.services.product-service.loadbalancer.servers.server1.weight=10"
  - "traefik.http.services.product-service.loadbalancer.servers.server2.weight=5"
```

## Security Configuration

### TLS/SSL
```yaml
# HTTPS configuration
labels:
  - "traefik.http.routers.product-service.entrypoints=websecure"
  - "traefik.http.routers.product-service.tls=true"
  - "traefik.http.routers.product-service.tls.certresolver=letsencrypt"
```

### Headers Middleware
```yaml
# Security headers
labels:
  - "traefik.http.middlewares.security-headers.headers.frameDeny=true"
  - "traefik.http.middlewares.security-headers.headers.contentTypeNosniff=true"
  - "traefik.http.middlewares.security-headers.headers.browserXssFilter=true"
```

## Observability

### Metrics (Prometheus)
```yaml
# Prometheus metrics endpoint
http://traefik:8080/metrics

# Key metrics:
- traefik_service_requests_total
- traefik_service_request_duration_seconds
- traefik_service_open_connections
- traefik_entrypoint_requests_total
```

### Logging
```yaml
# Access logs format
{
  "time": "2025-01-28T10:15:30Z",
  "status": 200,
  "method": "GET",
  "path": "/api/v1/products",
  "duration": "25ms",
  "service": "product-service",
  "client": "192.168.1.100"
}
```

### Distributed Tracing
```yaml
# Jaeger integration
command:
  - "--tracing.jaeger=true"
  - "--tracing.jaeger.localAgentHostPort=jaeger:6831"
  - "--tracing.jaeger.samplingType=probabilistic"
  - "--tracing.jaeger.samplingParam=0.1"
```

## Service Discovery

### Docker Provider
Automatic discovery of Docker containers:
```yaml
providers:
  docker:
    endpoint: "unix:///var/run/docker.sock"
    watch: true
    exposedByDefault: false
```

### Kubernetes Provider (Future)
```yaml
providers:
  kubernetes:
    endpoint: "https://kubernetes.default"
    token: "${KUBERNETES_TOKEN}"
    certAuthFilePath: "/var/run/secrets/kubernetes.io/serviceaccount/ca.crt"
```

## Health Checks

### Service Health
```python
# FastAPI health endpoint
@app.get("/health")
async def health_check():
    return {
        "status": "healthy",
        "service": "product-service",
        "version": "1.0.0",
        "timestamp": datetime.utcnow().isoformat()
    }
```

### Traefik Configuration
```yaml
healthcheck:
  path: /health
  interval: 10s
  timeout: 3s
  retries: 3
```

## Best Practices

### Do's
- ✅ Use labels for dynamic configuration
- ✅ Implement health checks for all services
- ✅ Use middleware for cross-cutting concerns
- ✅ Monitor Traefik metrics
- ✅ Use circuit breakers for resilience

### Don'ts
- ❌ Don't expose services directly
- ❌ Don't skip health checks
- ❌ Don't hardcode service addresses
- ❌ Don't ignore rate limiting
- ❌ Don't bypass Traefik for external access

## Migration to Kubernetes

When moving to K3s/K8s:

### Traefik Ingress Controller
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: product-service
  annotations:
    traefik.ingress.kubernetes.io/router.middlewares: auth,rate-limit
spec:
  rules:
    - host: api.firstviscount.com
      http:
        paths:
          - path: /api/v1/products
            pathType: Prefix
            backend:
              service:
                name: product-service
                port:
                  number: 8082
```

---

*Traefik provides the service mesh capabilities we need without the complexity of full service mesh solutions.*