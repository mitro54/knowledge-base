# Feature Flags

Feature flags (also known as feature toggles or feature switches) are a software development technique that allows teams to conditionally enable or disable features at runtime without deploying new code.

## Key Characteristics
- Runtime configuration without code changes
- Decouples deployment from release
- Enables gradual rollouts and A/B testing
- Supports kill switches for emergency feature disabling

---

## Problem Statement

Software teams face several challenges when releasing new features:

- **Risk of breaking changes**: New features may introduce bugs that affect all users immediately
- **Incomplete features**: Code may be merged before the feature is fully complete
- **Testing complexity**: It's difficult to test all combinations of features
- **Rollback difficulty**: Removing a problematic feature requires a new deployment
- **User segmentation**: Different user groups may need different feature sets

Without feature flags, teams must choose between:
- Keeping incomplete code in branches (branching problems)
- Releasing incomplete features to all users
- Complex deployment pipelines for different user segments

---

## Solution

Feature flags provide a mechanism to wrap feature code in conditional logic that can be controlled externally:

```
if (featureFlagService.isEnabled("new-checkout-flow", userId)) {
    showNewCheckoutFlow();
} else {
    showLegacyCheckoutFlow();
}
```

### Key Components

1. **Flag Definition**: Boolean or multi-value configuration stored externally
2. **Evaluation Engine**: Service that determines flag state based on rules
3. **Targeting Rules**: Conditions for which users/contexts see which variant
4. **Management Interface**: UI or API for controlling flag states

### Common Flag Types

| Type | Purpose | Example |
|------|---------|---------|
| Release Flag | Gradual feature rollout | 10% of users get new UI |
| Experiment Flag | A/B testing | Variant A vs Variant B |
| Permission Flag | Access control | Premium users only |
| Ops Flag | Emergency kill switch | Disable problematic feature |

---

## When to Use

### Appropriate Scenarios

- **Continuous Deployment**: Deploy code before enabling features
- **Canary Releases**: Gradually expose features to subsets of users
- **A/B Testing**: Compare different implementations
- **Feature Previews**: Allow early access for beta testers
- **Emergency Kill Switches**: Quickly disable problematic features
- **Environment-specific Features**: Different features per environment

### Prerequisites

- Feature flag service or library
- Configuration management system
- Monitoring to track flag impact
- Process for flag cleanup

---

## Tradeoffs

### Advantages

- **Reduced Risk**: Features can be disabled instantly without redeployment
- **Faster Feedback**: Get real user data on features before full rollout
- **Better CI/CD**: Merge smaller changes more frequently
- **Flexible Rollouts**: Target specific users, regions, or segments
- **Simplified Branching**: Reduce long-lived feature branches

### Disadvantages

- **Code Complexity**: Adds conditional logic throughout codebase
- **Technical Debt**: Flags may be forgotten and accumulate
- **Testing Overhead**: Must test all flag combinations
- **Performance Impact**: Flag evaluation adds latency (typically minimal)
- **Management Overhead**: Requires governance and cleanup processes

### Performance Considerations

- Flag evaluation should be cached
- Use client-side evaluation for high-scale applications
- Consider flag consolidation to reduce evaluation overhead

---

## Implementation Example

### Basic Implementation (Node.js)

```javascript
// Feature flag service
class FeatureFlagService {
    constructor() {
        this.flags = new Map();
        this.userTargets = new Map();
    }

    setFlag(flagName, defaultValue, rules = {}) {
        this.flags.set(flagName, { defaultValue, rules });
    }

    isEnabled(flagName, context = {}) {
        const flag = this.flags.get(flagName);
        if (!flag) return false;

        // Check percentage rollout
        if (flag.rules.percentage) {
            const hash = this.hash(context.userId || context.sessionId);
            return (hash % 100) < flag.rules.percentage;
        }

        // Check user list
        if (flag.rules.users && context.userId) {
            return flag.rules.users.includes(context.userId);
        }

        // Check environment
        if (flag.rules.environments) {
            return flag.rules.environments.includes(context.env);
        }

        return flag.defaultValue;
    }

    hash(str) {
        let hash = 0;
        for (let i = 0; i < str.length; i++) {
            hash = ((hash << 5) - hash) + str.charCodeAt(i);
            hash |= 0;
        }
        return Math.abs(hash);
    }
}

// Usage
const flagService = new FeatureFlagService();

// Configure flag
flagService.setFlag('new-checkout', false, {
    percentage: 25,  // 25% of users
    users: ['user123', 'user456'],  // Specific users always get it
    environments: ['staging', 'production']
});

// Check flag
if (flagService.isEnabled('new-checkout', { userId: 'user789', env: 'production' })) {
    renderNewCheckout();
} else {
    renderLegacyCheckout();
}
```

### Configuration-Based Example (JSON)

```json
{
  "flags": {
    "new-checkout-flow": {
      "enabled": true,
      "rollout": {
        "type": "percentage",
        "value": 25
      },
      "targeting": [
        { "attribute": "user_tier", "operator": "in", "values": ["premium", "enterprise"] },
        { "attribute": "country", "operator": "in", "values": ["US", "CA", "UK"] }
      ]
    },
    "dark-mode": {
      "enabled": true,
      "rollout": {
        "type": "all"
      }
    }
  }
}
```

---

## Anti-Pattern

### ❌ What NOT to Do

```javascript
// ANTI-PATTERN: Hardcoded feature flags
if (USER_IS_ADMIN && ENABLE_NEW_FEATURE === true) {
    // Feature code
}

// ANTI-PATTERN: Feature flag sprawl - forgotten flags
if (FLAG_OLD_CHECKOUT) {  // From 2019, never removed
    legacyCode();
} else if (FLAG_NEW_CHECKOUT) {  // From 2020
    newerCode();
} else if (FLAG_REDESIGNED_CHECKOUT) {  // From 2021
    newestCode();
}

// ANTI-PATTERN: Nested feature flags
if (FLAG_A) {
    if (FLAG_B) {
        if (FLAG_C) {
            // Deeply nested, hard to test
        }
    }
}
```

### Warning Signs

- Feature flags scattered throughout codebase without documentation
- Flags that have been active/inactive for months without review
- Complex nested flag conditions
- Flags used as a substitute for proper refactoring
- No process for flag retirement

---

## Related Patterns

- **Deployment Strategies**: Feature flags enable canary and blue-green deployments
  - See: [Deployment Strategies](02-Deployment-Strategies.md)
  
- **CI/CD**: Feature flags support continuous deployment practices
  - See: [CI/CD](01-CI-CD.md)
  
- **A/B Testing**: Experiment flags are a subset of feature flags
  - Related to testing strategies in [Testing Strategies](06-Testing-Engineering/05-Testing-Strategies.md)
  
- **Configuration Management**: Flags are a form of runtime configuration
  - See: [Configuration Management](03-Config-Management.md)