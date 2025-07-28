# Migration Paths

## Overview

This document outlines migration strategies for when our system outgrows initial technology choices. Each migration path includes triggers, strategies, and rollback plans.

## Redpanda → Apache Kafka

### Migration Triggers
- Event throughput > 100K messages/second sustained
- Need for Kafka-specific features (Streams, Connect)
- Multi-datacenter replication requirements
- Advanced stream processing needs

### Migration Strategy

#### Phase 1: Compatibility Verification
```python
# Test Kafka client compatibility
from aiokafka import AIOKafkaProducer

# Redpanda uses Kafka protocol
producer = AIOKafkaProducer(
    bootstrap_servers='redpanda:9092',  # Works with Redpanda
    # Later: bootstrap_servers='kafka:9092'
)
```

#### Phase 2: Parallel Running
```yaml
# docker-compose.yml
services:
  redpanda:
    # Existing Redpanda
  
  kafka:
    image: confluentinc/cp-kafka:latest
    environment:
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka:9093
```

#### Phase 3: Dual Publishing
```python
# Publish to both during transition
async def publish_event(event):
    await redpanda_producer.send(topic, event)
    await kafka_producer.send(topic, event)
```

#### Phase 4: Consumer Migration
1. Stop Redpanda consumers
2. Note offset positions
3. Start Kafka consumers from saved offsets
4. Verify no message loss

#### Phase 5: Cutover
1. Stop dual publishing
2. Publish only to Kafka
3. Decommission Redpanda
4. Update all configurations

### Rollback Plan
- Keep Redpanda running for 30 days
- Maintain dual publishing capability
- Test rollback procedures weekly
- Document offset mappings

---

## Docker Compose → Kubernetes (K3s)

### Migration Triggers
- Need for auto-scaling
- Multi-node deployment required
- Advanced deployment strategies needed
- Service mesh requirements

### Migration Strategy

#### Phase 1: Containerization Verification
```dockerfile
# Ensure all services have production-ready images
FROM python:3.11-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8080"]
```

#### Phase 2: Kubernetes Manifests
```yaml
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: product-catalog
spec:
  replicas: 2
  selector:
    matchLabels:
      app: product-catalog
  template:
    metadata:
      labels:
        app: product-catalog
    spec:
      containers:
      - name: product-catalog
        image: firstviscount/product-catalog:latest
        ports:
        - containerPort: 8082
        env:
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: product-catalog-secret
              key: database-url
```

#### Phase 3: Service Discovery Migration
```yaml
# From Docker Compose networks
# To Kubernetes services
apiVersion: v1
kind: Service
metadata:
  name: product-catalog
spec:
  selector:
    app: product-catalog
  ports:
    - protocol: TCP
      port: 8082
      targetPort: 8082
```

#### Phase 4: Configuration Migration
```bash
# From .env files
# To ConfigMaps and Secrets
kubectl create configmap app-config --from-env-file=.env
kubectl create secret generic app-secret --from-env-file=.env.secret
```

#### Phase 5: Progressive Rollout
1. Deploy Platform Coordination to K3s
2. Verify service discovery works
3. Deploy one service at a time
4. Maintain Docker Compose as fallback
5. Full cutover after stability proven

### Rollback Plan
- Keep Docker Compose files updated
- Maintain compatibility in services
- Test monthly rollback drills
- Document all K8s-specific features used

---

## PostgreSQL → Distributed Database

### Migration Triggers
- Single database becomes bottleneck
- Need for geographic distribution
- Sharding requirements
- Read replica lag issues

### Options Evaluation

#### Option 1: PostgreSQL with Citus
```sql
-- Distributed tables
SELECT create_distributed_table('orders', 'customer_id');
SELECT create_distributed_table('order_items', 'order_id');
```

#### Option 2: CockroachDB
```python
# Compatible with PostgreSQL driver
DATABASE_URL = "postgresql://root@cockroachdb:26257/defaultdb"
```

#### Option 3: Vitess
- MySQL compatible
- Proven at scale
- Complex operation

### Migration Strategy

#### Phase 1: Data Model Review
- Identify sharding keys
- Remove cross-shard joins
- Denormalize where needed

#### Phase 2: Application Changes
```python
# Add sharding key to queries
async def get_order(order_id: str, customer_id: str):
    # customer_id helps route to correct shard
    return await db.fetch_one(
        "SELECT * FROM orders WHERE order_id = $1 AND customer_id = $2",
        order_id, customer_id
    )
```

#### Phase 3: Dual Writing
```python
async def create_order(order_data):
    # Write to both databases
    async with old_db.transaction():
        old_result = await old_db.execute(INSERT_ORDER, order_data)
    
    async with new_db.transaction():
        new_result = await new_db.execute(INSERT_ORDER, order_data)
    
    return old_result  # Use old as source of truth initially
```

#### Phase 4: Verification
- Compare data between systems
- Monitor query performance
- Verify consistency

#### Phase 5: Cutover
1. Stop writes to old database
2. Final sync
3. Switch reads to new database
4. Monitor for issues
5. Decommission old database

---

## FastAPI → Alternative Frameworks

### Migration Triggers
- Performance bottlenecks
- Missing enterprise features
- Team expertise changes
- Better ecosystem fit needed

### Potential Targets

#### Go + Gin/Echo
```go
// Similar routing style
func main() {
    r := gin.Default()
    r.GET("/products", getProducts)
    r.GET("/products/:id", getProduct)
    r.Run(":8082")
}
```

#### Java + Spring Boot
```java
@RestController
@RequestMapping("/products")
public class ProductController {
    @GetMapping
    public List<Product> getProducts() {
        return productService.findAll();
    }
}
```

### Migration Strategy

#### Phase 1: API Compatibility Layer
```python
# Create adapter layer
class APIAdapter:
    def __init__(self, old_client, new_client):
        self.old_client = old_client
        self.new_client = new_client
    
    async def get_product(self, product_id):
        if feature_flag.is_enabled("use_new_service"):
            return await self.new_client.get_product(product_id)
        return await self.old_client.get_product(product_id)
```

#### Phase 2: Service by Service
1. Choose least critical service
2. Reimplement in new framework
3. Deploy alongside old service
4. Use feature flags for gradual rollout
5. Monitor performance and errors
6. Complete cutover
7. Repeat for next service

---

## Monolith Fallback Plan

### When to Consider
- Microservices overhead exceeds benefits
- Team size reduced significantly
- Complexity blocking feature delivery
- Cost constraints

### Consolidation Strategy

#### Phase 1: Identify Boundaries
```python
# Modular monolith structure
app/
├── modules/
│   ├── products/
│   ├── orders/
│   ├── inventory/
│   └── shared/
├── api/
└── main.py
```

#### Phase 2: Merge Services
1. Start with least independent services
2. Merge database schemas
3. Combine API endpoints
4. Maintain module boundaries
5. Keep event publishing for future split

#### Phase 3: Simplify Infrastructure
- Single deployment unit
- One database
- Remove service mesh
- Simplify monitoring

### Reversal Strategy
- Keep module boundaries clear
- Maintain API contracts
- Continue event publishing
- Document split points
- Can re-extract services later

---

## General Migration Principles

### Always
- ✅ Maintain backward compatibility
- ✅ Use feature flags
- ✅ Have rollback plans
- ✅ Test migrations thoroughly
- ✅ Monitor during transition

### Never
- ❌ Big bang migrations
- ❌ Skip compatibility testing
- ❌ Migrate without metrics
- ❌ Ignore team feedback
- ❌ Rush migrations

---

*These migration paths ensure we're never locked into technology choices while avoiding premature optimization.*