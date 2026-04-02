# Performance Patterns

## Title & Summary

Performance patterns are architectural and implementation strategies designed to optimize system responsiveness, throughput, and resource utilization. This pattern covers techniques for measuring, analyzing, and improving system performance across multiple dimensions including latency, throughput, scalability, and efficiency.

## Problem Statement

Modern applications face increasing performance demands:

- **User Expectations**: Users expect sub-second response times; 47% expect pages to load in 2 seconds or less
- **Resource Constraints**: Computing resources are expensive; inefficient use increases costs
- **Scale Challenges**: Systems must perform well under varying loads from idle to peak traffic
- **Complex Dependencies**: Microservices architectures introduce network latency and coordination overhead
- **Data Volume**: Large datasets require efficient processing and retrieval strategies
- **Real-time Requirements**: Many applications require low-latency, real-time processing

Poor performance leads to:
- Lost revenue (Amazon: $16M/hour per second of downtime)
- User abandonment (40% abandon sites taking >3 seconds to load)
- Increased infrastructure costs
- Poor search engine rankings
- Negative brand perception

## Solution

### Core Performance Dimensions

**1. Latency Optimization**
- Time from request initiation to response completion
- Critical for user-facing applications and real-time systems
- Measured in milliseconds (ms)

**2. Throughput Optimization**
- Number of requests processed per unit time
- Critical for batch processing and high-volume systems
- Measured in requests/second (RPS) or transactions/second (TPS)

**3. Resource Efficiency**
- CPU, memory, disk I/O, and network utilization
- Critical for cost optimization and scalability
- Measured as utilization percentages and cost per transaction

**4. Scalability**
- Ability to maintain performance under increasing load
- Critical for growing systems
- Measured as linear vs. non-linear performance degradation

### Performance Patterns Catalog

#### 1. Caching Patterns

**Multi-Level Caching**
```
┌─────────────────────────────────────────────────────┐
│                    Client                           │
│              (Browser Cache)                        │
└────────────────────┬────────────────────────────────┘
                     │
┌────────────────────▼────────────────────────────────┐
│              CDN / Edge Cache                       │
│         (Geo-distributed, TTL-based)               │
└────────────────────┬────────────────────────────────┘
                     │
┌────────────────────▼────────────────────────────────┐
│              Application Cache                      │
│         (Redis, Memcached - In-Memory)             │
└────────────────────┬────────────────────────────────┘
                     │
┌────────────────────▼────────────────────────────────┐
│              Database Cache                        │
│         (Query Cache, Buffer Pool)                 │
└────────────────────┬────────────────────────────────┘
                     │
┌────────────────────▼────────────────────────────────┐
│              Storage Layer                         │
│         (Disk, SSD, HDD)                           │
└─────────────────────────────────────────────────────┘
```

**Cache Strategies:**
- **Cache-Aside (Lazy Loading)**: Application checks cache, loads from backend on miss
- **Read-Through**: Cache automatically loads from backend on miss
- **Write-Through**: Writes go to both cache and backend
- **Write-Behind**: Writes go to cache, asynchronously flushed to backend
- **Write-Around**: Writes go directly to backend, cache not updated

#### 2. Asynchronous Processing

**Message Queue Pattern**
```
┌─────────┐    ┌──────────┐    ┌──────────┐    ┌─────────┐
│ Producer│ ──▶│ Message  │ ──▶│  Consumer│ ──▶│ Storage │
│         │    │  Queue   │    │          │    │         │
└─────────┘    └──────────┘    └──────────┘    └─────────┘
                 (Kafka,
                  RabbitMQ,
                  SQS)
```

**Benefits:**
- Decouples producers from consumers
- Enables load leveling and burst handling
- Allows parallel processing
- Improves system resilience

#### 3. Load Distribution

**Load Balancing Strategies**
- **Round Robin**: Equal distribution across servers
- **Least Connections**: Routes to server with fewest active connections
- **IP Hash**: Consistent routing based on client IP
- **Weighted**: Distribution based on server capacity
- **Geographic**: Routes to nearest datacenter

**Auto-Scaling Pattern**
```
┌─────────────────────────────────────────────────────┐
│                    Load Balancer                    │
└────────────┬────────────────┬───────────────────────┘
             │                │
    ┌────────▼──────┐  ┌──────▼────────┐
    │   Server Pool │  │   Server Pool │
    │   (Scale Up)  │  │   (Scale Down)│
    └───────────────┘  └───────────────┘
             ▲                ▲
             └────────────────┘
                  │
         ┌────────▼──────┐
         │   Metrics     │
         │   Collector   │
         └───────────────┘
```

#### 4. Database Performance

**Query Optimization**
- Index selection and optimization
- Query plan analysis
- Connection pooling
- Read replicas for read-heavy workloads
- Database sharding for write-heavy workloads

**Connection Pooling**
```
┌─────────────────────────────────────────────────────┐
│                  Application                        │
└────────────────────┬────────────────────────────────┘
                     │
┌────────────────────▼────────────────────────────────┐
│              Connection Pool                       │
│         (Fixed size, reusable connections)         │
└────────────────────┬────────────────────────────────┘
                     │
┌────────────────────▼────────────────────────────────┐
│              Database Server                       │
└─────────────────────────────────────────────────────┘
```

#### 5. Content Delivery

**CDN Pattern**
```
                    ┌──────────┐
                    │  Origin  │
                    │  Server  │
                    └────┬─────┘
                         │
              ┌──────────┼──────────┐
              ▼          ▼          ▼
        ┌────────┐  ┌────────┐  ┌────────┐
        │ CDN    │  │ CDN    │  │ CDN    │
        │ Edge   │  │ Edge   │  │ Edge   │
        │ US-East│  │  EU    │  │  Asia  │
        └────┬───┘  └────┬───┘  └────┬───┘
             │           │           │
        ┌────▼───────────▼───────────▼────┐
        │           Clients               │
        └─────────────────────────────────┘
```

#### 6. Parallel Processing

**Map-Reduce Pattern**
```
Input Data
    │
    ▼
┌─────────┐
│  MAP    │──┬──────────┬──────────┐
└─────────┘  │          │          │
             ▼          ▼          ▼
          ┌─────┐    ┌─────┐    ┌─────┐
          │Reduce│   │Reduce│   │Reduce│
          └───┬──┘    └───┬──┘    └───┬──┘
              │           │           │
              └───────────┴───────────┘
                      │
                      ▼
                Output
```

### Performance Measurement Framework

**Key Metrics:**
| Metric | Description | Target |
|--------|-------------|--------|
| Response Time | Time to first byte | < 200ms |
| Time to Interactive | Page usable | < 3.5s |
| Throughput | Requests/second | System-dependent |
| Error Rate | Failed requests | < 0.1% |
| Resource Utilization | CPU/Memory/Disk | < 70% average |

**APM Tools:**
- Application Performance Monitoring: New Relic, Datadog, AppDynamics
- Distributed Tracing: Jaeger, Zipkin, AWS X-Ray
- Profiling: pprof, VisualVM, Chrome DevTools

## When to Use

### Use Performance Patterns When:

1. **User-Facing Applications**: Any application where response time affects user satisfaction
2. **High-Traffic Systems**: Systems handling >1000 concurrent users
3. **Real-Time Systems**: Trading platforms, gaming, live collaboration
4. **Data-Intensive Applications**: Analytics, ML inference, search
5. **Cost-Sensitive Environments**: Cloud environments where resource efficiency = cost savings
6. **Regulated Industries**: Financial services, healthcare with SLA requirements

### Performance Budget Guidelines:

| System Type | Response Time | Throughput | Availability |
|-------------|---------------|------------|--------------|
| Web Application | < 200ms | 100+ RPS | 99.9% |
| API Service | < 100ms | 1000+ RPS | 99.95% |
| Real-Time System | < 50ms | Variable | 99.99% |
| Batch Processing | Variable | Maximize | 99.9% |

## Tradeoffs

### Performance vs. Consistency
```
High Performance ──────────────────────────────────▶
    │
    │  Cache Invalidation Complexity
    │  Eventual Consistency Required
    │  Data Staleness Possible
    ▼
Low Consistency
```

### Performance vs. Cost
- More caching = higher memory costs but lower compute costs
- More replicas = higher storage costs but better read performance
- CDN = additional cost but better global performance

### Performance vs. Complexity
- Simple systems are easier to debug but may not scale
- Complex caching strategies improve performance but add failure modes
- Async processing improves throughput but complicates error handling

### Key Tradeoff Decisions:

| Decision | Performance Benefit | Cost/Complexity |
|----------|---------------------|-----------------|
| Add Cache Layer | 10-100x faster reads | Cache invalidation complexity |
| Add Read Replicas | Linear read scaling | Replication lag, write amplification |
| Async Processing | Higher throughput | Eventual consistency, error handling |
| Database Sharding | Linear write scaling | Cross-shard queries complex |
| CDN | Global low latency | Cache invalidation, cost |

## Implementation Example

### Complete Performance-Optimized E-Commerce System

```python
# Performance-optimized e-commerce service

import asyncio
from functools import lru_cache
from redis import Redis
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker

# Configuration
CACHE_TTL = 300  # 5 minutes
CACHE_PREFIX = "ecom:"
DB_POOL_SIZE = 20
DB_MAX_OVERFLOW = 20

class PerformanceOptimizedService:
    def __init__(self):
        # Multi-level caching
        self.redis = Redis(host='redis', decode_responses=True)
        
        # Connection pooling
        self.engine = create_engine(
            'postgresql://user:pass@db:5432/ecommerce',
            pool_size=DB_POOL_SIZE,
            max_overflow=DB_MAX_OVERFLOW,
            pool_pre_ping=True,
            pool_recycle=3600
        )
        self.Session = sessionmaker(bind=self.engine)
        
    # L1: In-memory cache (fastest)
    @lru_cache(maxsize=1000)
    def get_product_hot(self, product_id: int) -> dict:
        """Hot product cache - in-memory, LRU eviction"""
        return self._fetch_product_from_db(product_id)
    
    # L2: Distributed cache
    def get_product(self, product_id: int) -> dict:
        """Two-level cache with cache-aside pattern"""
        cache_key = f"{CACHE_PREFIX}product:{product_id}"
        
        # Try distributed cache first
        cached = self.redis.get(cache_key)
        if cached:
            return self._deserialize(cached)
        
        # Cache miss - fetch from database
        product = self._fetch_product_from_db(product_id)
        
        # Populate cache
        self.redis.setex(cache_key, CACHE_TTL, self._serialize(product))
        
        return product
    
    # Async processing for non-critical paths
    async def process_order_async(self, order_data: dict) -> str:
        """Async order processing with message queue"""
        order_id = self._create_order_record(order_data)
        
        # Fire-and-forget for non-critical operations
        asyncio.create_task(self._notify_inventory(order_id))
        asyncio.create_task(self._update_recommendations(order_id))
        asyncio.create_task(self._send_confirmation_email(order_id))
        
        return order_id
    
    # Database query optimization
    def get_product_with_stats(self, product_id: int) -> dict:
        """Optimized query with proper indexing"""
        query = """
            SELECT p.*, 
                   COALESCE(pr.rating_avg, 0) as avg_rating,
                   COALESCE(pr.rating_count, 0) as rating_count
            FROM products p
            LEFT JOIN LATERAL (
                SELECT AVG(rating) as rating_avg, COUNT(*) as rating_count
                FROM product_reviews 
                WHERE product_id = p.id 
                AND reviewed_at > NOW() - INTERVAL '90 days'
            ) pr ON true
            WHERE p.id = :product_id
            AND p.is_active = true
        """
        # Uses composite index: (id, is_active)
        return self._execute_query(query, {"product_id": product_id})
    
    # Batch processing for efficiency
    def get_products_batch(self, product_ids: list) -> list:
        """Batch fetch with N+1 query prevention"""
        # Single query instead of N queries
        query = """
            SELECT * FROM products 
            WHERE id IN :ids 
            AND is_active = true
        """
        return self._execute_query(query, {"ids": product_ids})
    
    # Read replica for read-heavy workloads
    def get_product_recommendations(self, user_id: int, product_id: int) -> list:
        """Use read replica for recommendation queries"""
        # Primary database for writes
        # Read replica for reads (eventual consistency acceptable)
        replica_session = self._get_replica_session()
        
        recommendations = replica_session.query(Recommendation).filter(
            Recommendation.similar_to == product_id
        ).limit(10).all()
        
        return recommendations

# Performance monitoring decorator
def timed_operation(name: str):
    def decorator(func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            start = time.perf_counter()
            try:
                result = func(*args, **kwargs)
                return result
            finally:
                duration = (time.perf_counter() - start) * 1000
                metrics_client.timing(f"{name}.duration", duration)
                if duration > 1000:  # Log slow operations
                    logger.warning(f"Slow operation: {name} took {duration:.2f}ms")
        return wrapper
    return decorator

# Usage
class ProductService(PerformanceOptimizedService):
    @timed_operation("product.get")
    def get_product_details(self, product_id: int) -> dict:
        return self.get_product(product_id)
```

### Infrastructure Configuration (Kubernetes)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: product-service
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  template:
    spec:
      containers:
      - name: product-service
        image: product-service:latest
        resources:
          requests:
            memory: "256Mi"
            cpu: "250m"
          limits:
            memory: "512Mi"
            cpu: "500m"
        readinessProbe:
          httpGet:
            path: /health/ready
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 10
        livenessProbe:
          httpGet:
            path: /health/live
            port: 8080
          initialDelaySeconds: 15
          periodSeconds: 20
---
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: product-service-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: product-service
  minReplicas: 3
  maxReplicas: 20
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
```

## Anti-Pattern

### Performance Anti-Patterns

**1. N+1 Query Problem**
```python
# ❌ ANTI-PATTERN: N+1 queries
def get_users_with_orders(user_ids):
    users = User.query.filter(User.id.in_(user_ids)).all()  # 1 query
    result = []
    for user in users:
        orders = Order.query.filter_by(user_id=user.id).all()  # N queries!
        result.append({"user": user, "orders": orders})
    return result

# ✅ SOLUTION: Eager loading
def get_users_with_orders(user_ids):
    users = User.query.filter(
        User.id.in_(user_ids)
    ).options(
        joinedload(User.orders)  # Single query with JOIN
    ).all()
    return [{"user": u, "orders": u.orders} for u in users]
```

**2. Blocking I/O in Async Context**
```python
# ❌ ANTI-PATTERN: Blocking I/O blocks event loop
async def process_files(file_paths):
    results = []
    for path in file_paths:
        # This blocks the entire event loop!
        data = read_large_file(path)  # Synchronous file read
        results.append(process(data))
    return results

# ✅ SOLUTION: Use async I/O
async def process_files(file_paths):
    loop = asyncio.get_event_loop()
    results = []
    for path in file_paths:
        # Run blocking operation in thread pool
        data = await loop.run_in_executor(None, read_large_file, path)
        results.append(process(data))
    return results
```

**3. Cache Stampede**
```python
# ❌ ANTI-PATTERN: Cache stampede on expiration
def get_expensive_data(key):
    value = cache.get(key)
    if value is None:  # Race condition!
        value = compute_expensive_data()
        cache.set(key, value, ttl=300)
    return value

# ✅ SOLUTION: Mutex or probabilistic early refresh
def get_expensive_data(key):
    value = cache.get(key)
    if value is None:
        with cache_lock(key):  # Mutual exclusion
            value = cache.get(key)  # Double-check
            if value is None:
                value = compute_expensive_data()
                cache.set(key, value, ttl=300)
    return value
```

**4. Premature Optimization**
- Optimizing before measuring
- Optimizing the wrong bottleneck
- Over-optimizing infrequently executed code paths
- Sacrificing readability for marginal gains

**5. Ignoring Cache Locality**
```python
# ❌ ANTI-PATTERN: Poor cache locality
def sum_matrix_columns(matrix):
    total = 0
    for col in range(cols):      # Column-major access
        for row in range(rows):  # Jumps in memory
            total += matrix[row][col]
    return total

# ✅ SOLUTION: Row-major access
def sum_matrix_columns(matrix):
    total = 0
    for row in range(rows):      # Row-major access
        for col in range(cols):  # Sequential memory access
            total += matrix[row][col]
    return total
```

## Related Patterns

### System Design Patterns
- [[04-Scalability-Patterns]] - Performance and scalability are closely related
- [[05-Reliability-Patterns]] - Circuit breakers and timeouts affect performance
- [[06-Availability-Patterns]] - CDN and edge computing improve both

### Design Patterns
- [[02-Design-Patterns/01-Creational-Patterns]] - Object pools for expensive object creation
- [[02-Design-Patterns/02-Structural-Patterns]] - Flyweight pattern for memory efficiency

### Infrastructure Patterns
- [[09-Infrastructure/01-Containerization]] - Resource limits and requests
- [[09-Infrastructure/02-Orchestration]] - Auto-scaling and pod distribution

### Observability Patterns
- [[10-Observability/04-Metrics]] - Performance monitoring and alerting
- [[10-Observability/03-Tracing]] - Distributed tracing for bottleneck identification

---

*Last Updated: 2024-01-15*
*Related: Performance Engineering, Systems Design, Optimization*