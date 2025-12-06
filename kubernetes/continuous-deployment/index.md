# Continuous Deployment on Kubernetes

## What This Is

Continuous deployment on Kubernetes fundamentally differs from traditional deployment approaches. Instead of writing scripts that choreograph deployment steps, you declare desired state and let control loops converge actual state toward it. This section explores the theoretical foundations and practical implementations that make Kubernetes-native CD possible.

This guide covers the paradigm shift from imperative to declarative deployment, the control loop architecture that enables self-healing, and how Kubernetes primitives combine to enable modern deployment patterns like canary releases, blue/green deployments, and GitOps workflows.

---

## Core Topics

### 1. [Control Loop Theory](./control-loop-theory.md)
The heart of Kubernetes - continuous reconciliation loops. Covers:
- **Observe-compare-act-repeat pattern** from control theory
- **Level-triggered vs. edge-triggered systems** (why Kubernetes self-heals)
- Reconciliation mechanics (watch API, requeue, convergence)
- Eventual consistency guarantees
- Multiple controllers on single objects (HPA + Deployment)
- Failure modes and recovery patterns
- Performance characteristics (reconciliation intervals, convergence time)

**Key Concepts**: Control loops as fundamental architecture, self-healing through continuous reconciliation, level-triggered resilience

### 2. [Declarative State Model](./declarative-state-model.md)
Desired vs. actual state separation. Covers:
- **Imperative vs. declarative** operational models (beyond syntax)
- Why declarative wins for CD (idempotency, drift detection, auditability)
- **Client-side apply**: Three-way merge, annotation limitations
- **Server-side apply**: Field-level ownership, managed fields
- Structured merge diff vs. text diff
- Multi-actor scenarios (GitOps + HPA, manual kubectl)
- GitOps implications and source of truth

**Key Concepts**: State separation, field ownership, conflict resolution, Git as source of truth

### 3. [Configuration Management](./configuration-management.md)
Templates, rendering, and the GitOps source of truth debate. Covers:
- **The parameterization problem**: Managing multiple environments
- **Helm**: Templating engine, values.yaml, chart structure
- **Kustomize**: Base + overlays, patch strategies
- **Alternative approaches**: ytt, cue, jsonnet
- **Rendered manifests debate**: Template-only vs. rendered-only vs. hybrid
- Environment promotion patterns (dev → staging → prod)
- Auditability and review workflows
- ArgoCD/Flux rendering strategies

**Key Concepts**: Source of truth definition, template vs. rendered state, environment management

### 4. [Deployment Controllers](./deployment-controllers.md)
How Kubernetes orchestrates updates. Covers:
- **Deployment → ReplicaSet → Pod hierarchy** (why three levels?)
- Rolling update mechanics (maxSurge, maxUnavailable calculations)
- **Deployment controller implementation** (reconciliation logic)
- ReplicaSet revision history
- StatefulSets for ordered deployments
- DaemonSets for node-level services
- When to use which controller
- Deployment strategies (RollingUpdate vs. Recreate)

**Key Concepts**: Controller hierarchy, rolling update guarantees, stateful vs. stateless workloads

### 5. [Health Checks and Probes](./health-checks-probes.md)
Foundation of safe deployments. Covers:
- **Liveness vs. readiness vs. startup probes** (distinct purposes)
- Integration with rolling updates (readiness gates traffic)
- Probe types (HTTP, TCP, exec, gRPC)
- Timing configuration (initialDelay, period, timeout, threshold)
- Probe design patterns (what to check, dependencies)
- Common anti-patterns (probes that lie, cascade failures)
- Readiness gates for custom validation

**Key Concepts**: Health as precondition for traffic, probe design philosophy, failure propagation

### 6. [Progressive Delivery on Kubernetes](./progressive-delivery-k8s.md)
Advanced deployment patterns. Covers:
- **Canary deployments**: Multiple deployment approach vs. service mesh
- **Flagger**: Automated progressive delivery, metric analysis
- **Argo Rollouts**: Blue/green, canary with analysis templates
- Traffic splitting strategies (replica-based vs. mesh-based)
- Automated promotion based on Prometheus metrics
- Rollback automation and decision thresholds
- Comparison: Native K8s vs. Flagger vs. Argo vs. service mesh

**Key Concepts**: Metrics-driven automation, blast radius control, progressive validation

### 7. [Service Mesh Traffic Management](./service-mesh-traffic-management.md)
Fine-grained traffic control. Covers:
- **Why kube-proxy isn't enough** for advanced CD patterns
- **Istio**: VirtualServices, DestinationRules, weighted routing
- **Linkerd**: Simpler service mesh, TrafficSplits (SMI)
- Header-based routing (canary for specific users)
- Circuit breaking during deployments
- Retry policies and timeouts
- Observability benefits (distributed tracing, golden metrics)

**Key Concepts**: L7 traffic management, service mesh value proposition, complexity trade-offs

### 8. [GitOps Workflows](./gitops-workflows.md)
Git as source of truth. Covers:
- **Pull-based vs. push-based deployment** (security implications)
- **ArgoCD**: Application CRDs, sync strategies, health assessment
- **Flux**: GitOps toolkit, Kustomize/Helm controllers
- Multi-environment strategies (mono-repo vs. multi-repo)
- Automated sync vs. manual approval gates
- Secret management (sealed secrets, external secrets operator)
- Drift detection and remediation

**Key Concepts**: Declarative cluster state, continuous reconciliation, audit trail in Git

### 9. [Observability for Rollouts](./observability-rollouts.md)
Seeing what's happening. Covers:
- **Deployment events and conditions** (Progressing, Available, ReplicaFailure)
- Metrics for rollout health (pod churn, ready replicas, rollout duration)
- Prometheus metrics from kube-state-metrics
- Deployment annotations for correlation (version markers)
- Distributed tracing during version transitions
- Dashboard design for rollout monitoring
- Alerting on failed/stuck deployments

**Key Concepts**: Deployment observability, metric correlation, rollout health signals

### 10. [Rollback Strategies](./rollback-strategies.md)
When things go wrong. Covers:
- **kubectl rollout undo** mechanics (ReplicaSet history)
- Revision history limits and garbage collection
- GitOps rollback (git revert vs. git reset)
- Automated rollback triggers (metric thresholds)
- Rollback vs. roll-forward decision framework
- Database migration considerations (backward compatibility)
- Partial rollback strategies (per-environment, per-region)

**Key Concepts**: Rollback mechanisms, automated vs. manual, stateful rollback challenges

---

## The Paradigm Shift

### Imperative (Traditional)

```bash
# deploy.sh - You choreograph HOW
for instance in $(get_instances); do
  stop_instance $instance
  deploy_code $instance
  start_instance $instance
  health_check $instance
done
```

**Properties**: Sequential, fragile, one-shot execution, manual recovery

### Declarative (Kubernetes)

```yaml
# deployment.yaml - You declare WHAT
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  replicas: 6
  template:
    spec:
      containers:
      - name: myapp
        image: myapp:1.2.0
```

**Properties**: Continuous reconciliation, self-healing, eventual consistency, automatic recovery

---

## How Kubernetes Enables CD Patterns

Kubernetes primitives map directly to deployment strategies from release engineering:

### Rolling Updates (Native)
```yaml
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxSurge: 2
    maxUnavailable: 1
```
Built-in. Update image, apply, controller handles gradual rollout.

### Blue/Green (Service Switching)
```yaml
# Two Deployments: myapp-blue, myapp-green
# Service selector switches: version: blue → version: green
```
Instant cutover by changing Service selector.

### Canary (Multiple Deployments or Service Mesh)
```yaml
# Deployment 1: 9 replicas (v1.1.0)
# Deployment 2: 1 replica (v1.2.0)
# Service selects both → 90/10 split
```
Or use Istio/Linkerd for precise percentage control.

### Progressive Delivery (Flagger, Argo Rollouts)
```yaml
apiVersion: flagger.app/v1beta1
kind: Canary
spec:
  analysis:
    stepWeight: 10  # Increase by 10% each interval
    metrics:
    - name: request-success-rate
      thresholdRange:
        min: 99
```
Automated canary with metric analysis and rollback.

---

## GitOps Model

### Traditional CI/CD (Push)
```
Code → CI builds → CI kubectl apply → Cluster
```
CI has cluster write access. Audit trail in CI logs.

### GitOps (Pull)
```
Code → CI builds → CI commits manifest → Git
                                          ↓
                          GitOps Controller watches Git
                                          ↓
                                       Cluster
```
No CI cluster access. Git is source of truth. Continuous sync.

**Tools**: ArgoCD, Flux

---

## Configuration Management Approaches

### Helm (Templating)
```yaml
replicas: {{ .Values.replicas }}
image: {{ .Values.image }}:{{ .Values.tag }}
```
**Pros**: Flexible, packages as charts, widely adopted
**Cons**: Template complexity, values.yaml can grow large

### Kustomize (Overlays)
```
base/
  deployment.yaml
overlays/
  dev/kustomization.yaml
  prod/kustomization.yaml
```
**Pros**: Template-free, built into kubectl, composable
**Cons**: Patch syntax, less flexible than templates

### The Rendered Manifests Debate
- **Templates only**: DRY, but deployed state opaque
- **Rendered only**: Explicit, reviewable, verbose
- **Hybrid**: Both committed (best auditability)

---

## Server-Side Apply: Field Ownership

Multiple actors (GitOps controller, HPA, manual kubectl) can manage the same resource without conflict.

```yaml
metadata:
  managedFields:
  - manager: argocd-controller
    fieldsV1:
      f:spec:
        f:template: {}
  - manager: horizontal-pod-autoscaler
    fieldsV1:
      f:spec:
        f:replicas: {}
```

ArgoCD owns template (image, env vars). HPA owns replicas (scaling). No conflict.

---

## Core Principles

### Control Loops Over Scripts
Continuous reconciliation beats one-shot execution. System self-heals.

### Declarative Over Imperative
State in Git, not commands in logs. Idempotent applies enable automation.

### Level-Triggered Over Edge-Triggered
React to state mismatch, not events. Resilient to missed events.

### GitOps Over Push-Based CD
Git as source of truth. Pull-based sync. Better security and auditability.

---

## Key Mental Models

### Kubernetes CD is Control Theory Applied
- **Thermostat analogy**: Observe temperature, compare to desired, heat/cool accordingly
- **Kubernetes**: Observe cluster state, compare to manifest, create/delete resources
- Continuous feedback loop ensures convergence

### Desired State vs. Actual State
- **Desired**: What you declare in manifests (Git)
- **Actual**: What's running in cluster (etcd)
- **Controller's job**: Make Actual → Desired

### Field-Level Ownership Enables Multi-Actor
- Each manager (ArgoCD, HPA, kubectl) owns specific fields
- No annotation hacks, structured merge diff
- Conflicts are explicit and resolvable

---

## Interview Readiness

When discussing Kubernetes CD, demonstrate depth:

**Question**: "How does Kubernetes handle deployments?"

**Shallow**: "It does rolling updates automatically."

**Deep**: "Kubernetes uses level-triggered control loops. The Deployment controller continuously reconciles desired state (from the manifest) with actual state (cluster resources). It creates a new ReplicaSet for the new version, scales it up while scaling down the old ReplicaSet, respecting maxSurge and maxUnavailable constraints. If a pod fails readiness probes, the rollout pauses but doesn't fail - old version keeps serving traffic. This eventual consistency model, combined with health checks as preconditions for traffic, enables zero-downtime deployments with automatic rollback on failure."

**Question**: "GitOps vs. traditional CI/CD?"

**Deep**: "GitOps is declarative and pull-based. Git contains the desired state, a controller in the cluster continuously syncs cluster state to Git. This differs from imperative CI/CD where CI executes kubectl apply commands. GitOps provides better security (no CI cluster credentials), auditability (Git history is deployment history), and self-healing (controller re-applies if manual changes cause drift). The level-triggered reconciliation loop is more resilient than event-driven CI/CD pipelines."

---

## Quick Reference

### Deployment Strategy Decision Matrix

| Strategy | Complexity | Blast Radius | Rollback Speed | Traffic Control |
|----------|------------|--------------|----------------|-----------------|
| **Native Rolling** | Low | Medium (gradual) | Medium | Replica-based |
| **Flagger Canary** | Medium | Low (progressive) | Fast (auto) | Mesh or replica |
| **Argo Rollouts** | Medium | Low (progressive) | Fast (auto) | Mesh integration |
| **Service Mesh** | High | Variable | Instant | Fine-grained % |

### Health Check Types

```yaml
livenessProbe:   # Is container alive? Restart if fails
readinessProbe:  # Ready for traffic? Remove from Service if fails
startupProbe:    # Still starting? Delay liveness checks
```

### GitOps Tools Comparison

| Feature | ArgoCD | Flux |
|---------|--------|------|
| **Architecture** | Single controller | Toolkit (modular) |
| **UI** | Built-in dashboard | External (Weave GitOps) |
| **Config** | Application CRDs | GitRepository + Kustomization |
| **Multi-tenancy** | Projects, RBAC | Namespace isolation |
| **Complexity** | Batteries-included | Composable, flexible |

---

Kubernetes didn't just provide better deployment tooling. It provided a fundamentally different operational model: continuous reconciliation toward declared desired state via level-triggered control loops. This architectural shift enables self-healing, eventual consistency, and GitOps workflows that traditional imperative deployment systems cannot match.
