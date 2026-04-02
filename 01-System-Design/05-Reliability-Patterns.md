# Reliability Patterns

## Title & Summary

Reliability Patterns are architectural techniques that enable systems to continue functioning correctly despite component failures, network issues, or other adverse conditions. These patterns focus on building resilient systems that can withstand failures gracefully, recover automatically, and maintain service availability.

## Problem Statement

Modern distributed systems face numerous reliability challenges:

- **Component Failures**: Individual services, databases, or infrastructure components can fail
- **Network Issues**: Packet loss, latency spikes, and connection drops are common
- **Cascading Failures**: One failure can propagate and bring down entire systems
- **Data Loss**: Failures can result in permanent loss of critical data
- **Service Degradation**: Partial failures can significantly impact user experience
- **Recovery Complexity**: Manual intervention is slow and error-prone

## Solution

### Core Reliability Patterns

**1. Circuit Breaker Pattern**
Prevents cascading failures by stopping calls to failing services temporarily.

```
┌─────────────┐     ┌──────────────┐     ┌─────────────┐
│   Client    │────▶│ Circuit      │────▶│  Service    │
│             │     │  Breaker     │     │             │
└─────────────┘     └──────────────┘     └─────────────┘
                       │  │  │
                       ▼  ▼  ▼
                    CLOSED OPEN HALF-OPEN
```

**2. Retry Pattern**
Automatically retries failed operations with exponential backoff.

**3. Bulkhead Pattern**
Isolates resources so failures in one area don't affect others.

**4. Timeout Pattern**
Prevents indefinite waiting by setting maximum wait times.

**5. Fallback Pattern**
Provides alternative responses when primary operations fail.

**6. Health Check Pattern**
Monitors system health for automatic recovery and load balancing.

### Implementation Strategies

**Circuit Breaker States:**
- **CLOSED**: Normal operation; requests pass through
- **OPEN**: Failure threshold exceeded; requests fail fast
- **HALF-OPEN**: Testing if service recovered; limited requests allowed

**Retry Strategies:**
- **Fixed Delay**: Same delay between retries
- **Exponential Backoff**: Increasing delay (2^n seconds)
- **Jitter**: Random delay to prevent thundering herd

## When to Use

| Pattern | Use Case |
|---------|----------|
| Circuit Breaker | Preventing cascading failures in microservices |
| Retry | Handling transient network failures |
| Bulkhead | Isolating critical functions from non-critical |
| Timeout | Preventing indefinite blocking |
| Fallback | Providing degraded but functional service |
| Health Check | Monitoring for automatic recovery |

## Tradeoffs

### Circuit Breaker

| Advantage | Disadvantage |
|-----------|--------------|
| Prevents cascading failures | May hide recovering services temporarily |
| Fast failure response | Adds complexity to error handling |
| Protects system resources | Requires tuning of thresholds |

### Retry

| Advantage | Disadvantage |
|-----------|--------------|
| Handles transient failures | Can amplify load on failing services |
| Improves success rate | Increases latency on failures |
| Simple to implement | May retry non-retryable errors |

### Bulkhead

| Advantage | Disadvantage |
|-----------|--------------|
| Contains failures | Resource fragmentation |
| Protects critical paths | More complex resource management |
| Enables graceful degradation | May underutilize resources |

## Implementation Example

### Circuit Breaker Implementation

```javascript
class CircuitBreaker {
    constructor(options = {}) {
        this.failureThreshold = options.failureThreshold || 5;
        this.resetTimeout = options.resetTimeout || 30000; // 30 seconds
        this halfOpenMaxCalls = options.halfOpenMaxCalls || 1;
        
        this.state = 'CLOSED';
        this.failureCount = 0;
        this.lastFailureTime = null;
        this.halfOpenCalls = 0;
    }
    
    async execute(operation, context = {}) {
        if (this.state === 'OPEN') {
            if (this.shouldReset()) {
                this.transitionToHalfOpen();
            } else {
                throw new Error('Circuit breaker is OPEN');
            }
        }
        
        if (this.state === 'HALF-OPEN' && this.halfOpenCalls >= this.halfOpenMaxCalls) {
            throw new Error('Circuit breaker is HALF-OPEN, max calls reached');
        }
        
        try {
            const result = await operation();
            this.onSuccess();
            return result;
        } catch (error) {
            this.onFailure();
            throw error;
        }
    }
    
    onSuccess() {
        if (this.state === 'HALF-OPEN') {
            this.transitionToClosed();
        } else {
            this.failureCount = 0;
        }
    }
    
    onFailure() {
        this.failureCount++;
        this.lastFailureTime = Date.now();
        
        if (this.state === 'HALF-OPEN') {
            this.transitionToOpen();
        } else if (this.failureCount >= this.failureThreshold) {
            this.transitionToOpen();
        }
    }
    
    transitionToOpen() {
        this.state = 'OPEN';
        console.log('Circuit breaker OPENED');
    }
    
    transitionToHalfOpen() {
        this.state = 'HALF-OPEN';
        this.halfOpenCalls = 0;
        console.log('Circuit breaker HALF-OPEN');
    }
    
    transitionToClosed() {
        this.state = 'CLOSED';
        this.failureCount = 0;
        console.log('Circuit breaker CLOSED');
    }
    
    shouldReset() {
        return this.state === 'OPEN' && 
               this.lastFailureTime && 
               (Date.now() - this.lastFailureTime) > this.resetTimeout;
    }
    
    getState() {
        return {
            state: this.state,
            failureCount: this.failureCount,
            lastFailureTime: this.lastFailureTime
        };
    }
}

// Usage
const circuitBreaker = new CircuitBreaker({
    failureThreshold: 5,
    resetTimeout: 30000
});

async function callExternalService() {
    return circuitBreaker.execute(async () => {
        // Your external service call here
        const response = await fetch('https://api.example.com/data');
        return response.json();
    });
}
```

### Retry with Exponential Backoff

```javascript
async function retryWithBackoff(operation, options = {}) {
    const maxRetries = options.maxRetries || 3;
    const baseDelay = options.baseDelay || 1000;
    const maxDelay = options.maxDelay || 30000;
    const jitter = options.jitter !== false;
    
    let lastError;
    
    for (let attempt = 0; attempt <= maxRetries; attempt++) {
        try {
            return await operation();
        } catch (error) {
            lastError = error;
            
            if (attempt === maxRetries) {
                break;
            }
            
            // Calculate delay with exponential backoff
            let delay = Math.min(baseDelay * Math.pow(2, attempt), maxDelay);
            
            // Add jitter to prevent thundering herd
            if (jitter) {
                delay = delay * (0.5 + Math.random() * 0.5);
            }
            
            console.log(`Retry ${attempt + 1}/${maxRetries} after ${Math.round(delay)}ms`);
            await new Promise(resolve => setTimeout(resolve, delay));
        }
    }
    
    throw lastError;
}

// Usage
async function fetchWithRetry(url) {
    return retryWithBackoff(async () => {
        const response = await fetch(url);
        if (!response.ok) {
            throw new Error(`HTTP ${response.status}`);
        }
        return response.json();
    }, { maxRetries: 3, baseDelay: 1000 });
}
```

### Bulkhead Pattern

```javascript
class Bulkhead {
    constructor(options = {}) {
        this.maxConcurrent = options.maxConcurrent || 10;
        this.maxWaiting = options.maxWaiting || 100;
        this.timeout = options.timeout || 30000;
        
        this.active = 0;
        this.queue = [];
    }
    
    async execute(operation, context = {}) {
        return new Promise((resolve, reject) => {
            const entry = { operation, context, resolve, reject };
            
            if (this.active < this.maxConcurrent) {
                this.processEntry(entry);
            } else if (this.queue.length < this.maxWaiting) {
                this.queue.push(entry);
            } else {
                reject(new Error('Bulkhead is full'));
            }
        });
    }
    
    processEntry(entry) {
        this.active++;
        
        const timeoutId = setTimeout(() => {
            this.cleanup(entry);
            entry.reject(new Error('Bulkhead timeout'));
        }, this.timeout);
        
        entry.operation()
            .then(result => {
                clearTimeout(timeoutId);
                this.cleanup(entry);
                entry.resolve(result);
            })
            .catch(error => {
                clearTimeout(timeoutId);
                this.cleanup(entry);
                entry.reject(error);
            });
    }
    
    cleanup(entry) {
        this.active--;
        
        // Process next queued entry
        if (this.queue.length > 0 && this.active < this.maxConcurrent) {
            const nextEntry = this.queue.shift();
            this.processEntry(nextEntry);
        }
    }
    
    getStatus() {
        return {
            active: this.active,
            queued: this.queue.length,
            maxConcurrent: this.maxConcurrent,
            maxWaiting: this.maxWaiting
        };
    }
}

// Usage - Separate bulkheads for different operations
const criticalBulkhead = new Bulkhead({ maxConcurrent: 5, maxWaiting: 10 });
const nonCriticalBulkhead = new Bulkhead({ maxConcurrent: 20, maxWaiting: 50 });

// Critical operations get priority
async function processPayment(payment) {
    return criticalBulkhead.execute(() => paymentService.process(payment));
}

// Non-critical operations can be throttled
async function sendNotification(notification) {
    return nonCriticalBulkhead.execute(() => notificationService.send(notification));
}
```

### Health Check Implementation

```javascript
class HealthChecker {
    constructor() {
        this.checks = new Map();
        this.lastCheckResults = new Map();
    }
    
    addCheck(name, checkFn, options = {}) {
        this.checks.set(name, {
            fn: checkFn,
            timeout: options.timeout || 5000,
            critical: options.critical !== false
        });
    }
    
    async runAllChecks() {
        const results = {};
        let isHealthy = true;
        
        for (const [name, check] of this.checks) {
            try {
                const result = await Promise.race([
                    check.fn(),
                    new Promise((_, reject) => 
                        setTimeout(() => reject(new Error('Timeout')), check.timeout)
                    )
                ]);
                
                results[name] = { status: 'healthy', ...result };
                this.lastCheckResults.set(name, { status: 'healthy', time: Date.now() });
                
                if (check.critical && result.status !== 'healthy') {
                    isHealthy = false;
                }
            } catch (error) {
                results[name] = { status: 'unhealthy', error: error.message };
                this.lastCheckResults.set(name, { status: 'unhealthy', time: Date.now() });
                
                if (check.critical) {
                    isHealthy = false;
                }
            }
        }
        
        return {
            status: isHealthy ? 'healthy' : 'unhealthy',
            checks: results,
            timestamp: new Date().toISOString()
        };
    }
    
    getHealthEndpoint() {
        return async (req, res) => {
            const health = await this.runAllChecks();
            res.status(health.status === 'healthy' ? 200 : 503).json(health);
        };
    }
}

// Usage
const healthChecker = new HealthChecker();

// Database health check
healthChecker.addCheck('database', async () => {
    try {
        await db.query('SELECT 1');
        return { status: 'healthy', ping: 'ok' };
    } catch (error) {
        return { status: 'unhealthy', error: error.message };
    }
}, { critical: true });

// Redis health check
healthChecker.addCheck('redis', async () => {
    try {
        const ping = await redis.ping();
        return { status: 'healthy', ping };
    } catch (error) {
        return { status: 'unhealthy', error: error.message };
    }
}, { critical: true });

// External service health check (non-critical)
healthChecker.addCheck('external-api', async () => {
    try {
        const response = await fetch('https://api.example.com/health', { timeout: 2000 });
        return { status: response.ok ? 'healthy' : 'degraded', status: response.status };
    } catch (error) {
        return { status: 'unhealthy', error: error.message };
    }
}, { critical: false, timeout: 2000 });

// Add to Express app
app.get('/health', healthChecker.getHealthEndpoint());
```

## Anti-Pattern

### Common Reliability Mistakes

**1. Retry Storm (Thundering Herd)**
```
❌ BAD: All clients retry at the same time
for (const client of clients) {
    await retry(operation, { delay: 5000 }); // All retry after 5s!
}
```

```
✅ GOOD: Add jitter to spread retries
for (const client of clients) {
    await retry(operation, { 
        delay: 5000 + Math.random() * 5000 
    });
}
```

**2. Infinite Retry Loops**
```
❌ BAD: No limit on retries
while (true) {
    try {
        return await operation();
    } catch (error) {
        await sleep(1000);
    }
}
```

```
✅ GOOD: Limit retries with backoff
for (let i = 0; i < maxRetries; i++) {
    try {
        return await operation();
    } catch (error) {
        if (i === maxRetries - 1) throw error;
        await sleep(Math.pow(2, i) * 1000);
    }
}
```

**3. Catching All Exceptions**
```
❌ BAD: Swallowing all errors
try {
    await operation();
} catch (error) {
    console.log('Error occurred'); // Lost important info!
}
```

```
✅ GOOD: Specific error handling with logging
try {
    await operation();
} catch (error) {
    if (error instanceof RetryableError) {
        // Retry logic
    } else {
        logger.error('Non-retryable error', error);
        throw error;
    }
}
```

**4. No Circuit Breaker on External Calls**
```
❌ BAD: Direct calls without protection
async function getExternalData() {
    return await fetch('https://external-api.com/data'); // Can hang forever!
}
```

```
✅ GOOD: Protected external calls
async function getExternalData() {
    return await circuitBreaker.execute(async () => {
        return await fetch('https://external-api.com/data', { timeout: 5000 });
    });
}
```

## Related Patterns

- **[Scalability Patterns](./04-Scalability-Patterns.md)** - Scaling for reliability
- **[Availability Patterns](./06-Availability-Patterns.md)** - High availability techniques
- **[Fault Tolerance](../05-Safety-Engineering/02-Fault-Tolerance.md)** - Resilience patterns
- **[Error Handling](../05-Safety-Engineering/03-Error-Handling.md)** - Exception strategies
- **[Monitoring](../10-Observability/02-Monitoring.md)** - Health monitoring
- **[Deployment Strategies](../11-DevOps/02-Deployment-Strategies.md)** - Safe deployments