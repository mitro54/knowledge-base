# Security Patterns

## Title & Summary

Security patterns are proven solutions to common security problems in software architecture and design. This document covers authentication, authorization, encryption, and secure coding practices that protect systems from vulnerabilities and attacks.

## Problem Statement

Modern software systems face numerous security threats:

- **Unauthorized Access**: Attackers attempting to gain access to protected resources
- **Data Breaches**: Sensitive information being exposed or stolen
- **Injection Attacks**: Malicious code injected through input fields
- **Session Hijacking**: Attackers taking over user sessions
- **Cross-Site Scripting (XSS)**: Malicious scripts injected into web pages
- **Cross-Site Request Forgery (CSRF)**: Unauthorized commands from trusted users
- **Man-in-the-Middle Attacks**: Interception of communication channels

Without proper security patterns, systems are vulnerable to these attacks, leading to data loss, reputation damage, and financial losses.

## Solution

### Authentication Patterns

**Multi-Factor Authentication (MFA)**
```
Authentication = Something You Know + Something You Have + Something You Are
```

- **Knowledge Factor**: Passwords, PINs, security questions
- **Possession Factor**: Tokens, smartphones, smart cards
- **Inherence Factor**: Biometrics (fingerprint, face, voice)

**OAuth 2.0 / OpenID Connect**
- Delegated authorization framework
- Token-based authentication
- Supports multiple grant types (authorization code, implicit, password, client credentials)

**JWT (JSON Web Tokens)**
- Stateless authentication tokens
- Contains payload with claims
- Signed to prevent tampering

### Authorization Patterns

**Role-Based Access Control (RBAC)**
```
User → Role → Permission → Resource
```

**Attribute-Based Access Control (ABAC)**
```
Access = f(User Attributes, Resource Attributes, Environment, Action)
```

**Permission-Based Access Control**
- Fine-grained permissions
- Direct user-permission mapping

### Encryption Patterns

**Data at Rest Encryption**
- Database encryption (TDE, column-level)
- File encryption (AES-256)
- Key management (HSM, KMS)

**Data in Transit Encryption**
- TLS/SSL for network communication
- Certificate pinning for mobile apps
- End-to-end encryption for messaging

### Secure Coding Patterns

**Input Validation**
```python
# Whitelist approach - only allow known safe inputs
ALLOWED_CHARS = set('abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789_-')
def validate_username(username):
    return all(c in ALLOWED_CHARS for c in username) and len(username) <= 32
```

**Output Encoding**
- HTML encoding for web output
- URL encoding for parameters
- JavaScript encoding for dynamic content

**Parameterized Queries**
```python
# Prevent SQL injection
cursor.execute("SELECT * FROM users WHERE username = %s", (username,))
# NOT: cursor.execute(f"SELECT * FROM users WHERE username = '{username}'")
```

## When to Use

| Pattern | Use When |
|---------|----------|
| MFA | Protecting sensitive data, financial systems, admin access |
| OAuth 2.0 | Third-party integrations, single sign-on scenarios |
| JWT | Stateless microservices, mobile app authentication |
| RBAC | Systems with clear role hierarchies (admin, user, guest) |
| ABAC | Complex access control with many attributes |
| TLS 1.3 | All network communication, especially over public networks |
| AES-256 | Encrypting sensitive data at rest |

## Tradeoffs

### Authentication Tradeoffs

| Approach | Security | Usability | Performance |
|----------|----------|-----------|-------------|
| Password Only | Low | High | High |
| Password + OTP | Medium | Medium | Medium |
| Password + Biometric | High | High | Medium |
| Certificate-Based | Very High | Low | High |
| Biometric Only | High | Very High | Medium |

### Authorization Tradeoffs

| Approach | Granularity | Complexity | Performance |
|----------|-------------|------------|-------------|
| RBAC | Medium | Low | High |
| ABAC | Very High | High | Medium |
| Permission-Based | High | Medium | Medium |
| ACL | High | Medium | Low (at scale) |

### Encryption Tradeoffs

| Approach | Security | Performance | Key Management |
|----------|----------|-------------|----------------|
| AES-128 | High | Very High | Simple |
| AES-256 | Very High | High | Simple |
| RSA-2048 | High | Low | Complex |
| RSA-4096 | Very High | Very Low | Complex |
| ChaCha20 | Very High | Very High | Simple |

## Implementation Example

### Complete Authentication & Authorization System

```python
from functools import wraps
from datetime import datetime, timedelta
import jwt
import hashlib
import secrets

# Configuration
SECRET_KEY = secrets.token_urlsafe(32)
ALGORITHM = "HS256"
ACCESS_TOKEN_EXPIRE_MINUTES = 30
REFRESH_TOKEN_EXPIRE_DAYS = 7

class SecurityService:
    def __init__(self):
        self.password_hash_rounds = 100000
        
    def hash_password(self, password: str, salt: str = None) -> tuple:
        """Hash password with bcrypt-like approach"""
        if not salt:
            salt = secrets.token_hex(16)
        hashed = hashlib.pbkdf2_hmac(
            'sha256',
            password.encode('utf-8'),
            salt.encode('utf-8'),
            self.password_hash_rounds
        )
        return salt, hashed.hex()
    
    def verify_password(self, password: str, salt: str, hashed: str) -> bool:
        """Verify password against hash"""
        _, new_hash = self.hash_password(password, salt)
        return secrets.compare_digest(new_hash, hashed)
    
    def create_access_token(self, user_id: str, roles: list = None) -> str:
        """Create JWT access token"""
        payload = {
            'sub': user_id,
            'roles': roles or [],
            'exp': datetime.utcnow() + timedelta(minutes=ACCESS_TOKEN_EXPIRE_MINUTES),
            'iat': datetime.utcnow(),
            'type': 'access'
        }
        return jwt.encode(payload, SECRET_KEY, algorithm=ALGORITHM)
    
    def create_refresh_token(self, user_id: str) -> str:
        """Create JWT refresh token"""
        payload = {
            'sub': user_id,
            'exp': datetime.utcnow() + timedelta(days=REFRESH_TOKEN_EXPIRE_DAYS),
            'iat': datetime.utcnow(),
            'type': 'refresh'
        }
        return jwt.encode(payload, SECRET_KEY, algorithm=ALGORITHM)
    
    def decode_token(self, token: str) -> dict:
        """Decode and verify JWT token"""
        try:
            payload = jwt.decode(token, SECRET_KEY, algorithms=[ALGORITHM])
            return payload
        except jwt.ExpiredSignatureError:
            raise ValueError("Token expired")
        except jwt.InvalidTokenError:
            raise ValueError("Invalid token")

# RBAC Authorization
class AuthorizationService:
    def __init__(self):
        # Role hierarchy
        self.role_hierarchy = {
            'super_admin': ['admin', 'moderator', 'user', 'guest'],
            'admin': ['moderator', 'user', 'guest'],
            'moderator': ['user', 'guest'],
            'user': ['guest'],
            'guest': []
        }
        
        # Role permissions
        self.role_permissions = {
            'super_admin': ['*'],  # All permissions
            'admin': ['read:', 'write:', 'delete:', 'admin:'],
            'moderator': ['read:', 'write:', 'moderate:'],
            'user': ['read:', 'write:own:'],
            'guest': ['read:public:']
        }
    
    def has_permission(self, user_roles: list, required_permission: str) -> bool:
        """Check if user has required permission"""
        for role in user_roles:
            permissions = self.role_permissions.get(role, [])
            if '*' in permissions:
                return True
            for perm in permissions:
                if self._permission_matches(perm, required_permission):
                    return True
        return False
    
    def _permission_matches(self, pattern: str, permission: str) -> bool:
        """Check if permission matches pattern"""
        if pattern.endswith('*'):
            return permission.startswith(pattern.rstrip('*'))
        return pattern == permission
    
    def requires_role(self, *required_roles):
        """Decorator to require specific roles"""
        def decorator(func):
            @wraps(func)
            def wrapper(user, *args, **kwargs):
                if not any(role in required_roles for role in user.get('roles', [])):
                    raise PermissionError(f"Required roles: {required_roles}")
                return func(user, *args, **kwargs)
            return wrapper
        return decorator
    
    def requires_permission(self, permission: str):
        """Decorator to require specific permission"""
        def decorator(func):
            @wraps(func)
            def wrapper(user, *args, **kwargs):
                if not self.has_permission(user.get('roles', []), permission):
                    raise PermissionError(f"Required permission: {permission}")
                return func(user, *args, **kwargs)
            return wrapper
        return decorator

# Usage Example
security = SecurityService()
authz = AuthorizationService()

# Authentication
def login(username: str, password: str) -> dict:
    # Get user from database
    user = get_user_from_db(username)
    if not user or not security.verify_password(password, user['salt'], user['password_hash']):
        raise ValueError("Invalid credentials")
    
    # Generate tokens
    access_token = security.create_access_token(user['id'], user['roles'])
    refresh_token = security.create_refresh_token(user['id'])
    
    return {
        'access_token': access_token,
        'refresh_token': refresh_token,
        'token_type': 'Bearer',
        'expires_in': ACCESS_TOKEN_EXPIRE_MINUTES * 60
    }

# Protected endpoints
@authz.requires_role('admin', 'super_admin')
def delete_user(requester, user_id: str):
    """Only admins can delete users"""
    return delete_user_from_db(user_id)

@authz.requires_permission('write:own:')
def update_profile(requester, updates: dict):
    """Users can only update their own profile"""
    return update_user_profile(requester['sub'], updates)
```

## Anti-Pattern

### ❌ Security Anti-Patterns

**1. Hardcoded Credentials**
```python
# ANTI-PATTERN: Never hardcode secrets
DB_PASSWORD = "super_secret_password_123"
API_KEY = "sk-1234567890abcdef"

# PATTERN: Use environment variables or secret management
import os
from dotenv import load_dotenv

load_dotenv()
DB_PASSWORD = os.getenv('DB_PASSWORD')
API_KEY = os.getenv('API_KEY')
```

**2. Rolling Your Own Cryptography**
```python
# ANTI-PATTERN: Custom encryption is dangerous
def encrypt(data, key):
    return ''.join(chr(ord(c) ^ ord(key[i % len(key)])) for i, c in enumerate(data))

# PATTERN: Use established libraries
from cryptography.fernet import Fernet
fernet = Fernet(key)  # Properly generated key
encrypted = fernet.encrypt(data.encode())
```

**3. Storing Passwords in Plain Text**
```python
# ANTI-PATTERN: Never store plain text passwords
user['password'] = password

# PATTERN: Always hash with proper algorithm
salt, hashed = security.hash_password(password)
user['password_salt'] = salt
user['password_hash'] = hashed
```

**4. Insufficient Input Validation**
```python
# ANTI-PATTERN: Trusting user input
query = f"SELECT * FROM users WHERE id = {user_input}"

# PATTERN: Parameterized queries
cursor.execute("SELECT * FROM users WHERE id = %s", (user_input,))
```

**5. Missing Rate Limiting**
```python
# ANTI-PATTERN: No protection against brute force
def login(username, password):
    return authenticate(username, password)

# PATTERN: Implement rate limiting
from ratelimit import limits

@limits(calls=5, period=60)  # 5 attempts per minute
def login(username, password):
    return authenticate(username, password)
```

## Related Patterns

- **[Error Handling](../05-Safety-Engineering/03-Error-Handling.md)** - Secure error messages that don't leak information
- **[Input Validation](../05-Safety-Engineering/04-Input-Validation.md)** - First line of defense against injection attacks
- **[Secure Communication](../05-Safety-Engineering/05-Secure-Communication.md)** - TLS/SSL for data in transit
- **[Fault Tolerance](../05-Safety-Engineering/02-Fault-Tolerance.md)** - Security incident response and recovery
- **[Monitoring](../10-Observability/02-Monitoring.md)** - Security event monitoring and alerting
- **[Logging](../10-Observability/01-Logging.md)** - Security audit logging