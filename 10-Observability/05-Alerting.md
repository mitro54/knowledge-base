# Alerting

Alerting is the practice of configuring automated notifications that trigger when specific conditions are met, enabling teams to respond quickly to issues before they impact users.

## Key Characteristics
- Automated detection of anomalous conditions
- Multi-channel notification delivery
- Escalation policies for unacknowledged alerts
- Integration with incident management systems

---

## Problem Statement

Without effective alerting, teams face significant challenges:

- **Delayed Detection**: Issues may exist for hours before discovery
- **User Impact**: Problems affect users before teams are aware
- **Reactive vs Proactive**: Teams respond to user complaints rather than preventing issues
- **On-Call Burden**: Engineers overwhelmed by noise or missing critical alerts
- **Blame Culture**: Post-incident finger-pointing instead of learning

Common scenarios without alerting:
- Database connection pool exhausted for 2 hours before discovery
- Payment processing failures go unnoticed until customers complain
- Memory leak causes gradual degradation over days

---

## Solution

Effective alerting combines monitoring data with intelligent notification:

### Alert Components

1. **Detection**: Identify conditions that warrant attention
2. **Evaluation**: Determine if thresholds are breached
3. **Notification**: Deliver alerts through appropriate channels
4. **Acknowledgment**: Confirm someone is addressing the issue
5. **Escalation**: Escalate if not acknowledged within timeframe

### Alert Types

| Type | Trigger | Example |
|------|---------|---------|
| Threshold | Metric exceeds value | CPU > 90% for 5 min |
| Anomaly | Statistical deviation | Traffic 3σ below normal |
| Event-based | Specific event occurs | Deployment started |
| Synthetic | Health check fails | API endpoint returns 500 |

### Alert Severity Levels

```
CRITICAL (P0): Production down, immediate action required
HIGH (P1): Major functionality impaired, urgent response
MEDIUM (P2): Minor issues, respond during business hours
LOW (P3): Informational, address when convenient
```

---

## When to Use

### Appropriate Alert Conditions

- **Service Availability**: HTTP status codes, health checks
- **Performance Degradation**: Response time, throughput drops
- **Resource Exhaustion**: CPU, memory, disk, connection limits
- **Error Rates**: Increased 4xx/5xx responses
- **Business Metrics**: Payment failures, signup drops
- **Infrastructure**: Node failures, network issues

### Alert Design Principles

| Principle | Description |
|-----------|------------|
| Alert on Symptoms, Not Causes | Alert on what users experience |
| Every Alert Needs a Runbook | Clear action steps for responders |
| Page for Actionable Alerts Only | If no action, don't page |
| Default to Under-Alerting | Add alerts gradually |

---

## Tradeoffs

### Advantages

- **Faster Detection**: Issues identified before user impact
- **Reduced MTTR**: Quick response minimizes damage
- **Proactive Operations**: Fix issues before they escalate
- **Accountability**: Clear ownership of alerts
- **Learning**: Alert patterns reveal systemic issues

### Disadvantages

- **Alert Fatigue**: Too many alerts cause responders to ignore them
- **False Positives**: Incorrect alerts waste time and erode trust
- **On-Call Burden**: 24/7 coverage requires rotation and compensation
- **Complexity**: Configuring effective alerts requires expertise

### Performance Considerations

- Alert evaluation adds load to monitoring systems
- High-frequency alerts can overwhelm notification channels
- Consider alert aggregation and rate limiting

---

## Implementation Example

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
        annotations:
          summary: "Service {{ $labels.instance }} is down"
          description: "Service has been down for more than 1 minute"
          runbook_url: "https://runbooks.example.com/service-down"

      # High: High Error Rate
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
          description: "Error rate is {{ $value | humanizePercentage }}"

      # Medium: High Latency
      - alert: HighLatency
        expr: histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m])) > 1
        for: 10m
        labels:
          severity: medium
        annotations:
          summary: "High latency detected"
          description: "P95 latency is {{ $value }}s"

      # Low: Disk Space Warning
      - alert: DiskSpaceWarning
        expr: (node_filesystem_avail_bytes / node_filesystem_size_bytes) * 100 < 20
        for: 15m
        labels:
          severity: low
        annotations:
          summary: "Low disk space on {{ $labels.instance }}"
```

### Alertmanager Configuration

```yaml
global:
  resolve_timeout: 5m
  slack_api_url: 'https://hooks.slack.com/services/XXX'

route:
  group_by: ['alertname', 'severity']
  group_wait: 30s
  group_interval: 5m
  repeat_interval: 4h
  receiver: 'slack-default'
  routes:
    - match:
        severity: critical
      receiver: 'pagerduty-critical'
      continue: true
    - match:
        severity: high
      receiver: 'slack-high'

receivers:
  - name: 'slack-default'
    slack_configs:
      - channel: '#alerts'
        title: '{{ .GroupLabels.alertname }}'
        text: '{{ range .Alerts }}{{ .Annotations.summary }}{{ end }}'

  - name: 'pagerduty-critical'
    pagerduty_configs:
      - service_key: 'your-pagerduty-service-key'
        severity: critical

inhibit_rules:
  - source_match:
      severity: 'critical'
    target_match:
      severity: 'high'
    equal: ['alertname', 'instance']
```

### Alert Runbook Template

```markdown
# Runbook: High Error Rate

## Alert Details
- **Alert**: HighErrorRate
- **Severity**: HIGH
- **Threshold**: > 5% error rate for 5 minutes

## Impact
Users may experience failed requests and poor experience.

## Diagnosis Steps
1. Check error logs: `kubectl logs -l app=api --tail=200`
2. Check recent deployments: `kubectl rollout history deployment/api`
3. Check dependent services: `curl -s https://status.internal/services`

## Resolution Options
- **If recent deployment**: Rollback with `kubectl rollout undo deployment/api`
- **If dependency issue**: Check status page, contact team
- **If unknown**: Enable debug logging, investigate

## Verification
- Error rate returns to < 1%
- User reports resolved

## Escalation
If unresolved in 15 minutes, escalate to on-call lead.
```

---

## Anti-Pattern

### ❌ What NOT to Do

```
ANTI-PATTERN: Alerting on everything
- Alert on every metric crossing any threshold
- No prioritization or severity levels
- Result: Alert fatigue, ignored alerts

ANTI-PATTERN: Alerting on causes, not symptoms
- Alert on "CPU high" instead of "API slow"
- Engineers fix symptoms users don't care about
- Users still have bad experience

ANTI-PATTERN: No runbooks
- Alerts fire but no one knows what to do
- Panic and guesswork during incidents
- Inconsistent responses

ANTI-PATTERN: Alerting without acknowledgment
- No way to confirm someone is working on it
- Multiple people investigate same issue
- No escalation if ignored
```

### Warning Signs

- On-call engineers disable alerts they don't like
- Same alerts fire repeatedly without resolution
- Teams complain about "noise" constantly
- Critical alerts get buried in notification channels
- No one knows who to page for specific issues

---

## Related Patterns

- **Monitoring**: Alerting depends on monitoring data
  - See: [Monitoring](02-Monitoring.md)
  
- **Logging**: Logs help diagnose alert conditions
  - See: [Logging](01-Logging.md)
  
- **Metrics**: Quantitative data for alert thresholds
  - See: [Metrics](04-Metrics.md)
  
- **Tracing**: Helps identify root cause of alerts
  - See: [Tracing](03-Tracing.md)
  
- **Incident Management**: Alerting feeds into incident response
  - Related to deployment and operational practices
