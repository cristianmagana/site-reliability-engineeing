# Progressive Delivery on Kubernetes: Automated Canary and Blue/Green

## The Gap Between Native K8s and Progressive Delivery

Kubernetes provides rolling updates out of the box. But progressive delivery requires more:

**What native K8s provides**:
- Rolling updates (gradual pod replacement)
- Replica-based traffic distribution (9 pods old, 1 pod new = 90/10 split)
- Health checks (readiness gates traffic)

**What native K8s doesn't provide**:
- Automated canary analysis (check metrics, decide to proceed or rollback)
- Fine-grained traffic splitting (5% traffic independent of replica count)
- Automated rollback based on error rates, latency
- Progressive stages (1% → 5% → 25% → 100% automatically)

**The tools that bridge the gap**: Flagger, Argo Rollouts, service meshes.

---

## Canary Deployments: Replica-Based vs. Mesh-Based

### Approach 1: Multiple Deployments (Replica-Based)

**How it works**: Create two Deployments, adjust replica counts for traffic split.

```yaml
# Stable deployment (v1.0.0)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-stable
spec:
  replicas: 9  # 90% of traffic
  template:
    metadata:
      labels:
        app: myapp
        version: stable
    spec:
      containers:
      - name: myapp
        image: myapp:1.0.0

---
# Canary deployment (v1.1.0)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-canary
spec:
  replicas: 1  # 10% of traffic
  template:
    metadata:
      labels:
        app: myapp
        version: canary
    spec:
      containers:
      - name: myapp
        image: myapp:1.1.0

---
# Service selects both
apiVersion: v1
kind: Service
metadata:
  name: myapp
spec:
  selector:
    app: myapp  # Selects both stable and canary
  ports:
  - port: 80
    targetPort: 8080
```

**Traffic distribution**:
```
Service endpoints:
  stable: 9 pods
  canary: 1 pod

Random load balancing → ~90% to stable, ~10% to canary
```

**Pros**:
- Simple, no extra tools
- Works with standard kube-proxy
- No service mesh required

**Cons**:
- Coarse-grained (10% requires 1 canary pod + 9 stable pods)
- Can't do 5% without 1 canary + 19 stable (20 total pods)
- No automated analysis
- Manual replica adjustment

### Approach 2: Service Mesh (Traffic-Based)

**How it works**: Service mesh (Istio, Linkerd) routes traffic based on percentage, independent of replicas.

```yaml
# Stable deployment (any replica count)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-stable
spec:
  replicas: 3
  template:
    metadata:
      labels:
        app: myapp
        version: stable

---
# Canary deployment (any replica count)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-canary
spec:
  replicas: 3
  template:
    metadata:
      labels:
        app: myapp
        version: canary

---
# Istio VirtualService: 95% to stable, 5% to canary
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: myapp
spec:
  hosts:
  - myapp
  http:
  - match:
    - uri:
        prefix: /
    route:
    - destination:
        host: myapp
        subset: stable
      weight: 95  # 95% to stable
    - destination:
        host: myapp
        subset: canary
      weight: 5   # 5% to canary
```

**Pros**:
- Fine-grained traffic control (1%, 5%, any percentage)
- Independent of replica count
- Can split traffic by headers, user ID, etc.

**Cons**:
- Requires service mesh (complexity, resource overhead)
- Still manual (must update VirtualService weights)
- No automated analysis

---

## Flagger: Automated Progressive Delivery

**What it is**: Kubernetes controller that automates canary deployments with metric analysis.

### How Flagger Works

**You provide**:
1. A Deployment (your application)
2. A Canary resource (Flagger CRD defining rollout strategy)
3. Metrics (Prometheus queries for success rate, latency)

**Flagger does**:
1. Creates canary Deployment automatically
2. Gradually shifts traffic (0% → 10% → 20% → ... → 100%)
3. Monitors metrics at each step
4. Automatically promotes or rolls back based on metrics

### Flagger Architecture

```
                  ┌─────────────┐
                  │   Flagger   │  (Controller)
                  └──────┬──────┘
                         │
         ┌───────────────┼───────────────┐
         │               │               │
         ↓               ↓               ↓
   ┌──────────┐   ┌──────────┐   ┌──────────┐
   │ Primary  │   │  Canary  │   │Prometheus│
   │Deployment│   │Deployment│   │  Metrics │
   └──────────┘   └──────────┘   └──────────┘
         │               │
         └───────┬───────┘
                 ↓
          ┌──────────┐
          │ Service  │
          │  Mesh    │  (Istio/Linkerd)
          └──────────┘
```

### Flagger Canary Resource

```yaml
apiVersion: flagger.app/v1beta1
kind: Canary
metadata:
  name: myapp
  namespace: default
spec:
  # Target deployment to manage
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: myapp

  # Service mesh provider
  provider: istio  # or linkerd, nginx, gloo, etc.

  # Service configuration
  service:
    port: 80
    targetPort: 8080

  # Analysis configuration
  analysis:
    interval: 1m           # Check metrics every 1 minute
    threshold: 5           # Allow 5 failed checks before rollback
    maxWeight: 50          # Max canary weight (don't go above 50%)
    stepWeight: 10         # Increase by 10% each iteration

    # Metrics to check
    metrics:
    - name: request-success-rate
      thresholdRange:
        min: 99            # Must maintain 99% success rate
      interval: 1m

    - name: request-duration
      thresholdRange:
        max: 500           # P99 latency must be < 500ms
      interval: 1m

  # Metric queries (Prometheus)
  metricsServer: http://prometheus:9090
```

### Flagger Rollout Process

**Initial state**:
```
Primary deployment: myapp-primary (3 replicas, v1.0.0)
Canary deployment: myapp (0 replicas, v1.0.0)
Traffic: 100% to primary
```

**User updates Deployment**:
```bash
kubectl set image deployment/myapp myapp=myapp:1.1.0
```

**Flagger detects change and starts canary**:

```
Iteration 0 (0:00):
  Canary deployment: Scale to 3 replicas (v1.1.0)
  Traffic: 0% canary, 100% primary
  Wait for canary pods to be ready...

Iteration 1 (0:01):
  Canary pods ready ✓
  Traffic: 10% canary, 90% primary
  Check metrics...
    request-success-rate: 99.5% ✓ (>= 99)
    request-duration: 350ms ✓ (<= 500ms)
  Result: PROCEED

Iteration 2 (0:02):
  Traffic: 20% canary, 80% primary
  Check metrics...
    request-success-rate: 99.2% ✓
    request-duration: 380ms ✓
  Result: PROCEED

Iteration 3 (0:03):
  Traffic: 30% canary, 70% primary
  Check metrics...
    request-success-rate: 98.5% ✗ (< 99)
  Failed check count: 1 / 5

Iteration 4 (0:04):
  Traffic: still 30% (don't increase due to previous failure)
  Check metrics...
    request-success-rate: 98.2% ✗
  Failed check count: 2 / 5

Iteration 5 (0:05):
  Failed check count: 3 / 5
  (If reaches 5, automatic rollback)

Iteration 6 (0:06):
  Check metrics...
    request-success-rate: 99.1% ✓ (recovered)
  Failed check count: Reset to 0
  Traffic: 40% canary, 60% primary
  Result: PROCEED

...

Iteration 10 (0:10):
  Traffic: 50% canary, 50% primary (reached maxWeight)
  Check metrics...
    All healthy ✓
  Result: PROMOTE

Promotion (0:11):
  Primary deployment updated: v1.0.0 → v1.1.0
  Canary deployment scaled to 0
  Traffic: 100% to primary (now v1.1.0)

Canary complete ✓
```

**If metrics fail threshold times (5)**:
```
Rollback:
  Traffic: 0% canary, 100% primary
  Canary deployment scaled to 0
  Alert: Canary failed, investigate metrics
```

### Flagger Metric Queries

Flagger queries Prometheus for metrics.

**Example Prometheus queries**:

```yaml
metrics:
- name: request-success-rate
  thresholdRange:
    min: 99
  interval: 1m
  # Prometheus query
  templateRef:
    name: success-rate
    namespace: flagger-system

---
# ConfigMap defining the query
apiVersion: v1
kind: ConfigMap
metadata:
  name: success-rate
  namespace: flagger-system
data:
  query: |
    sum(
      rate(
        http_requests_total{
          app="{{ .Name }}",
          status!~"5.."
        }[{{ .Interval }}]
      )
    ) /
    sum(
      rate(
        http_requests_total{
          app="{{ .Name }}"
        }[{{ .Interval }}]
      )
    ) * 100
```

**Variables available**:
- `{{ .Name }}`: Canary name
- `{{ .Namespace }}`: Namespace
- `{{ .Interval }}`: Analysis interval

---

## Argo Rollouts: Advanced Deployment Strategies

**What it is**: Kubernetes controller providing advanced deployment strategies (canary, blue/green) as CRD.

**Differs from Flagger**:
- More deployment strategies (Flagger focuses on canary)
- Tighter GitOps integration (Argo CD)
- Analysis templates (reusable metric checks)

### Argo Rollout Resource

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: myapp
spec:
  replicas: 5
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
        image: myapp:1.0.0

  # Canary strategy
  strategy:
    canary:
      steps:
      - setWeight: 20      # 20% traffic to canary
      - pause: {duration: 5m}  # Wait 5 minutes

      - setWeight: 40
      - pause: {duration: 5m}

      - setWeight: 60
      - pause: {duration: 5m}

      - setWeight: 80
      - pause: {duration: 5m}

      # Automated analysis
      - setWeight: 100
        analysis:
          templates:
          - templateName: success-rate
          args:
          - name: service-name
            value: myapp

      # Traffic routing (for service mesh)
      trafficRouting:
        istio:
          virtualService:
            name: myapp
            routes:
            - primary
```

### Blue/Green with Argo Rollouts

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: myapp
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
  template:
    spec:
      containers:
      - name: myapp
        image: myapp:1.0.0

  strategy:
    blueGreen:
      # Active service points to stable
      activeService: myapp-active

      # Preview service points to new version
      previewService: myapp-preview

      # Auto-promotion after 30s (or manual approval)
      autoPromotionSeconds: 30

      # Keep old version for rollback
      scaleDownDelaySeconds: 300
```

**How it works**:

```
1. Deploy new version (green)
   Active service: blue (v1.0.0)
   Preview service: green (v1.1.0) ← New version available for testing

2. Test preview service (internal testing, smoke tests)
   curl http://myapp-preview/

3. Auto-promotion (after 30s) or manual
   argo rollouts promote myapp

4. Switch active service to green
   Active service: green (v1.1.0)
   Preview service: (none)

5. Keep blue for rollback (5 minutes)
   After 5 minutes, blue scaled down

6. If rollback needed within 5 minutes:
   argo rollouts abort myapp
   Active service switches back to blue
```

### Analysis Templates

Reusable metric checks.

```yaml
apiVersion: argoproj.io/v1alpha1
kind: AnalysisTemplate
metadata:
  name: success-rate
spec:
  args:
  - name: service-name

  metrics:
  - name: success-rate
    interval: 1m
    successCondition: result >= 0.99
    provider:
      prometheus:
        address: http://prometheus:9090
        query: |
          sum(rate(http_requests_total{service="{{args.service-name}}",status!~"5.."}[1m])) /
          sum(rate(http_requests_total{service="{{args.service-name}}"}[1m]))
```

**Use in Rollout**:

```yaml
strategy:
  canary:
    steps:
    - setWeight: 20
    - analysis:
        templates:
        - templateName: success-rate
        args:
        - name: service-name
          value: myapp
```

**If analysis fails**: Rollback automatically.

---

## Comparison: Native K8s vs. Flagger vs. Argo Rollouts

| Feature | Native K8s | Flagger | Argo Rollouts |
|---------|-----------|---------|---------------|
| **Rolling Update** | ✅ Built-in | ✅ (as baseline) | ✅ |
| **Blue/Green** | Manual (Service selector) | ❌ | ✅ |
| **Canary** | Manual (multiple Deployments) | ✅ Automated | ✅ Automated |
| **Traffic %** | Replica-based | Mesh-based | Mesh-based |
| **Metric Analysis** | ❌ | ✅ Prometheus | ✅ Prometheus, Datadog, etc. |
| **Auto Rollback** | ❌ | ✅ | ✅ |
| **Progressive Stages** | ❌ | ✅ | ✅ |
| **Webhooks** | ❌ | ✅ (for notifications) | ✅ |
| **Manual Gates** | ❌ | ✅ (webhooks) | ✅ (pause steps) |
| **GitOps Native** | N/A | Works with ArgoCD/Flux | ✅ ArgoCD integration |
| **Service Mesh Required** | ❌ | ✅ (for fine-grained %) | ✅ (for fine-grained %) |

---

## Automated Rollback: Decision Logic

Both Flagger and Argo Rollouts use similar decision logic.

### Metric Evaluation

```
For each analysis interval:
  1. Query metric from provider (Prometheus)
  2. Evaluate threshold
     - successCondition: result >= 0.99
     - failureCondition: result < 0.95
  3. Record result
  4. Check consecutive failures against threshold

If consecutiveFailures >= threshold:
  ROLLBACK

If all metrics pass:
  PROCEED to next stage
```

### Example: Flagger Decision Tree

```
Analysis runs every 1 minute
Threshold: 5 consecutive failures

Minute 1: Metrics pass ✓
Minute 2: Metrics pass ✓
Minute 3: Metrics fail ✗ (count: 1)
Minute 4: Metrics fail ✗ (count: 2)
Minute 5: Metrics pass ✓ (count: reset to 0)
Minute 6: Metrics fail ✗ (count: 1)
Minute 7: Metrics fail ✗ (count: 2)
Minute 8: Metrics fail ✗ (count: 3)
Minute 9: Metrics fail ✗ (count: 4)
Minute 10: Metrics fail ✗ (count: 5 - ROLLBACK!)

Action: Shift traffic to 0% canary, scale down canary
```

### Rollback Process

**Flagger**:
```
1. Set canary weight to 0% (all traffic to primary)
2. Scale canary deployment to 0 replicas
3. Update Canary resource status: Failed
4. Send webhook notification (Slack, etc.)
5. Wait for manual investigation and re-deploy
```

**Argo Rollouts**:
```
1. Abort rollout
2. Scale down new ReplicaSet to 0
3. Ensure old ReplicaSet is at full scale
4. Update Rollout status: Degraded
5. Require manual intervention to retry
```

---

## Header-Based Canary (Advanced)

Send specific users to canary based on headers.

**Istio VirtualService**:

```yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: myapp
spec:
  hosts:
  - myapp
  http:
  # Route internal employees to canary
  - match:
    - headers:
        x-user-group:
          exact: "internal"
    route:
    - destination:
        host: myapp
        subset: canary

  # Route beta users to canary
  - match:
    - headers:
        x-user-group:
          exact: "beta"
    route:
    - destination:
        host: myapp
        subset: canary

  # Everyone else to stable
  - route:
    - destination:
        host: myapp
        subset: stable
```

**Flagger integration**:

```yaml
spec:
  canaryAnalysis:
    match:
    - headers:
        x-user-group:
          exact: "beta"
```

---

## Interview Deep Dive

**Question**: "How do you implement automated canary deployments on Kubernetes?"

**Shallow answer**: "Use Flagger or Argo Rollouts."

**Deep answer**:
"Automated canary deployments on Kubernetes require three components: traffic management, metric collection, and decision automation.

Native Kubernetes provides rolling updates but not canary - you can manually create two Deployments and adjust replicas for traffic distribution, but this is coarse-grained (10% requires 1 canary + 9 stable pods) and manual.

For fine-grained control, you need a service mesh like Istio or Linkerd. The mesh routes traffic by percentage independent of replica count - you can have 3 canary pods and 3 stable pods, but route only 5% traffic to canary via VirtualService weight configuration.

The automation layer comes from tools like Flagger or Argo Rollouts. These are Kubernetes controllers that watch your Deployment, detect image changes, and orchestrate progressive rollout. They create a canary Deployment, gradually shift traffic (10% → 20% → 30%), and at each stage query Prometheus for metrics like error rate and latency.

If metrics exceed thresholds (e.g., error rate > 1%), the tool tracks consecutive failures. After N failures (configurable), it automatically rolls back - shifts traffic to 0% canary, scales down the canary Deployment. If all stages pass, it promotes the canary by updating the primary Deployment and cleaning up the canary.

The key distinction from manual processes: humans define the success criteria (error rate < 1%), the tool enforces it automatically, including rollback. This enables safe, hands-off deployments even at 2 AM."

**Question**: "Flagger vs. Argo Rollouts - when to use each?"

**Deep answer**:
"Both automate progressive delivery but have different philosophies and ecosystems.

Flagger is specialized for canary deployments. It's provider-agnostic (works with Istio, Linkerd, nginx, AWS App Mesh) and focuses on automated metric analysis with Prometheus. It's excellent if you primarily need canary with automated rollback and already have a service mesh. Flagger creates separate Deployments for primary and canary, managing them through its controller.

Argo Rollouts is more comprehensive - it supports canary, blue/green, and custom strategies via a Rollout CRD that replaces Deployment. It integrates tightly with Argo CD for GitOps workflows and supports multiple metric providers (Prometheus, Datadog, New Relic). Rollouts has richer analysis templates that are reusable across multiple rollouts.

Choose Flagger if: You want a lightweight tool focused on canary, you're comfortable with Deployments and want to add canary on top, you need webhook integrations for notifications.

Choose Argo Rollouts if: You're using Argo CD for GitOps, you want blue/green in addition to canary, you need reusable analysis templates across many services, you want a unified Rollout abstraction instead of managing Deployments separately.

Both require a service mesh for fine-grained traffic splitting. Without a mesh, you're limited to replica-based traffic distribution which is coarse."

---

## Key Takeaways

1. **Native K8s limitations**: Rolling updates, no automated canary/blue-green, no metric analysis
2. **Service mesh enables fine-grained traffic**: 5% canary without 20 total pods
3. **Flagger**: Automated canary with Prometheus metric analysis, multi-provider support
4. **Argo Rollouts**: Comprehensive strategies (canary, blue/green), GitOps integration, analysis templates
5. **Automated rollback**: Metric thresholds trigger automatic rollback, no manual intervention
6. **Progressive stages**: 10% → 20% → 50% → 100% with validation at each stage
7. **Header-based routing**: Target specific users for canary (internal, beta, geographic)

Progressive delivery on Kubernetes requires extending the platform. Native capabilities provide the foundation (rolling updates, health checks), but automation requires controllers (Flagger, Argo Rollouts) that orchestrate traffic shifts, monitor metrics, and make promotion/rollback decisions. Combined with service mesh for traffic management and Prometheus for observability, you can achieve fully automated, metrics-driven deployments.
