# Observability for Rollouts: Seeing What's Happening

## The Blind Deployment Problem

Traditional deployment: Deploy, hope it works, find out hours later from user complaints.

**Without observability**:
```bash
kubectl apply -f deployment.yaml
# Deployment updated
# ...is it working? ¯\_(ツ)_/¯
```

**Questions you can't answer**:
- Is the new version healthy?
- Is error rate higher than old version?
- Is latency worse?
- Are pods crashing?
- When did the rollout complete?

**Kubernetes observability** gives you the data to answer these questions in real-time.

---

## The Three Pillars of Observability

### Metrics: Aggregated Numbers

**What they are**: Time-series data (counters, gauges, histograms)

**Examples**:
- Request rate: 150 req/s
- Error rate: 2.5%
- P99 latency: 350ms
- Pod count: 6 ready, 0 not ready

**Use case**: Dashboards, alerts, trend analysis

### Logs: Event Records

**What they are**: Timestamped event messages

**Examples**:
```
2024-01-15 10:32:15 INFO  Request processed successfully
2024-01-15 10:32:16 ERROR Failed to connect to database
2024-01-15 10:32:17 WARN  High memory usage: 85%
```

**Use case**: Debugging, root cause analysis

### Traces: Request Journey

**What they are**: End-to-end request flow across services

**Example**:
```
Trace ID: abc123

Span 1: API Gateway (50ms)
  Span 2: User Service (20ms)
    Span 3: Database Query (15ms)
  Span 4: Product Service (25ms)
    Span 5: Cache Lookup (5ms)
```

**Use case**: Understanding latency, finding bottlenecks

---

## Deployment Events and Conditions

Kubernetes records deployment lifecycle as **events** and **conditions**.

### Events: What Happened

```bash
kubectl describe deployment myapp

Events:
  Type    Reason             Age   Message
  ----    ------             ----  -------
  Normal  ScalingReplicaSet  5m    Scaled up replica set myapp-abc to 6
  Normal  ScalingReplicaSet  4m    Scaled up replica set myapp-xyz to 2
  Normal  ScalingReplicaSet  3m    Scaled down replica set myapp-abc to 4
  Normal  ScalingReplicaSet  2m    Scaled up replica set myapp-xyz to 4
  Normal  ScalingReplicaSet  1m    Scaled down replica set myapp-abc to 0
```

**Tells you**: Timeline of rollout (scale up new, scale down old)

**Events are ephemeral**: Only kept for 1 hour by default.

### Conditions: Current State

```bash
kubectl get deployment myapp -o yaml

status:
  conditions:
  - type: Progressing
    status: "True"
    reason: NewReplicaSetAvailable
    message: ReplicaSet "myapp-xyz" has successfully progressed.

  - type: Available
    status: "True"
    reason: MinimumReplicasAvailable
    message: Deployment has minimum availability.

  observedGeneration: 15
  replicas: 6
  updatedReplicas: 6
  readyReplicas: 6
  availableReplicas: 6
```

**Key fields**:
- `Progressing`: Is rollout making progress?
- `Available`: Are minimum replicas available?
- `replicas`: Total desired
- `updatedReplicas`: Running new version
- `readyReplicas`: Passing readiness probes
- `availableReplicas`: Ready and available for traffic

**Healthy deployment**:
```
replicas: 6
updatedReplicas: 6  (all pods updated)
readyReplicas: 6    (all pods ready)
availableReplicas: 6 (all pods available)
```

**Unhealthy deployment** (stuck):
```
replicas: 6
updatedReplicas: 2  (only 2 pods updated)
readyReplicas: 4    (2 new pods not ready)
availableReplicas: 4

Condition:
  type: Progressing
  status: "False"
  reason: ProgressDeadlineExceeded
  message: ReplicaSet "myapp-xyz" has timed out progressing.
```

Rollout stuck, new pods failing to become ready.

---

## Metrics for Rollout Health

### Deployment-Level Metrics (kube-state-metrics)

**kube-state-metrics** exposes Kubernetes object state as Prometheus metrics.

**Deployment metrics**:

```promql
# Desired vs. actual replicas
kube_deployment_spec_replicas{deployment="myapp"}
kube_deployment_status_replicas{deployment="myapp"}
kube_deployment_status_replicas_ready{deployment="myapp"}
kube_deployment_status_replicas_updated{deployment="myapp"}

# Deployment conditions
kube_deployment_status_condition{deployment="myapp",condition="Progressing",status="true"}
kube_deployment_status_condition{deployment="myapp",condition="Available",status="true"}
```

**Alerting rules**:

```yaml
# Alert if deployment not progressing
- alert: DeploymentNotProgressing
  expr: |
    kube_deployment_status_condition{condition="Progressing",status="false"} == 1
  for: 5m
  annotations:
    summary: "Deployment {{ $labels.deployment }} not progressing"

# Alert if replicas not ready
- alert: DeploymentReplicasNotReady
  expr: |
    kube_deployment_status_replicas_ready != kube_deployment_spec_replicas
  for: 10m
  annotations:
    summary: "Deployment {{ $labels.deployment }} has unhealthy replicas"
```

### Pod-Level Metrics

**Pod churn** (how many pods created/deleted):

```promql
# Pod creation rate
rate(kube_pod_created{namespace="production"}[5m])

# Pod deletion rate
rate(kube_pod_deleted{namespace="production"}[5m])
```

High churn during rollout is expected. Sustained high churn indicates instability (pods crashing and restarting).

**Pod restarts**:

```promql
# Restart count
kube_pod_container_status_restarts_total{pod=~"myapp-.*"}

# Restart rate (alerts on crash loops)
rate(kube_pod_container_status_restarts_total{pod=~"myapp-.*"}[5m]) > 0
```

**Pod status phases**:

```promql
# Pods in each phase
kube_pod_status_phase{namespace="production",phase="Running"}
kube_pod_status_phase{namespace="production",phase="Pending"}
kube_pod_status_phase{namespace="production",phase="Failed"}
```

Pending pods during rollout: normal (waiting to schedule/start)
Failed pods: investigate (image pull error, OOM, etc.)

### Application-Level Metrics

**Golden signals** (from your application):

**Latency**:
```promql
# P50, P95, P99 latency
histogram_quantile(0.50, rate(http_request_duration_seconds_bucket{job="myapp"}[5m]))
histogram_quantile(0.95, rate(http_request_duration_seconds_bucket{job="myapp"}[5m]))
histogram_quantile(0.99, rate(http_request_duration_seconds_bucket{job="myapp"}[5m]))
```

**Traffic**:
```promql
# Requests per second
rate(http_requests_total{job="myapp"}[5m])
```

**Errors**:
```promql
# Error rate
rate(http_requests_total{job="myapp",status=~"5.."}[5m])
/
rate(http_requests_total{job="myapp"}[5m])
```

**Saturation**:
```promql
# Memory usage percentage
container_memory_usage_bytes{pod=~"myapp-.*"}
/
container_spec_memory_limit_bytes{pod=~"myapp-.*"}
```

---

## Deployment Annotations: Correlation Markers

Mark deployments in metrics to correlate with changes.

### Grafana Annotations

Create annotation in Grafana when deployment happens:

```bash
# CI/CD script
curl -X POST https://grafana.example.com/api/annotations \
  -H "Authorization: Bearer $GRAFANA_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "dashboardId": 1,
    "time": '$(date +%s000)',
    "tags": ["deployment", "myapp", "v1.2.0"],
    "text": "Deployed myapp v1.2.0"
  }'
```

**On dashboard**: Vertical line showing deployment time.

```
Latency (ms)
500 |                              ▲ Deploy v1.2.0
400 |                              |
300 |         _____________________|_____
200 |________/                     |     \______
    |______|______|______|______|__|______|______
      10:00  10:15  10:30  10:45  11:00  11:15
```

See latency spike after deployment → investigate.

### Prometheus Labels

Label metrics with deployment version:

```yaml
# ServiceMonitor (Prometheus Operator)
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: myapp
spec:
  selector:
    matchLabels:
      app: myapp
  endpoints:
  - port: metrics
    relabelings:
    - sourceLabels: [__meta_kubernetes_pod_label_version]
      targetLabel: version  # Add version label to metrics
```

**Metrics now have version**:

```promql
http_requests_total{job="myapp",version="1.2.0"}
http_requests_total{job="myapp",version="1.1.0"}
```

**Compare old vs. new**:

```promql
# Error rate for new version
rate(http_requests_total{job="myapp",version="1.2.0",status=~"5.."}[5m])
/
rate(http_requests_total{job="myapp",version="1.2.0"}[5m])

# Error rate for old version
rate(http_requests_total{job="myapp",version="1.1.0",status=~"5.."}[5m])
/
rate(http_requests_total{job="myapp",version="1.1.0"}[5m])
```

If new version error rate > old version error rate → rollback.

---

## Distributed Tracing During Rollouts

### The Version Transition Problem

During rollout, requests span both old and new versions:

```
Request 1:
  API Gateway (v1.2.0) → User Service (v1.2.0) → Database

Request 2:
  API Gateway (v1.2.0) → User Service (v1.1.0) → Database

Request 3:
  API Gateway (v1.1.0) → User Service (v1.2.0) → Database
```

Mixed versions in single trace.

### Tracing with Version Tags

**Add version to trace spans**:

```go
import "go.opentelemetry.io/otel"

func handleRequest(ctx context.Context, req *http.Request) {
    tracer := otel.Tracer("myapp")
    ctx, span := tracer.Start(ctx, "handle_request")
    defer span.End()

    // Tag span with version
    span.SetAttributes(attribute.String("app.version", "1.2.0"))

    // Process request...
}
```

**Trace UI** (Jaeger):

```
Trace ID: abc123
Duration: 150ms

Span: API Gateway
  app.version: 1.2.0
  duration: 50ms

  Span: User Service
    app.version: 1.1.0  ← Old version
    duration: 80ms

    Span: Database Query
      duration: 70ms
```

**Insight**: User Service (old version) is slow. New version might fix it.

**Query traces by version**:

```
service.name=user-service AND app.version=1.2.0
```

Compare latency distribution:
- v1.1.0: P99 = 500ms
- v1.2.0: P99 = 300ms (faster!)

---

## Dashboard Design for Rollouts

### Dashboard 1: Deployment Health

**Panels**:

1. **Deployment Status** (single stat):
   - Desired replicas vs. ready replicas
   - "6 / 6 Ready" (green) or "4 / 6 Ready" (yellow)

2. **Rollout Progress** (gauge):
   - `updatedReplicas / replicas * 100`
   - "100% Updated" (done) or "33% Updated" (in progress)

3. **Pod Phases** (stacked bar):
   - Running: 6
   - Pending: 0
   - Failed: 0

4. **Recent Events** (table):
   - Timestamp, Reason, Message
   - ScalingReplicaSet, NewReplicaSetAvailable, etc.

5. **Conditions** (status panel):
   - Progressing: True ✓
   - Available: True ✓

### Dashboard 2: Version Comparison

**Panels**:

1. **Request Rate by Version** (graph):
   ```promql
   sum(rate(http_requests_total{job="myapp"}[5m])) by (version)
   ```
   Shows traffic shifting from v1.1.0 to v1.2.0

2. **Error Rate by Version** (graph):
   ```promql
   sum(rate(http_requests_total{status=~"5..",job="myapp"}[5m])) by (version)
   /
   sum(rate(http_requests_total{job="myapp"}[5m])) by (version)
   ```
   Compare error rates

3. **Latency by Version** (graph):
   ```promql
   histogram_quantile(0.99,
     sum(rate(http_request_duration_seconds_bucket{job="myapp"}[5m])) by (version, le)
   )
   ```
   Compare P99 latency

4. **Memory Usage by Version** (graph):
   ```promql
   avg(container_memory_usage_bytes{pod=~"myapp-.*"}) by (version)
   ```
   Detect memory leaks in new version

### Dashboard 3: Rollout Timeline

**Panels**:

1. **Replica Count Over Time** (graph):
   ```promql
   kube_deployment_status_replicas{deployment="myapp"}
   kube_deployment_status_replicas_ready{deployment="myapp"}
   kube_deployment_status_replicas_updated{deployment="myapp"}
   ```
   See rollout progression

2. **Pod Lifecycle Events** (annotations):
   - PodCreated, PodReady, PodDeleted
   - Visualize pod churn

3. **Error Rate Timeline** (graph with annotations):
   - Deployment annotations mark version changes
   - See if errors correlate with deployments

---

## Alerting on Failed Rollouts

### Alert 1: Deployment Not Progressing

```yaml
- alert: DeploymentStuck
  expr: |
    kube_deployment_status_condition{
      condition="Progressing",
      status="false"
    } == 1
  for: 10m
  labels:
    severity: warning
  annotations:
    summary: "Deployment {{ $labels.deployment }} stuck for 10 minutes"
    description: "Check pod status and events"
```

**Trigger**: Deployment has `Progressing=False` for 10 minutes.

**Action**: Check pod logs, describe deployment, check for resource constraints.

### Alert 2: High Error Rate After Deployment

```yaml
- alert: PostDeploymentErrorSpike
  expr: |
    (
      rate(http_requests_total{status=~"5..",job="myapp"}[5m])
      /
      rate(http_requests_total{job="myapp"}[5m])
    ) > 0.05  # 5% error rate
  for: 5m
  labels:
    severity: critical
  annotations:
    summary: "High error rate after deployment: {{ $value | humanizePercentage }}"
    description: "Consider rollback"
```

**Trigger**: Error rate > 5% for 5 minutes.

**Action**: Check if recent deployment, consider rollback.

### Alert 3: Canary Performing Worse

```yaml
- alert: CanaryHigherLatency
  expr: |
    histogram_quantile(0.99,
      rate(http_request_duration_seconds_bucket{version="canary"}[5m])
    )
    >
    histogram_quantile(0.99,
      rate(http_request_duration_seconds_bucket{version="stable"}[5m])
    ) * 1.5  # 50% higher
  for: 5m
  labels:
    severity: warning
  annotations:
    summary: "Canary P99 latency 50% higher than stable"
    description: "Investigate performance regression"
```

**Trigger**: Canary P99 latency > 1.5x stable P99.

**Action**: Automated rollback (if integrated with Flagger) or manual investigation.

---

## Real-User Monitoring (RUM) vs. Synthetic

### Synthetic Monitoring

**What it is**: Automated tests simulating user behavior.

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: smoke-tests
spec:
  schedule: "*/5 * * * *"  # Every 5 minutes
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: test
            image: test-runner
            command: ["run-smoke-tests.sh"]
          restartPolicy: Never
```

**Pros**: Predictable, catches obvious breakage, runs 24/7
**Cons**: Doesn't reflect real user experience, limited coverage

### Real-User Monitoring

**What it is**: Metrics from actual user requests.

```javascript
// Frontend RUM (browser)
window.addEventListener('load', () => {
  const loadTime = performance.timing.loadEventEnd - performance.timing.navigationStart;

  fetch('/api/metrics', {
    method: 'POST',
    body: JSON.stringify({
      metric: 'page_load_time',
      value: loadTime,
      version: '1.2.0'
    })
  });
});
```

**Pros**: Real user experience, captures edge cases, geographic diversity
**Cons**: Depends on traffic, harder to debug

### Combining Both

**Deployment workflow**:

1. Deploy canary (5% traffic)
2. Synthetic tests run against canary → Pass
3. Monitor RUM metrics from real users hitting canary
4. Compare RUM metrics: canary vs. stable
5. If canary RUM metrics degrade → Rollback
6. If canary RUM metrics same/better → Promote

---

## Interview Deep Dive

**Question**: "How do you know if a deployment is healthy?"

**Shallow answer**: "Check if pods are running."

**Deep answer**:
"Deployment health is multi-dimensional - you need to check infrastructure health, application health, and business metrics.

At the infrastructure layer, check deployment conditions. The Progressing condition should be True, indicating the rollout is making progress. The Available condition should be True, meaning minimum replicas are available. Check that replicas match the desired count: replicas, updatedReplicas, readyReplicas, and availableReplicas should all equal the spec.

At the application layer, use golden signals. Compare the new version to the old version: error rate should not increase, latency (P99, P95) should not regress, and request rate should be stable. Check resource utilization - memory usage should be expected, not indicating a leak. Pod restarts should be zero or minimal, not a crash loop.

At the business layer, monitor real user metrics. Page load times, transaction completion rates, user-reported errors. These can catch issues that infrastructure metrics miss - for example, a frontend bug that doesn't cause 5xx errors but breaks the checkout flow.

The key is comparing versions. Absolute metrics can be misleading - '2% error rate' could be normal or catastrophic depending on baseline. Comparing new version to old version (2% vs. 0.1%) reveals regressions. Use version labels in Prometheus metrics to enable this comparison. Distributed tracing with version tags helps understand which version is slow in multi-service traces.

Automated tools like Flagger codify this analysis - query Prometheus for canary vs. stable metrics, apply thresholds, decide to promote or rollback. Without automation, you need dashboards showing side-by-side comparisons and alerts for regression patterns."

**Question**: "What metrics would you alert on during a deployment?"

**Deep answer**:
"Alert strategy should balance early detection with avoiding alert fatigue. I'd implement three tiers.

Tier 1 - Critical (immediate page): Deployment stuck for 10+ minutes (Progressing condition False), error rate spike above 5%, complete outage (all pods unavailable). These indicate the deployment is definitively broken and requires immediate intervention.

Tier 2 - Warning (Slack notification): Canary error rate higher than stable, latency regression (P99 >50% increase), high pod restart rate (indicating crashes), resource saturation (memory >90%). These suggest the deployment might be problematic but need investigation before rollback.

Tier 3 - Informational (logged, not alerted): Deployment completed, rollout duration exceeds baseline, pod churn rate. These are useful for post-deployment analysis but don't require real-time action.

Critical alerts should integrate with automated rollback - if error rate exceeds threshold for 5 consecutive minutes, trigger rollback automatically. Warnings should pause automated rollout progression until investigated. Informational metrics should be queryable for debugging but not create noise.

The version comparison is crucial - don't alert on absolute values, alert on relative changes. 'Error rate = 2%' is meaningless without context. 'Canary error rate (2%) > 2x stable error rate (0.5%)' is actionable.

Also consider traffic volume - a canary getting 5% traffic with 100 requests/second has statistical significance. A canary with 1 request/second does not - wait for more data before alerting on rate changes."

---

## Key Takeaways

1. **Three pillars**: Metrics (dashboards, alerts), Logs (debugging), Traces (request journey)
2. **Deployment events and conditions**: Progressing, Available, replica counts
3. **Version comparison**: Label metrics with version, compare canary vs. stable
4. **Golden signals**: Latency, traffic, errors, saturation - monitor all four
5. **Deployment annotations**: Mark deployments on graphs, correlate metrics with changes
6. **Distributed tracing**: Tag spans with version, analyze mixed-version traces
7. **Automated alerting**: Stuck deployments, error spikes, latency regressions

Observability turns deployments from blind operations into data-driven decisions. Without metrics, you're deploying and hoping. With comprehensive observability - comparing versions, correlating changes with metrics, automated alerts on regressions - you can deploy confidently and catch issues before users do.
