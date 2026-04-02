# Metrics

## Title & Summary

Metrics are quantitative measurements of system behavior, performance, and business operations collected over time. Effective metrics collection and analysis enables data-driven decisions, capacity planning, performance optimization, and early detection of anomalies through statistical analysis.

## Problem Statement

Systems without proper metrics face significant challenges:

- **Blind Operations**: No quantitative data to support decisions
- **Unknown Performance Baselines**: Cannot measure improvement or degradation
- **Capacity Planning Difficulties**: Unable to predict resource needs
- **SLA Measurement Impossible**: Cannot prove service level compliance
- **Trend Analysis Missing**: No historical data for pattern recognition
- **Anomaly Detection Absent**: Cannot identify unusual behavior automatically

## Solution

### Core Metric Types

**Counters** - Monotonically increasing values (e.g., request counts, errors)
**Gauges** - Instantaneous values that can go up or down (e.g., memory usage, temperature)
**Histograms** - Value distributions with buckets (e.g., response times)
**Summaries** - Pre-computed percentiles (e.g., p50, p95, p99 latencies)

### Implementation Example

```python
import time
import threading
from collections import defaultdict
from dataclasses import dataclass, field
from typing import Dict, List, Optional
from enum import Enum

class MetricType(Enum):
    COUNTER = "counter"
    GAUGE = "gauge"
    HISTOGRAM = "histogram"
    SUMMARY = "summary"

@dataclass
class Metric:
    name: str
    metric_type: MetricType
    description: str
    labels: List[str] = field(default_factory=list)
    value: float = 0.0
    timestamp: float = field(default_factory=time.time)

class Counter:
    """Monotonically increasing counter"""
    def __init__(self, name: str, description: str, labels: List[str] = None):
        self.name = name
        self.description = description
        self.labels = labels or []
        self._values: Dict[tuple, float] = defaultdict(float)
        self._lock = threading.Lock()
    
    def inc(self, value: float = 1.0, label_values: List[str] = None) -> 'Counter':
        with self._lock:
            if label_values:
                key = tuple(zip(self.labels, label_values))
                self._values[key] += value
            else:
                self._values[()] += value
        return self
    
    def get(self, label_values: List[str] = None) -> float:
        with self._lock:
            if label_values:
                key = tuple(zip(self.labels, label_values))
                return self._values.get(key, 0.0)
            return sum(self._values.values())
    
    def collect(self) -> List[Dict]:
        metrics = []
        for key, value in self._values.items():
            metric = {
                'name': self.name,
                'type': MetricType.COUNTER.value,
                'help': self.description,
                'value': value,
                'labels': dict(key) if key else {}
            }
            metrics.append(metric)
        return metrics

class Gauge:
    """Instantaneous value that can go up or down"""
    def __init__(self, name: str, description: str, labels: List[str] = None):
        self.name = name
        self.description = description
        self.labels = labels or []
        self._values: Dict[tuple, float] = {}
        self._lock = threading.Lock()
    
    def set(self, value: float, label_values: List[str] = None) -> 'Gauge':
        with self._lock:
            if label_values:
                key = tuple(zip(self.labels, label_values))
                self._values[key] = value
            else:
                self._values[()] = value
        return self
    
    def inc(self, value: float = 1.0, label_values: List[str] = None) -> 'Gauge':
        current = self.get(label_values)
        self.set(current + value, label_values)
        return self
    
    def dec(self, value: float = 1.0, label_values: List[str] = None) -> 'Gauge':
        current = self.get(label_values)
        self.set(current - value, label_values)
        return self
    
    def get(self, label_values: List[str] = None) -> float:
        with self._lock:
            if label_values:
                key = tuple(zip(self.labels, label_values))
                return self._values.get(key, 0.0)
            if () in self._values:
                return self._values[()]
            return 0.0
    
    def collect(self) -> List[Dict]:
        metrics = []
        for key, value in self._values.items():
            metric = {
                'name': self.name,
                'type': MetricType.GAUGE.value,
                'help': self.description,
                'value': value,
                'labels': dict(key) if key else {}
            }
            metrics.append(metric)
        return metrics

class Histogram:
    """Distribution of values into buckets"""
    def __init__(self, name: str, description: str, labels: List[str] = None,
                 buckets: List[float] = None):
        self.name = name
        self.description = description
        self.labels = labels or []
        self.buckets = buckets or [0.005, 0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 
                                   1.0, 2.5, 5.0, 10.0, float('inf')]
        self._counts: Dict[tuple, List[int]] = defaultdict(lambda: [0] * (len(self.buckets) + 1))
        self._sums: Dict[tuple, float] = defaultdict(float)
        self._lock = threading.Lock()
    
    def observe(self, value: float, label_values: List[str] = None) -> 'Histogram':
        with self._lock:
            key = tuple(zip(self.labels, label_values)) if label_values else ()
            self._sums[key] += value
            
            for i, bucket in enumerate(self.buckets):
                if value <= bucket:
                    self._counts[key][i] += 1
                    break
            else:
                self._counts[key][len(self.buckets)] += 1  # +Inf bucket
        
        return self
    
    def get_percentile(self, percentile: float, label_values: List[str] = None) -> float:
        with self._lock:
            key = tuple(zip(self.labels, label_values)) if label_values else ()
            counts = self._counts[key]
            total = sum(counts[:-1])  # Exclude +Inf bucket
            
            if total == 0:
                return 0.0
            
            target = percentile / 100.0 * total
            cumulative = 0
            
            for i, count in enumerate(counts[:-1]):
                cumulative += count
                if cumulative >= target:
                    return self.buckets[i]
            
            return self.buckets[-1]
    
    def collect(self) -> List[Dict]:
        metrics = []
        for key in set(list(self._counts.keys()) + list(self._sums.keys())):
            counts = self._counts[key]
            total = sum(counts[:-1])
            
            # Count metric
            metrics.append({
                'name': f'{self.name}_count',
                'type': MetricType.COUNTER.value,
                'help': f'Count of {self.description}',
                'value': total,
                'labels': dict(key) if key else {}
            })
            
            # Sum metric
            metrics.append({
                'name': f'{self.name}_sum',
                'type': MetricType.COUNTER.value,
                'help': f'Sum of {self.description}',
                'value': self._sums.get(key, 0.0),
                'labels': dict(key) if key else {}
            })
            
            # Bucket metrics
            for i, bucket in enumerate(self.buckets):
                metrics.append({
                    'name': f'{self.name}_bucket',
                    'type': MetricType.COUNTER.value,
                    'help': f'Buckets for {self.description}',
                    'value': sum(counts[:i+1]),
                    'labels': dict(key) if key else {},
                    'le': str(bucket)
                })
        
        return metrics

# Application Metrics
class ApplicationMetrics:
    def __init__(self):
        # Request metrics
        self.http_requests_total = Counter(
            'http_requests_total',
            'Total HTTP requests',
            labels=['method', 'endpoint', 'status_code']
        )
        
        self.http_request_duration = Histogram(
            'http_request_duration_seconds',
            'HTTP request duration in seconds',
            labels=['method', 'endpoint'],
            buckets=[0.01, 0.05, 0.1, 0.25, 0.5, 1.0, 2.5, 5.0, 10.0]
        )
        
        # System metrics
        self.active_connections = Gauge(
            'active_connections',
            'Number of active connections'
        )
        
        self.memory_usage_bytes = Gauge(
            'memory_usage_bytes',
            'Current memory usage in bytes'
        )
        
        self.cpu_usage_percent = Gauge(
            'cpu_usage_percent',
            'Current CPU usage percentage'
        )
        
        # Business metrics
        self.orders_total = Counter(
            'orders_total',
            'Total orders processed',
            labels=['status', 'payment_method']
        )
        
        self.order_value = Histogram(
            'order_value_dollars',
            'Order value in dollars',
            buckets=[10, 25, 50, 100, 250, 500, 1000, 2500, 5000, float('inf')]
        )
        
        self.revenue_total = Counter(
            'revenue_total_dollars',
            'Total revenue in dollars',
            labels=['currency']
        )
    
    def record_request(self, method: str, endpoint: str, status_code: int, 
                      duration_ms: float):
        self.http_requests_total.inc(
            label_values=[method, endpoint, str(status_code)]
        )
        self.http_request_duration.observe(
            duration_ms / 1000,
            label_values=[method, endpoint]
        )
    
    def record_order(self, status: str, payment_method: str, value: float):
        self.orders_total.inc(
            label_values=[status, payment_method]
        )
        self.order_value.observe(value)
        self.revenue_total.inc(value, label_values=['USD'])

# Usage
metrics = ApplicationMetrics()

def handle_request(request):
    start_time = time.time()
    try:
        response = process_request(request)
        status_code = 200
    except Exception as e:
        status_code = 500
        raise
    finally:
        duration_ms = (time.time() - start_time) * 1000
        metrics.record_request(
            request.method,
            request.path,
            status_code,
            duration_ms
        )
```

### Prometheus Exporter

```python
from prometheus_client import start_http_server, CollectorRegistry

# Create custom registry
registry = CollectorRegistry()

# Register metrics with custom registry
REQUEST_COUNT = Counter(
    'http_requests_total',
    'Total HTTP requests',
    ['method', 'endpoint', 'status_code'],
    registry=registry
)

REQUEST_LATENCY = Histogram(
    'http_request_latency_seconds',
    'Request latency in seconds',
    ['endpoint'],
    registry=registry
)

# Start metrics server
start_http_server(8000, registry=registry)
```

## When to Use

**Always Track:**
- Request rates and latencies (RED method: Rate, Errors, Duration)
- Error rates and types
- Resource utilization (CPU, memory, disk, network)
- Queue lengths and processing times
- External dependency health

**Track Based on Context:**
- Business metrics (orders, revenue, users)
- Custom application metrics
- Cache hit/miss ratios
- Database query performance
- Background job completion rates

**Cardinality Guidelines:**
- Low cardinality (< 10 values): endpoint, status_code, method
- Medium cardinality (10-100 values): user_type, region, service
- High cardinality (100+ values): Use sparingly, consider sampling

## Tradeoffs

| Aspect | Benefits | Costs |
|--------|----------|-------|
| **High Cardinality** | Detailed analysis, precise filtering | Storage explosion, query slowdown |
| **Frequent Collection** | Real-time visibility | Resource overhead, storage costs |
| **Many Metrics** | Comprehensive coverage | Complexity, noise, alert fatigue |
| **Fine Buckets** | Precise percentile calculation | More storage, complex queries |
| **Long Retention** | Historical analysis, trend detection | Storage costs, slower queries |

## Anti-Pattern

### ❌ Common Metrics Anti-Patterns

```python
# ANTI-PATTERN 1: High cardinality labels
def record_user_request(user_id, endpoint):
    # Creating unique time series for every user
    metrics.counter('requests_total', 
                   labels={'user_id': user_id, 'endpoint': endpoint}).inc()
    # Creates millions of time series!

# ANTI-PATTERN 2: Using gauges for counts
def track_active_users():
    # Gauge should not be used for monotonically increasing values
    metrics.gauge('total_users_ever', total_users).set()
    # Should use Counter instead

# ANTI-PATTERN 3: Missing labels for important dimensions
def record_request():
    # No way to filter by endpoint or status
    metrics.counter('requests_total').inc()

# ANTI-PATTERN 4: Inappropriate bucket boundaries
def record_latency():
    # Buckets don't match actual latency distribution
    metrics.histogram('latency', buckets=[1, 10, 100]).observe(latency_ms)
    # Most values fall in last bucket, no useful distribution
```

### ✅ Correct Patterns

```python
# CORRECT: Appropriate label cardinality
def record_user_request(user_id, endpoint):
    # Aggregate at appropriate level
    metrics.counter('requests_total', 
                   labels={'endpoint': endpoint, 'user_type': get_user_type(user_id)}).inc()

# CORRECT: Using right metric types
def track_active_users():
    # Counter for total users ever
    metrics.counter('users_registered_total').inc()
    # Gauge for current active users
    metrics.gauge('active_users').set(current_active_count)

# CORRECT: Meaningful labels
def record_request():
    metrics.counter('requests_total', 
                   labels={'endpoint': endpoint, 'method': method, 'status': status}).inc()

# CORRECT: Appropriate bucket boundaries
def record_latency():
    # Buckets match actual distribution
    metrics.histogram('latency_ms', 
                     buckets=[1, 5, 10, 25, 50, 100, 250, 500, 1000, 2500, 5000]).observe(latency_ms)
```

## Related Patterns

- **[Logging](./01-Logging.md)** - Event recording
- **[Monitoring](./02-Monitoring.md)** - System health tracking
- **[Tracing](./03-Tracing.md)** - Request flow tracking
- **[Alerting](./05-Alerting.md)** - Notification on threshold breaches
- **[Performance](../01-System-Design/07-Performance.md)** - System performance optimization