# Deployment Strategies

## Overview

Deployment strategies are approaches for releasing new versions of applications to production environments. Different strategies balance risk, downtime, complexity, and cost to meet organizational requirements for reliability and speed of delivery.

### Key Characteristics
- **Risk Mitigation**: Minimize impact of failed deployments
- **Rollback Capability**: Ability to revert to previous version
- **Downtime Management**: Control over service availability during deployment
- **Traffic Control**: Managing user traffic during transitions
- **Resource Efficiency**: Optimizing infrastructure usage during deployment

---

## Problem Statement

Deploying new versions of applications presents several challenges:

- **Service Downtime**: Users cannot access the application during deployment
- **Deployment Failures**: Bugs or configuration errors can break production
- **Data Migration Issues**: Schema changes may fail or corrupt data
- **State Management**: In-flight requests may be lost during transitions
- **Rollback Complexity**: Reverting changes can be as complex as the original deployment
- **User Impact**: All users affected simultaneously by any issues

### Consequences
- Lost revenue during downtime
- Damaged user trust and reputation
- Emergency hotfixes under pressure
- Team burnout from deployment anxiety
- Difficulty meeting SLA requirements

---

## Solution

Different deployment strategies address these challenges in various ways:

### Big Bang Deployment
Deploy entire new version at once, replacing the old version completely.

```
[Old Version] → [Downtime] → [New Version]
```

### Rolling Deployment
Gradually replace instances with new version while maintaining availability.

```
[Old: ●●●●] → [Old: ●●●○] → [Old: ●●○○] → [New: ○○○○]
```

### Blue-Green Deployment
Maintain two identical environments; switch traffic between them.

```
Blue (Active) ← Traffic → Green (Standby)
         Switch Traffic
Blue (Standby) ← Traffic → Green (Active)
```

### Canary Deployment
Gradually route increasing traffic to new version.

```
100% Old → 90% Old / 10% New → 50/50 → 10% Old / 90% New → 100% New
```

### Shadow Deployment
Route copy of production traffic to new version without affecting users.

```
User → Load Balancer → Production (responses sent)
                ↳ Shadow (responses discarded)
```

### A/B Testing Deployment
Route traffic based on user segments for comparison testing.

```
User A → Version A
User B → Version B
Compare metrics and user behavior
```

---

## When to Use

### Big Bang Deployment
**Use when:**
- Small applications with minimal user base
- Acceptable maintenance windows exist
- Simple applications without complex state
- Limited infrastructure resources

**Avoid when:**
- High availability is critical
- Large user base with global distribution
- Complex state management required

### Rolling Deployment
**Use when:**
- Stateless applications
- Database schema is backward compatible
- Moderate availability requirements
- Limited infrastructure budget

**Avoid when:**
- Zero downtime is mandatory
- Complex state synchronization needed
- Database migrations required

### Blue-Green Deployment
**Use when:**
- Zero downtime is required
- Quick rollback is essential
- Sufficient infrastructure for duplicate environment
- State is externalized (database, cache)

**Avoid when:**
- Infrastructure costs are a major concern
- State is stored in application instances
- Database schema changes break backward compatibility

### Canary Deployment
**Use when:**
- Risk mitigation is critical
- Traffic routing capabilities exist
- Monitoring and metrics are available
- Gradual validation is important

**Avoid when:**
- Simple applications with low risk
- No traffic routing infrastructure
- Immediate full rollout is required

### Shadow Deployment
**Use when:**
- Performance validation needed
- Complex system behavior must be tested
- Production-like traffic patterns required
- No user impact acceptable during testing

**Avoid when:**
- Write operations need testing
- Infrastructure for traffic duplication unavailable
- Quick validation needed

### A/B Testing Deployment
**Use when:**
- Feature comparison needed
- User behavior analysis required
- Data-driven decisions important
- Marketing or UX changes being tested

**Avoid when:**
- Backend-only changes
- Performance improvements
- Bug fixes

---

## Tradeoffs

| Strategy | Downtime | Complexity | Cost | Rollback Speed | Risk Level |
|----------|----------|------------|------|----------------|------------|
| Big Bang | High | Low | Low | Medium | High |
| Rolling | Low | Medium | Medium | Medium | Medium |
| Blue-Green | None | Medium | High | Fast | Low |
| Canary | None | High | Medium | Medium | Low |
| Shadow | None | High | High | N/A | Lowest |
| A/B Testing | None | High | Medium | Medium | Low |

### Performance Considerations
- **Blue-Green**: Requires 2x infrastructure capacity
- **Canary**: Traffic routing adds latency overhead
- **Rolling**: Brief performance degradation during transition
- **Shadow**: Requires additional resources for shadow instances

---

## Implementation Example

### Blue-Green Deployment (Kubernetes)

```yaml
# Blue environment (current production)
apiVersion: v1
kind: Service
metadata:
  name: myapp-production
spec:
  selector:
    app: myapp
    version: blue
  ports:
    - port: 80
      targetPort: 8080

---
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
# Green environment (new version, standby)
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

**Deployment Script:**
```bash
#!/bin/bash
# Blue-Green Deployment Script

set -e

NEW_VERSION="2.0.0"
CURRENT_ENV="blue"
NEW_ENV="green"

# 1. Deploy new version to standby environment
echo "Deploying version $NEW_VERSION to $NEW_ENV environment..."
kubectl set image deployment/myapp-$NEW_ENV myapp=myapp:$NEW_VERSION

# 2. Wait for readiness
echo "Waiting for $NEW_ENV environment to be ready..."
kubectl rollout status deployment/myapp-$NEW_ENV

# 3. Run smoke tests against new environment
echo "Running smoke tests..."
curl -f http://myapp-$NEW_ENV.health-check.local/health || {
    echo "Smoke tests failed! Rolling back..."
    exit 1
}

# 4. Switch traffic by updating service selector
echo "Switching traffic from $CURRENT_ENV to $NEW_ENV..."
kubectl patch service myapp-production -p "{\"spec\":{\"selector\":{\"version\":\"$NEW_ENV\"}}}"

# 5. Verify switch
echo "Verifying deployment..."
sleep 10
curl -f http://myapp-production.local/health

echo "Deployment complete!"

# 6. Cleanup old environment (optional, after verification period)
# kubectl delete deployment/myapp-$CURRENT_ENV
```

### Canary Deployment (Istio)

```yaml
# VirtualService for canary deployment
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
        subset: v2  # New version (canary)
      weight: 10    # 10% traffic
    - destination:
        host: myapp
        subset: v1  # Old version (stable)
      weight: 90    # 90% traffic
    - match:
      - headers:
          canary:
            exact: "true"
      route:
      - destination:
          host: myapp
          subset: v2
        weight: 100  # Internal traffic always goes to canary
```

**Progressive Canary Rollout Script:**
```bash
#!/bin/bash
# Progressive Canary Rollout

CANARY_PERCENTAGES=(10 25 50 75 100)
SLEEP_BETWEEN_STEPS=300  # 5 minutes

for percentage in "${CANARY_PERCENTAGES[@]}"; do
    echo "Setting canary to ${percentage}%"
    
    # Update Istio VirtualService
    istioctl apply -f - <<EOF
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
        subset: v2
      weight: $percentage
    - destination:
        host: myapp
        subset: v1
      weight: $((100 - percentage))
EOF

    # Check error rates and metrics
    ERROR_RATE=$(get_error_rate_metric "myapp-v2")
    
    if (( $(echo "$ERROR_RATE > 0.01" | bc -l) )); then
        echo "Error rate ($ERROR_RATE) exceeds threshold (1%)! Rolling back..."
        rollback_canary
        exit 1
    fi

    echo "Canary at ${percentage}% - Error rate: $ERROR_RATE - OK"
    
    if [ $percentage -lt 100 ]; then
        echo "Waiting ${SLEEP_BETWEEN_STEPS}s before next step..."
        sleep $SLEEP_BETWEEN_STEPS
    fi
done

echo "Canary deployment complete! 100% traffic on new version."
```

### Rolling Deployment (Kubernetes)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  replicas: 6
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 2          # Maximum extra pods during update
      maxUnavailable: 1    # Maximum pods that can be unavailable
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
      - name: myapp
        image: myapp:2.0.0
        readinessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 10
          periodSeconds: 5
```

---

## Anti-Pattern

### Common Mistakes and Pitfalls

#### ❌ Database Schema Changes Without Migration Strategy
```bash
# ANTI-PATTERN: Changing schema during deployment
# Version 1.0 expects: users(id, name, email)
# Version 2.0 expects: users(id, name, email, phone) ← New column

# If deployment fails halfway, database is in inconsistent state!
```

**Correct Approach:**
```sql
-- Expand/Contract pattern
-- Phase 1: Expand (backward compatible)
ALTER TABLE users ADD COLUMN phone VARCHAR(20);

-- Deploy version 2.0 (reads/writes both old and new schema)

-- Phase 2: Contract (after verification)
-- Remove old code paths, then eventually remove old columns
```

#### ❌ Stateful Blue-Green Without State Migration
```
Blue: User sessions stored in app memory
Green: New instances, no session data
Result: All users logged out after switch!
```

**Correct Approach:**
- Externalize state to Redis, database, or distributed cache
- Use sticky sessions during transition
- Implement session migration strategy

#### ❌ Canary Without Proper Monitoring
```bash
# ANTI-PATTERN: No validation between canary steps
for pct in 10 25 50 75 100; do
    set_canary $pct
    sleep 10  # Not enough time to detect issues!
done
```

**Correct Approach:**
```bash
for pct in 10 25 50 75 100; do
    set_canary $pct
    wait_for_stability  # Wait for metrics to stabilize
    check_error_rates   # Verify error rates acceptable
    check_latency       # Verify latency within bounds
    check_business_metrics  # Verify key metrics healthy
done
```

#### ❌ Ignoring In-Flight Requests
Switching traffic while requests are being processed causes request failures.

#### ❌ No Health Checks Between Steps
Proceeding with deployment without verifying each step's success.

---

## Related Patterns

### See Also
- [CI/CD](./01-CI-CD.md) - Pipeline automation for deployments
- [Feature Flags](./04-Feature-Flags.md) - Controlling feature availability
- [Release Management](./05-Release-Management.md) - Coordinating releases

### Complementary Patterns
- [Service Mesh](09-Infrastructure/04-Service-Mesh.md) - Traffic management capabilities
- [Orchestration](09-Infrastructure/02-Orchestration.md) - Container management
- [Fault Tolerance](05-Safety-Engineering/02-Fault-Tolerance.md) - Handling deployment failures

### Alternative Approaches
- [Event-Driven Architecture](01-System-Design/03-Event-Driven-Architecture.md) - Asynchronous deployment triggers
- [Microservices Architecture](01-System-Design/02-Microservices-Architecture.md) - Independent service deployment