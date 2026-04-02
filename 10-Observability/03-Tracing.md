# Tracing

## Title & Summary

Tracing is the practice of tracking requests as they flow through distributed systems, providing visibility into service interactions, latency breakdowns, and error propagation. Distributed tracing enables understanding of complex request flows across microservices, helping identify performance bottlenecks and failure points.

## Problem Statement

Distributed systems without tracing face significant challenges:

- **Opaque Request Flows**: Impossible to see how requests traverse services
- **Latency Attribution**: Cannot determine which service causes slowdowns
- **Error Source Identification**: Difficult to pinpoint where failures originate
- **Dependency Mapping**: Unknown service-to-service communication patterns
- **Debugging Complexity**: No way to follow a request across service boundaries
- **Performance Optimization Blindspots**: Cannot identify optimization opportunities

## Solution

### Core Tracing Concepts

**Trace**: A complete journey of a request through the system
**Span**: A single unit of work within a trace (e.g., a function call, HTTP request)
**Context**: Trace ID and Span ID propagated across service boundaries

**Trace Context Propagation**
- Trace ID: Unique identifier for the entire trace
- Span ID: Unique identifier for each span
- Parent Span ID: Links child spans to parent spans

### Implementation Example

```python
import uuid
import time
import json
from dataclasses import dataclass, field
from typing import Dict, List, Optional
from contextvars import ContextVar

@dataclass
class Span:
    trace_id: str
    span_id: str
    parent_span_id: Optional[str]
    operation_name: str
    service_name: str
    start_time: float
    end_time: Optional[float] = None
    tags: Dict[str, str] = field(default_factory=dict)
    logs: List[Dict] = field(default_factory=list)
    errors: List[Dict] = field(default_factory=list)
    
    def finish(self):
        self.end_time = time.time()
    
    @property
    def duration_ms(self) -> float:
        if self.end_time:
            return (self.end_time - self.start_time) * 1000
        return 0

@dataclass
class TraceContext:
    trace_id: str
    span_id: str
    parent_span_id: Optional[str] = None
    sampled: bool = True

# Context variable for current span
current_span: ContextVar[Optional[Span]] = ContextVar('current_span', default=None)
current_context: ContextVar[Optional[TraceContext]] = ContextVar('current_context', default=None)

class Tracer:
    def __init__(self, service_name: str, sampling_rate: float = 0.1):
        self.service_name = service_name
        self.sampling_rate = sampling_rate
        self.traces: Dict[str, List[Span]] = {}
    
    def should_sample(self, trace_id: str = None) -> bool:
        """Deterministic sampling based on trace ID"""
        if trace_id:
            # Use trace ID for consistent sampling decision
            hash_value = int(trace_id[:8], 16)
            return (hash_value % 100) < (self.sampling_rate * 100)
        return hash(uuid.uuid4()) % 100 < (self.sampling_rate * 100)
    
    def start_trace(self, operation_name: str, tags: Dict = None) -> Span:
        """Start a new trace"""
        trace_id = str(uuid.uuid4())
        span_id = str(uuid.uuid4())
        
        sampled = self.should_sample(trace_id)
        context = TraceContext(
            trace_id=trace_id,
            span_id=span_id,
            sampled=sampled
        )
        current_context.set(context)
        
        span = Span(
            trace_id=trace_id,
            span_id=span_id,
            parent_span_id=None,
            operation_name=operation_name,
            service_name=self.service_name,
            start_time=time.time(),
            tags=tags or {}
        )
        current_span.set(span)
        
        if sampled:
            self.traces[trace_id] = []
        
        return span
    
    def start_span(self, operation_name: str, parent_span: Span = None, 
                   tags: Dict = None) -> Span:
        """Start a new span within existing trace"""
        context = current_context.get()
        if not context:
            raise ValueError("No active trace context")
        
        span_id = str(uuid.uuid4())
        parent_span_id = parent_span.span_id if parent_span else context.span_id
        
        span = Span(
            trace_id=context.trace_id,
            span_id=span_id,
            parent_span_id=parent_span_id,
            operation_name=operation_name,
            service_name=self.service_name,
            start_time=time.time(),
            tags=tags or {}
        )
        
        old_span = current_span.get()
        current_span.set(span)
        
        if context.sampled:
            self.traces[context.trace_id].append(span)
        
        return span
    
    def finish_span(self, span: Span):
        """Finish a span"""
        span.finish()
        # Restore previous span
        context = current_context.get()
        if context and context.sampled and context.trace_id in self.traces:
            # Span already added on creation
            pass
    
    def add_log(self, span: Span, message: str, data: Dict = None):
        """Add a log to a span"""
        log_entry = {
            'timestamp': time.time(),
            'message': message,
            'data': data or {}
        }
        span.logs.append(log_entry)
    
    def record_error(self, span: Span, error: Exception):
        """Record an error on a span"""
        error_entry = {
            'timestamp': time.time(),
            'type': type(error).__name__,
            'message': str(error)
        }
        span.errors.append(error_entry)
        span.tags['error'] = 'true'
    
    def get_trace_context_headers(self) -> Dict[str, str]:
        """Get headers for propagating trace context"""
        context = current_context.get()
        if not context:
            return {}
        
        headers = {
            'x-trace-id': context.trace_id,
            'x-span-id': context.span_id,
            'x-sampled': '1' if context.sampled else '0'
        }
        if context.parent_span_id:
            headers['x-parent-span-id'] = context.parent_span_id
        return headers
    
    def extract_trace_context(self, headers: Dict[str, str]) -> Optional[TraceContext]:
        """Extract trace context from headers"""
        trace_id = headers.get('x-trace-id')
        if not trace_id:
            return None
        
        sampled = headers.get('x-sampled') == '1'
        
        return TraceContext(
            trace_id=trace_id,
            span_id=headers.get('x-span-id', str(uuid.uuid4())),
            parent_span_id=headers.get('x-parent-span-id'),
            sampled=sampled
        )

# Usage Example
tracer = Tracer('order-service', sampling_rate=1.0)

def process_order(order_id: str):
    # Start trace
    trace_span = tracer.start_trace('process_order', {'order_id': order_id})
    
    try:
        # Span for validation
        with tracer.start_span('validate_order') as span:
            validate_order(order_id)
        
        # Span for payment
        with tracer.start_span('process_payment') as span:
            span.tags['payment_method'] = 'credit_card'
            process_payment(order_id)
        
        # Span for inventory
        with tracer.start_span('reserve_inventory') as span:
            reserve_inventory(order_id)
        
        tracer.add_log(trace_span, 'Order processed successfully', 
                       {'order_id': order_id})
        
    except Exception as e:
        tracer.record_error(trace_span, e)
        raise
    finally:
        tracer.finish_span(trace_span)

# HTTP Middleware for trace propagation
def trace_middleware(request, response):
    # Extract incoming trace context
    incoming_context = tracer.extract_trace_context(request.headers)
    if incoming_context:
        current_context.set(incoming_context)
    
    # Add outgoing trace headers to response
    outgoing_headers = tracer.get_trace_context_headers()
    response.headers.update(outgoing_headers)
    
    return response
```

### OpenTelemetry Integration

```python
from opentelemetry import trace
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import ConsoleSpanExporter, BatchSpanProcessor
from opentelemetry.exporter.jaeger import JaegerExporter

# Configure tracer provider
provider = TracerProvider()
processor = BatchSpanProcessor(JaegerExporter(
    endpoint='http://jaeger-collector:14268/api/v2/spans'
))
provider.add_span_processor(processor)
trace.set_tracer_provider(provider)

# Get tracer
tracer = trace.get_tracer('order-service')

@tracer.start_as_current_span("process_order")
def process_order(order_id: str):
    with tracer.start_as_current_span("validate_order"):
        validate_order(order_id)
    
    with tracer.start_as_current_span("process_payment"):
        process_payment(order_id)
    
    return order_id
```

## When to Use

**Always Trace:**
- HTTP requests across service boundaries
- Database queries
- Cache operations
- Message queue publish/subscribe
- External API calls

**Trace Based on Context:**
- Long-running background jobs
- Scheduled tasks
- Event processing pipelines
- Complex business operations

**Sampling Strategies:**
- Development: 100% sampling
- Staging: 50% sampling
- Production: 1-10% sampling (adjust based on traffic)
- Error traces: Always sample (100%)

## Tradeoffs

| Aspect | Benefits | Costs |
|--------|----------|-------|
| **100% Sampling** | Complete visibility, no missed traces | High overhead, storage costs |
| **Low Sampling** | Lower overhead, cheaper storage | May miss important traces |
| **High Cardinality Tags** | Detailed filtering and analysis | Storage explosion, query slowdown |
| **Detailed Spans** | Fine-grained visibility | More data to process and store |
| **Context Propagation** | Complete trace reconstruction | Header overhead, complexity |

## Anti-Pattern

### ❌ Common Tracing Anti-Patterns

```python
# ANTI-PATTERN 1: Creating spans for everything
def process_data(items):
    for item in items:
        with tracer.start_span(f'process_item_{item.id}'):
            process_item(item)
    # Creates thousands of spans for simple loops

# ANTI-PATTERN 2: Not propagating trace context
def call_external_service(data):
    # Missing trace context propagation
    response = requests.post('http://external/api', json=data)
    # Trace breaks at service boundary

# ANTI-PATTERN 3: Overly verbose span names
def handle_request():
    with tracer.start_span('user_service_get_user_by_id_endpoint_handler'):
        # Too specific, hard to aggregate
        pass

# ANTI-PATTERN 4: Storing sensitive data in spans
def process_payment(card_number):
    with tracer.start_span('process_payment') as span:
        span.set_tag('card_number', card_number)  # NEVER!
        span.set_tag('cvv', cvv)  # NEVER!
```

### ✅ Correct Patterns

```python
# CORRECT: Appropriate span granularity
def process_data(items):
    with tracer.start_span('process_batch'):
        span.set_attribute('batch.size', len(items))
        for item in items:
            process_item(item)
    # Single span for batch operation

# CORRECT: Proper trace context propagation
def call_external_service(data):
    headers = tracer.get_trace_context_headers()
    response = requests.post('http://external/api', json=data, headers=headers)
    # Trace continues across service boundary

# CORRECT: Meaningful span names
def handle_request():
    with tracer.start_span('get_user'):
        # Clear, aggregatable name
        pass

# CORRECT: Sanitized span attributes
def process_payment(card_number):
    with tracer.start_span('process_payment') as span:
        span.set_attribute('card_last_four', card_number[-4:])
        span.set_attribute('amount', amount)
        # No sensitive data
```

## Related Patterns

- **[Logging](./01-Logging.md)** - Event recording
- **[Monitoring](./02-Monitoring.md)** - System health tracking
- **[Metrics](./04-Metrics.md)** - Quantitative measurements
- **[Event-Driven Architecture](../01-System-Design/03-Event-Driven.md)** - Event flow tracking
- **[Microservices](../01-System-Design/02-Microservices.md)** - Service communication