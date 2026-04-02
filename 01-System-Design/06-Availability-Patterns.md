# Availability Patterns

## Title & Summary

Availability Patterns are architectural strategies that ensure systems remain accessible and operational despite failures, maintenance, or high load. These patterns focus on achieving high availability (HA) through redundancy, failover mechanisms, and careful system design to meet Service Level Objectives (SLOs) and Service Level Agreements (SLAs).

## Problem Statement

Systems face availability challenges when:

- **Component Failures**: Servers, databases, or network equipment fail unexpectedly
- **Planned Maintenance**: Updates and patches require system downtime
- **Traffic Spikes**: Sudden load increases overwhelm capacity
- **Data Center Outages**: Power failures, network issues, or disasters
- **Cascading Failures**: One failure triggers multiple subsequent failures
- **Geographic Issues**: Regional outages affect all users in an area

## Solution

### Understanding Availability

**Availability Calculation:**
```
Availability = (Uptime / Total Time) × 100%

"Nine" System Availability:
- 90% (one nine)    = 7.3 days downtime/year
- 99% (two nines)   = 3.65 days downtime/year
- 99.9% (three nines) = 8.76 hours downtime/year
- 99.99% (four nines) = 52.56 minutes downtime/year
- 99.999% (five nines) = 5.26 minutes downtime/year
```

### Core Availability Patterns

**1. Redundancy Pattern**
Multiple copies of components to eliminate single points of failure.

```
┌─────────────────────────────────────────────────┐
│              Load Balancer                       │
│         (Active-Active or Active-Passive)        │
└──────────────────┬──────────────────────────────┘
                   │
        ┌──────────┼──────────┐
        ▼          ▼          ▼
   ┌───────┐  ┌───────┐  ┌───────┐
   │ Server│  │ Server│  │ Server│
   │   1   │  │   2   │  │   3   │
   └───┬───┘  └───┬───┘  └───┬───┘
       │          │          │
       └──────────┴──────────┘
                │
        ┌───────┴───────┐
        ▼               ▼
   ┌───────┐       ┌───────┐
   │ DB    │       │ DB    │
   │Primary│◀─────▶│Replica│
   └───────┘       └───────┘
```

**2. Failover Pattern**
Automatic switching to backup systems when primary fails.

**3. Multi-Region Deployment**
Deploying across geographic regions for disaster recovery.

**4. Health Check & Auto-Recovery**
Continuous monitoring with automatic remediation.

**5. Graceful Degradation**
Maintaining core functionality when non-critical features fail.

### Implementation Strategies

**Active-Active vs Active-Passive:**

| Aspect | Active-Active | Active-Passive |
|--------|---------------|----------------|
| Resource Utilization | 100% | 50% |
| Failover Time | Near-zero | Seconds to minutes |
| Complexity | High | Medium |
| Cost | Lower (better utilization) | Higher (idle resources) |
| Best For | Stateless services | Stateful services |

## When to Use

| Pattern | Use Case | Target Availability |
|---------|----------|---------------------|
| Active-Active | Stateless web services | 99.9%+ |
| Active-Passive | Database systems | 99.99%+ |
| Multi-Region | Global applications | 99.99%+ |
| Graceful Degradation | Complex systems | Maintain core functionality |
| Health Checks | All production systems | Required for HA |

## Tradeoffs

### Active-Active

| Advantage | Disadvantage |
|-----------|--------------|
| Better resource utilization | Complex state synchronization |
| Zero failover time | Split-brain scenarios possible |
| Higher throughput | More complex debugging |

### Active-Passive

| Advantage | Disadvantage |
|-----------|--------------|
| Simpler implementation | Idle resources (wasted cost) |
| No split-brain issues | Failover latency |
| Easier to maintain | Capacity limitations during failover |

### Multi-Region

| Advantage | Disadvantage |
|-----------|--------------|
| Disaster recovery | Higher latency for cross-region |
| Geographic redundancy | Data consistency challenges |
| Regulatory compliance | Significantly higher cost |

## Implementation Example

### Multi-Region Active-Passive Setup

```javascript
// Region configuration
const REGIONS = {
    PRIMARY: 'us-east-1',
    SECONDARY: 'us-west-2'
};

class AvailabilityManager {
    constructor() {
        this.currentRegion = REGIONS.PRIMARY;
        this.failoverInProgress = false;
        this.healthCheckInterval = null;
    }
    
    // Health check for current region
    async checkRegionHealth(region) {
        const checks = [
            this.checkLoadBalancer(region),
            this.checkApplicationServers(region),
            this.checkDatabase(region),
            this.checkCache(region)
        ];
        
        const results = await Promise.allSettled(checks);
        const allHealthy = results.every(r => r.status === 'fulfilled');
        
        return {
            region,
            healthy: allHealthy,
            timestamp: Date.now(),
            checks: results.map((r, i) => ({
                name: ['lb', 'app', 'db', 'cache'][i],
                status: r.status === 'fulfilled' ? 'healthy' : 'unhealthy'
            }))
        };
    }
    
    async checkLoadBalancer(region) {
        const response = await fetch(
            `https://${region}.example.com/health`,
            { timeout: 5000 }
        );
        if (!response.ok) throw new Error('LB unhealthy');
    }
    
    async checkApplicationServers(region) {
        const response = await fetch(
            `https://app.${region}.example.com/health`,
            { timeout: 5000 }
        );
        if (!response.ok) throw new Error('App servers unhealthy');
    }
    
    async checkDatabase(region) {
        const client = await this.getDbClient(region);
        try {
            await client.query('SELECT 1');
        } finally {
            client.release();
        }
    }
    
    async checkCache(region) {
        const redis = this.getRedisClient(region);
        const ping = await redis.ping();
        if (ping !== 'PONG') throw new Error('Cache unhealthy');
    }
    
    // Automatic failover
    async startHealthMonitoring() {
        const FAILOVER_THRESHOLD = 3; // Consecutive failures
        let consecutiveFailures = 0;
        
        this.healthCheckInterval = setInterval(async () => {
            try {
                const health = await this.checkRegionHealth(this.currentRegion);
                
                if (health.healthy) {
                    consecutiveFailures = 0;
                } else {
                    consecutiveFailures++;
                    console.warn(`Region ${this.currentRegion} unhealthy (${consecutiveFailures}/${FAILOVER_THRESHOLD})`);
                    
                    if (consecutiveFailures >= FAILOVER_THRESHOLD && !this.failoverInProgress) {
                        await this.performFailover();
                    }
                }
            } catch (error) {
                console.error('Health check error:', error);
            }
        }, 10000); // Check every 10 seconds
    }
    
    async performFailover() {
        this.failoverInProgress = true;
        console.log(`Initiating failover from ${this.currentRegion}`);
        
        const targetRegion = this.currentRegion === REGIONS.PRIMARY 
            ? REGIONS.SECONDARY 
            : REGIONS.PRIMARY;
        
        try {
            // 1. Verify target region is healthy
            const targetHealth = await this.checkRegionHealth(targetRegion);
            if (!targetHealth.healthy) {
                throw new Error(`Target region ${targetRegion} not healthy`);
            }
            
            // 2. Update DNS to point to target region
            await this.updateDNS(targetRegion);
            
            // 3. Promote read replica if applicable
            await this.promoteDatabase(targetRegion);
            
            // 4. Update configuration
            this.currentRegion = targetRegion;
            
            // 5. Notify stakeholders
            await this.notifyFailover(targetRegion);
            
            console.log(`Failover complete. Now serving from ${targetRegion}`);
        } catch (error) {
            console.error('Failover failed:', error);
            throw error;
        } finally {
            this.failoverInProgress = false;
        }
    }
    
    async updateDNS(targetRegion) {
        // Update DNS to point to target region
        // Implementation depends on DNS provider
        console.log(`Updating DNS to point to ${targetRegion}`);
    }
    
    async promoteDatabase(targetRegion) {
        // Promote read replica to primary
        console.log(`Promoting database in ${targetRegion}`);
    }
    
    async notifyFailover(targetRegion) {
        // Send notifications via email, Slack, PagerDuty, etc.
        console.log(`Notifying stakeholders of failover to ${targetRegion}`);
    }
    
    // Manual failover for testing/maintenance
    async manualFailover(targetRegion) {
        if (this.failoverInProgress) {
            throw new Error('Failover already in progress');
        }
        
        const health = await this.checkRegionHealth(targetRegion);
        if (!health.healthy) {
            throw new Error(`Target region ${targetRegion} is not healthy`);
        }
        
        await this.performFailover();
    }
}

// Usage
const availabilityManager = new AvailabilityManager();
availabilityManager.startHealthMonitoring();
```

### Graceful Degradation Implementation

```javascript
class GracefulDegradation {
    constructor(app) {
        this.app = app;
        this.disabledFeatures = new Set();
    }
    
    // Decorator for feature availability
    withFallback(featureName, handler, fallback) {
        return async (...args) => {
            if (this.disabledFeatures.has(featureName)) {
                console.log(`Using fallback for ${featureName}`);
                return fallback(...args);
            }
            
            try {
                return await handler(...args);
            } catch (error) {
                console.error(`${featureName} failed, using fallback`, error);
                this.disabledFeatures.add(featureName);
                return fallback(...args);
            }
        };
    }
    
    // Disable a feature
    disableFeature(featureName) {
        this.disabledFeatures.add(featureName);
        console.log(`Feature ${featureName} disabled`);
    }
    
    // Re-enable a feature
    enableFeature(featureName) {
        this.disabledFeatures.delete(featureName);
        console.log(`Feature ${featureName} re-enabled`);
    }
    
    // Health endpoint showing feature status
    getFeatureStatus() {
        return Array.from(this.disabledFeatures);
    }
}

// Usage in Express app
const app = express();
const degradation = new GracefulDegradation(app);

// Original handlers
async function getRecommendations(user) {
    // ML-based recommendations (expensive)
    return await mlService.getRecommendations(user);
}

async function getPopularItems() {
    // Fallback: just return popular items
    return await db.query('SELECT * FROM items ORDER BY popularity DESC LIMIT 10');
}

async function getPersonalizedContent(user) {
    // Personalized content
    return await personalizationService.getContent(user);
}

async function getDefaultContent() {
    // Fallback: default content for all users
    return await db.query('SELECT * FROM content LIMIT 20');
}

// Wrap with graceful degradation
app.get('/recommendations', degradation.withFallback(
    'recommendations',
    getRecommendations,
    getPopularItems
));

app.get('/content', degradation.withFallback(
    'personalized-content',
    getPersonalizedContent,
    getDefaultContent
));

// Health endpoint
app.get('/health/features', (req, res) => {
    res.json({
        disabledFeatures: degradation.getFeatureStatus(),
        timestamp: new Date().toISOString()
    });
});
```

### Circuit Breaker with Fallback (Combined Pattern)

```javascript
class AvailabilityService {
    constructor() {
        this.circuitBreakers = new Map();
        this.fallbacks = new Map();
    }
    
    configure(serviceName, options) {
        this.circuitBreakers.set(serviceName, new CircuitBreaker({
            failureThreshold: options.failureThreshold || 5,
            resetTimeout: options.resetTimeout || 30000
        }));
        
        if (options.fallback) {
            this.fallbacks.set(serviceName, options.fallback);
        }
    }
    
    async call(serviceName, operation, defaultFallback = () => null) {
        const circuitBreaker = this.circuitBreakers.get(serviceName);
        const fallback = this.fallbacks.get(serviceName) || defaultFallback;
        
        try {
            return await circuitBreaker.execute(operation);
        } catch (error) {
            console.log(`Service ${serviceName} unavailable, using fallback`);
            return fallback();
        }
    }
}

// Usage
const availabilityService = new AvailabilityService();

// Configure services with fallbacks
availabilityService.configure('recommendations', {
    failureThreshold: 3,
    resetTimeout: 60000,
    fallback: () => ({ items: [], source: 'fallback' })
});

availabilityService.configure('user-profile', {
    failureThreshold: 5,
    resetTimeout: 30000,
    fallback: () => ({ name: 'Guest', avatar: '/default-avatar.png' })
});

// Call services with automatic fallback
app.get('/dashboard', async (req, res) => {
    const recommendations = await availabilityService.call(
        'recommendations',
        () => recommendationService.getForUser(req.user.id)
    );
    
    const userProfile = await availabilityService.call(
        'user-profile',
        () => userService.getProfile(req.user.id)
    );
    
    res.json({ recommendations, userProfile });
});
```

## Anti-Pattern

### Common Availability Mistakes

**1. Single Point of Failure**
```
❌ BAD: No redundancy
┌─────────┐     ┌─────────┐     ┌─────────┐
│  Load   │────▶│  App    │────▶│  DB     │
│ Balancer│     │ Server  │     │ Server  │
└─────────┘     └─────────┘     └─────────┘
   (Single)      (Single)       (Single)
```

```
✅ GOOD: Redundant components
┌─────────┐     ┌─────────┐     ┌─────────┐
│  Load   │────▶│  App    │────▶│  DB     │
│ Balancer│     │ Servers │     │ Cluster │
└─────────┘     └─────────┘     └─────────┘
   (Active-    (Multiple)      (Primary +
    Active)                    Replica)
```

**2. Silent Failures**
```
❌ BAD: Swallowing errors
try {
    await externalService.call();
} catch (error) {
    // Silently ignored!
}
```

```
✅ GOOD: Proper error handling with fallback
try {
    return await externalService.call();
} catch (error) {
    logger.error('External service failed', error);
    metrics.increment('external_service_failures');
    return fallbackService.call();
}
```

**3. No Health Checks**
```
❌ BAD: Load balancer sends traffic to dead servers
servers:
  - server1
  - server2  # Could be down!
  - server3
```

```
✅ GOOD: Health checks configured
servers:
  - server1
  - server2
  - server3
health_check:
  path: /health
  interval: 10s
  timeout: 5s
  unhealthy_threshold: 3
```

**4. Cascading Failures**
```
❌ BAD: No circuit breakers, failures cascade
Service A → Service B → Service C → Database
     ↓         ↓          ↓
  All hang when database is slow!
```

```
✅ GOOD: Circuit breakers prevent cascade
Service A →[CB]→ Service B →[CB]→ Service C →[CB]→ Database
     ↓          ↓           ↓
  Fast fail if any component is unhealthy
```

## Related Patterns

- **[Reliability Patterns](./05-Reliability-Patterns.md)** - Resilience techniques
- **[Scalability Patterns](./04-Scalability-Patterns.md)** - Scaling for availability
- **[Fault Tolerance](../05-Safety-Engineering/02-Fault-Tolerance.md)** - Resilience patterns
- **[Monitoring](../10-Observability/02-Monitoring.md)** - Health monitoring
- **[Deployment Strategies](../11-DevOps/02-Deployment-Strategies.md)** - Zero-downtime deployments
- **[Replication](../07-Distributed-Systems/04-Replication.md)** - Data redundancy