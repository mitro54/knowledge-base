# Alerting

Alerting is the practice of configuring automated notifications that trigger when specific conditions are met, enabling teams to respond quickly to issues before they impact users.

## Summary

Alerting serves as the reactive component of the observability stack. While monitoring tells you *where* a problem might be, alerting tells you *when* a problem requires human intervention. Effective alerting combines data from metrics, logs, and traces with intelligent notification rules, escalation policies, and actionable documentation (runbooks) to ensure that issues are resolved efficiently and reliably.

**Key Characteristics:**
- **Automated detection of anomalous conditions**: Scans data continuously for breaches.
- **Multi-channel notification delivery**: Slack, Email, SMS, PagerDuty, Webhooks.
- **Escalation policies for unacknowledged alerts**: Ensures accountability and resolution.
- **Integration with incident management systems**: Connects alerts to Jira, ServiceNow, etc.
- **State management**: Tracks alert status (Firing, Resolved, Silenced, Inhibited).
- **Threshold and Anomaly intelligence**: Supports both fixed and statistical triggers.

---

## Problem Statement

### The Challenge

Modern distributed systems are too complex for manual monitoring. Without automated alerting, operational teams are blind to the real-time health of their applications, leading to prolonged outages and eroded user trust. The challenge is not just detecting problems, but doing so without creating "noise" that leads to alert fatigue.

### Context

- **Historical Context**: Early systems relied on periodic manual checks or "red/green" dashboards that required constant human eyes.
- **Team Context**: On-call rotations require clear, unambiguous signals to effectively manage responsibilities outside normal business hours.
- **Technical Context**: Distributed microservices mean a failure in one component can cause a cascade; alerting must identify the root cause or at least the primary symptom quickly.

### Consequences of Not Addressing

Without effective alerting, teams face significant challenges:
- **Delayed Detection**: Issues may exist for hours or days before discovery.
- **User Impact**: Problems affect users before teams are even aware of a failure.
- **Reactive vs Proactive**: Teams respond to social media complaints rather than preventing outages.
- **On-Call Burden**: Engineers overwhelmed by noise or missing critical alerts entirely.
- **Blame Culture**: Post-incident finger-pointing instead of data-driven learning.
- **MTTR Inflation**: Mean Time To Resolution increases as diagnosis starts from zero every time.

---

## Solution

### The Alerting Pipeline Approach

Effective alerting combines monitoring data with intelligent notification logic across a structured pipeline:

```
┌─────────────────┐      ┌─────────────────┐      ┌─────────────────┐
│  Data Sources   │      │  Alert Engine   │      │  Notification   │
│ (Prometheus,    │─────▶│ (Evaluation,    │─────▶│  Management     │
│  ELK, Datadog)  │      │  Thresholds)    │      │ (Alertmanager)  │
└─────────────────┘      └─────────────────┘      └────────┬────────┘
                                                           │
                                             ┌─────────────┴─────────────┐
                                             ▼                           ▼
                                     ┌───────────────┐           ┌───────────────┐
                                     │   Channels    │           │  Responders   │
                                     │ (Slack, Pager,│           │  (Runbooks,   │
                                     │  SMS, Email)  │           │   Action)     │
                                     └───────────────┘           └───────────────┘
```

### Key Components

1. **Detection (Sources)**: Identify conditions that warrant attention by querying metrics or logs.
2. **Evaluation (Engine)**: Determine if thresholds are breached over a specific duration (e.g., `for: 5m`).
3. **Notification Management**: Handles deduplication, grouping, inhibition, and routing of alerts.
4. **Channels**: Delivery mechanisms that reach the right person at the right time.
5. **Acknowledgment & Escalation**: Confirm someone is addressing the issue or scale the alert up the chain.
6. **Runbooks**: Standardized procedures mapping specific alerts to resolution steps.

### How It Addresses the Problem

- **Instant Recognition**: Bridges the gap between event occurrence and team awareness.
- **Prioritization**: Categorizes issues so critical failures get paged while minor warnings wait for morning.
- **Contextual Awareness**: Groups related alerts to prevent notification storms during cascaded failures.
- **Actionable Intelligence**: Provides links to dashboards and runbooks directly in the notification.

---

## When to Use

### Appropriate Scenarios

| Scenario | Suitability |
|----------|-------------|
| Critical Production Services | ⭐⭐⭐⭐⭐ Essential |
| Resource Exhaustion (Disk/CPU) | ⭐⭐⭐⭐⭐ Essential |
| Financial/Business Transactions | ⭐⭐⭐⭐⭐ Essential |
| Security Breaches/Audit Failures | ⭐⭐⭐⭐⭐ Essential |
| Performance Degradation (Latency) | ⭐⭐⭐⭐ Recommended |
| Staging/Dev Environment Issues | ⭐⭐⭐ Optional |

### Prerequisites

- **Monitoring Baseline**: You cannot alert on what you don't measure.
- **Defined Thresholds**: Knowledge of what constitutes "normal" vs "broken".
- **On-Call Rotation**: A designated person or team responsible for responding.
- **Notification Infrastructure**: Working integrations with Slack, PagerDuty, or similar.

### Indicators for Suitability

- **SLA/SLO Requirements**: You have legal or business commitments to uptime.
- **Hidden Failures**: Silent failures (like background job drops) aren't visible on the UI.
- **Scale**: You have more than 5 services or 10 nodes to manage manually.
- **High-Stakes Changes**: Frequent deployments require immediate feedback on regressions.

---

## Tradeoffs

### Advantages

| Advantage | Description |
|-----------|-------------|
| **Faster Detection** | Issues identified before user impact reaches critical mass. |
| **Reduced MTTR** | Quick notification with context minimizes diagnosis time. |
| **Proactive Ops** | Fix issues before they escalate into full-scale outages. |
| **Accountability** | Clear ownership via acknowledgment and escalation flows. |
| **Learning** | Alert patterns reveal systemic weaknesses and technical debt. |

### Disadvantages

| Disadvantage | Description |
|--------------|-------------|
| **Alert Fatigue** | Too many low-priority alerts cause responders to ignore them. |
| **False Positives** | Flapping or incorrect alerts erode trust in the system. |
| **Operational Burden** | 24/7 coverage requires rotation, compensation, and management. |
| **Maintenance** | Alert rules must be updated as the application evolves. |
| **Complexity** | Distributed systems require complex "inhibitions" to avoid noise. |

### Performance Considerations

- **Evaluation Load**: Heavy/complex PromQL or SQL queries for alerts can strain the monitoring server.
- **Notification Storms**: Un-grouped alerts can overwhelm Slack channels or PagerDuty APIs.
- **Resolution Latency**: High "evaluation intervals" (e.g. 5m) add delay to the actual alert firing.

### Complexity Implications

- **Initial Complexity**: Low to Medium—setting up basic "up/down" alerts is simple.
- **Long-term Complexity**: High—maintaining accurate thresholds and non-noisy anomaly detection is difficult.
- **Operational Complexity**: Medium—requires consistent on-call management and runbook maintenance.

---

## Implementation Example

### Basic Alerting Structure (Terraform/YAML)

```
monitoring-infra/
├── alerts/
│   ├── app-alerts.yaml      # Business and application rules
│   ├── infra-alerts.yaml    # Node and container rules
│   └── custom-alerts.yaml   # Ad-hoc/temporary rules
├── alertmanager/
│   └── config.yaml          # Routing and notification logic
└── dashboards/
    └── alerting-health.json # Dashboard to track alert volume
```

### Prometheus Alerting Rules

```yaml
groups:
  - name: application-alerts
    rules:
      # Critical: Service Down
      - alert: ServiceDown
        expr: up == 0
        for: 1m
        labels:
          severity: critical
          tier: backend
        annotations:
          summary: "Service {{ $labels.instance }} is down"
          description: "Service has been down for more than 1 minute. Check pod logs."
          runbook_url: "https://runbooks.example.com/service-down"

      # High: High Error Rate (Symptoms-based)
      - alert: HighErrorRate
        expr: |
          (
            sum(rate(http_requests_total{status=~"5.."}[5m]))
            /
            sum(rate(http_requests_total[5m]))
          ) > 0.05
        for: 5m
        labels:
          severity: high
        annotations:
          summary: "High error rate detected"
          description: "Error rate is {{ $value | humanizePercentage }}. Check downstream dependencies."

      # Medium: High Latency (SLA impact)
      - alert: HighLatency
        expr: histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m])) > 1
        for: 10m
        labels:
          severity: medium
        annotations:
          summary: "High latency detected"
          description: "P95 latency is {{ $value }}s. Check for database locks."

      # Low: Disk Space Warning (Capacity Planning)
      - alert: DiskSpaceWarning
        expr: (node_filesystem_avail_bytes / node_filesystem_size_bytes) * 100 < 20
        for: 15m
        labels:
          severity: low
        annotations:
          summary: "Low disk space on {{ $labels.instance }}"
```

### Alertmanager Configuration (Routing & Grouping)

```yaml
global:
  resolve_timeout: 5m
  slack_api_url: 'https://hooks.slack.com/services/XXX'

route:
  group_by: ['alertname', 'severity', 'cluster']
  group_wait: 30s
  group_interval: 5m
  repeat_interval: 4h
  receiver: 'slack-default'
  routes:
    # Route criticals to PagerDuty
    - match:
        severity: critical
      receiver: 'pagerduty-urgent'
      continue: true
    # Route high severity to specific engineering channel
    - match:
        severity: high
      receiver: 'slack-engineering'

receivers:
  - name: 'slack-default'
    slack_configs:
      - channel: '#alerts-general'
        title: '{{ .CommonLabels.alertname }}'
        text: '{{ range .Alerts }}{{ .Annotations.summary }}\n{{ end }}'

  - name: 'pagerduty-urgent'
    pagerduty_configs:
      - service_key: 'pd-api-key-here'
        severity: critical

inhibit_rules:
  # If a node is down, don't alert on services on that node
  - source_match:
      alertname: 'NodeDown'
    target_match:
      severity: 'critical'
    equal: ['node']
```

---

## Anti-Pattern

### Common Mistakes

#### 1. Alerting on Causes, Not Symptoms
Alerting on "CPU > 80%" instead of "Request Latency > 2s". High CPU might be normal during batch jobs, but high latency always affects users.
```yaml
# ❌ ANTI-PATTERN: Cause-based alerting
- alert: HighCPU
  expr: node_cpu_utilization > 0.8  # No context on user impact
```

#### 2. The Golden Signal Trap (Alert Fatigue)
Alerting on every minor spike in every metric without duration or grouping.
```yaml
# ❌ ANTI-PATTERN: Instant alerts
- alert: TransientError
  expr: rate(errors[1s]) > 0  # Fires on every tiny blip, creating noise
```

#### 3. No Runbook Reference
Alerts that fire with "Service Error" but provide no link to logs, dashboards, or resolution steps.
```yaml
# ❌ ANTI-PATTERN: Vague alerts
- alert: SomethingWrong
  annotations:
    summary: "Error occurred" # Not actionable
```

### Warning Signs

- **On-call engineers disable alerts** manually because they are "annoying".
- **Silent failures**: Critical systems fail but no alert fires.
- **Inbox Zero impossibility**: Thousands of unread alerts in notification channels.
- **Missing MTTA (Mean Time To Acknowledge)**: Alerts fire but no one clicks "acknowledge".

### What NOT to Do

1. **Don't** page for non-actionable alerts. If it can wait until Monday, use Slack/Email only.
2. **Don't** alert on every small variance. Use statistical windows (e.g., 5-15 mins).
3. **Don't** assume thresholds stay static. Review them every quarter.
4. **Don't** ignore "resolved" notifications. Ensure your system clears alerts automatically.
5. **Don't** set up alerting without a dedicated channel or person to receive it.

---

## Related Patterns

### Complementary Patterns

- [Monitoring](10-Observability/02-Monitoring.md) - The data source for all alerting rules.
- [Logging](10-Observability/01-Logging.md) - Used to provide detailed context to alert notifications.
- [Metrics](10-Observability/04-Metrics.md) - Quantitative data for objective threshold settings.
- [Tracing](10-Observability/03-Tracing.md) - Helps identify *where* in a request chain a failure occurred.

### Alternative Approaches

- **Log-based Alerting**: Using ELK or Grafana Loki to alert on specific error strings in logs.
- **Synthetic Monitoring**: Using external probes (like Pingdom) to alert on end-to-end availability.
- **Anomaly Detection (AIOps)**: Using ML models to detect deviation from seasonal patterns (e.g. Prophet).

### Evolution Path

- Start with **Infrastructure Alerts** (Up/Down, Disk, CPU).
- Move to **Symptom-based Alerts** (Error Rates, Latency).
- Implement **Business Logic Alerts** (Failed checkouts, high-value transaction drops).
- Adopt **Anomaly Detection** for complex, scale-dependent patterns.

### See Also

- [SLOs and SLIs](04-Best-Practices/03-Design-Principles.md) - Fundamental concepts for defining what "good" looks like.
- [Incident Management](11-DevOps/05-Release-Management.md) - The process that begins when an alert fires.
- [Chaos Engineering](05-Safety-Engineering/02-Fault-Tolerance.md) - Testing your alerting by intentionally failing systems.
