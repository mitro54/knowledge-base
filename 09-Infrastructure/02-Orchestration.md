# Orchestration

## Title & Summary

**Orchestration** is the automated coordination, management, and scheduling of containers across multiple hosts. This pattern covers Kubernetes architecture, core concepts (pods, services, deployments), scaling strategies, and patterns for running containerized applications at scale.

---

## Problem Statement

Running containers in production introduces complex challenges:

- **Manual scaling** doesn't adapt to load changes
- **Service discovery** becomes difficult with dynamic container IPs
- **Self-healing** requires constant monitoring and intervention
- **Rolling updates** are complex to implement manually
- **Resource allocation** across multiple hosts is inefficient

### Example Scenario

```
Without Orchestration:
├── Developer manually starts 10 containers
├── One container crashes → Manual restart required
├── Traffic spikes → Manual scaling needed
├── Deployment → Manual rolling update
└── Load balancing → Manual configuration

With Orchestration:
├── Declare desired state (10 replicas)
├── Auto-healing on crash
├── Auto-scaling on demand
├── Automated rolling updates
└── Built-in service discovery and load balancing
```

---

## Solution

### Kubernetes Architecture

```
┌─────────────────────────────────────────────────┐
│              Kubernetes Cluster                 │
│  ┌───────────────────────────────────────────┐  │
│  │           Control Plane                    │  │
│  │  ┌───────────┐  ┌───────────┐            │  │
│  │  │  API      │  │  Scheduler│            │  │
│  │  │  Server   │  │           │            │  │
│  │  └─────┬─────┘  └───────────┘            │  │
│  │        │                                 │  │
│  │  ┌─────▼─────┐  ┌───────────┐            │  │
│  │  │ Controller│  │  etcd     │            │  │
│  │  │ Managers  │  │  (State)  │            │  │
│  │  └───────────┘  └───────────┘            │  │
│  └───────────────────────────────────────────┘  │
│                        ↓                        │
│  ┌───────────────────────────────────────────┐  │
│  │              Worker Nodes                  │  │
│  │  ┌───────────┐  ┌───────────┐            │  │
│  │  │   Node 1  │  │   Node 2  │  ...       │  │
│  │  │ ┌───────┐ │  │ ┌───────┐ │            │  │
│  │  │ │ Kubelet│ │  │ │Kubelet │ │            │  │
│  │  │ └───┬───┘ │  │ └───┬───┘ │            │  │
│  │  │     │     │  │     │     │            │  │
│  │  │ ┌───▼─────▼──┐  ┌───▼─────▼──┐       │  │
│  │  │ │  Pods      │  │  Pods      │       │  │
│  │  │ └────────────┘  └────────────┘       │  │
│  │  └───────────┘  └───────────┘            │  │
│  └───────────────────────────────────────────┘  │
└─────────────────────────────────────────────────┘
```

### Core Kubernetes Concepts

#### Pod - The Smallest Deployable Unit

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
  labels:
    app: web
    tier: frontend
spec:
  containers:
  - name: nginx
    image: nginx:1.21
    ports:
    - containerPort: 80
    env:
    - name: ENVIRONMENT
      value: "production"
    resources:
      requests:
        memory: "64Mi"
        cpu: "250m"
      limits:
        memory: "128Mi"
        cpu: "500m"
    livenessProbe:
      httpGet:
        path: /health
        port: 80
      initialDelaySeconds: 10
      periodSeconds: 5
```

#### Deployment - Declarative Updates

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
  labels:
    app: web-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web-app
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  template:
    metadata:
      labels:
        app: web-app
    spec:
      containers:
      - name: web-app
        image: myapp:v1.2.0
        ports:
        - containerPort: 3000
        env:
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: db-secret
              key: url
        resources:
          requests:
            memory: "256Mi"
            cpu: "500m"
          limits:
            memory: "512Mi"
            cpu: "1000m"
        livenessProbe:
          httpGet:
            path: /health
            port: 3000
          initialDelaySeconds: 15
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /ready
            port: 3000
          initialDelaySeconds: 5
          periodSeconds: 5
```

#### Service - Stable Network Endpoint

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-app-service
spec:
  type: ClusterIP  # ClusterIP, NodePort, LoadBalancer
  selector:
    app: web-app
  ports:
  - port: 80
    targetPort: 3000
    protocol: TCP
```

### Scaling Strategies

#### Horizontal Pod Autoscaler (HPA)

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: web-app-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: web-app
  minReplicas: 3
  maxReplicas: 20
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
      - type: Percent
        value: 10
        periodSeconds: 60
    scaleUp:
      stabilizationWindowSeconds: 0
      policies:
      - type: Percent
        value: 100
        periodSeconds: 15
      - type: Pods
        value: 4
        periodSeconds: 15
      selectPolicy: Max
```

#### Cluster Autoscaler

```
Cluster Autoscaler Decision Process:

1. Detect pods pending due to resource constraints
2. Attempt to fit pending pods on existing nodes
3. If no fit, calculate nodes to add
4. Select node group and instance type
5. Scale up node group
6. Pods scheduled on new nodes

Scale Down Process:

1. Identify underutilized nodes (<50% CPU/Memory)
2. Check if pods can be rescheduled
3. Drain node safely
4. Remove node from cluster
```

### Service Types and Networking

```
Service Types:

ClusterIP (Internal):
┌─────────────────────────────────┐
│      Kubernetes Cluster        │
│  ┌───────┐     ┌───────┐       │
│  │Pod A  │     │Pod B  │       │
│  └──┬────┘     └──┬────┘       │
│     │            │             │
│     └────┬───────┘             │
│          ▼                     │
│    ┌───────────┐               │
│    │ Service   │ 10.96.0.1    │
│    │ (ClusterIP)│              │
│    └───────────┘               │
└─────────────────────────────────┘
Only accessible within cluster

NodePort (External Access):
┌─────────────────────────────────┐
│      Kubernetes Cluster        │
│  ┌───────┐     ┌───────┐       │
│  │Pod A  │     │Pod B  │       │
│  └──┬────┘     └──┬────┘       │
│     │            │             │
│     └────┬───────┘             │
│          ▼                     │
│    ┌───────────┐               │
│    │ Service   │ 10.96.0.1    │
│    │ (NodePort)│ :30007       │
│    └───────────┘               │
└─────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────┐
│  Node 1: 192.168.1.10:30007    │
│  Node 2: 192.168.1.11:30007    │
└─────────────────────────────────┘
Accessible via any node IP

LoadBalancer (Cloud):
┌─────────────────────────────────┐
│     Cloud Load Balancer        │
│      203.0.113.50:80           │
└──────────────┬──────────────────┘
               │
               ▼
┌─────────────────────────────────┐
│      Kubernetes Cluster        │
│  ┌───────┐     ┌───────┐       │
│  │Pod A  │     │Pod B  │       │
│  └──┬────┘     └──┬────┘       │
│     └────┬───────┘             │
│          ▼                     │
│    ┌───────────┐               │
│    │ Service   │               │
│    │(LoadBalancer)│            │
│    └───────────┘               │
└─────────────────────────────────┘
Cloud provider creates external LB
```

### Ingress - HTTP Routing

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: main-ingress
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - api.example.com
    - www.example.com
    secretName: main-tls
  rules:
  - host: api.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 80
  - host: www.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: web-service
            port:
              number: 80
      - path: /api
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 80
```

---

## When to Use

### Use Kubernetes When:
- ✅ Managing containers across multiple hosts
- ✅ Need automatic scaling and self-healing
- ✅ Running microservices architecture
- ✅ Require high availability
- ✅ Need declarative configuration

### Use HPA When:
- ✅ Workload varies over time
- ✅ Can define scaling metrics
- ✅ Want to optimize costs

### Use StatefulSets When:
- ✅ Running stateful applications (databases)
- ✅ Need stable network identifiers
- ✅ Require ordered deployment/scaling

---

## Tradeoffs

| Approach | Complexity | Flexibility | Learning Curve | Operations |
|---------|----------|-----------|--------------|-----------|
| **Manual Containers** | Low | High | Low | High |
| **Docker Swarm** | Medium | Medium | Medium | Medium |
| **Kubernetes** | High | Very High | High | Low (automated) |

### Quantitative Comparison

```
Operational Overhead:

Manual Container Management:
- Scaling: Manual, 5-10 minutes per action
- Healing: Manual monitoring, 5+ minutes response
- Updates: Manual rolling, 10-30 minutes
- Service Discovery: Manual DNS/config

Kubernetes:
- Scaling: Automatic, <30 seconds
- Healing: Automatic, <10 seconds
- Updates: Automated rolling, 5-15 minutes
- Service Discovery: Built-in DNS

Cost Analysis (100 pods):
Manual: 2 ops engineers × $100k = $200k/year
Kubernetes: 1 ops engineer × $100k = $100k/year
Savings: $100k/year + reduced downtime
```

---

## Implementation Example

### Complete Application Deployment

```yaml
# Namespace for isolation
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    env: production
---
# ConfigMap for configuration
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: production
data:
  APP_ENV: "production"
  LOG_LEVEL: "info"
  REDIS_HOST: "redis-service"
---
# Secret for sensitive data
apiVersion: v1
kind: Secret
metadata:
  name: app-secrets
  namespace: production
type: Opaque
stringData:
  DATABASE_PASSWORD: "secure-password"
  API_KEY: "api-key-value"
---
# Deployment with multiple containers
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
  namespace: production
  labels:
    app: web-app
    version: v1.2.0
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web-app
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  template:
    metadata:
      labels:
        app: web-app
        version: v1.2.0
    spec:
      serviceAccountName: app-service-account
      containers:
      - name: web-app
        image: myregistry/web-app:v1.2.0
        ports:
        - containerPort: 3000
        envFrom:
        - configMapRef:
            name: app-config
        - secretRef:
            name: app-secrets
        resources:
          requests:
            memory: "256Mi"
            cpu: "500m"
          limits:
            memory: "512Mi"
            cpu: "1000m"
        livenessProbe:
          httpGet:
            path: /health
            port: 3000
          initialDelaySeconds: 15
          periodSeconds: 10
          timeoutSeconds: 3
          failureThreshold: 3
        readinessProbe:
          httpGet:
            path: /ready
            port: 3000
          initialDelaySeconds: 5
          periodSeconds: 5
          timeoutSeconds: 1
          failureThreshold: 3
        volumeMounts:
        - name: cache-volume
          mountPath: /app/cache
      volumes:
      - name: cache-volume
        emptyDir: {}
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchExpressions:
                - key: app
                  operator: In
                  values:
                  - web-app
              topologyKey: kubernetes.io/hostname
---
# Service for internal communication
apiVersion: v1
kind: Service
metadata:
  name: web-app-service
  namespace: production
spec:
  type: ClusterIP
  selector:
    app: web-app
  ports:
  - name: http
    port: 80
    targetPort: 3000
---
# Horizontal Pod Autoscaler
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: web-app-hpa
  namespace: production
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: web-app
  minReplicas: 3
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
---
# Ingress for external access
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: web-app-ingress
  namespace: production
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - api.example.com
    secretName: api-tls
  rules:
  - host: api.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: web-app-service
            port:
              number: 80
```

---

## Anti-Pattern

### ❌ Anti-Pattern: Running Database in Pod

```yaml
# Bad: Stateful data in regular Pod
apiVersion: v1
kind: Pod
metadata:
  name: database
spec:
  containers:
  - name: postgres
    image: postgres:15
    # No persistent storage - data lost on restart!
```

### ❌ Anti-Pattern: No Resource Limits

```yaml
# Bad: No resource constraints
spec:
  containers:
  - name: app
    image: myapp
    # Missing resources section
    # Can consume all node resources!
```

### ❌ Anti-Pattern: Running as Root

```yaml
# Bad: Security risk
spec:
  containers:
  - name: app
    image: myapp
    # No securityContext - runs as root
```

### ✅ Correct Approach

```yaml
# Good: StatefulSet for databases with persistent storage
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: database
spec:
  serviceName: database
  replicas: 1
  template:
    spec:
      containers:
      - name: postgres
        image: postgres:15
        volumeMounts:
        - name: data
          mountPath: /var/lib/postgresql/data
        securityContext:
          runAsNonRoot: true
          runAsUser: 999
          allowPrivilegeEscalation: false
        resources:
          requests:
            memory: "512Mi"
            cpu: "500m"
          limits:
            memory: "1Gi"
            cpu: "1000m"
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 10Gi
```

---

## Related Patterns

- **[Containerization](01-Containerization.md)** - Container fundamentals
- **[Service Mesh](04-Service-Mesh.md)** - Advanced service communication
- **[Infrastructure as Code](03-IaC.md)** - Declarative infrastructure
- **[CI/CD](../11-DevOps/01-CI-CD.md)** - Deployment pipelines
- **[Observability](../10-Observability/01-Logging.md)** - Monitoring orchestrated apps