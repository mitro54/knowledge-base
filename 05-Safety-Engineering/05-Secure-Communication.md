# Secure Communication

## Title & Summary

**Secure Communication** encompasses the protocols, patterns, and practices for protecting data as it travels between systems, services, and users. This includes encryption in transit, secure protocol selection, certificate management, and defense against network-based attacks.

## Problem Statement

Data transmitted over networks is vulnerable to multiple threats:

- **Eavesdropping**: Attackers intercept unencrypted traffic
- **Man-in-the-Middle (MitM)**: Attackers intercept and modify communications
- **Session Hijacking**: Stolen session tokens allow unauthorized access
- **Replay Attacks**: Captured messages are resent to cause unauthorized actions
- **DNS Spoofing**: Users redirected to malicious servers
- **Certificate Forgery**: Fake certificates enable impersonation

## Solution

### Core Principles

**1. Encryption in Transit**
- Use TLS 1.2+ (preferably TLS 1.3) for all communications
- Enable Perfect Forward Secrecy (PFS)
- Use strong cipher suites (AES-256-GCM, ChaCha20-Poly1305)

**2. Certificate Management**
- Use certificates from trusted Certificate Authorities (CAs)
- Implement certificate pinning for critical applications
- Automate certificate renewal (Let's Encrypt, ACME)
- Monitor certificate expiration

**3. Protocol Security**
- Disable insecure protocols (SSLv2, SSLv3, TLS 1.0, TLS 1.1)
- Use HSTS (HTTP Strict Transport Security)
- Implement OCSP Stapling for certificate validation

### Implementation Example

```python
import ssl
import socket
from cryptography import x509
from cryptography.hazmat.primitives import hashes, serialization
from cryptography.hazmat.primitives.asymmetric import rsa
from cryptography.hazmat.backends import default_backend

class SecureCommunication:
    def __init__(self, cert_path, key_path, ca_path=None):
        self.cert_path = cert_path
        self.key_path = key_path
        self.ca_path = ca_path
        
    def create_secure_context(self):
        """Create TLS context with security best practices"""
        context = ssl.SSLContext(ssl.PROTOCOL_TLS_SERVER)
        
        # Load certificates
        context.load_cert_chain(
            certfile=self.cert_path,
            keyfile=self.key_path
        )
        
        # Security settings
        context.minimum_version = ssl.TLSVersion.TLSv1_2
        context.options |= ssl.OP_NO_COMPRESSION  # Disable compression (CRIME attack)
        context.options |= ssl.OP_CIPHER_SERVER_PREFERENCE  # Server cipher preference
        
        # Strong cipher suites only
        context.set_ciphers(
            'ECDHE-ECDSA-AES256-GCM-SHA384:'
            'ECDHE-RSA-AES256-GCM-SHA384:'
            'ECDHE-ECDSA-CHACHA20-POLY1305:'
            'ECDHE-RSA-CHACHA20-POLY1305'
        )
        
        # Require certificate verification for client connections
        if self.ca_path:
            context.load_verify_locations(cafile=self.ca_path)
            context.verify_mode = ssl.CERT_REQUIRED
            
        return context
    
    def create_client_context(self):
        """Create secure client context"""
        context = ssl.create_default_context()
        
        # Security settings
        context.minimum_version = ssl.TLSVersion.TLSv1_2
        context.check_hostname = True
        
        return context

# HTTP Headers for Security
SECURITY_HEADERS = {
    'Strict-Transport-Security': 'max-age=31536000; includeSubDomains; preload',
    'X-Content-Type-Options': 'nosniff',
    'X-Frame-Options': 'DENY',
    'X-XSS-Protection': '1; mode=block',
    'Content-Security-Policy': "default-src 'self'",
    'Referrer-Policy': 'strict-origin-when-cross-origin',
    'Permissions-Policy': 'geolocation=(), microphone=(), camera=()'
}
```

### Certificate Pinning

```python
import hashlib

class CertificatePinning:
    def __init__(self, expected_hashes):
        """
        expected_hashes: List of SHA256 hashes of pinned certificates
        """
        self.expected_hashes = expected_hashes
    
    def get_certificate_hash(self, cert_pem):
        """Calculate SHA256 hash of certificate"""
        cert = x509.load_pem_x509_certificate(
            cert_pem.encode(),
            default_backend()
        )
        return hashlib.sha256(cert.public_bytes(
            encoding=serialization.Encoding.DER
        )).hexdigest()
    
    def verify_pin(self, cert_pem):
        """Verify certificate against pinned hashes"""
        cert_hash = self.get_certificate_hash(cert_pem)
        return cert_hash in self.expected_hashes
```

## Tradeoffs & Considerations

| Aspect | Consideration |
|--------|---------------|
| **Performance** | TLS adds latency (~1-2 RTTs for handshake) |
| **Compatibility** | TLS 1.3 not supported by very old clients |
| **Complexity** | Certificate management adds operational overhead |
| **Cost** | EV certificates cost more than DV certificates |
| **Flexibility** | Certificate pinning can break with cert rotation |

### Performance Optimization

```python
# TLS Session Resumption - Reduces handshake overhead
context.session_cache = ssl.SSLSessionCache(1024 * 1024)
context.session_timeout = 300  # 5 minutes

# OCSP Stapling - Improves certificate validation speed
# Configure in web server (nginx example):
# ssl_stapling on;
# ssl_stapling_verify on;
```

## Anti-Patterns

### ❌ BAD: Insecure Default Settings

```python
# Using default TLS settings (may allow weak protocols)
context = ssl.SSLContext(ssl.PROTOCOL_TLS)

# Not verifying certificates
context.verify_mode = ssl.CERT_NONE  # DANGEROUS!

# Using weak cipher suites
context.set_ciphers('ALL:!aNULL:!eNULL')  # Too permissive
```

### ❌ BAD: Hardcoded Certificates

```python
# Never hardcode certificates or keys
CERTIFICATE = """-----BEGIN CERTIFICATE-----
MIID... (actual certificate content)
-----END CERTIFICATE-----"""
```

### ❌ BAD: Ignoring Certificate Errors

```python
# Never ignore certificate verification errors
import urllib3
urllib3.disable_warnings(urllib3.exceptions.InsecureRequestWarning)  # BAD!
response = requests.get('https://example.com', verify=False)  # BAD!
```

### ❌ BAD: Using Deprecated Protocols

```python
# Never use SSLv3, TLS 1.0, or TLS 1.1
context = ssl.SSLContext(ssl.PROTOCOL_SSLv3)  # VULNERABLE!
context = ssl.SSLContext(ssl.PROTOCOL_TLSv1)  # VULNERABLE!
```

## When to Use

| Scenario | Recommendation |
|----------|----------------|
| **Public-facing APIs** | TLS 1.2+ with HSTS, certificate from trusted CA |
| **Internal Services** | TLS with mutual authentication (mTLS) |
| **Mobile Apps** | Certificate pinning for critical endpoints |
| **IoT Devices** | Lightweight TLS (DTLS) with pre-shared keys |
| **Legacy Systems** | TLS 1.2 minimum, plan migration path |

## Related Patterns

- **[Security Patterns](./01-Security-Patterns.md)** - Authentication and authorization
- **[Input Validation](./04-Input-Validation.md)** - Protecting against injection attacks
- **[Error Handling](./03-Error-Handling.md)** - Secure error messages
- **[Fault Tolerance](./02-Fault-Tolerance.md)** - Handling network failures securely

## Verification Checklist

- [ ] TLS 1.2+ enforced, older protocols disabled
- [ ] Strong cipher suites configured
- [ ] HSTS enabled with appropriate max-age
- [ ] Certificate from trusted CA
- [ ] Certificate expiration monitoring in place
- [ ] Perfect Forward Secrecy enabled
- [ ] OCSP Stapling configured (optional but recommended)
- [ ] Security headers set on all responses
- [ ] No mixed content (HTTP resources on HTTPS pages)
- [ ] Certificate pinning for critical applications