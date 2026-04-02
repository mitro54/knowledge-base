# Input Validation

## Title & Summary

Input validation is the process of ensuring that data entered into a system meets expected criteria before processing. This document covers validation strategies, sanitization techniques, and defense-in-depth approaches to prevent invalid or malicious data from compromising system integrity.

## Problem Statement

Unvalidated input is one of the most common sources of security vulnerabilities and system failures:

- **Security Vulnerabilities**: SQL injection, XSS, command injection, path traversal
- **Data Corruption**: Invalid data types, out-of-range values, malformed structures
- **System Crashes**: Type errors, null pointer exceptions, buffer overflows
- **Business Logic Errors**: Invalid states, constraint violations
- **Regulatory Compliance**: GDPR, HIPAA, PCI-DSS require input validation
- **User Experience**: Confusing error messages, data loss

### Common Attack Vectors

```
SQL Injection:    ' OR 1=1 --
XSS Attack:       <script>alert('xss')</script>
Command Injection: ; rm -rf /
Path Traversal:   ../../../etc/passwd
LDAP Injection:   *)(uid=*))(|(uid=*
```

## Solution

### Validation Strategies

**1. Whitelist vs Blacklist**

```
WHITELIST (Recommended): Allow only known-good values
BLACKLIST (Avoid): Block known-bad values
```

Whitelist is more secure because:
- You know what's allowed
- New attack vectors don't bypass validation
- Easier to reason about security

**2. Defense in Depth Layers**

```
┌─────────────────────────────────────────┐
│  Client-Side Validation                 │
│  - UX improvement, not security         │
├─────────────────────────────────────────┤
│  API Gateway / Edge Validation          │
│  - Rate limiting, basic structure       │
├─────────────────────────────────────────┤
│  Application Validation                 │
│  - Business logic, type checking        │
├─────────────────────────────────────────┤
│  Database Constraints                   │
│  - Type, length, unique, foreign keys   │
├─────────────────────────────────────────┤
│  ORM / Query Parameterization           │
│  - SQL injection prevention             │
└─────────────────────────────────────────┘
```

**3. Validation Types**

| Type | Purpose | Example |
|------|---------|---------|
| Type Validation | Correct data type | String, Integer, Boolean |
| Range Validation | Within bounds | Age: 0-150 |
| Format Validation | Correct pattern | Email, Phone, UUID |
| Length Validation | Size constraints | Password: 8-128 chars |
| Value Validation | Allowed values | Status: active, inactive |
| Cross-Field Validation | Field relationships | End date > Start date |

### Sanitization Techniques

**1. Encoding**
```
HTML Entity: < → <
URL Encode:   space → %20
Base64:      binary → ASCII
```

**2. Escaping**
```
SQL: ' → ''
Shell: $ → \$
Regex: . → \.
```

**3. Filtering**
```
Remove: Script tags, null bytes
Truncate: Limit string length
Normalize: Unicode normalization
```

## When to Use

| Technique | Use For | Don't Use For |
|-----------|---------|---------------|
| Whitelist | Enum values, allowed characters | Complex patterns |
| Regex | Format validation (email, phone) | Security-critical parsing |
| Type Casting | Type validation | Security sanitization |
| Encoding | Output contexts (HTML, URL) | Input validation |
| Parameterization | SQL queries | String concatenation |

## Tradeoffs

### Client-Side vs Server-Side

| Aspect | Client-Side | Server-Side |
|--------|-------------|-------------|
| Performance | Reduces server load | Always required |
| Security | Can be bypassed | Trustworthy |
| UX | Immediate feedback | Network latency |
| Use Case | UX enhancement | Security requirement |

### Strict vs Lenient Validation

| Aspect | Strict | Lenient |
|--------|--------|---------|
| Security | Higher | Lower |
| UX | May reject valid input | More forgiving |
| Compatibility | May break integrations | Better compatibility |
| Debugging | Clearer errors | Harder to diagnose |

## Implementation Example

### Comprehensive Input Validation System

```python
import re
import json
from typing import Any, Dict, List, Optional, Callable, Union
from dataclasses import dataclass, field
from enum import Enum
from functools import wraps
import html
import uuid

# Validation Error Types
class ValidationErrorType(Enum):
    TYPE_ERROR = "type_error"
    REQUIRED = "required"
    MIN_LENGTH = "min_length"
    MAX_LENGTH = "max_length"
    MIN_VALUE = "min_value"
    MAX_VALUE = "max_value"
    PATTERN_MISMATCH = "pattern_mismatch"
    INVALID_EMAIL = "invalid_email"
    INVALID_URL = "invalid_url"
    INVALID_UUID = "invalid_uuid"
    NOT_IN_ENUM = "not_in_enum"
    INVALID_JSON = "invalid_json"
    CUSTOM = "custom"

@dataclass
class ValidationError:
    field: str
    error_type: ValidationErrorType
    message: str
    value: Any = None
    
    def to_dict(self) -> dict:
        return {
            "field": self.field,
            "type": self.error_type.value,
            "message": self.message,
            "value": str(self.value) if self.value is not None else None
        }

# Validator Base Class
class Validator:
    def __init__(self, field_name: str = ""):
        self.field_name = field_name
        self.validators: List[Callable] = []
        
    def validate(self, value: Any) -> List[ValidationError]:
        errors = []
        for validator in self.validators:
            try:
                validator(value)
            except ValidationError as e:
                errors.append(ValidationError(
                    field=self.field_name or e.field,
                    error_type=e.error_type,
                    message=e.message,
                    value=value
                ))
        return errors
    
    def must_be_type(self, expected_type: type):
        def validator(value: Any):
            if value is not None and not isinstance(value, expected_type):
                raise ValidationError(
                    field=self.field_name,
                    error_type=ValidationErrorType.TYPE_ERROR,
                    message=f"Expected {expected_type.__name__}, got {type(value).__name__}"
                )
        self.validators.append(validator)
        return self
    
    def required(self):
        def validator(value: Any):
            if value is None or value == "":
                raise ValidationError(
                    field=self.field_name,
                    error_type=ValidationErrorType.REQUIRED,
                    message="This field is required"
                )
        self.validators.append(validator)
        return self
    
    def min_length(self, min_len: int):
        def validator(value: Any):
            if value is not None and len(value) < min_len:
                raise ValidationError(
                    field=self.field_name,
                    error_type=ValidationErrorType.MIN_LENGTH,
                    message=f"Minimum length is {min_len}"
                )
        self.validators.append(validator)
        return self
    
    def max_length(self, max_len: int):
        def validator(value: Any):
            if value is not None and len(value) > max_len:
                raise ValidationError(
                    field=self.field_name,
                    error_type=ValidationErrorType.MAX_LENGTH,
                    message=f"Maximum length is {max_len}"
                )
        self.validators.append(validator)
        return self
    
    def min_value(self, min_val: Union[int, float]):
        def validator(value: Any):
            if value is not None and value < min_val:
                raise ValidationError(
                    field=self.field_name,
                    error_type=ValidationErrorType.MIN_VALUE,
                    message=f"Minimum value is {min_val}"
                )
        self.validators.append(validator)
        return self
    
    def max_value(self, max_val: Union[int, float]):
        def validator(value: Any):
            if value is not None and value > max_val:
                raise ValidationError(
                    field=self.field_name,
                    error_type=ValidationErrorType.MAX_VALUE,
                    message=f"Maximum value is {max_val}"
                )
        self.validators.append(validator)
        return self
    
    def pattern(self, pattern: str, error_msg: str = None):
        regex = re.compile(pattern)
        def validator(value: Any):
            if value is not None and not regex.match(str(value)):
                raise ValidationError(
                    field=self.field_name,
                    error_type=ValidationErrorType.PATTERN_MISMATCH,
                    message=error_msg or f"Must match pattern: {pattern}"
                )
        self.validators.append(validator)
        return self
    
    def email(self):
        email_pattern = r'^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$'
        def validator(value: Any):
            if value is not None and not re.match(email_pattern, str(value)):
                raise ValidationError(
                    field=self.field_name,
                    error_type=ValidationErrorType.INVALID_EMAIL,
                    message="Invalid email address"
                )
        self.validators.append(validator)
        return self
    
    def url(self):
        url_pattern = r'^https?://[^\s]+$'
        def validator(value: Any):
            if value is not None and not re.match(url_pattern, str(value)):
                raise ValidationError(
                    field=self.field_name,
                    error_type=ValidationErrorType.INVALID_URL,
                    message="Invalid URL"
                )
        self.validators.append(validator)
        return self
    
    def uuid(self):
        def validator(value: Any):
            if value is not None:
                try:
                    uuid.UUID(str(value))
                except ValueError:
                    raise ValidationError(
                        field=self.field_name,
                        error_type=ValidationErrorType.INVALID_UUID,
                        message="Invalid UUID format"
                    )
        self.validators.append(validator)
        return self
    
    def enum(self, allowed_values: List[Any]):
        def validator(value: Any):
            if value is not None and value not in allowed_values:
                raise ValidationError(
                    field=self.field_name,
                    error_type=ValidationErrorType.NOT_IN_ENUM,
                    message=f"Must be one of: {allowed_values}"
                )
        self.validators.append(validator)
        return self
    
    def custom(self, validator_func: Callable, error_msg: str):
        def validator(value: Any):
            if not validator_func(value):
                raise ValidationError(
                    field=self.field_name,
                    error_type=ValidationErrorType.CUSTOM,
                    message=error_msg
                )
        self.validators.append(validator)
        return self

# Schema Validator
class SchemaValidator:
    def __init__(self):
        self.fields: Dict[str, Validator] = {}
        
    def add_field(self, name: str, validator: Validator):
        self.fields[name] = validator
        return self
    
    def string(self, name: str, **kwargs) -> 'SchemaValidator':
        validator = Validator(name).must_be_type(str)
        for key, value in kwargs.items():
            if hasattr(validator, key):
                getattr(validator, key)(value)
        self.fields[name] = validator
        return self
    
    def integer(self, name: str, **kwargs) -> 'SchemaValidator':
        validator = Validator(name).must_be_type(int)
        for key, value in kwargs.items():
            if hasattr(validator, key):
                getattr(validator, key)(value)
        self.fields[name] = validator
        return self
    
    def number(self, name: str, **kwargs) -> 'SchemaValidator':
        validator = Validator(name).must_be_type((int, float))
        for key, value in kwargs.items():
            if hasattr(validator, key):
                getattr(validator, key)(value)
        self.fields[name] = validator
        return self
    
    def validate(self, data: Dict[str, Any]) -> Dict[str, List[Dict]]:
        all_errors = {}
        
        for field_name, validator in self.fields.items():
            value = data.get(field_name)
            errors = validator.validate(value)
            if errors:
                all_errors[field_name] = [e.to_dict() for e in errors]
        
        return all_errors

# Sanitization Functions
class Sanitizer:
    @staticmethod
    def html_encode(value: str) -> str:
        """Encode HTML special characters"""
        return html.escape(str(value))
    
    @staticmethod
    def html_decode(value: str) -> str:
        """Decode HTML entities"""
        from html import unescape
        return unescape(str(value))
    
    @staticmethod
    def strip_html(value: str) -> str:
        """Remove HTML tags"""
        import re
        return re.sub(r'<[^>]+>', '', str(value))
    
    @staticmethod
    def truncate(value: str, max_length: int) -> str:
        """Truncate string to max length"""
        return str(value)[:max_length]
    
    @staticmethod
    def normalize_whitespace(value: str) -> str:
        """Normalize whitespace"""
        return ' '.join(str(value).split())
    
    @staticmethod
    def lowercase(value: str) -> str:
        """Convert to lowercase"""
        return str(value).lower()
    
    @staticmethod
    def uppercase(value: str) -> str:
        """Convert to uppercase"""
        return str(value).upper()
    
    @staticmethod
    def remove_non_alphanumeric(value: str) -> str:
        """Remove non-alphanumeric characters"""
        return re.sub(r'[^a-zA-Z0-9]', '', str(value))
    
    @staticmethod
    def safe_json_parse(value: str, default: Any = None) -> Any:
        """Safely parse JSON"""
        try:
            return json.loads(value)
        except (json.JSONDecodeError, TypeError):
            return default

# Security-Focused Input Validator
class SecurityInputValidator:
    """Specialized validator for security-sensitive inputs"""
    
    # Dangerous patterns to detect
    DANGEROUS_PATTERNS = [
        (r'<script[^>]*>', 'Script tag detected'),
        (r'javascript:', 'JavaScript protocol'),
        (r'on\w+\s*=', 'Event handler'),
        (r';\s*rm\s+', 'Command injection attempt'),
        (r'\.\./', 'Path traversal attempt'),
        (r"'\s*OR\s+'?\d+'?\s*=\s*'?\d+", 'SQL injection attempt'),
        (r'union\s+select', 'SQL UNION injection'),
    ]
    
    def __init__(self, strict_mode: bool = True):
        self.strict_mode = strict_mode
        
    def validate_input(self, field_name: str, value: Any) -> List[ValidationError]:
        errors = []
        
        # Type check
        if not isinstance(value, (str, bytes)):
            return errors
        
        value_str = str(value)
        
        # Check for dangerous patterns
        for pattern, message in self.DANGEROUS_PATTERNS:
            if re.search(pattern, value_str, re.IGNORECASE):
                errors.append(ValidationError(
                    field=field_name,
                    error_type=ValidationErrorType.CUSTOM,
                    message=f"Security violation: {message}"
                ))
                if self.strict_mode:
                    break
        
        # Check for null bytes
        if '\x00' in value_str:
            errors.append(ValidationError(
                field=field_name,
                error_type=ValidationErrorType.CUSTOM,
                message="Null byte detected"
            ))
        
        # Check for overly long input (potential DoS)
        if len(value_str) > 10000:
            errors.append(ValidationError(
                field=field_name,
                error_type=ValidationErrorType.MAX_LENGTH,
                message="Input too long (max 10000 characters)"
            ))
        
        return errors

# Usage Examples
def example_usage():
    # Example 1: User registration validation
    registration_schema = SchemaValidator()
    registration_schema.string("email", required=True, email=True, max_length=255)
    registration_schema.string("password", required=True, min_length=8, max_length=128)
    registration_schema.string("username", required=True, min_length=3, max_length=30, 
                               pattern=r'^[a-zA-Z0-9_]+$')
    registration_schema.integer("age", min_value=13, max_value=150)
    
    user_data = {
        "email": "invalid-email",
        "password": "short",
        "username": "valid_user",
        "age": 25
    }
    
    errors = registration_schema.validate(user_data)
    print(f"Validation Errors: {json.dumps(errors, indent=2)}")
    
    # Example 2: Security-focused validation
    security_validator = SecurityInputValidator(strict_mode=True)
    
    malicious_input = "<script>alert('xss')</script>"
    security_errors = security_validator.validate_input("comment", malicious_input)
    
    if security_errors:
        print(f"Security violations detected: {security_errors}")
    
    # Example 3: Sanitization pipeline
    user_comment = "<script>alert('xss')</script>Hello <b>World</b>"
    
    # Sanitize the input
    sanitized = Sanitizer.strip_html(user_comment)  # Remove HTML
    sanitized = Sanitizer.html_encode(sanitized)     # Encode special chars
    sanitized = Sanitizer.truncate(sanitized, 500)   # Limit length
    
    print(f"Original: {user_comment}")
    print(f"Sanitized: {sanitized}")
    
    # Example 4: Combined validation and sanitization
    def process_user_input(raw_input: Dict) -> Dict:
        # Validate first
        schema = SchemaValidator()
        schema.string("username", required=True, min_length=3, max_length=30)
        schema.string("email", required=True, email=True)
        
        errors = schema.validate(raw_input)
        if errors:
            raise ValueError(f"Validation failed: {errors}")
        
        # Then sanitize
        sanitized_input = {
            "username": Sanitizer.lowercase(Sanitizer.normalize_whitespace(raw_input["username"])),
            "email": Sanitizer.lowercase(raw_input["email"].strip())
        }
        
        return sanitized_input

if __name__ == "__main__":
    example_usage()
```

## Anti-Pattern

### ❌ Input Validation Anti-Patterns

**1. Client-Side Only Validation**
```python
# ANTI-PATTERN: Relying only on frontend validation
# Frontend JavaScript
if (email.indexOf('@') === -1) {
    showError("Invalid email");
    return;
}
// Backend accepts anything!

# PATTERN: Validate on both client and server
# Frontend: UX improvement
# Backend: Security requirement
```

**2. Using Blacklists**
```python
# ANTI-PATTERN: Blocking known bad values
blocked_chars = ['<', '>', '"', "'", ';']
for char in user_input:
    if char in blocked_chars:
        raise Error("Invalid character")
# Attacker finds unblocked dangerous char

# PATTERN: Whitelist allowed values
allowed_pattern = r'^[a-zA-Z0-9_]+$'
if not re.match(allowed_pattern, username):
    raise Error("Invalid username")
```

**3. Trusting Parsed Data**
```python
# ANTI-PATTERN: Assuming parsing validates
age = int(user_input)  # What if it's "25abc"?
# Or worse:
data = json.loads(user_input)  # Could be malicious object

# PATTERN: Validate after parsing
try:
    age = int(user_input)
    if age < 0 or age > 150:
        raise ValueError("Age out of range")
except ValueError:
    raise ValidationError("Invalid age")
```

**4. Inconsistent Validation**
```python
# ANTI-PATTERN: Different validation at different layers
def create_user(data):
    # API layer validation
    if not data.get("email"):
        return error("Email required")
    
    # Service layer - different validation!
    if len(data["email"]) > 100:
        return error("Email too long")
    
    # Database layer - yet another check
    # Might fail with cryptic error

# PATTERN: Centralized validation schema
user_schema = SchemaValidator()
user_schema.string("email", required=True, email=True, max_length=255)
```

**5. Using Validation for Sanitization**
```python
# ANTI-PATTERN: Trying to sanitize instead of validate
user_input = re.sub(r'[<>"\'&]', '', user_input)  # Strip dangerous chars
# Attacker uses other vectors

# PATTERN: Validate input, encode output
if not is_valid_input(user_input):
    reject()
# Later, when displaying:
display(Sanitizer.html_encode(user_input))
```

## Related Patterns

- **[Security Patterns](../05-Safety-Engineering/01-Security-Patterns.md)** - Overall security architecture
- **[Error Handling](../05-Safety-Engineering/03-Error-Handling.md)** - Validation error responses
- **[Fault Tolerance](../05-Safety-Engineering/02-Fault-Tolerance.md)** - Handling invalid input gracefully
- **[Clean Code](../04-Best-Practices/01-Clean-Code.md)** - Validation as part of clean interfaces
- **[Testing Pyramid](../06-Testing-Engineering/01-Testing-Pyramid.md)** - Testing validation logic