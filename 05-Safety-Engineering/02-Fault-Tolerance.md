# Fault Tolerance

## Title & Summary

Fault tolerance is the ability of a system to continue operating properly in the event of the failure of some of its components. This document covers patterns and techniques for building resilient systems that can withstand failures without complete system outage.

## Problem Statement

System failures are inevitable in production environments:

- **Hardware Failures**: Server crashes, disk failures, network equipment malfunctions
- **Software Bugs**: Unhandled exceptions, memory leaks, deadlocks
- **Network Issues**: Packet loss, latency spikes, connection timeouts
- **External Dependencies**: Third-party API failures, database unavailability
- **Resource Exhaustion**: Memory overflow, CPU saturation, disk space depletion
- **Cascading Failures**: One component failure triggering failures in dependent components

Without fault tolerance, a single point of failure can bring down entire systems, causing downtime, data loss, and revenue impact.

## Solution

### Core Fault Tolerance Patterns

**1. Circuit Breaker Pattern**
```
States: CLOSED → OPEN → HALF-OPEN → CLOSED
```

- **CLOSED**: Normal operation, requests flow through
- **OPEN**: Failure threshold reached, requests fail fast
- **HALF-OPEN**: Testing if service recovered, limited requests allowed

**2. Retry Pattern**
```
Retry with Exponential Backoff + Jitter
```

- **Simple Retry**: Fixed number of attempts
- **Exponential Backoff**: Increasing delay between retries
- **Jitter**: Randomization to prevent thundering herd

**3. Bulkhead Pattern**
```
Isolate resources into separate pools
```

- Resource isolation by component or request type
- Prevents single component failure from exhausting all resources
- Similar to ship bulkheads containing water damage

**4. Timeout Pattern**
```
Set maximum wait time for operations
```

- Prevents indefinite waiting on failed operations
- Enables faster failure detection
- Should be combined with retry logic

**5. Fallback Pattern**
```
Provide alternative when primary fails
```

- Cached responses
- Default values
- Degraded functionality
- Alternative service providers

**6. Health Check Pattern**
```
Regularly verify component health
```

- Liveness probes: Is the service alive?
- Readiness probes: Is the service ready to serve traffic?
- Startup probes: Is the service starting up correctly?

### Implementation Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    Load Balancer                             │
├──────────────────┬──────────────────┬───────────────────────┤
│   Circuit        │   Circuit        │   Circuit             │
│   Breaker        │   Breaker        │   Breaker             │
├──────────────────┼──────────────────┼───────────────────────┤
│   Bulkhead       │   Bulkhead       │   Bulkhead            │
│   Pool A         │   Pool B         │   Pool C              │
├──────────────────┼──────────────────┼───────────────────────┤
│   Service A      │   Service B      │   Service C           │
│   (Replicas)     │   (Replicas)     │   (Replicas)          │
└──────────────────┴──────────────────┴───────────────────────┘
```

## When to Use

| Pattern | Use When | Typical Configuration |
|---------|----------|----------------------|
| Circuit Breaker | External service calls, database queries | 5 failures, 30s timeout, 10s half-open |
| Retry with Backoff | Transient failures, network issues | 3 retries, 1s base, 2x multiplier |
| Bulkhead | Multi-tenant systems, mixed workloads | Separate pools per tenant/workload |
| Timeout | All external calls, long operations | 1-30s depending on operation type |
| Fallback | Non-critical features, read operations | Cache TTL, default values |
| Health Check | All services in production | 10-30s interval, 3 failures = unhealthy |

## Tradeoffs

### Circuit Breaker Tradeoffs

| Aspect | Benefit | Cost |
|--------|---------|------|
| Failure Isolation | Prevents cascading failures | May hide intermittent issues |
| Fast Failure | Quick fail-over to fallback | False positives possible |
| Recovery Testing | Half-open state validates recovery | May overload recovering service |

### Retry Tradeoffs

| Aspect | Benefit | Cost |
|--------|---------|------|
| Resilience | Handles transient failures | Increases latency on failure |
| Success Rate | Improves overall success rate | May amplify failures (thundering herd) |
| Complexity | Simple to implement | Can mask underlying issues |

### Bulkhead Tradeoffs

| Aspect | Benefit | Cost |
|--------|---------|------|
| Isolation | Failure containment | Resource fragmentation |
| Predictability | Consistent performance per pool | Higher total resource usage |
| Multi-tenancy | Tenant isolation | More complex configuration |

## Implementation Example

### Comprehensive Fault Tolerance Implementation

```python
import asyncio
import random
import time
from enum import Enum
from functools import wraps
from dataclasses import dataclass
from typing import Callable, Optional, Any
from collections import defaultdict

# Circuit Breaker States
class CircuitState(Enum):
    CLOSED = "closed"
    OPEN = "open"
    HALF_OPEN = "half_open"

@dataclass
class CircuitBreakerConfig:
    failure_threshold: int = 5
    recovery_timeout: float = 30.0
    half_open_max_calls: int = 3
    success_threshold: int = 2

class CircuitBreaker:
    def __init__(self, config: CircuitBreakerConfig = None):
        self.config = config or CircuitBreakerConfig()
        self.state = CircuitState.CLOSED
        self.failure_count = 0
        self.success_count = 0
        self.last_failure_time = None
        self._half_open_calls = 0
        
    def _is_recovery_period_elapsed(self) -> bool:
        if self.last_failure_time is None:
            return True
        elapsed = time.time() - self.last_failure_time
        return elapsed >= self.config.recovery_timeout
    
    def can_execute(self) -> bool:
        if self.state == CircuitState.CLOSED:
            return True
        elif self.state == CircuitState.OPEN:
            if self._is_recovery_period_elapsed():
                self.state = CircuitState.HALF_OPEN
                self._half_open_calls = 0
                return True
            return False
        else:  # HALF_OPEN
            return self._half_open_calls < self.config.half_open_max_calls
    
    def record_success(self):
        if self.state == CircuitState.HALF_OPEN:
            self.success_count += 1
            if self.success_count >= self.config.success_threshold:
                self._reset()
        elif self.state == CircuitState.CLOSED:
            self.failure_count = max(0, self.failure_count - 1)
    
    def record_failure(self):
        self.failure_count += 1
        self.last_failure_time = time.time()
        
        if self.state == CircuitState.HALF_OPEN:
            self._open_circuit()
        elif self.state == CircuitState.CLOSED:
            if self.failure_count >= self.config.failure_threshold:
                self._open_circuit()
    
    def _open_circuit(self):
        self.state = CircuitState.OPEN
        self.success_count = 0
        
    def _reset(self):
        self.state = CircuitState.CLOSED
        self.failure_count = 0
        self.success_count = 0
        
    def get_state(self) -> str:
        return self.state.value

def circuit_breaker(config: CircuitBreakerConfig = None):
    """Decorator for circuit breaker pattern"""
    cb = CircuitBreaker(config)
    
    def decorator(func: Callable):
        @wraps(func)
        async def wrapper(*args, **kwargs):
            if not cb.can_execute():
                raise ServiceUnavailableError(
                    f"Circuit breaker is {cb.get_state()} for {func.__name__}"
                )
            
            try:
                result = await func(*args, **kwargs)
                cb.record_success()
                return result
            except Exception as e:
                cb.record_failure()
                raise
        
        wrapper.circuit_breaker = cb
        return wrapper
    return decorator

# Retry with Exponential Backoff
@dataclass
class RetryConfig:
    max_attempts: int = 3
    base_delay: float = 1.0
    max_delay: float = 60.0
    exponential_base: float = 2.0
    jitter: bool = True

class RetryHandler:
    def __init__(self, config: RetryConfig = None):
        self.config = config or RetryConfig()
        
    def calculate_delay(self, attempt: int) -> float:
        delay = self.config.base_delay * (self.config.exponential_base ** attempt)
        delay = min(delay, self.config.max_delay)
        if self.config.jitter:
            delay *= (0.5 + random.random() * 0.5)  # 50-100% of calculated delay
        return delay
    
    async def execute_with_retry(
        self, 
        func: Callable, 
        *args, 
        retryable_exceptions: tuple = (Exception,),
        **kwargs
    ) -> Any:
        last_exception = None
        
        for attempt in range(self.config.max_attempts):
            try:
                if asyncio.iscoroutinefunction(func):
                    return await func(*args, **kwargs)
                else:
                    return func(*args, **kwargs)
            except retryable_exceptions as e:
                last_exception = e
                if attempt < self.config.max_attempts - 1:
                    delay = self.calculate_delay(attempt)
                    await asyncio.sleep(delay)
        
        raise last_exception

def retry(config: RetryConfig = None):
    """Decorator for retry pattern"""
    handler = RetryHandler(config)
    
    def decorator(func: Callable):
        @wraps(func)
        async def wrapper(*args, **kwargs):
            return await handler.execute_with_retry(func, *args, **kwargs)
        return wrapper
    return decorator

# Bulkhead Pattern
class Bulkhead:
    def __init__(self, pool_size: int = 10):
        self.semaphore = asyncio.Semaphore(pool_size)
        self.pool_size = pool_size
        
    async def execute(self, func: Callable, *args, **kwargs) -> Any:
        async with self.semaphore:
            if asyncio.iscoroutinefunction(func):
                return await func(*args, **kwargs)
            else:
                return func(*args, **kwargs)
    
    def get_available_slots(self) -> int:
        return self.semaphore._value

def bulkhead(pool_size: int = 10):
    """Decorator for bulkhead pattern"""
    bulkhead = Bulkhead(pool_size)
    
    def decorator(func: Callable):
        @wraps(func)
        async def wrapper(*args, **kwargs):
            return await bulkhead.execute(func, *args, **kwargs)
        wrapper.bulkhead = bulkhead
        return wrapper
    return decorator

# Timeout Pattern
async def with_timeout(coro, timeout: float):
    """Execute coroutine with timeout"""
    try:
        return await asyncio.wait_for(coro, timeout=timeout)
    except asyncio.TimeoutError:
        raise TimeoutError(f"Operation timed out after {timeout} seconds")

def timeout(seconds: float):
    """Decorator for timeout pattern"""
    def decorator(func: Callable):
        @wraps(func)
        async def wrapper(*args, **kwargs):
            return await with_timeout(func(*args, **kwargs), seconds)
        return wrapper
    return decorator

# Fallback Pattern
class FallbackHandler:
    def __init__(self, fallback_value: Any = None, fallback_func: Callable = None):
        self.fallback_value = fallback_value
        self.fallback_func = fallback_func
        
    async def execute(
        self, 
        primary_func: Callable, 
        *args, 
        exceptions: tuple = (Exception,),
        **kwargs
    ) -> Any:
        try:
            if asyncio.iscoroutinefunction(primary_func):
                return await primary_func(*args, **kwargs)
            else:
                return primary_func(*args, **kwargs)
        except exceptions:
            if self.fallback_func:
                if asyncio.iscoroutinefunction(self.fallback_func):
                    return await self.fallback_func(*args, **kwargs)
                else:
                    return self.fallback_func(*args, **kwargs)
            return self.fallback_value

def fallback(value: Any = None, func: Callable = None):
    """Decorator for fallback pattern"""
    handler = FallbackHandler(value, func)
    
    def decorator(primary_func: Callable):
        @wraps(primary_func)
        async def wrapper(*args, **kwargs):
            return await handler.execute(primary_func, *args, **kwargs)
        return wrapper
    return decorator

# Combined Pattern Example
class ResilientService:
    def __init__(self):
        self.circuit_breaker = CircuitBreaker()
        self.retry_handler = RetryHandler(RetryConfig(max_attempts=3, base_delay=0.5))
        self.bulkhead = Bulkhead(pool_size=5)
        
    @circuit_breaker(CircuitBreakerConfig(failure_threshold=5, recovery_timeout=30))
    @retry(RetryConfig(max_attempts=3, base_delay=0.5))
    @timeout(10.0)
    async def call_external_service(self, request: dict) -> dict:
        """External service call with full resilience"""
        # Simulate external call
        await asyncio.sleep(random.uniform(0.1, 0.5))
        if random.random() < 0.1:  # 10% failure rate
            raise ConnectionError("External service unavailable")
        return {"status": "success", "data": request}
    
    @bulkhead(pool_size=10)
    @fallback(value={"cached": True, "data": []})
    async def get_user_data(self, user_id: str) -> dict:
        """Get user data with bulkhead and fallback"""
        return await self.call_external_service({"user_id": user_id})

# Health Check Implementation
class HealthChecker:
    def __init__(self):
        self.checks = {}
        self.health_status = {
            "status": "healthy",
            "checks": {},
            "timestamp": None
        }
        
    def register_check(self, name: str, check_func: Callable):
        """Register a health check"""
        self.checks[name] = check_func
        
    async def run_all_checks(self):
        """Run all registered health checks"""
        results = {}
        all_healthy = True
        
        for name, check_func in self.checks.items():
            try:
                if asyncio.iscoroutinefunction(check_func):
                    result = await check_func()
                else:
                    result = check_func()
                results[name] = {"status": "healthy", "details": result}
            except Exception as e:
                results[name] = {"status": "unhealthy", "error": str(e)}
                all_healthy = False
        
        self.health_status = {
            "status": "healthy" if all_healthy else "unhealthy",
            "checks": results,
            "timestamp": time.time()
        }
        
        return self.health_status

# Usage Example
async def main():
    service = ResilientService()
    health_checker = HealthChecker()
    
    # Register health checks
    health_checker.register_check("database", lambda: check_database_connection())
    health_checker.register_check("cache", lambda: check_cache_connection())
    health_checker.register_check("external_api", lambda: check_external_api())
    
    # Run health checks
    health = await health_checker.run_all_checks()
    print(f"System Health: {health['status']}")
    
    # Call resilient service
    try:
        result = await service.get_user_data("user123")
        print(f"Result: {result}")
    except Exception as e:
        print(f"Service call failed: {e}")

# Custom Exceptions
class ServiceUnavailableError(Exception):
    """Raised when service is unavailable"""
    pass

class CircuitBreakerOpenError(ServiceUnavailableError):
    """Raised when circuit breaker is open"""
    pass
```

## Anti-Pattern

### ❌ Fault Tolerance Anti-Patterns

**1. Retry Storm (Thundering Herd)**
```python
# ANTI-PATTERN: All clients retry at the same time
for _ in range(MAX_RETRIES):
    try:
        return service.call()
    except:
        continue  # Immediate retry - causes thundering herd

# PATTERN: Exponential backoff with jitter
for attempt in range(MAX_RETRIES):
    try:
        return await service.call()
    except:
        delay = base_delay * (2 ** attempt) * random.uniform(0.5, 1.5)
        await asyncio.sleep(delay)
```

**2. Infinite Retry Loops**
```python
# ANTI-PATTERN: No max retry limit
while True:
    try:
        return service.call()
    except:
        await asyncio.sleep(1)

# PATTERN: Limited retries with proper backoff
for attempt in range(MAX_RETRIES):
    try:
        return await service.call()
    except:
        if attempt == MAX_RETRIES - 1:
            raise
        await asyncio.sleep(calculate_backoff(attempt))
```

**3. Catching All Exceptions**
```python
# ANTI-PATTERN: Catching everything including programming errors
try:
    result = service.call()
except:
    return fallback_value  # Hides bugs!

# PATTERN: Catch only expected exceptions
try:
    result = await service.call()
except (ConnectionError, TimeoutError, ServiceUnavailableError):
    return await fallback_handler.get_cached_value()
```

**4. No Circuit Breaker for External Calls**
```python
# ANTI-PATTERN: Direct calls without protection
def get_user_data(user_id):
    return external_api.get_user(user_id)  # Can hang forever

# PATTERN: Protected external calls
@circuit_breaker()
@timeout(5.0)
@retry()
async def get_user_data(user_id):
    return await external_api.get_user(user_id)
```

**5. Shared Resources Without Bulkhead**
```python
# ANTI-PATTERN: All requests share same connection pool
pool = ConnectionPool(size=10)
async def handle_request():
    conn = await pool.get()  # One slow request blocks all

# PATTERN: Bulkhead isolation
user_pool = ConnectionPool(size=5)
admin_pool = ConnectionPool(size=3)
report_pool = ConnectionPool(size=2)
```

## Related Patterns

- **[Security Patterns](../05-Safety-Engineering/01-Security-Patterns.md)** - Secure fallback mechanisms
- **[Error Handling](../05-Safety-Engineering/03-Error-Handling.md)** - Proper error propagation
- **[Scalability Patterns](../01-System-Design/04-Scalability-Patterns.md)** - Scaling for fault tolerance
- **[Reliability Patterns](../01-System-Design/05-Reliability-Patterns.md)** - Overall system reliability
- **[Monitoring](../10-Observability/02-Monitoring.md)** - Circuit breaker state monitoring
- **[Logging](../10-Observability/01-Logging.md)** - Failure and recovery logging