# Configuration Management

## Overview

Configuration Management is the practice of systematically managing and controlling the configuration of software systems, infrastructure, and applications. It ensures consistency, reproducibility, and traceability across environments while enabling efficient changes and rollbacks.

### Key Characteristics
- **Centralized Configuration**: Single source of truth for all settings
- **Version Control**: All configurations tracked and versioned
- **Environment Separation**: Different configurations per environment
- **Automation**: Programmatic configuration application
- **Audit Trail**: Complete history of configuration changes

---

## Problem Statement

Managing configuration across multiple environments and systems presents significant challenges:

- **Configuration Drift**: Environments diverge over time due to manual changes
- **Inconsistency**: Different settings across development, staging, and production
- **Manual Errors**: Human mistakes in copying or typing configuration values
- **Lack of Traceability**: Unknown who changed what and when
- **Difficult Rollbacks**: No easy way to revert to previous configuration state
- **Secret Management**: Sensitive data scattered across systems insecurely
- **Environment Parity**: "Works on my machine" syndrome

### Consequences
- Production bugs caused by configuration differences
- Security vulnerabilities from exposed secrets
- Extended troubleshooting time due to environment differences
- Compliance violations from untracked changes
- Service outages from incorrect configurations

---

## Solution

Configuration Management addresses these challenges through systematic approaches:

### Configuration as Code
Treat configuration files as code with version control, reviews, and automation.

```
Configuration Files → Git Repository → Automated Application
```

### Configuration Layers
Organize configuration into logical layers:

```
┌─────────────────────────────────────┐
│   Environment-Specific Config       │  (dev, staging, prod)
├─────────────────────────────────────┤
│   Application-Specific Config       │  (per service settings)
├─────────────────────────────────────┤
│   Organization/Team Config          │  (shared defaults)
├─────────────────────────────────────┤
│   Platform/Infrastructure Config    │  (base settings)
└─────────────────────────────────────┘
```

### Configuration Sources Hierarchy
```
1. Command-line arguments (highest priority)
2. Environment variables
3. Configuration files
4. Service configuration store
5. Hardcoded defaults (lowest priority)
```

### Configuration Management Tools
- **Infrastructure Tools**: Ansible, Chef, Puppet, SaltStack
- **Cloud Native**: Kubernetes ConfigMaps, Secrets
- **Configuration Stores**: Consul, etcd, AWS Parameter Store
- **Secrets Management**: HashiCorp Vault, AWS Secrets Manager

---

## When to Use

### Appropriate Scenarios
- Multi-environment deployments (dev, staging, production)
- Microservices architectures with many services
- Compliance requirements for configuration audit trails
- Teams practicing Infrastructure as Code
- Applications requiring environment-specific settings

### Prerequisites
- Version control system (Git)
- Automation capabilities (scripts, CI/CD)
- Understanding of environment separation
- Security practices for sensitive data

### Indicators for Adoption
- Manual configuration changes causing incidents
- Different behaviors across environments
- Difficulty reproducing issues
- Configuration stored in multiple locations
- Secrets embedded in code or configuration files

---

## Tradeoffs

### Advantages
- **Consistency**: Identical configurations across environments
- **Reproducibility**: Can recreate any environment from configuration
- **Auditability**: Complete history of all changes
- **Automation**: Configuration applied automatically
- **Collaboration**: Team can review configuration changes
- **Rollback**: Easy to revert to previous configuration

### Disadvantages
- **Initial Setup**: Time investment to establish practices
- **Learning Curve**: Team must learn tools and patterns
- **Complexity**: Can become complex with many services
- **Tool Overhead**: Additional infrastructure to manage

### Performance Considerations
- Configuration fetch latency in distributed systems
- Cache invalidation for configuration updates
- Storage costs for configuration history
- Network overhead for centralized configuration stores

---

## Implementation Example

### Twelve-Factor App Configuration

```bash
# .env file (development only - NEVER commit to Git)
DATABASE_URL=postgresql://localhost:5432/myapp_dev
REDIS_URL=redis://localhost:6379
API_KEY=dev-api-key-12345
LOG_LEVEL=debug

# Production: Use environment variables or secrets manager
# Never store secrets in code or version control
```

### Application Configuration (Node.js)

```javascript
// config/index.js
const dotenv = require('dotenv');
const path = require('path');

// Load environment-specific config
const envFile = path.resolve(__dirname, `../.env.${process.env.NODE_ENV || 'development'}`);
try {
    dotenv.config({ path: envFile });
} catch (e) {
    // File doesn't exist, that's okay
}

const config = {
    // Base configuration (defaults)
    nodeEnv: process.env.NODE_ENV || 'development',
    port: parseInt(process.env.PORT, 10) || 3000,
    logLevel: process.env.LOG_LEVEL || 'info',
    
    // Database configuration
    database: {
        host: process.env.DATABASE_HOST || 'localhost',
        port: parseInt(process.env.DATABASE_PORT, 10) || 5432,
        name: process.env.DATABASE_NAME || 'myapp',
        user: process.env.DATABASE_USER,
        password: process.env.DATABASE_PASSWORD,
        poolSize: parseInt(process.env.DATABASE_POOL_SIZE, 10) || 10,
    },
    
    // Redis configuration
    redis: {
        host: process.env.REDIS_HOST || 'localhost',
        port: parseInt(process.env.REDIS_PORT, 10) || 6379,
        password: process.env.REDIS_PASSWORD,
    },
    
    // API configuration
    api: {
        version: process.env.API_VERSION || 'v1',
        rateLimit: parseInt(process.env.API_RATE_LIMIT, 10) || 100,
        timeout: parseInt(process.env.API_TIMEOUT, 10) || 30000,
    },
    
    // Feature flags
    features: {
        newCheckout: process.env.FEATURE_NEW_CHECKOUT === 'true',
        darkMode: process.env.FEATURE_DARK_MODE === 'true',
    },
};

// Validate required configuration
function validateConfig() {
    const required = ['DATABASE_USER', 'DATABASE_PASSWORD'];
    const missing = required.filter(key => !process.env[key]);
    
    if (missing.length > 0 && config.nodeEnv !== 'development') {
        throw new Error(`Missing required configuration: ${missing.join(', ')}`);
    }
}

validateConfig();
module.exports = config;
```

### Kubernetes ConfigMaps and Secrets

```yaml
# ConfigMap for non-sensitive configuration
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: production
data:
  app.properties: |
    log.level=info
    cache.size=1000
    max.connections=100
  feature-flags.json: |
    {
      "newCheckout": true,
      "darkMode": false,
      "betaFeatures": false
    }

---
# Secret for sensitive configuration (base64 encoded)
apiVersion: v1
kind: Secret
metadata:
  name: app-secrets
  namespace: production
type: Opaque
data:
  database-url: cG9zdGdyZXNxbDovL3VzZXI6cGFzc0BkYi5sb2NhbGhvc3Q6NTQzMi9teWFwcA==
  api-key: cHJvZC1hcGkta2V5LTEyMzQ1Njc4OQ==
  jwt-secret: c3VwZXItc2VjcmV0LWp3dC1rZXktdG8ta2VlcC1zYWZl

---
# Deployment using ConfigMap and Secret
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  template:
    spec:
      containers:
      - name: myapp
        image: myapp:latest
        env:
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: app-secrets
              key: database-url
        - name: API_KEY
          valueFrom:
            secretKeyRef:
              name: app-secrets
              key: api-key
        - name: LOG_LEVEL
          valueFrom:
            configMapKeyRef:
              name: app-config
              key: log.level
        volumeMounts:
        - name: config-volume
          mountPath: /app/config
      volumes:
      - name: config-volume
        configMap:
          name: app-config
```

### Ansible Configuration Management

```yaml
# playbook.yml
---
- name: Configure web servers
  hosts: webservers
  become: yes
  
  vars:
    app_name: "myapp"
    app_port: 8080
    app_env: "{{ 'production' if inventory_hostname in groups.prod else 'staging' }}"
    
  tasks:
    - name: Install required packages
      apt:
        name:
          - nginx
          - nodejs
          - npm
        state: present
    
    - name: Create application directory
      file:
        path: "/opt/{{ app_name }}"
        state: directory
        owner: www-data
        group: www-data
        mode: '0755'
    
    - name: Deploy application configuration
      template:
        src: "templates/app.conf.j2"
        dest: "/opt/{{ app_name }}/config/app.conf"
        owner: www-data
        group: www-data
        mode: '0644'
      notify: Restart application
    
    - name: Deploy Nginx configuration
      template:
        src: "templates/nginx.conf.j2"
        dest: "/etc/nginx/sites-available/{{ app_name }}"
        owner: root
        group: root
        mode: '0644'
      notify: Reload nginx
    
    - name: Enable application site
      file:
        src: "/etc/nginx/sites-available/{{ app_name }}"
        dest: "/etc/nginx/sites-enabled/{{ app_name }}"
        state: link
      notify: Reload nginx

  handlers:
    - name: Restart application
      systemd:
        name: "{{ app_name }}"
        state: restarted
    
    - name: Reload nginx
      systemd:
        name: nginx
        state: reloaded
```

### HashiCorp Vault Integration

```bash
# Vault configuration for dynamic secrets

# Enable Kubernetes auth method
vault auth enable kubernetes

# Configure Kubernetes auth
vault write auth/kubernetes/config \
    kubernetes_host="https://kubernetes.default.svc" \
    kubernetes_ca_cert=@/var/run/secrets/kubernetes.io/serviceaccount/ca.crt \
    token_reviewer_jwt="$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)"

# Create role for application
vault write auth/kubernetes/role/myapp \
    bound_service_account_names=myapp-sa \
    bound_service_account_namespaces=production \
    ttl=1h \
    policies=app-policy

# Enable database secrets engine
vault secrets enable database

# Configure PostgreSQL connection
vault write database/config/postgres \
    plugin_name=postgresql-database-plugin \
    allowed_roles="myapp-role" \
    connection_url="postgresql://{{username}}:{{password}}@localhost:5432/myapp?sslmode=disable" \
    username="vault_user" \
    password="vault_password"

# Create dynamic role
vault write database/roles/myapp-role \
    db_name=postgres \
    creation_statements="CREATE USER {{name}} WITH PASSWORD '{{password}}' VALID UNTIL '{{expiration}}'; GRANT SELECT, INSERT, UPDATE ON ALL TABLES IN SCHEMA public TO {{name}};" \
    default_ttl="1h" \
    max_ttl="24h"
```

---

## Anti-Pattern

### Common Mistakes and Pitfalls

#### ❌ Hardcoding Configuration Values
```javascript
// ANTI-PATTERN: Hardcoded values
const config = {
    databaseUrl: 'postgresql://user:pass@localhost:5432/myapp',
    apiKey: 'sk-1234567890abcdef',
    secretKey: 'my-super-secret-key-12345'
};
```

**Correct Approach:**
```javascript
// Use environment variables with sensible defaults
const config = {
    databaseUrl: process.env.DATABASE_URL || 'postgresql://localhost:5432/myapp',
    apiKey: process.env.API_KEY,  // Required, no default
    secretKey: process.env.SECRET_KEY,  // Required, no default
};
```

#### ❌ Storing Secrets in Version Control
```bash
# ANTI-PATTERN: .env file committed to Git
$ git log --all --full-history -- .env
commit abc123 - Added database credentials
```

**Correct Approach:**
```bash
# Add to .gitignore
echo ".env" >> .gitignore
echo ".env.*" >> .gitignore

# Provide template without actual values
cat > .env.example << 'EOF'
DATABASE_URL=postgresql://user:password@host:5432/database
API_KEY=your-api-key-here
SECRET_KEY=your-secret-key-here
EOF
```

#### ❌ Mixing Environments
```javascript
// ANTI-PATTERN: Same config for all environments
if (process.env.NODE_ENV === 'production') {
    config.debug = true;  // Debug enabled in production!
    config.logLevel = 'debug';
}
```

**Correct Approach:**
```javascript
// Environment-specific configuration
const envConfig = {
    development: { debug: true, logLevel: 'debug' },
    staging: { debug: false, logLevel: 'info' },
    production: { debug: false, logLevel: 'warn' },
};
Object.assign(config, envConfig[process.env.NODE_ENV]);
```

#### ❌ Configuration Drift Through Manual Changes
Making direct changes to production servers without updating configuration management.

#### ❌ Storing Secrets in Plain Text
Even in "secure" locations, secrets should be encrypted at rest.

---

## Related Patterns

### See Also
- [CI/CD](./01-CI-CD.md) - Automated configuration deployment
- [Infrastructure as Code](09-Infrastructure/03-IaC.md) - Infrastructure configuration
- [Feature Flags](./04-Feature-Flags.md) - Runtime configuration of features

### Complementary Patterns
- [Containerization](09-Infrastructure/01-Containerization.md) - Configuration in containers
- [Orchestration](09-Infrastructure/02-Orchestration.md) - Configuration management at scale
- [Security Patterns](05-Safety-Engineering/01-Security-Patterns.md) - Secure configuration practices

### Alternative Approaches
- [Service Mesh](09-Infrastructure/04-Service-Mesh.md) - Distributed configuration management
- [Event-Driven Infrastructure](09-Infrastructure/05-Event-Driven-Infrastructure.md) - Reactive configuration updates