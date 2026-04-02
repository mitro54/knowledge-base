# Error Handling

## Title & Summary

Error handling is the systematic approach to detecting, reporting, and recovering from errors in software systems. This document covers error handling strategies, exception management, and techniques for building robust systems that gracefully handle failures.

## Problem Statement

Errors are inevitable in software systems:

- **Runtime Errors**: Division by zero, null pointer dereference, type mismatches
- **Resource Errors**: File not found, connection refused, out of memory
- **Validation Errors**: Invalid input, malformed data, constraint violations
- **Business Logic Errors**: Insufficient funds, item out of stock, permission denied
- **System Errors**: Database unavailable, network timeout, service degraded
- **Unexpected Errors**: Bugs, edge cases, unhandled scenarios

Poor error handling leads to:
- Crashes and system instability
- Security vulnerabilities through error messages
- Poor user experience
- Difficult debugging and troubleshooting
- Data corruption or loss

## Solution

### Error Handling Strategies

**1. Exception-Based Error Handling**
```
Try-Catch-Finally Pattern
```

- Use exceptions for exceptional conditions
- Catch specific exceptions, not generic ones
- Always clean up resources in finally blocks
- Re-throw when you can't handle the error

**2. Result/Optional Pattern**
```
Function Returns: Result<T, E> or Option<T>
```

- Explicit error handling in return type
- No silent failures
- Forces caller to handle both success and error cases

**3. Error Codes**
```
Return: (status_code, result_or_error)
```

- Numeric or enum-based error identification
- Common in systems programming
- Requires documentation for each code

**4. Continuation/Error Callback Pattern**
```
Callback: (error, result) => void
```

- Node.js style error-first callbacks
- Error as first parameter
- Result only present if no error

### Error Classification

**Recoverable Errors**
- Can be handled and system continues
- Examples: validation errors, transient network issues
- Strategy: Retry, fallback, or user notification

**Non-Recoverable Errors**
- System cannot continue safely
- Examples: corruption, security violations
- Strategy: Fail fast, log, alert

**Expected Errors**
- Anticipated in normal operation
- Examples: resource not found, permission denied
- Strategy: Handle gracefully, minimal logging

**Unexpected Errors**
- Bugs or unhandled scenarios
- Examples: null pointer, assertion failures
- Strategy: Log extensively, alert, fail safely

### Error Handling Layers

```
┌─────────────────────────────────────────┐
│         Presentation Layer              │
│   - User-friendly error messages        │
│   - Error pages, notifications          │
├─────────────────────────────────────────┤
│           API Layer                     │
│   - HTTP status codes                   │
│   - Structured error responses          │
├─────────────────────────────────────────┤
│         Service Layer                   │
│   - Business logic errors               │
│   - Transaction management              │
├─────────────────────────────────────────┤
│        Data Access Layer                │
│   - Database errors                     │
│   - Connection handling                 │
├─────────────────────────────────────────┤
│         Infrastructure Layer            │
│   - Network errors                      │
│   - Resource errors                     │
└─────────────────────────────────────────┘
```

## When to Use

| Strategy | Best For | Example |
|----------|----------|---------|
| Exceptions | Unexpected, exceptional conditions | Database connection failed |
| Result Type | Expected errors, functional style | Parse operation that may fail |
| Error Codes | Systems programming, APIs | Operating system errors |
| Callbacks | Async operations, event-driven | File I/O, network requests |
| Try-Catch | Resource management, cleanup | File handling, connections |

## Tradeoffs

### Exception-Based Tradeoffs

| Aspect | Benefit | Cost |
|--------|---------|------|
| Separation | Error logic separate from main flow | Performance overhead |
| Propagation | Automatically bubbles up | Can hide where error occurred |
| Safety | Forces handling or crashes | Can be caught and ignored |

### Result Type Tradeoffs

| Aspect | Benefit | Cost |
|--------|---------|------|
| Explicit | Error handling visible in type | More verbose code |
| Performance | No exception overhead | Manual propagation needed |
| Safety | Cannot forget to handle | Easy to ignore with match |

### Error Message Tradeoffs

| Aspect | Detailed | Generic |
|--------|----------|---------|
| Debugging | Easy to diagnose | Hard to diagnose |
| Security | May leak info | Safer |
| User Experience | Confusing for users | User-friendly |

## Implementation Example

### Comprehensive Error Handling System

```python
import traceback
import logging
from typing import Optional, Tuple, TypeVar, Generic, Callable, Any
from dataclasses import dataclass
from enum import IntEnum
from functools import wraps
import uuid

# Error Types
class ErrorType(IntEnum):
    VALIDATION_ERROR = 400
    UNAUTHORIZED = 401
    FORBIDDEN = 403
    NOT_FOUND = 404
    CONFLICT = 409
    INTERNAL_ERROR = 500
    SERVICE_UNAVAILABLE = 503

@dataclass
class AppError(Exception):
    """Base application error"""
    error_type: ErrorType
    message: str
    details: Optional[dict] = None
    request_id: Optional[str] = None
    traceback_info: Optional[str] = None
    
    def __init__(self, error_type: ErrorType, message: str, 
                 details: dict = None, request_id: str = None):
        self.error_type = error_type
        self.message = message
        self.details = details
        self.request_id = request_id or str(uuid.uuid4())[:8]
        self.traceback_info = traceback.format_exc()
        super().__init__(self.message)
    
    def to_dict(self) -> dict:
        return {
            "error": {
                "type": self.error_type.name,
                "code": self.error_type.value,
                "message": self.message,
                "request_id": self.request_id,
                "details": self.details
            }
        }

# Specific Error Types
class ValidationError(AppError):
    def __init__(self, message: str, field_errors: dict = None):
        super().__init__(
            ErrorType.VALIDATION_ERROR, 
            message, 
            {"field_errors": field_errors}
        )

class NotFoundError(AppError):
    def __init__(self, resource_type: str, resource_id: str):
        super().__init__(
            ErrorType.NOT_FOUND,
            f"{resource_type} with id '{resource_id}' not found",
            {"resource_type": resource_type, "resource_id": resource_id}
        )

class UnauthorizedError(AppError):
    def __init__(self, reason: str = "Authentication required"):
        super().__init__(ErrorType.UNAUTHORIZED, reason)

class ForbiddenError(AppError):
    def __init__(self, reason: str = "Permission denied"):
        super().__init__(ErrorType.FORBIDDEN, reason)

class ServiceUnavailableError(AppError):
    def __init__(self, service: str = "Unknown service"):
        super().__init__(
            ErrorType.SERVICE_UNAVAILABLE,
            f"Service '{service}' is temporarily unavailable"
        )

# Result Type (Functional Style)
T = TypeVar('T')
E = TypeVar('E')

class Result(Generic[T, E]):
    """Functional result type for explicit error handling"""
    
    def __init__(self, value: T = None, error: E = None):
        self.value = value
        self.error = error
        self.is_ok = error is None
        self.is_err = error is not None
    
    @classmethod
    def ok(cls, value: T) -> 'Result[T, E]':
        return cls(value=value)
    
    @classmethod
    def err(cls, error: E) -> 'Result[T, E]':
        return cls(error=error)
    
    def map(self, func: Callable[[T], T]) -> 'Result[T, E]':
        if self.is_ok:
            return Result.ok(func(self.value))
        return Result.err(self.error)
    
    def map_err(self, func: Callable[[E], E]) -> 'Result[T, E]':
        if self.is_err:
            return Result.err(func(self.error))
        return Result.ok(self.value)
    
    def and_then(self, func: Callable[[T], 'Result[T, E]']) -> 'Result[T, E]':
        if self.is_ok:
            return func(self.value)
        return self
    
    def or_else(self, func: Callable[[E], 'Result[T, E]']) -> 'Result[T, E]':
        if self.is_err:
            return func(self.error)
        return self
    
    def unwrap(self) -> T:
        if self.is_ok:
            return self.value
        raise self.error
    
    def unwrap_or(self, default: T) -> T:
        return self.value if self.is_ok else default

# Error Handler Decorator
def handle_errors(default_error: AppError = None):
    """Decorator to handle errors in functions"""
    def decorator(func: Callable):
        @wraps(func)
        def wrapper(*args, **kwargs):
            try:
                return func(*args, **kwargs)
            except AppError as e:
                logging.error(f"AppError: {e.message} (Request: {e.request_id})")
                raise
            except Exception as e:
                error = AppError(
                    ErrorType.INTERNAL_ERROR,
                    "An unexpected error occurred",
                    {"original_error": str(e)},
                    request_id=str(uuid.uuid4())[:8]
                )
                logging.error(f"Unexpected error: {error.traceback_info}")
                if default_error:
                    raise default_error
                raise error
        return wrapper
    return decorator

# Error Handler Middleware (for web frameworks)
class ErrorHandlerMiddleware:
    def __init__(self, app):
        self.app = app
        
    async def __call__(self, request, call_next):
        try:
            response = await call_next(request)
            return response
        except AppError as e:
            return self._handle_app_error(e)
        except Exception as e:
            return self._handle_unexpected_error(e)
    
    def _handle_app_error(self, error: AppError) -> dict:
        log_level = logging.WARNING if error.error_type < 500 else logging.ERROR
        log_message = f"{error.error_type.name}: {error.message}"
        if error.details:
            log_message += f" - {error.details}"
        logging.log(log_level, log_message)
        
        return {
            "status": error.error_type.value,
            "error": error.to_dict()["error"]
        }
    
    def _handle_unexpected_error(self, error: Exception) -> dict:
        app_error = AppError(
            ErrorType.INTERNAL_ERROR,
            "Internal server error",
            request_id=str(uuid.uuid4())[:8]
        )
        logging.error(f"Unexpected error: {traceback.format_exc()}")
        return self._handle_app_error(app_error)

# Retry with Error Handling
async def retry_with_handling(
    func: Callable,
    max_retries: int = 3,
    retryable_errors: tuple = (ConnectionError, TimeoutError),
    backoff_base: float = 1.0
):
    """Execute function with retry and proper error handling"""
    last_error = None
    
    for attempt in range(max_retries):
        try:
            return await func()
        except retryable_errors as e:
            last_error = e
            if attempt < max_retries - 1:
                import asyncio
                await asyncio.sleep(backoff_base * (2 ** attempt))
            continue
        except Exception as e:
            # Non-retryable error, raise immediately
            raise
    
    # All retries exhausted
    raise ServiceUnavailableError(
        f"Operation failed after {max_retries} attempts",
        details={"last_error": str(last_error)}
    )

# Transaction Error Handler
class TransactionErrorHandler:
    def __init__(self, transaction_manager):
        self.tm = transaction_manager
        
    async def execute_with_transaction(self, operation: Callable):
        """Execute operation within transaction with proper error handling"""
        transaction = None
        try:
            transaction = await self.tm.begin()
            result = await operation(transaction)
            await self.tm.commit(transaction)
            return result
        except ValidationError as e:
            await self.tm.rollback(transaction)
            logging.warning(f"Transaction rolled back: {e.message}")
            raise
        except Exception as e:
            await self.tm.rollback(transaction)
            logging.error(f"Transaction failed: {traceback.format_exc()}")
            raise AppError(
                ErrorType.INTERNAL_ERROR,
                "Transaction failed",
                details={"original_error": str(e)}
            )

# Usage Examples
async def example_usage():
    # Example 1: Exception-based error handling
    @handle_errors()
    async def get_user(user_id: str):
        if not user_id:
            raise ValidationError("User ID is required", {"field": "user_id"})
        
        user = await database.get_user(user_id)
        if not user:
            raise NotFoundError("User", user_id)
        
        return user
    
    # Example 2: Result type for parsing
    def parse_int(value: str) -> Result[int, str]:
        try:
            return Result.ok(int(value))
        except ValueError:
            return Result.err(f"Cannot parse '{value}' as integer")
    
    result = parse_int("123")
    if result.is_ok:
        print(f"Parsed value: {result.value}")
    else:
        print(f"Parse error: {result.error}")
    
    # Example 3: Chaining with and_then
    def safe_divide(a: float, b: float) -> Result[float, str]:
        if b == 0:
            return Result.err("Division by zero")
        return Result.ok(a / b)
    
    result = (
        parse_int("100")
        .and_then(lambda x: parse_int("5").map(lambda y: safe_divide(x, y)))
    )
    
    # Example 4: Error handling in API
    async def api_handler(request):
        try:
            user_id = request.params.get("id")
            user = await get_user(user_id)
            return {"user": user}
        except AppError as e:
            return e.to_dict()
```

## Anti-Pattern

### ❌ Error Handling Anti-Patterns

**1. Empty Catch Blocks**
```python
# ANTI-PATTERN: Silently swallowing errors
try:
    process_data(data)
except:
    pass  # What happened?

# PATTERN: Log and handle appropriately
try:
    process_data(data)
except ValidationError as e:
    logger.warning(f"Validation failed: {e}")
    return error_response(str(e))
except Exception as e:
    logger.error(f"Unexpected error: {e}", exc_info=True)
    raise
```

**2. Catching Generic Exceptions**
```python
# ANTI-PATTERN: Catching everything
try:
    result = risky_operation()
except Exception:
    return default_value  # Hides real problems

# PATTERN: Catch specific exceptions
try:
    result = risky_operation()
except ConnectionError:
    return cached_value
except TimeoutError:
    raise ServiceUnavailableError("Operation timed out")
```

**3. Using Exceptions for Control Flow**
```python
# ANTI-PATTERN: Exceptions as control mechanism
def get_config_value(key):
    try:
        return config[key]
    except KeyError:
        return default_value  # Should use .get() instead

# PATTERN: Use proper control flow
def get_config_value(key):
    return config.get(key, default_value)
```

**4. Returning Null for Errors**
```python
# ANTI-PATTERN: Null means error
def find_user(user_id):
    user = database.query("SELECT * FROM users WHERE id = ?", user_id)
    return user[0] if user else None  # Caller must check for None

# PATTERN: Explicit error handling
def find_user(user_id):
    user = database.query("SELECT * FROM users WHERE id = ?", user_id)
    if not user:
        raise NotFoundError("User", user_id)
    return user[0]
```

**5. Leaking Sensitive Information**
```python
# ANTI-PATTERN: Exposing internal details
try:
    process_payment(payment)
except Exception as e:
    return {"error": str(e)}  # May expose SQL, paths, etc.

# PATTERN: Generic user message, detailed log
try:
    process_payment(payment)
except PaymentError as e:
    logger.error(f"Payment failed: {e}", exc_info=True)
    return {"error": "Payment processing failed. Please try again."}
```

## Related Patterns

- **[Security Patterns](../05-Safety-Engineering/01-Security-Patterns.md)** - Secure error messages
- **[Fault Tolerance](../05-Safety-Engineering/02-Fault-Tolerance.md)** - Error recovery strategies
- **[Input Validation](../05-Safety-Engineering/04-Input-Validation.md)** - Validation error handling
- **[Logging](../10-Observability/01-Logging.md)** - Error logging best practices
- **[Monitoring](../10-Observability/02-Monitoring.md)** - Error rate monitoring
- **[Testing Strategies](../06-Testing-Engineering/05-Testing-Strategies.md)** - Error case testing