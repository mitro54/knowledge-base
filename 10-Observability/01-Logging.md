# Logging

## Title & Summary

Logging is the practice of recording events, errors, and diagnostic information in a structured, persistent format. Effective logging provides visibility into system behavior, enables troubleshooting, supports security auditing, and forms the foundation of observability in modern systems.

## Problem Statement

Systems without proper logging face significant challenges:

- **Debugging Difficulties**: Impossible to trace issues without visibility into system behavior
- **Security Blindspots**: Cannot detect or investigate security incidents
- **Performance Unknowns**: No data to identify bottlenecks or anomalies
- **Compliance Failures**: Unable to meet audit and regulatory requirements
- **Mean Time to Resolution (MTTR) Increase**: Longer troubleshooting times without logs
- **Post-Incident Analysis Impossible**: Cannot reconstruct what happened during failures

## Solution

### Core Logging Principles

**Structured Logging**
- Logs formatted as structured data (JSON, key-value pairs)
- Enables efficient parsing, filtering, and analysis
- Consistent field names across the application

**Log Levels**
- **DEBUG**: Detailed information for debugging
- **INFO**: General operational information
- **WARN**: Potential issues that don't prevent operation
- **ERROR**: Errors that affect functionality
- **FATAL**: Critical errors causing system termination

**Log Context**
- Request IDs for tracing across services
- User IDs for user-specific debugging
- Timestamps in UTC with timezone awareness
- Service name and instance identifier

### Implementation Example

```python
import logging
import json
import uuid
from datetime import datetime
from typing import Any, Dict

class StructuredLogger:
    def __init__(self, service_name: str, log_level: int = logging.INFO):
        self.service_name = service_name
        self.instance_id = str(uuid.uuid4())[:8]
        
        # Configure formatter
        formatter = logging.Formatter('%(message)s')
        
        # Console handler
        console_handler = logging.StreamHandler()
        console_handler.setFormatter(formatter)
        
        # File handler
        file_handler = logging.FileHandler(f'{service_name}.log')
        file_handler.setFormatter(formatter)
        
        self.logger = logging.getLogger(service_name)
        self.logger.setLevel(log_level)
        self.logger.addHandler(console_handler)
        self.logger.addHandler(file_handler)
    
    def _build_log_entry(self, level: str, message: str, **context) -> str:
        entry = {
            "timestamp": datetime.utcnow().isoformat() + "Z",
            "level": level,
            "service": self.service_name,
            "instance": self.instance_id,
            "message": message,
            "context": context
        }
        return json.dumps(entry)
    
    def debug(self, message: str, **context):
        self.logger.debug(self._build_log_entry("DEBUG", message, **context))
    
    def info(self, message: str, **context):
        self.logger.info(self._build_log_entry("INFO", message, **context))
    
    def warning(self, message: str, **context):
        self.logger.warning(self._build_log_entry("WARN", message, **context))
    
    def error(self, message: str, exception: Exception = None, **context):
        if exception:
            context["exception"] = str(exception)
            context["exception_type"] = type(exception).__name__
            context["traceback"] = self._format_traceback(exception)
        self.logger.error(self._build_log_entry("ERROR", message, **context))
    
    def _format_traceback(self, exception: Exception) -> str:
        import traceback
        return ''.join(traceback.format_exception(type(exception), exception, exception.__traceback__))

# Usage
logger = StructuredLogger("order-service")

def process_order(order_id: str, user_id: str):
    request_id = str(uuid.uuid4())
    
    logger.info("Order processing started", 
                order_id=order_id, 
                user_id=user_id,
                request_id=request_id)
    
    try:
        # Process order logic
        result = perform_order_operations(order_id, user_id)
        
        logger.info("Order processing completed",
                    order_id=order_id,
                    request_id=request_id,
                    duration_ms=calculate_duration())
        
        return result
    except PaymentError as e:
        logger.error("Payment processing failed",
                     order_id=order_id,
                     request_id=request_id,
                     exception=e)
        raise
```

### Log Aggregation Architecture

```yaml
# Example: ELK Stack Configuration
apiVersion: v1
kind: ConfigMap
metadata:
  name: fluentd-config
data:
  fluent.conf: |
    <source>
      @type tail
      path /var/log/containers/*.log
      pos_file /var/log/fluentd/containers.log.pos
      tag kubernetes.*
      read_from_head true
      <parse>
        @type json
      </parse>
    </source>
    
    <filter kubernetes.**>
      @type kubernetes_metadata
      kubernetes_url https://kubernetes.default.svc
    </filter>
    
    <match kubernetes.**>
      @type elasticsearch
      host elasticsearch.logging.svc
      port 9200
      logstash_format true
      logstash_prefix applogs
    </match>
```

## When to Use

**Always Log:**
- Application startup and shutdown
- Configuration changes
- Authentication and authorization events
- Business-critical operations (orders, payments)
- Errors and exceptions
- Performance metrics (request duration, queue sizes)
- External service calls

**Log at Appropriate Levels:**
- DEBUG: Variable states, detailed flow information (development only)
- INFO: Normal operations, user actions, state changes
- WARN: Degraded performance, retry attempts, fallback activation
- ERROR: Failed operations, exceptions, data validation failures

**Avoid Logging:**
- Sensitive data (passwords, tokens, PII)
- Excessive debug information in production
- Large payloads or binary data
- High-frequency events without sampling

## Tradeoffs

| Aspect | Benefits | Costs |
|--------|----------|-------|
| **Detail Level** | Better debugging capability | Increased storage, potential performance impact |
| **Structured Format** | Easy parsing and analysis | Slightly more CPU for serialization |
| **Synchronous Logging** | Guaranteed order, simplicity | Can block application threads |
| **Asynchronous Logging** | Non-blocking, better performance | Potential log loss on crash, ordering issues |
| **Log Aggregation** | Centralized analysis, retention | Infrastructure cost, complexity |
| **High Retention** | Historical analysis, compliance | Storage costs, slower queries |

## Anti-Pattern

### ❌ Common Logging Anti-Patterns

```python
# ANTI-PATTERN 1: Logging sensitive information
class PaymentProcessor:
    def process_payment(self, card_number: str, cvv: str):
        logger.info(f"Processing payment for card: {card_number}, CVV: {cvv}")
        # NEVER log sensitive data!
        return self.gateway.charge(card_number, cvv)

# ANTI-PATTERN 2: Using logs for debugging in production
def calculate_total(items):
    for i in range(len(items)):
        logger.debug(f"Item {i}: {items[i]}, running total: {total}")
        # Excessive logging impacts performance
        total += items[i].price
    return total

# ANTI-PATTERN 3: Inconsistent log levels
def validate_user(user):
    if not user.email:
        logger.error("User has no email")  # Should be validation error, not system error
    if user.age < 18:
        logger.warn("User is minor")  # Inconsistent level usage
    return True

# ANTI-PATTERN 4: String concatenation in logs
def log_user_action(user_id, action, details):
    logger.info("User " + user_id + " performed action " + action + " with details " + str(details))
    # Hard to parse, no structure
```

### ✅ Correct Patterns

```python
# CORRECT: Sanitized logging with context
class PaymentProcessor:
    def process_payment(self, card_number: str, cvv: str):
        # Mask card number for logging
        masked_card = f"****-****-****-{card_number[-4:]}"
        logger.info("Processing payment", 
                    card_last_four=card_number[-4:],
                    amount=amount,
                    user_id=user_id)
        return self.gateway.charge(card_number, cvv)

# CORRECT: Appropriate log levels and structure
def validate_user(user):
    errors = []
    if not user.email:
        errors.append("missing_email")
        logger.warning("Validation warning", 
                       field="email", 
                       error="missing",
                       user_id=user.id)
    
    if user.age < 18:
        errors.append("age_restriction")
    
    if errors:
        raise ValidationError(errors)
    
    return True

# CORRECT: Structured logging with correlation
def log_user_action(user_id: str, action: str, details: Dict):
    logger.info("User action performed",
                user_id=user_id,
                action=action,
                details=details,
                request_id=get_request_id())
```

## Related Patterns

- **[Monitoring](./02-Monitoring.md)** - Real-time system health tracking
- **[Tracing](./03-Tracing.md)** - Request flow tracking across services
- **[Metrics](./04-Metrics.md)** - Quantitative measurements
- **[Alerting](./05-Alerting.md)** - Notification on issues
- **[Error Handling](../05-Safety-Engineering/03-Error-Handling.md)** - Exception management strategies