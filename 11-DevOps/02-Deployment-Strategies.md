# Deployment Strategies

Deployment strategies are formal approaches for releasing new versions of applications to production environments while managing risk and availability.

## Summary

Deployment strategies define the mechanics of how software is transitioned from a staged artifact to a live, user-facing service. In modern DevOps, the choice of strategy is driven by the need to balance deployment frequency with system reliability. High-performing teams utilize automated strategies like Blue-Green or Canary deployments to achieve zero-downtime releases and rapid rollback capabilities. The objective is to decouple the process of "deployment" (installing the code) from "release" (exposing it to users), thereby reducing the blast radius of potential failures.

**Key Characteristics:**
- **Risk Mitigation**: Strategies designed to minimize the impact of failed deployments on the user base.
- **Rollback Capability**: The ability to instantly revert to a previously known good state.
- **Downtime Management**: Eliminating or strictly controlling service unavailability during transitions.
- **Traffic Control**: Granular management of user requests (sharding, weighting) during the rollout.
- **Resource Efficiency**: Optimizing the use of compute resources (CPU, Memory) during the upgrade process.
- **Automation First**: Every stage of the deployment is defined as code (e.g., K8s controller logic).

---

## Problem Statement

### The Challenge

Updating live software systems is inherently risky. Traditional "replace and restart" methods often lead to service outages, state corruption, and inconsistent user experiences. Without a structured deployment strategy, teams are forced into "Big Bang" releases that are difficult to test in production conditions and even harder to roll back when things go wrong.

### Context

- **Historical Context**: Early web services required "Maintenance Mode" pages during deployments, stopping all business operations.
- **Technical Context**: Modern cloud-native applications run across hundreds of containers; updating them one-by-one requires sophisticated orchestration.
- **Business Context**: In the 24/7 global economy, even 10 minutes of downtime can result in massive revenue loss and brand damage.

### Consequences of Not Addressing

- **Service Downtime**: Users cannot access critical functionality during updates, violating SLAs.
- **Deployment Failures**: Small bugs in new versions can propagate to 100% of the user base instantly.
- **Data Migration Issues**: Schema changes can break old application versions if not managed with "Expand/Contract" patterns.
- **State Management Failures**: In-flight requests or active sessions are dropped during restarts.
- **Rollback Complexity**: Manual "undo" operations are error-prone and slow.
- **Team Fatigue**: High-stakes, manual deployments lead to developer burnout and avoidance of frequent releases.

---

## Solution

### The Multi-Strategy Approach

Deployment strategies provide controlled mechanisms to transition between Version A (Stable) and Version B (New). The selection of a strategy depends on infrastructure capabilities and risk tolerance.

```
      Traditional (High Risk)              Modern (Low Risk)
    ┌─────────────────────────┐        ┌─────────────────────────┐
    │       Big Bang          │        │      Blue-Green         │
    │  [Old] ─▶ [Off] ─▶ [New]│        │  [Blue] ─◀─▶ [Green]    │
    └─────────────────────────┘        └─────────────────────────┘
                 │                                  │
                 ▼                                  ▼
    ┌─────────────────────────┐        ┌─────────────────────────┐
    │       Rolling           │        │       Canary            │
    │  ●●●● ─▶ ●●●○ ─▶ ○○○○   │        │  100% ─▶ 90/10 ─▶ 100%  │
    └─────────────────────────┘        └─────────────────────────┘
```

### Core Strategies

1. **Recreate (Big Bang)**: Terminate version A then start version B. Simple but involves downtime.
2. **Rolling Update**: Incrementally replace instances of version A with version B. Used by default in Kubernetes.
3. **Blue-Green**: Run two identical environments. Switch traffic from one to the other at the router/load-balancer level.
4. **Canary**: Release version B to a small subset of users first, then progressively widen the rollout based on telemetry.
5. **Shadow**: Mirror production traffic to version B in the background to validate performance without user impact.
6. **A/B Testing**: Route traffic based on user attributes (e.g., location, tier) to measure product-specific outcomes.

### How It Addresses the Problem

- **Zero Downtime**: Modern strategies use overlapping lifecycle management to keep services available.
- **Validation in Production**: Canary and Shadow strategies allow for "testing in prod" with zero risk.
- **Instant Rollback**: Blue-Green allows for an immediate switch back to the "Blue" environment if "Green" fails.
- **Reduced Blast Radius**: Canary limits the impact of bugs to a tiny percentage of users.

---

## When to Use

### Appropriate Scenarios

| Strategy | Suitability | Risk Level | Infrastructure Cost |
|----------|-------------|------------|---------------------|
| Big Bang | ⭐ (Legacy/Dev) | High | Low |
| Rolling | ⭐⭐⭐ (General) | Medium | Medium |
| Blue-Green | ⭐⭐⭐⭐⭐ (Zero-Downtime) | Low | High (2x) |
| Canary | ⭐⭐⭐⭐⭐ (Critical) | Lowest | Medium |
| Shadow | ⭐⭐⭐⭐ (Perf Validation) | Lowest | High |

### Prerequisites

- **Orchestration**: A system like Kubernetes, ECS, or Nomad to manage container lifecycles.
- **Service Discovery**: Automated way for clients/load balancers to find new instances.
- **Health Probes**: Automated "Liveness" and "Readiness" checks.
- **Externalized State**: Sessions and databases must be decoupled from app instances.
- **Infrastructure as Code (IaC)**: Reproducible environment definitions.

### Indicators for Suitability

- **SLA Requirements**: If 99.9% uptime is required, Blue-Green or Canary is mandatory.
- **Traffic Volume**: High-traffic systems benefit most from Canary to detect subtle performance regressions.
- **Resource Constraints**: If you cannot afford 2x infrastructure, Rolling is the best compromise.
- **Release Frequency**: Multiple daily deployments require fully automated, low-risk strategies (Canary).

---

## Tradeoffs

### Advantages

| Strategy | Key Advantage | Description |
|----------|---------------|-------------|
| **Big Bang** | Simplicity | No state synchronization or version overlap issues. |
| **Rolling** | Resource Neutral | Doesn't require extra servers; uses current capacity. |
| **Blue-Green** | Instant Rollback | Reverting is as simple as flipping a load balancer switch. |
| **Canary** | Blast Radius Control | Caught bugs only affect 1-5% of traffic. |
| **Shadow** | Real-world Load | Tests system behavior with actual production data. |
| **A/B Testing** | Data-driven | Validates business metrics (conversion, engagement). |

### Disadvantages

| Strategy | Key Drawback | Description |
|----------|--------------|-------------|
| **Big Bang** | Downtime | Complete service interruption during deploy. |
| **Rolling** | Slow Rollback | Must roll back instance by instance, taking time. |
| **Blue-Green** | Cost | Requires doubling your server count during deploy. |
| **Canary** | Complexity | Requires advanced traffic routing (Service Mesh). |
| **Shadow** | Write Complexity | Hard to test stateful operations (DB writes) without side effects. |
| **A/B Testing** | Overlap Issues | Can lead to data inconsistency for users switching segments. |

### Performance Considerations

- **Initialization Latency**: New versions often have "cold start" performance dips. Rolling updates must account for this via "minReadySeconds".
- **Network Overhead**: Shadow deployments double the incoming traffic load on the network layer.
- **Database Load**: Blue-Green switches can cause a sudden surge of connections to the shared database.

### Complexity Implications

- **Initial Complexity**: Rolling is built-in to most tools; Canary requires Istio/Linkerd.
- **Operational Complexity**: Canary requires automated "judgment" (checking metrics and auto-promoting).
- **Automation Level**: High—manual Blue-Green is dangerous; it must be fully scripted or orchestrated.

---

## Implementation Example

### Basic Project Structure for Deployment

```
deployment-infra/
├── k8s/
│   ├── base/               # Stable definitions
│   ├── blue-green/         # Strategy overlays
│   └── canary/             # Istio/Flagger configs
├── scripts/
│   ├── deploy-rolling.sh   # Default script
│   └── promote-canary.sh   # Evaluation logic
└── monitoring/
    └── deployment-health.promql # Health validation queries
```

### Blue-Green Deployment (Kubernetes)

Using labels to switch traffic between two distinct deployments.

```yaml
# 1. The Production Service (pointing to Blue)
apiVersion: v1
kind: Service
metadata:
  name: myapp-production
spec:
  selector:
    app: myapp
    version: blue # Change this to 'green' to switch traffic
  ports:
    - port: 80
      targetPort: 8080

---
# 2. Blue Deployment (v1.0.0)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-blue
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
      version: blue
  template:
    metadata:
      labels:
        app: myapp
        version: blue
    spec:
      containers:
      - name: myapp
        image: myapp:1.0.0

---
# 3. Green Deployment (v2.0.0 - Standby)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-green
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
      version: green
  template:
    metadata:
      labels:
        app: myapp
        version: green
    spec:
      containers:
      - name: myapp
        image: myapp:2.0.0
```

**Traffic Switch Script:**
```bash
#!/bin/bash
# Blue-Green Traffic Switcher

# 1. Verify Green is ready
kubectl rollout status deployment/myapp-green

# 2. Run Smoke Tests
curl -s http://myapp-green:8080/health | grep "UP" || exit 1

# 3. Flip the Switch
kubectl patch service myapp-production -p '{"spec":{"selector":{"version":"green"}}}'

echo "Traffic successfully routed to Green (v2.0.0)"
```

### Canary Deployment (Istio Service Mesh)

Using weighted routing to gradually introduce a new version.

```yaml
# Istio VirtualService for Traffic Weighting
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: myapp-canary
spec:
  hosts:
  - myapp
  http:
  - route:
    - destination:
        host: myapp
        subset: v2  # Canary
      weight: 10    # 10% Traffic
    - destination:
        host: myapp
        subset: v1  # Stable
      weight: 90    # 90% Traffic
```

### Rolling Update (Standard K8s Strategy)

```yaml
spec:
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 25%       # Allow 25% extra pods during deploy
      maxUnavailable: 0   # Ensure zero downtime by never killing a pod before its replacement is ready
```

---

## Anti-Pattern

### Common Mistakes

#### 1. Database Schema Mismatch
Deploying a new code version that requires a new database column BEFORE the column exists.
```php
// ❌ ANTI-PATTERN: Code assumes column exists
$user->phone = $request->phone; // Crashes if DB hasn't been migrated yet
```
**Fix**: Use the "Expand and Contract" pattern where DB changes are backward compatible.

#### 2. Ignoring Health Probes
Proceeding with a rolling update even when new pods are crashing.
```yaml
# ❌ ANTI-PATTERN: No readiness probe
containers:
- name: app
  image: broken-image:latest
# K8s will still kill old pods as soon as containers start, leading to total outage
```
**Fix**: Always define `readinessProbe` and `livenessProbe`.

#### 3. Stateful Blue-Green
Switching traffic from Blue to Green when user sessions are stored in Blue's local memory.
**Result**: Every user is logged out and loses their shopping cart.
**Fix**: Use a distributed session store (Redis) or sticky sessions.

### Warning Signs

- **"Release Windows"**: You are only allowed to deploy at 3:00 AM on Sundays.
- **Rollback Fear**: Rolling back a deployment takes longer than 15 minutes.
- **Inconsistent Tests**: Smoke tests pass but real users experience immediate errors.
- **Manual Load Balancer Updates**: Someone has to log into a console to point to new IPs.

### What NOT to Do

1. **Don't** assume a successful container start means a successful deployment. Verify business logic.
2. **Don't** perform a Big Bang deploy on any service with an uptime SLA > 90%.
3. **Don't** skip staging. Testing in prod via Canary is a *supplement* to staging, not a replacement.
4. **Don't** leave old Blue versions running forever; clean up orphans to save cost.
5. **Don't** ignore the database. Deployment strategies MUST include a data migration plan.

---

## Related Patterns

### Complementary Patterns

- [CI/CD Pipeline](./01-CI-CD.md) - The automation trigger for deployment strategies.
- [Feature Flags](./04-Feature-Flags.md) - Allows code deployment without functional release.
- [Circuit Breaker](05-Safety-Engineering/02-Fault-Tolerance.md) - Protects against cascading failures during rollout.
- [Observability](10-Observability/02-Monitoring.md) - Essential for Canary analysis and rollbacks.

### Alternative Approaches

- **GitOps**: Using tools like ArgoCD to implement these strategies via Git state.
- **Manual Promotion**: For high-regulation environments (Banking/Medical).
- **Serverless**: Where the provider handles the deployment strategy (e.g., AWS Lambda Aliases).

### Evolution Path

- **Manual Restart**: High risk, high downtime.
- **Scripted Rolling**: Low cost, manageable risk.
- **Automated Blue-Green**: Zero downtime, high cost.
- **Progressive Delivery (Canary)**: Lowest risk, highest engineering maturity.

### See Also

- [Service Mesh](09-Infrastructure/04-Service-Mesh.md) - The infrastructure needed for advanced Canary.
- [Infrastructure as Code](09-Infrastructure/03-IaC.md) - Provisioning the environments for deployment.
- [Domain-Driven Design](01-System-Design/02-Microservices-Architecture.md) - Defining service boundaries for independent deployment.