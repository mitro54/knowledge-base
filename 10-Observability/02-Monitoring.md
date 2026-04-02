# Monitoring

## Title & Summary

Monitoring is the continuous observation of system health, performance, and availability through automated collection and analysis of operational data. Effective monitoring enables proactive issue detection, capacity planning, and ensures system reliability through real-time visibility.

## Problem Statement

Systems without proper monitoring suffer from:

- **Reactive Problem Detection**: Issues discovered only after users report them
- **Unknown System State**: No visibility into current health or performance
- **Capacity Blindspots**: Unable to predict resource exhaustion
- **Slow Incident Response**: No early warning system for degradation
- **SLA Violations**: Cannot prove or measure service level compliance
- **Post-Mortem Difficulties**: Lack of historical data for analysis

## Solution

### Core Monitoring Components

**Health Checks**
- Liveness probes: Is the service alive?
- Readiness probes: Is the service ready to serve traffic?
- Startup probes: Is the service starting up correctly?

**Metrics Collection**
- System metrics: CPU, memory, disk, network
- Application metrics: Request rates, latencies, error rates
- Business metrics: Orders, users, revenue

**Alerting Integration**
- Threshold-based alerts
- Anomaly detection
- Multi-channel notifications

### Implementation Example

```python
import time
import psutil
from prometheus_client import Counter, Histogram, Gauge, Summary
from dataclasses import dataclass
from typing import Dict, List

# Define metrics
REQUEST_COUNT = Counter(
    'http_requests_total',
    'Total HTTP requests',
    ['method', 'endpoint', 'status_code']
)

REQUEST_DURATION = Histogram(
    'http_request_duration_seconds',
    'HTTP request duration in seconds',
    ['method', 'endpoint'],
    buckets=(0.005, 0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1.0, 2.5, 5.0, 10.0)
)

ACTIVE_CONNECTIONS = Gauge(
    'active_connections',
    'Number of active connections'
)

MEMORY_USAGE = Gauge(
    'memory_usage_bytes',
    'Current memory usage in bytes'
)

CPU_USAGE = Gauge(
    'cpu_usage_percent',
    'Current CPU usage percentage'
)

@dataclass
class HealthStatus:
    status: str  # 'healthy', 'degraded', 'unhealthy'
    checks: Dict[str, bool]
    message: str = ""

class SystemMonitor:
    def __init__(self, thresholds: Dict = None):
        self.thresholds = thresholds or {
            'cpu_warning': 70,
            'cpu_critical': 90,
            'memory_warning': 70,
            'memory_critical': 90,
            'disk_warning': 80,
            'disk_critical': 95,
            'request_latency_warning_ms': 500,
            'request_latency_critical_ms': 1000
        }
    
    def get_system_metrics(self) -> Dict:
        """Collect system-level metrics"""
        metrics = {
            'cpu_percent': psutil.cpu_percent(interval=1),
            'memory_percent': psutil.virtual_memory().percent,
            'memory_available': psutil.virtual_memory().available,
            'disk_percent': psutil.disk_usage('/').percent,
            'disk_free': psutil.disk_usage('/').free,
            'network_io': self._get_network_io()
        }
        
        # Update Prometheus gauges
        CPU_USAGE.set(metrics['cpu_percent'])
        MEMORY_USAGE.set(psutil.virtual_memory().used)
        
        return metrics
    
    def _get_network_io(self) -> Dict:
        """Get network I/O statistics"""
        net_io = psutil.net_io_counters()
        return {
            'bytes_sent': net_io.bytes_sent,
            'bytes_recv': net_io.bytes_recv,
            'packets_sent': net_io.packets_sent,
            'packets_recv': net_io.packets_recv
        }
    
    def check_health(self) -> HealthStatus:
        """Perform comprehensive health check"""
        metrics = self.get_system_metrics()
        checks = {}
        
        # CPU check
        cpu_ok = metrics['cpu_percent'] < self.thresholds['cpu_critical']
        checks['cpu'] = cpu_ok
        
        # Memory check
        memory_ok = metrics['memory_percent'] < self.thresholds['memory_critical']
        checks['memory'] = memory_ok
        
        # Disk check
        disk_ok = metrics['disk_percent'] < self.thresholds['disk_critical']
        checks['disk'] = disk_ok
        
        # Determine overall status
        if all(checks.values()):
            status = 'healthy'
            message = 'All systems operational'
        elif any(checks.values()):
            status = 'degraded'
            failed = [k for k, v in checks.items() if not v]
            message = f'Degraded: {", ".join(failed)}'
        else:
            status = 'unhealthy'
            message = 'Critical system issues detected'
        
        return HealthStatus(status=status, checks=checks, message=message)
    
    def record_request(self, method: str, endpoint: str, status_code: int, duration_ms: float):
        """Record HTTP request metrics"""
        REQUEST_COUNT.labels(method=method, endpoint=endpoint, 
                           status_code=status_code).inc()
        REQUEST_DURATION.labels(method=method, endpoint=endpoint).observe(duration_ms / 1000)

# Usage in application
monitor = SystemMonitor()

def health_check_endpoint():
    """HTTP endpoint for health checks"""
    health = monitor.check_health()
    return {
        'status': health.status,
        'checks': health.checks,
        'message': health.message,
        'timestamp': time.time()
    }
```

### Kubernetes Health Check Configuration

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: application-pod
spec:
  containers:
  - name: app
    image: myapp:latest
    ports:
    - containerPort: 8080
    
    # Liveness probe - restart if unhealthy
    livenessProbe:
      httpGet:
        path: /health/live
        port: 8080
      initialDelaySeconds: 30
      periodSeconds: 10
      timeoutSeconds: 5
      failureThreshold: 3
    
    # Readiness probe - remove from service if unhealthy
    readinessProbe:
      httpGet:
        path: /health/ready
        port: 8080
      initialDelaySeconds: 5
      periodSeconds: 5
      timeoutSeconds: 3
      failureThreshold: 3
    
    # Startup probe - for slow-starting containers
    startupProbe:
      httpGet:
        path: /health/startup
        port: 8080
      failureThreshold: 30
      periodSeconds: 10
```

## When to Use

**Always Monitor:**
- Service availability (uptime)
- Response times and latencies
- Error rates and failure counts
- Resource utilization (CPU, memory, disk, network)
- Queue lengths and backlog
- External dependency health

**Monitor Based on Context:**
- Business metrics (orders, revenue, users)
- Custom application metrics
- Database connection pool status
- Cache hit rates
- Message queue throughput

**Alerting Thresholds:**
- Warning: 70-80% of capacity, degraded performance
- Critical: 90%+ of capacity, significant degradation
- Emergency: Service unavailable, data loss risk

## Tradeoffs

| Aspect | Benefits | Costs |
|--------|----------|-------|
| **High Cardinality** | Detailed insights, precise alerting | Storage costs, query performance |
| **Frequent Collection** | Real-time visibility, fast detection | Resource overhead, storage costs |
| **Many Metrics** | Comprehensive coverage | Complexity, alert fatigue |
| **Low Thresholds** | Early warning, proactive response | False positives, alert fatigue |
| **High Thresholds** | Fewer false positives | Late detection, more impact |

## Anti-Pattern

### ❌ Common Monitoring Anti-Patterns

```python
# ANTI-PATTERN 1: Monitoring everything at highest granularity
def record_user_action(user_id, action, details):
    # Creating a metric for every user action creates cardinality explosion
    metrics.gauge(f'user_{user_id}_last_action', action)
    metrics.counter(f'user_{user_id}_{action}_count').inc()

# ANTI-PATTERN 2: No differentiation between health check types
@app.route('/health')
def health():
    # Single endpoint for everything - can't distinguish liveness vs readiness
    return {'status': 'ok'}

# ANTI-PATTERN 3: Alerting on everything
def setup_alerts():
    # Alerting on every metric creates alert fatigue
    alert_if(cpu > 50)
    alert_if(memory > 50)
    alert_if(request_latency > 100ms)
    alert_if(error_rate > 0.1%)
    # Too many low-threshold alerts

# ANTI-PATTERN 4: Monitoring only infrastructure, not application
def monitor_system():
    # Only checking infrastructure metrics
    check_cpu_usage()
    check_memory_usage()
    check_disk_space()
    # Missing: application health, business metrics, user experience
```

### ✅ Correct Patterns

```python
# CORRECT: Appropriate metric cardinality
def record_user_action(user_id, action, details):
    # Aggregate at appropriate level
    metrics.counter('user_actions_total', labels={'action': action}).inc()
    metrics.histogram('user_action_duration', labels={'action': action}).observe(duration)

# CORRECT: Separate health check endpoints
@app.route('/health/live')
def liveness():
    """Is the application alive?"""
    return {'status': 'alive'}

@app.route('/health/ready')
def readiness():
    """Is the application ready to serve traffic?"""
    if not database_connected():
        return {'status': 'not_ready'}, 503
    if cache_unavailable():
        return {'status': 'not_ready'}, 503
    return {'status': 'ready'}

# CORRECT: Thoughtful alerting with appropriate thresholds
def setup_alerts():
    # Tiered alerting with appropriate thresholds
    alert_if(cpu > 90, severity='critical', message='High CPU usage')
    alert_if(cpu > 75, severity='warning', message='Elevated CPU usage')
    alert_if(error_rate > 1%, severity='critical', message='High error rate')
    alert_if(request_latency_p99 > 2s, severity='warning', message='High latency')

# CORRECT: Comprehensive monitoring including business metrics
def monitor_system():
    # Infrastructure metrics
    check_cpu_usage()
    check_memory_usage()
    
    # Application metrics
    check_request_rate()
    check_error_rate()
    check_latency_percentiles()
    
    # Business metrics
    check_order_rate()
    check_payment_success_rate()
    check_active_users()
```

## Related Patterns

- **[Logging](./01-Logging.md)** - Event recording and analysis
- **[Tracing](./03-Tracing.md)** - Request flow tracking
- **[Metrics](./04-Metrics.md)** - Quantitative measurements
- **[Alerting](./05-Alerting.md)** - Notification on issues
- **[Reliability](../01-System-Design/05-Reliability.md)** - System reliability principles