# Scalability Patterns

## Title & Summary

Scalability Patterns encompass architectural strategies and techniques for building systems that can handle growing workloads by adding resources. These patterns address both horizontal scaling (adding more machines) and vertical scaling (adding more power to existing machines), enabling systems to maintain performance under increasing load.

## Problem Statement

Systems face scalability challenges when:

- **User Growth**: Increasing user base overwhelms single-server capacity
- **Data Volume**: Large datasets exceed memory or storage limits
- **Transaction Load**: High request rates cause bottlenecks
- **Resource Contention**: CPU, memory, network, or I/O become saturated
- **Single Points of Limitation**: Individual components cannot scale independently
- **Geographic Distribution**: Users spread across regions experience latency

## Solution

### Scaling Approaches

**1. Vertical Scaling (Scale Up)**
- Add more resources (CPU, RAM, storage) to existing servers
- Simple to implement; no code changes required
- Limited by maximum hardware capacity
- Cost-effective for small to medium workloads

**2. Horizontal Scaling (Scale Out)**
- Add more servers to distribute load
- Theoretically unlimited scaling capacity
- Requires stateless design and load balancing
- More complex but more flexible long-term

### Core Scalability Patterns

**Load Balancing**
```
                    [Load Balancer]
                        /   |   \
                       /    |    \
              [Server 1] [Server 2] [Server 3]
```

**Caching Strategies**
- **Application Cache**: In-memory caching (Redis, Memcached)
- **CDN**: Edge caching for static content
- **Database Cache**: Query result caching
- **Browser Cache**: Client-side caching

**Database Scaling**
- **Read Replicas**: Multiple read-only copies
- **Sharding**: Horizontal data partitioning
- **Connection Pooling**: Efficient connection management

**Message Queues**
- Decouple producers from consumers
- Buffer bursts of traffic
- Enable asynchronous processing

## When to Use

| Pattern | Best For |
|---------|----------|
| Load Balancing | Distributing traffic across multiple servers |
| Caching | Reducing database load, improving response times |
| Read Replicas | Read-heavy workloads |
| Sharding | Large datasets exceeding single server capacity |
| Message Queues | Bursty traffic, asynchronous processing |
| CDN | Static content served to global users |
| Auto-scaling | Variable/predictable workloads |

## Tradeoffs

### Load Balancing

| Advantage | Disadvantage |
|-----------|--------------|
| Distributes load evenly | Adds infrastructure complexity |
| Enables zero-downtime deployments | Potential single point of failure |
| Improves fault tolerance | Session affinity challenges |

### Caching

| Advantage | Disadvantage |
|-----------|--------------|
| Dramatically reduces latency | Cache invalidation complexity |
| Reduces backend load | Memory consumption |
| Improves throughput | Data consistency challenges |

### Database Scaling

| Advantage | Disadvantage |
|-----------|--------------|
| Handles larger datasets | Cross-shard queries complex |
| Improves read performance | Replication lag possible |
| Enables geographic distribution | Increased operational complexity |

## Implementation Example

### Load Balancer Configuration (Nginx)

```nginx
upstream backend_servers {
    least_conn;
    server 10.0.1.10:8080 weight=5;
    server 10.0.1.11:8080 weight=3;
    server 10.0.1.12:8080 backup;
    
    keepalive 32;
}

server {
    listen 80;
    
    location / {
        proxy_pass http://backend_servers;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        
        # Health check configuration
        proxy_next_upstream error timeout http_500 http_502 http_503 http_504;
    }
    
    location /health {
        return 200 "healthy\n";
        add_header Content-Type text/plain;
    }
}
```

### Caching Layer with Redis

```javascript
const Redis = require('ioredis');
const redis = new Redis();

class CachedService {
    constructor(ttl = 300) {
        this.ttl = ttl; // 5 minutes default
    }
    
    async getWithCache(key, fetchFn) {
        // Try cache first
        const cached = await redis.get(key);
        if (cached) {
            console.log(`Cache hit for ${key}`);
            return JSON.parse(cached);
        }
        
        console.log(`Cache miss for ${key}, fetching...`);
        const data = await fetchFn();
        
        // Cache with TTL
        await redis.setex(key, this.ttl, JSON.stringify(data));
        return data;
    }
    
    async invalidatePattern(pattern) {
        const keys = await redis.keys(pattern);
        if (keys.length > 0) {
            await redis.del(...keys);
            console.log(`Invalidated ${keys.length} keys matching ${pattern}`);
        }
    }
}

// Usage
const userService = new CachedService(600); // 10 minute TTL

app.get('/users/:id', async (req, res) => {
    const user = await userService.getWithCache(
        `user:${req.params.id}`,
        () => db.users.findById(req.params.id)
    );
    res.json(user);
});
```

### Database Read Replicas

```javascript
const { Pool } = require('pg');

// Primary database for writes
const writePool = new Pool({
    host: 'primary-db.example.com',
    database: 'myapp',
    max: 20
});

// Read replicas for reads
const readPools = [
    new Pool({ host: 'replica-1.example.com', database: 'myapp' }),
    new Pool({ host: 'replica-2.example.com', database: 'myapp' })
];

class DatabaseService {
    async write(query, params) {
        const client = await writePool.connect();
        try {
            return await client.query(query, params);
        } finally {
            client.release();
        }
    }
    
    async read(query, params) {
        // Round-robin across read replicas
        const replicaIndex = this.currentReplica % readPools.length;
        this.currentReplica = (this.currentReplica + 1) % readPools.length;
        
        const client = await readPools[replicaIndex].connect();
        try {
            return await client.query(query, params);
        } finally {
            client.release();
        }
    }
}

const db = new DatabaseService();
db.currentReplica = 0;
```

### Auto-scaling Configuration (AWS-style)

```yaml
# Kubernetes Horizontal Pod Autoscaler
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: backend-hpa
  namespace: production
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: backend
  minReplicas: 3
  maxReplicas: 50
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
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
      - type: Percent
        value: 10
        periodSeconds: 60
    scaleUp:
      stabilizationWindowSeconds: 0
      policies:
      - type: Percent
        value: 100
        periodSeconds: 15
      - type: Pods
        value: 4
        periodSeconds: 15
      selectPolicy: Max
```

## Anti-Pattern

### Common Scalability Mistakes

**1. Scaling Before Measuring**
```
❌ BAD: Adding servers without understanding bottlenecks
- "Let's just add more servers"
- No profiling or monitoring
- Scaling the wrong component
```

```
✅ GOOD: Measure first, then scale strategically
- Profile to identify bottlenecks
- Monitor resource utilization
- Scale based on data
```

**2. Stateful Services Without Consideration**
```
❌ BAD: Storing session state in memory
class UserService {
    constructor() {
        this.sessions = new Map(); // Lost on restart!
    }
}
```

```
✅ GOOD: Externalize state to scalable stores
class UserService {
    constructor(redis) {
        this.sessionStore = redis; // Shared, persistent
    }
}
```

**3. N+1 Query Problem**
```
❌ BAD: Inefficient data fetching
async function getUsersWithOrders(userId) {
    const users = await db.query('SELECT * FROM users WHERE id = $1', [userId]);
    const orders = await db.query('SELECT * FROM orders WHERE user_id = $1', [userId]);
    // N+1 queries when fetching multiple users!
}
```

```
✅ GOOD: Single efficient query
async function getUsersWithOrders(userId) {
    return await db.query(`
        SELECT u.*, JSON_agg(o) as orders
        FROM users u
        LEFT JOIN orders o ON o.user_id = u.id
        WHERE u.id = $1
        GROUP BY u.id
    `, [userId]);
}
```

**4. Ignoring Cache Penetration**
```
❌ BAD: No protection for missing keys
async function getUser(id) {
    let user = cache.get(id);
    if (!user) {
        user = db.getUser(id); // Database hit for every missing key!
    }
    return user;
}
```

```
✅ GOOD: Cache null values briefly
async function getUser(id) {
    let user = cache.get(id);
    if (user === undefined) {
        user = await db.getUser(id);
        if (user) {
            cache.set(id, user, 300);
        } else {
            cache.set(id, NULL_VALUE, 60); // Prevent penetration
        }
    }
    return user === NULL_VALUE ? null : user;
}
```

## Related Patterns

- **[Microservices Architecture](./02-Microservices-Architecture.md)** - Independent service scaling
- **[Event-Driven Architecture](./03-Event-Driven-Architecture.md)** - Asynchronous scaling
- **[Reliability Patterns](./05-Reliability-Patterns.md)** - Scaling for resilience
- **[Availability Patterns](./06-Availability-Patterns.md)** - High availability through scaling
- **[Performance Patterns](./07-Performance-Patterns.md)** - Performance optimization techniques
- **[Partitioning](../07-Distributed-Systems/03-Partitioning.md)** - Data distribution strategies
- **[Replication](../07-Distributed-Systems/04-Replication.md)** - Data copying for scale