# Feature Flags

Feature flags (also known as feature toggles or feature switches) are a software development technique that allows teams to conditionally enable or disable features at runtime without deploying new code.

## Summary

Feature Flags (FF) represent a fundamental shift in software delivery, moving from "static releases" to "dynamic runtime control." By wrapping code in conditional branches managed by an external configuration service, teams can decouple the technical act of **deployment** (moving code to production) from the business act of **release** (exposing functionality to users). This enables "Dark Launching," where features are hidden from users while they are being tested under real production load. Feature flags are the engine behind Progressive Delivery, allowing for granular A/B testing, instant kill-switches, and zero-downtime maintenance.

**Key Characteristics:**
- **Runtime Control**: Changes to application behavior take effect instantly without a restart or redeployment.
- **Decoupled Release**: Code can be merged to `main` while still being hidden behind a flag, preventing "branch hell."
- **Targeted Exposure**: Granular rules (e.g., 10% of users, specific regions, internal employees) for who sees a feature.
- **Experimentation**: Native support for A/B testing and multivariate experiments with automatic data collection.
- **Safety Valve**: Functional "kill switches" to disable buggy features in milliseconds.
- **Environment Management**: Different flag states for Dev, Staging, and Production from a single control plane.

---

## Problem Statement

### The Challenge

Traditional release models are all-or-nothing. A single bug in a new feature can force a complete rollback of the entire application, reverting unrelated features and causing massive downtime. Furthermore, long-lived feature branches create complex merge conflicts, while "Big Bang" releases overwhelm support teams and users alike.

### Context

- **Historical Context**: Before feature flags, "turning off" a feature required editing a config file on every server and restarting the service.
- **Technical Context**: Modern SaaS platforms need to test "mobile app" features where users may be on dozens of different app versions simultaneously.
- **Team Context**: Marketing may want to launch a feature on a specific date (e.g., Black Friday) independently of the engineering sprint cycle.

### Consequences of Not Addressing

- **High-Risk Deployments**: Every release is a threat to system stability; one bug affects 100% of users.
- **Feature Branch Sprawl**: Developers keep code out of `main` for weeks, leading to "integration hell" during merges.
- **Slow Feedback Loops**: Teams must wait for a full deployment cycle to see how a feature performs in the wild.
- **Rollback Latency**: Removing a problematic feature takes minutes or hours (build + deploy time) instead of milliseconds.
- **Lack of Personalization**: Impossible to offer "Early Access" or "Beta" programs to specific users without custom code hacks.
- **Testing Blind Spots**: It is nearly impossible to test how new code behaves under real production traffic without actually releasing it.

---

## Solution

### The Feature Flag Lifecycle

Feature flags provide a mechanism to wrap code in conditional logic that checks an external "Source of Truth" at runtime.

```
    ┌─────────────┐       ┌─────────────┐       ┌─────────────┐
    │  Management │       │   Evaluation│       │   Application│
    │     UI      │──────▶│    Engine   │──────▶│     Code    │
    └─────────────┘       └─────────────┘       └──────┬──────┘
           ▲                      ▲                    │
           │              Rules:  │                    │
           │              [ %10,  │              Context:
           └──────────────[ Beta  │              [ user_id,
                          [ Prod  │              [ region ]
                                  └────────────────────┘
```

### Key Components

1. **Flag Definition**: A remote configuration (Boolean, String, JSON) that defines the feature's availability.
2. **Evaluation Engine**: The logic (usually a library or SDK) that determines the toggle state for a specific user.
3. **Context Provider**: Data passed during evaluation (e.g., `userId`, `userTier`, `deviceType`) to determine eligibility.
4. **Targeting Rules**: Logic that maps context to flag state (e.g., "If region is 'US' and tier is 'Premium', then ON").
5. **Analytics Integration**: Linking flag states to business metrics (e.g., conversion rate) to measure impact.
6. **Cleanup Process**: Periodic removal of "deprecated" flags to prevent technical debt.

### How It Addresses the Problem

- **Instant Reverts**: Buggy code stays in production but the flag is flipped to "OFF" in milliseconds.
- **Trunk-Based Development**: Incomplete code is merged to `main` safely behind a "false" flag.
- **Canary Rolling**: Progressively increase traffic from 1% to 100%, monitoring error rates at each step.
- **Experimentation**: Validates if Version A is actually better than Version B before making it the permanent standard.

---

## When to Use

### Appropriate Scenarios

| Scenario | Suitability | Priority |
|----------|-------------|----------|
| SaaS / Cloud-Native Apps | ⭐⭐⭐⭐⭐ Critical | High |
| A/B Testing & UI Experiments | ⭐⭐⭐⭐⭐ Critical | High |
| High-Frequency Deployments | ⭐⭐⭐⭐⭐ Critical | High |
| Mobile Applications | ⭐⭐⭐⭐ Recommended | Medium |
| Critical Infrastructure (Internal) | ⭐⭐⭐ Good (Kill switches) | Medium |
| Small Static Sites | ⭐⭐ Optional | Low |

### Prerequisites

- **FF Service/Library**: A robust system like LaunchDarkly, Flagsmith, or a custom internal API.
- **Context passing**: The application must have consistent access to user metadata during requests.
- **Monitoring**: Real-time error monitoring to detect when a new flag state causes issues.
- **Flag Hygiene**: A strict policy for removing flags once a feature is 100% rolled out.

### Indicators for Adoption

- **Releases happen < 1x a month**: You are batching too much risk.
- **Hotfixes are common**: You need a way to disable features without redeploying.
- **Product Managers want "Beta" testers**: You need targeted exposure logic.
- **Merge conflicts are painful**: Your feature branches are living too long.

---

## Tradeoffs

### Advantages

| Advantage | Description |
|-----------|-------------|
| **Reduced Blast Radius** | New code can be tested on 1% of users first. |
| **Separation of Concerns** | Engineers deploy; Product Managers release. |
| **Kill Switches** | Operational safety during high-traffic events (e.g., sale launches). |
| **Continuous Integration** | Supports merging half-finished features into the master branch. |
| **User Segmentation** | Allows for tailoring features based on user attributes or monetization tiers. |

### Disadvantages

| Disadvantage | Description |
|--------------|-------------|
| **Technical Debt** | Forgotten flags create "spaghetti code" that is hard to maintain. |
| **Testing Combinatorics** | Hard to test all possible combinations of active flags ($2^n$ complexity). |
| **Performance Overhead** | Every flag check adds a small delay (nanoseconds locally, milliseconds via API). |
| **Stale Cache Risk** | Users might see inconsistent states if flag updates take time to propagate. |
| **Complexity** | Adds conditional logic to every part of the UX and backend. |

### Performance Considerations

- **Local Evaluation**: Use SDKs that download rules once and evaluate locally to avoid per-request network calls.
- **Streaming Updates**: Use WebSockets or Server-Sent Events (SSE) to push flag changes to clients instantly.
- **Default Values**: Always provide a sensible hardcoded default if the FF service is unreachable.

### Complexity Implications

- **Initial Complexity**: Low—easy to wrap a single `if` statement.
- **Long-term Complexity**: High—requires a culture of cleanup and automated removal of old flags.
- **Operational Complexity**: Medium—the FF management dashboard becomes a critical Tier-1 tool.

---

## Implementation Example

### Basic Flag Service Structure (Node.js/TypeScript)

```typescript
// Feature flag service implementation
class FeatureFlagService {
    private flags = new Map<string, any>();

    // Simulated fetch from remote config
    async fetchRemoteFlags() {
        // In reality, this would hit dynamic config service
        this.flags.set('new-checkout-flow', {
            enabled: true,
            rollout: 25, // 25% of users
            users: ['beta-tester-1', 'admin-user']
        });
    }

    isEnabled(flagName: string, context: { userId: string, tier: string }): boolean {
        const flag = this.flags.get(flagName);
        if (!flag || !flag.enabled) return false;

        // 1. Mandatory override for specific users
        if (flag.users && flag.users.includes(context.userId)) return true;

        // 2. Percentage rollout based on stable hashing
        const userHash = this.getDeterministicHash(context.userId);
        if (userHash % 100 < flag.rollout) return true;

        return false;
    }

    private getDeterministicHash(id: string): number {
         let hash = 0;
         for (let i = 0; i < id.length; i++) {
             hash = ((hash << 5) - hash) + id.charCodeAt(i);
             hash |= 0;
         }
         return Math.abs(hash);
    }
}

// Usage in Application
const ff = new FeatureFlagService();
if (ff.isEnabled('new-checkout-flow', { userId: 'user_123', tier: 'premium' })) {
    startModernCheckout();
} else {
    startLegacyCheckout();
}
```

### JSON-Based Flag Configuration (Remote Storage)

```json
{
  "flags": {
    "v2-engine-migration": {
      "status": "enabled",
      "rules": [
        {
          "attribute": "region",
          "operator": "in",
          "values": ["US-EAST", "EU-WEST"]
        },
        {
          "attribute": "version",
          "operator": "gte",
          "values": "1.4.0"
        }
      ],
      "rollout_percentage": 10,
      "kill_switch": false
    }
  }
}
```

### Database-Backed Feature Flags (Relational)

For heavy enterprise applications, flags can be stored in the database for consistency with transactions.

```sql
SELECT flag_name, is_enabled, targeting_json 
FROM feature_flags 
WHERE flag_name = 'dark_mode_beta' 
  AND (target_all = true OR target_user_ids @> ARRAY['user789']);
```

---

## Anti-Pattern

### Common Mistakes

#### 1. Flag Sprawl (Zombie Flags)
Keeping flags in the code long after a feature has been 100% rolled out.
```javascript
// ❌ ANTI-PATTERN: Flag from 3 years ago
if (flags.isEnabled('checkout-redesign-2021')) { ... } 
```
**Fix**: Add "expiration dates" to flags and automate tickets for their removal.

#### 2. Nested Flag Combinations
Wrapping flags inside flags, creating impossible-to-test logic branches.
```javascript
// ❌ ANTI-PATTERN: N-level deep toggles
if (flagA) {
  if (flagB) {
     if (flagC) { ... } // Testing this state is statistically impossible
  }
}
```

#### 3. Using Flags for Secrets
Storing API keys or private certificates inside a feature flag "JSON" value.
**Fix**: Use a Secret Manager; flags are for *behavioral control*, not *data security*.

#### 4. Flag-Driven Outages
Changing a flag state in production without first testing that specific state in staging.
**Fix**: Always validate the "ON" state behind a flag in a lower environment first.

### Warning Signs

- **"I don't know what this flag does"**: Documentation lives only in one developer's head.
- **Flags used for permissions**: Using FF for RBAC (Role Based Access Control) instead of a dedicated auth service.
- **Flag evaluation happens in tight loops**: Calling an external API to check a flag inside a `for` loop (causes huge latency).

---

## Related Patterns

### Complementary Patterns

- [Deployment Strategies](./02-Deployment-Strategies.md) - Flags are the technical engine for Canary and Blue-Green.
- [CI/CD Pipeline](./01-CI-CD.md) - Enables merging code to the trunk even if it's not "live."
- [Config Management](./03-Config-Management.md) - Flags are a specific type of runtime configuration.
- [Observability](10-Observability/02-Monitoring.md) - Used to measure the business impact of a flag state.

### Alternative Approaches

- **Branching by Abstraction**: Re-architecting the system to allow two versions of code to coexist without an `if` statement.
- **Modular Packaging**: Deploying different plugin versions to different environments.
- **Manual Config Files**: Storing toggles in `config.yaml` and restarting the app to apply changes (Low maturity).

### See Also

- [Twelve-Factor App](04-Best-Practices/03-Design-Principles.md) - Externalizing configuration.
- [Testing Pyramid](06-Testing-Engineering/01-Testing-Pyramid.md) - How to test both states of a flag efficiently.
- [Trunk-Based Development](04-Best-Practices/05-Code-Organization.md) - The workflow that most benefits from feature flags.