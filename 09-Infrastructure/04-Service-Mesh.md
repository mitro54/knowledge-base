# Service Mesh

## Title & Summary

**Service Mesh** is a dedicated infrastructure layer for handling service-to-service communication in microservices architectures. This pattern covers traffic management, observability, and security capabilities provided by service meshes like Istio, Linkerd, and Consul Connect, enabling developers to focus on business logic while the mesh handles infrastructure concerns.

---

## Problem Statement

Microservices architectures face significant communication challenges:

- **Complex service discovery** - services need to find each other dynamically
- **Resilience patterns** - retries, timeouts, circuit breakers needed everywhere
- **Traffic management** - canary deployments, A/B testing, load balancing
- **Observability** - distributed tracing, metrics, logging across services
- **Security** - mTLS, authentication, authorization between services
- **Policy enforcement** - rate limiting, quotas, access control

### Example Scenario

```
Without Service Mesh:

┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│  Service A  │────▶│  Service B  │────▶│  Service C  │
└─────────────┘     └─────────────┘     └─────────────┘
       │                  │                  │
       ├── Service Discovery
       ├── Load Balancing
       ├── Retries
       ├── Timeouts
       ├── Circuit Breakers
       ├── mTLS
       ├── Metrics
       └── Tracing

Each service must implement ALL of these!
```

### Code Duplication Problem

```java
// Service A - implements resilience
@HystrixCommand(fallbackMethod = "fallback", timeout = 3000)
public Response callServiceB() {
    return restTemplate.getForObject("http://service-b/api", Response.class);
}

// Service B - implements resilience
@HystrixCommand(fallbackMethod = "fallback", timeout = 3000)
public Response callServiceC() {
    return restTemplate.getForObject("http://service-c/api", Response.class);
}

// Service C - implements resilience
@HystrixCommand(fallbackMethod = "fallback", timeout = 3000)
public Response callExternal() {
    return restTemplate.getForObject("http://external/api", Response.class);
}

// Same patterns repeated everywhere!
```

---

## Solution

### Service Mesh Architecture

```
Application Layer (Developer Focus)
┌─────────────────────────────────────────────────────────┐
│                                                         │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐              │
│  │Service A │  │Service B │  │Service C │              │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘              │
│       │             │             │                     │
└───────┼─────────────┼─────────────┼─────────────────────┘
        │             │             │
        ▼             ▼             ▼
Service Mesh Layer (Infrastructure)
┌─────────────────────────────────────────────────────────┐
│                                                         │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐              │
│  │ Sidecar  │  │ Sidecar  │  │ Sidecar  │              │
│  │   A      │  │   B      │  │   C      │              │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘              │
│       │             │             │                     │
│       └─────────────┴─────────────┘                     │
│                     │                                   │
│         ┌───────────▼───────────┐                       │
│         │   Control Plane       │                       │
│         │  (Istio/Linkerd)      │                       │
│         └───────────────────────┘                       │
│                                                         │
└─────────────────────────────────────────────────────────┘

Capabilities Provided:
├── Traffic Management (routing, load balancing)
├── Resilience (retries, timeouts, circuit breakers)
├── Security (mTLS, authentication, authorization)
├── Observability (metrics, traces, logs)
└── Policy Enforcement (rate limiting, quotas)
```

### Istio - Feature-Rich Service Mesh

```yaml
# Istio Gateway - Entry point for external traffic
apiVersion: networking.istio.io/v1beta1
kind: Gateway
metadata:
  name: my-gateway
spec:
  selector:
    istio: ingressgateway
  servers:
  - port:
      number: 80
      name: http
      protocol: HTTP
    hosts:
    - "*.example.com"
  - port:
      number: 443
      name: https
      protocol: HTTPS
    tls:
      mode: SIMPLE
      credentialName: my-gateway-cert
    hosts:
    - "*.example.com"

---
# VirtualService - Traffic routing rules
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: product-service
spec:
  hosts:
  - product-service
  http:
  # Route to different versions based on headers
  - match:
    - headers:
        version:
          exact: "v2"
    route:
    - destination:
        host: product-service
        subset: v2
      weight: 100
  
  # Default route to v1
  - route:
    - destination:
        host: product-service
        subset: v1
      weight: 100

---
# DestinationRule - Load balancing and traffic policies
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: product-service-dr
spec:
  host: product-service
  trafficPolicy:
    connectionPool:
      tcp:
        maxConnections: 100
      http:
        h2UpgradePolicy: UPGRADE
        maxRequestsPerConnection: 10
    loadBalancer:
      simple: ROUND_ROBIN
    outlierDetection:
      consecutive5xxErrors: 5
      interval: 30s
      baseEjectionTime: 30s
      maxEjectionPercent: 50
  subsets:
  - name: v1
    labels:
      version: v1
  - name: v2
    labels:
      version: v2

---
# Canary Deployment Configuration
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: product-canary
spec:
  hosts:
  - product-service
  http:
  - route:
    # 90% to stable v1
    - destination:
        host: product-service
        subset: v1
      weight: 90
    # 10% to canary v2
    - destination:
        host: product-service
        subset: v2
      weight: 10
```

### Advanced Traffic Management

```yaml
# A/B Testing with Header Manipulation
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: ab-test
spec:
  hosts:
  - web-service
  http:
  - match:
    - headers:
        cookie:
          regex: "user_id=(\d+)"
    route:
    - destination:
        host: web-service
        subset: variant-a
      weight: 50
    - destination:
        host: web-service
        subset: variant-b
      weight: 50

# Fault Injection for Chaos Engineering
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: fault-injection
spec:
  hosts:
  - payment-service
  http:
  - fault:
      delay:
        percentage:
          value: 1
        fixedDelay: 5s
      abort:
        percentage:
          value: 1
        httpStatus: 500
    route:
    - destination:
        host: payment-service
        subset: v1
      weight: 100

# Mirror Traffic for Testing
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: traffic-mirroring
spec:
  hosts:
  - order-service
  http:
  - route:
    - destination:
        host: order-service
        subset: production
      weight: 100
    mirror:
      host: order-service
      subset: staging
    mirror_percent: 10
```

### Linkerd - Lightweight Service Mesh

```yaml
# Linkerd ServiceProfile - API contract definition
apiVersion: linkerd.io/v1alpha2
kind: ServiceProfile
metadata:
  name: orders-api.serviceprofile.linkerd.io
spec:
  route:
  - condition:
      method: GET
      pathRegex: /orders
    name: GET /orders
    opaqueResponse: false
    isRetryable: true
    timeout: 1s
  - condition:
      method: POST
      pathRegex: /orders
    name: POST /orders
    opaqueResponse: false
    isRetryable: false
    timeout: 10s
  - condition:
      method: GET
      pathRegex: /orders/.+
    name: GET /orders/:id
    opaqueResponse: false
    isRetryable: true
    timeout: 1s
  - condition:
      method: "*"
      pathRegex: ".*"
    name: catch-all
    opaqueResponse: false
    isRetryable: false
    timeout: 1s

# Linkerd AuthorizationPolicy
apiVersion: security.linkerd.io/v1beta2
kind: AuthorizationPolicy
metadata:
  name: allow-orders-to-payments
spec:
  targetRef:
    kind: Service
    name: payments-api
  server:
    name: payments-api
    kind: Server
  authorization:
    when:
      client:
        kind: ServiceAccount
        name: orders-api
        namespace: orders
```

### mTLS Configuration

```yaml
# Istio PeerAuthentication - mTLS policy
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  namespace: istio-system
spec:
  mtls:
    mode: STRICT  # STRICT, PERMISSIVE, DISABLE

# Per-namespace mTLS policy
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: permissive-mtls
  namespace: production
spec:
  mtls:
    mode: PERMISSIVE  # Allow both mTLS and plaintext

# Workload-specific mTLS
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: legacy-workload
  namespace: production
spec:
  selector:
    matchLabels:
      app: legacy-service
  mtls:
    mode: DISABLE  # Legacy app doesn't support mTLS
```

---

## When to Use

### Use Service Mesh When:
- ✅ 10+ microservices
- ✅ Need advanced traffic management
- ✅ Require mTLS between services
- ✅ Need distributed observability
- ✅ Multiple teams managing services

### Don't Use When:
- ❌ Small number of services (< 5)
- ❌ Simple monolithic architecture
- ❌ Limited operational expertise
- ❌ Performance is critical (latency sensitive)

---

## Tradeoffs

| Aspect | Without Service Mesh | With Service Mesh |
|--------|---------------------|-------------------|
| **Development Complexity** | High (implement everywhere) | Low (mesh handles it) |
| **Operational Complexity** | Low | High (mesh to manage) |
| **Latency** | ~0ms | ~500μs-1ms per hop |
| **Resource Overhead** | None | ~100-200MB RAM per sidecar |
| **Debugging** | Application-level | Mesh + Application |
| **Flexibility** | Full control | Mesh capabilities only |

### Quantitative Impact

```
Performance Impact:

Without Service Mesh:
├── Request latency: 10ms
├── Memory overhead: 0MB
└── CPU overhead: 0%

With Service Mesh (Istio):
├── Request latency: 10.5-11ms (+500μs-1ms)
├── Memory overhead: 100-200MB per sidecar
└── CPU overhead: 2-5% per sidecar

With Service Mesh (Linkerd):
├── Request latency: 10.2-10.5ms (+200-500μs)
├── Memory overhead: 5-15MB per sidecar
└── CPU overhead: 1-2% per sidecar

Cost Analysis (100 services):
├── Istio sidecars: 100 × 200MB = 20GB RAM
├── Linkerd sidecars: 100 × 10MB = 1GB RAM
└── Decision depends on feature requirements vs cost
```

---

## Implementation Example

### Complete Istio Setup

```yaml
# Namespace with Istio injection enabled
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    istio-injection: enabled

---
# Deployment with sidecar injection
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service
  namespace: production
spec:
  replicas: 3
  selector:
    matchLabels:
      app: order-service
  template:
    metadata:
      labels:
        app: order-service
        version: v1
    spec:
      containers:
      - name: order-service
        image: myregistry/order-service:v1.2.0
        ports:
        - containerPort: 8080
        env:
        - name: DB_HOST
          value: "postgres-service"
        resources:
          requests:
            memory: "256Mi"
            cpu: "250m"
          limits:
            memory: "512Mi"
            cpu: "500m"
        livenessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 10
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /ready
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 5

---
# Service definition
apiVersion: v1
kind: Service
metadata:
  name: order-service
  namespace: production
spec:
  selector:
    app: order-service
  ports:
  - name: http
    port: 80
    targetPort: 8080
  - name: grpc
    port: 9090
    targetPort: 9090

---
# Request Authentication Policy
apiVersion: security.istio.io/v1beta1
kind: RequestAuthentication
metadata:
  name: order-service-auth
  namespace: production
spec:
  selector:
    matchLabels:
      app: order-service
  jwtRules:
  - issuer: "https://auth.example.com"
    audiences:
    - "order-service"

---
# Authorization Policy
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: order-service-policy
  namespace: production
spec:
  selector:
    matchLabels:
      app: order-service
  action: ALLOW
  rules:
  - from:
    - source:
        principals: ["cluster.local/ns/production/sa/user-service"]
    to:
    - operation:
        methods: ["GET", "POST"]
        paths: ["/orders/*"]
```

### Kubernetes Manifest with Service Mesh

```yaml
# Complete microservice with service mesh integration
apiVersion: apps/v1
kind: Deployment
metadata:
  name: payment-service
  namespace: production
spec:
  replicas: 3
  selector:
    matchLabels:
      app: payment-service
      version: v1
  template:
    metadata:
      labels:
        app: payment-service
        version: v1
      annotations:
        # Linkerd retry policy
        config.linkerd.io/retry: "500,503"
        config.linkerd.io/timeout: "1s"
    spec:
      serviceAccountName: payment-service
      containers:
      - name: payment-service
        image: myregistry/payment-service:v2.0.0
        ports:
        - containerPort: 8080
          name: http
        - containerPort: 9090
          name: grpc
        env:
        - name: SERVICE_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.labels['app']
        - name: POD_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        - name: ISTIO_PROXY_ADDR
          value: "127.0.0.1:15001"
        resources:
          requests:
            memory: "512Mi"
            cpu: "500m"
          limits:
            memory: "1Gi"
            cpu: "1000m"
        livenessProbe:
          httpGet:
            path: /actuator/health/liveness
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 10
          timeoutSeconds: 3
          failureThreshold: 3
        readinessProbe:
          httpGet:
            path: /actuator/health/readiness
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 5
          timeoutSeconds: 1
          failureThreshold: 3
      initContainers:
      - name: wait-for-db
        image: busybox
        command: ['sh', '-c', 'until nc -z postgres-service 5432; do sleep 1; done']
```

---

## Anti-Pattern

### ❌ Anti-Pattern: Over-Engineering

```yaml
# Bad: Using service mesh for simple architecture
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: simple-app
spec:
  hosts:
  - simple-app
  http:
  - route:
    - destination:
        host: simple-app
      weight: 100
# Just one service? No need for service mesh!
```

### ❌ Anti-Pattern: Ignoring Performance Impact

```yaml
# Bad: Not considering sidecar overhead
spec:
  containers:
  - name: latency-sensitive-service
    resources:
      requests:
        memory: "64Mi"   # Too small with sidecar!
        cpu: "50m"       # Too small with sidecar!
```

### ❌ Anti-Pattern: Complex Mesh Without Need

```yaml
# Bad: Over-complicated routing for simple case
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: over-engineered
spec:
  hosts:
  - my-service
  http:
  - match:
    - headers:
        x-request-id:
          regex: ".*"
    route:
    - destination:
        host: my-service
        subset: v1
      weight: 100
# Just use a simple Service!
```

### ✅ Correct Approach

```yaml
# Good: Simple service for simple needs
apiVersion: v1
kind: Service
metadata:
  name: simple-app
spec:
  selector:
    app: simple-app
  ports:
  - port: 80
    targetPort: 8080

# Good: Proper resource sizing with sidecar
spec:
  containers:
  - name: my-service
    resources:
      requests:
        memory: "256Mi"   # Plus ~100MB for sidecar
        cpu: "250m"       # Plus ~50m for sidecar
```

---

## Related Patterns

- **[Orchestration](02-Orchestration.md)** - Kubernetes service mesh integration
- **[Containerization](01-Containerization.md)** - Container-based services
- **[Event-Driven Infrastructure](05-Event-Driven-Infrastructure.md)** - Event routing
- **[Security Patterns](../05-Safety-Engineering/01-Security-Patterns.md)** - mTLS and authentication
- **[Observability](../10-Observability/01-Logging.md)** - Distributed tracing