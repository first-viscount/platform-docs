# Chaos Testing Experiments

## Purpose

Intentionally break things to learn how the system fails and build resilience. Each experiment should teach us something about distributed systems behavior.

## Experiment Framework

### For Each Experiment
1. **Hypothesis**: What we expect to happen
2. **Method**: How we'll break things
3. **Observation**: What actually happened
4. **Learning**: What we learned
5. **Improvement**: How we fixed it

## Planned Experiments

### Experiment 1: Service Discovery Failure
**Target**: Platform Coordination Service  
**Week**: 4

**Hypothesis**: Services will use cached discovery data and continue operating.

**Method**:
```bash
# Kill Platform Coordination service
docker-compose stop platform-coordination

# Observe service behavior
# Try to make inter-service calls
# Watch logs for errors
```

**Expected Observations**:
- Services continue with last known addresses
- New services cannot register
- Health checks fail but services remain functional

**Improvements to Test**:
- Cache TTL configuration
- Fallback mechanisms
- Circuit breaker behavior

---

### Experiment 2: Message Broker Partition
**Target**: Redpanda  
**Week**: 8

**Hypothesis**: Services will queue events locally and resume when connection restored.

**Method**:
```bash
# Block Redpanda port
iptables -A INPUT -p tcp --dport 9092 -j DROP

# Generate events
# Monitor service behavior
# Restore connection
iptables -D INPUT -p tcp --dport 9092 -j DROP
```

**Expected Observations**:
- Events queue in memory
- Services remain responsive
- Events replay when connection restored
- Some event ordering issues

**Improvements to Test**:
- Local queue size limits
- Event replay ordering
- Duplicate event handling

---

### Experiment 3: Database Connection Exhaustion
**Target**: PostgreSQL connections  
**Week**: 12

**Hypothesis**: Services will fail fast with clear error messages.

**Method**:
```python
# Exhaust connection pool
async def exhaust_connections():
    connections = []
    try:
        while True:
            conn = await database.connect()
            connections.append(conn)
    except Exception as e:
        print(f"Failed at {len(connections)} connections: {e}")
```

**Expected Observations**:
- Connection pool exhaustion at limit
- New requests fail immediately
- Health checks report unhealthy
- Service recovers when connections released

**Improvements to Test**:
- Connection pool sizing
- Timeout configurations
- Circuit breaker triggers
- Graceful degradation

---

### Experiment 4: Cascading Service Failure
**Target**: Order → Inventory → Product chain  
**Week**: 16

**Hypothesis**: Circuit breakers will prevent cascade failure.

**Method**:
```bash
# Slow down Inventory service
tc qdisc add dev eth0 root netem delay 5000ms

# Create orders rapidly
# Monitor all service health
# Watch circuit breaker states
```

**Expected Observations**:
- Order service circuit breaker opens
- Fallback behavior activates
- Other services remain healthy
- System recovers when latency removed

**Improvements to Test**:
- Circuit breaker thresholds
- Fallback strategies
- Timeout configurations
- Bulkhead isolation

---

### Experiment 5: Event Storm
**Target**: All services  
**Week**: 18

**Hypothesis**: Services will handle burst traffic with backpressure.

**Method**:
```python
# Generate massive event burst
async def event_storm():
    tasks = []
    for i in range(10000):
        task = producer.send('order-events', create_order_event())
        tasks.append(task)
    await asyncio.gather(*tasks)
```

**Expected Observations**:
- Consumer lag increases
- Processing slows but continues
- No data loss
- Recovery time proportional to burst

**Improvements to Test**:
- Consumer scaling
- Batch processing
- Backpressure mechanisms
- Priority queues

---

### Experiment 6: Split Brain Scenario
**Target**: Inventory service (multiple instances)  
**Week**: 19

**Hypothesis**: Distributed locking will prevent conflicting updates.

**Method**:
```bash
# Start two Inventory instances
docker-compose up -d --scale inventory=2

# Network partition between instances
# Send conflicting updates to each
# Restore network
# Check final state
```

**Expected Observations**:
- Distributed lock prevents conflicts
- One instance wins, other retries
- Eventual consistency achieved
- Audit trail shows all attempts

**Improvements to Test**:
- Lock timeout behavior
- Conflict resolution strategies
- Compensation logic
- Monitoring alerts

---

### Experiment 7: Memory Leak Simulation
**Target**: Product Catalog Service  
**Week**: 20

**Hypothesis**: Service will restart before affecting others.

**Method**:
```python
# Intentional memory leak
leak_list = []
@app.get("/leak")
async def create_leak():
    # Add 10MB to memory
    leak_list.append("x" * 10_000_000)
    return {"size": len(leak_list)}
```

**Expected Observations**:
- Memory usage increases steadily
- Health checks start failing
- Container restarts automatically
- Service recovers with no data loss

**Improvements to Test**:
- Memory limits
- Health check sensitivity
- Restart policies
- Monitoring alerts

---

## Chaos Testing Tools

### Manual Chaos
```bash
# Network delays
tc qdisc add dev eth0 root netem delay 100ms

# Packet loss
tc qdisc add dev eth0 root netem loss 10%

# Service killing
docker-compose kill <service>

# Resource limits
docker update --memory="50m" <container>
```

### Automated Chaos
```python
# Chaos module for testing
import random
import asyncio

class ChaosMonkey:
    def __init__(self, probability=0.1):
        self.probability = probability
    
    async def maybe_fail(self):
        if random.random() < self.probability:
            raise Exception("Chaos monkey strike!")
    
    async def maybe_delay(self):
        if random.random() < self.probability:
            await asyncio.sleep(random.uniform(1, 5))
```

## Learning Documentation

### Failure Catalog
Document each failure mode discovered:
1. How to trigger it
2. System behavior
3. Detection method
4. Recovery approach
5. Prevention strategy

### Resilience Patterns
Document patterns that work:
1. Circuit breaker configurations
2. Retry strategies
3. Timeout values
4. Fallback approaches
5. Monitoring alerts

## Chaos Testing Schedule

| Week | Experiment | Focus Area |
|------|-----------|------------|
| 4 | Service Discovery | Infrastructure |
| 8 | Message Broker | Events |
| 12 | Database | Persistence |
| 16 | Cascading Failure | Resilience |
| 18 | Event Storm | Performance |
| 19 | Split Brain | Consistency |
| 20 | Memory Leak | Resources |

## Success Criteria

### Good Chaos Testing
- Failures are controlled
- Learning is documented
- Improvements are implemented
- Confidence increases
- Real issues prevented

### Bad Chaos Testing
- Random destruction
- No documentation
- No improvements
- Team frustration
- Same failures repeat

---

*Remember: The goal is learning, not breaking. Every experiment should make the system stronger.*