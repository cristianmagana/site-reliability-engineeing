# Declarative State Model: Desired vs. Actual

## The Core Abstraction

Kubernetes separates two fundamental concepts that traditional deployment systems conflate:

```
Desired State:  What you want the system to look like
Actual State:   What the system currently looks like

Controller's Job:  Make Actual → Desired
```

This separation is the foundation of everything Kubernetes does.

---

## Imperative vs. Declarative: More Than Just Syntax

This isn't about `kubectl run` vs. `kubectl apply`. It's about fundamentally different operational models.

### Imperative: Commands That Modify State

You tell the system **how** to change state:

```bash
# Imperative commands
kubectl run myapp --image=myapp:1.0 --replicas=3
kubectl scale deployment myapp --replicas=5
kubectl set image deployment/myapp myapp=myapp:1.1
kubectl delete pod myapp-abc123
```

**Each command is an action**:
- "Run 3 instances of myapp:1.0"
- "Scale to 5 instances"
- "Change image to myapp:1.1"
- "Delete this specific pod"

**State is implicit**. To know current state, you must:
1. Track all commands executed
2. Query the system: `kubectl get deployment myapp -o yaml`

**History is in the commands**:
```
10:00 - kubectl run myapp --image=myapp:1.0 --replicas=3
10:05 - kubectl scale deployment myapp --replicas=5
10:10 - kubectl set image deployment/myapp myapp=myapp:1.1
```

What's running now? You must mentally apply these commands in sequence.

### Declarative: State That Gets Applied

You tell the system **what** you want the final state to be:

```yaml
# deployment.yaml - Declarative manifest
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  replicas: 5
  template:
    spec:
      containers:
      - name: myapp
        image: myapp:1.1
```

```bash
kubectl apply -f deployment.yaml
```

**The manifest is the desired state**:
- "I want 5 replicas"
- "Running image myapp:1.1"
- "With these exact specifications"

**Current state is explicit**: The YAML file declares it.

**History is in the manifests**:
```
commit abc123 (10:00)
  replicas: 3
  image: myapp:1.0

commit def456 (10:05)
  replicas: 5
  image: myapp:1.0

commit ghi789 (10:10)
  replicas: 5
  image: myapp:1.1
```

What's running now? Look at the latest manifest in Git.

---

## Why Declarative Wins for CD

### Property 1: Idempotency

**Imperative**:
```bash
kubectl run myapp --image=myapp:1.0 --replicas=3

# Run once:  Creates deployment with 3 replicas ✓
# Run twice: Error - deployment already exists ✗
```

Not idempotent. Running the same command twice gives different results.

**Declarative**:
```bash
kubectl apply -f deployment.yaml

# Run once:  Creates deployment ✓
# Run twice: No change (already matches) ✓
# Run 100 times: Still no change ✓
```

Idempotent. Applying the same manifest multiple times converges to the same state.

**Why this matters for CD**: Automation can safely retry. Network timeout? Apply again. Unsure if deploy succeeded? Apply again. Won't create duplicates.

### Property 2: Drift Detection

**Imperative**: No built-in concept of drift

```bash
# You executed this command yesterday
kubectl scale deployment myapp --replicas=5

# Today, someone manually changed it
kubectl scale deployment myapp --replicas=3

# How do you know it drifted?
# You must remember what you set it to yesterday
```

**Declarative**: Desired state is in Git

```yaml
# deployment.yaml in Git
spec:
  replicas: 5  # This is the source of truth
```

```bash
# Cluster has replicas: 3 (someone changed it manually)
# GitOps controller compares Git (desired) vs. Cluster (actual)
# Detects drift: 5 != 3
# Auto-corrects: Updates cluster to match Git
```

**Drift detection formula**:
```
Drift = Desired State (Git) - Actual State (Cluster)

If Drift != 0:
  Alert and/or auto-correct
```

### Property 3: Auditability

**Imperative**: History is in logs

```
# CI/CD logs
2024-01-15 10:05 - Executed: kubectl scale deployment myapp --replicas=5
2024-01-15 10:10 - Executed: kubectl set image deployment/myapp myapp=myapp:1.1
```

**To answer "what was deployed on Jan 15?"**:
- Search through logs
- Reconstruct state by applying commands mentally
- Hope logs are complete

**Declarative**: History is in Git

```bash
git log -- deployment.yaml

commit ghi789
Date: 2024-01-15 10:10
  Update image to myapp:1.1

commit def456
Date: 2024-01-15 10:05
  Scale to 5 replicas

# To see exact state on Jan 15
git show ghi789:deployment.yaml
```

**To answer "what was deployed on Jan 15?"**:
- `git show <commit>:deployment.yaml`
- Exact manifest used

Git is a time machine for cluster state.

### Property 4: Review Process

**Imperative**: Review commands

```
Pull request:
  Execute: kubectl scale deployment myapp --replicas=10
```

Reviewer sees the **action**, not the **result**.

**Questions**:
- What's the current replica count? (need to look at cluster)
- Is this change safe? (depends on current state)
- What's the full desired state after this? (mentally apply command)

**Declarative**: Review manifests

```
Pull request:
  deployment.yaml
  - replicas: 5
  + replicas: 10
```

Reviewer sees the **diff** of desired state.

**Questions**:
- What's changing? (5 → 10, explicit in diff)
- What's the final state? (10 replicas, right there)
- Is this safe? (Can see full context in manifest)

Much easier to review.

---

## Client-Side Apply: The Annotation Hack

Before server-side apply (Kubernetes 1.22+), `kubectl apply` used a clever but fragile mechanism.

### The Three-Way Merge Problem

When you `kubectl apply`, Kubernetes needs to merge:

1. **Last applied configuration**: What you applied last time
2. **Current configuration**: What's in the cluster now (might have been modified)
3. **New configuration**: What you're applying now

```
Last Applied:  replicas: 3, image: myapp:1.0
Current:       replicas: 5, image: myapp:1.0  (someone scaled it)
New:           replicas: 3, image: myapp:1.1

Question: What should final state be?
  replicas: 3 or 5?  (New says 3, but user scaled to 5)
  image: myapp:1.1    (Clear: New overrides)
```

### The Annotation Solution

Client-side apply stores the last applied config in an annotation:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
  annotations:
    kubectl.kubernetes.io/last-applied-configuration: |
      {
        "apiVersion": "apps/v1",
        "kind": "Deployment",
        "metadata": {"name": "myapp"},
        "spec": {
          "replicas": 3,
          "template": {
            "spec": {
              "containers": [{"name": "myapp", "image": "myapp:1.0"}]
            }
          }
        }
      }
spec:
  replicas: 5  # Current value (manually scaled)
  template:
    spec:
      containers:
      - name: myapp
        image: myapp:1.0
```

**Three-way merge logic**:

```python
def three_way_merge(last_applied, current, new):
    result = {}

    for field in new:
        if field not in last_applied:
            # New field added in this apply
            result[field] = new[field]
        elif new[field] != last_applied[field]:
            # Field was changed by user (in new config)
            result[field] = new[field]
        elif current[field] != last_applied[field]:
            # Field was modified externally (keep external modification)
            result[field] = current[field]
        else:
            # No changes, use new value (same as last)
            result[field] = new[field]

    return result
```

**Example**:

```
Field: replicas
  last_applied: 3
  current: 5  (scaled externally)
  new: 3

Check: new[replicas] (3) == last_applied[replicas] (3)  → True
       current[replicas] (5) != last_applied[replicas] (3) → True
       → Field modified externally, keep current value
Result: replicas: 5
```

User scaled to 5 manually. Next `kubectl apply` won't override it back to 3 (since new manifest still says 3, same as last applied).

### Problems with Client-Side Apply

**Problem 1: Annotation Size Limit**

Annotations have a 256KB limit. Large manifests can exceed this:

```yaml
# deployment.yaml (200KB manifest)
# Last-applied annotation: 200KB
# Total object size: 400KB+
# Risk: Hitting etcd size limits
```

**Problem 2: Multi-Actor Conflicts**

```
Actor A (kubectl): Manages replicas and image
Actor B (HPA): Manages replicas
Actor C (Manual admin): Changes both

Who owns replicas?
```

Client-side apply doesn't track field ownership. Annotation stores entire manifest, not individual field ownership.

**Problem 3: Incomplete Deletes**

```yaml
# First apply
spec:
  replicas: 3
  strategy:
    type: RollingUpdate

# Second apply (removed strategy)
spec:
  replicas: 5
  # strategy removed from manifest
```

Client-side apply doesn't always delete removed fields. They might remain in the cluster.

---

## Server-Side Apply: Field-Level Ownership

Kubernetes 1.22+ introduced server-side apply (SSA), solving client-side apply's limitations.

### The Core Concept: Field Managers

Each actor (kubectl, HPA, ArgoCD) is a **field manager**. API server tracks which manager owns which fields.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
  managedFields:
  - manager: kubectl
    operation: Apply
    apiVersion: apps/v1
    time: "2024-01-15T10:00:00Z"
    fieldsType: FieldsV1
    fieldsV1:
      f:spec:
        f:template:
          f:spec:
            f:containers:
              k:{"name":"myapp"}:
                f:image: {}

  - manager: horizontal-pod-autoscaler
    operation: Update
    apiVersion: apps/v1
    time: "2024-01-15T10:05:00Z"
    fieldsType: FieldsV1
    fieldsV1:
      f:spec:
        f:replicas: {}
```

**Field ownership**:
- `kubectl` owns: `spec.template.spec.containers[0].image`
- `horizontal-pod-autoscaler` owns: `spec.replicas`

### How It Works

**Apply with field manager**:

```bash
kubectl apply -f deployment.yaml --server-side --field-manager=kubectl
```

**What happens**:

1. Kubernetes parses the manifest
2. For each field in the manifest:
   - Check: Is this field already owned by another manager?
   - If no: Assign ownership to `kubectl`
   - If yes: Check for conflict (value different?)
3. If conflict: Fail with error or force ownership transfer
4. Update object and record field ownership in `managedFields`

**Example: No Conflict**

```yaml
# kubectl applies
spec:
  replicas: 3
  template:
    spec:
      containers:
      - image: myapp:1.0

# kubectl owns: replicas, image
```

```yaml
# HPA updates (different field)
spec:
  replicas: 5  # HPA owns this now

# kubectl owns: image
# HPA owns: replicas
```

Next time kubectl applies with `replicas: 3`, it sees HPA owns that field. Conflict!

### Conflict Resolution

**Scenario**: kubectl wants `replicas: 3`, HPA set `replicas: 5`

**Option 1: Force ownership transfer**

```bash
kubectl apply -f deployment.yaml --server-side --force-conflicts
```

kubectl becomes owner of `replicas`, overrides HPA's value to 3.

**Use case**: Emergency override, you know what you're doing.

**Option 2: Exclude field from management**

```yaml
# Don't specify replicas in manifest if HPA manages it
spec:
  # replicas: 3  ← Remove this line
  template:
    spec:
      containers:
      - image: myapp:1.1
```

kubectl only manages `image`, HPA continues managing `replicas`. No conflict.

**Use case**: Division of responsibility (GitOps manages image, HPA manages replicas).

### Structured Merge Diff

Server-side apply uses **structured merge diff**, not text diff.

**Client-side apply** (text-based):

```yaml
# Last applied
spec:
  containers:
  - name: app
    image: myapp:1.0
  - name: sidecar
    image: sidecar:1.0

# New apply (removed sidecar)
spec:
  containers:
  - name: app
    image: myapp:1.1

# Result: Both containers remain (deletion not detected reliably)
```

**Server-side apply** (structured):

```yaml
# kubectl owns: containers[app], containers[sidecar]

# New apply
spec:
  containers:
  - name: app
    image: myapp:1.1
  # sidecar not mentioned

# Result: kubectl removes containers[sidecar] (it owned it, now it's gone)
```

Server-side apply understands the structure. If you owned a field and stop specifying it, it gets deleted.

---

## Multi-Actor Scenarios

Real-world Kubernetes has multiple actors modifying resources.

### Scenario 1: GitOps + HPA

**Setup**:

```yaml
# In Git (managed by ArgoCD)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  replicas: 3  # Initial replica count
  template:
    spec:
      containers:
      - name: myapp
        image: myapp:1.0

---
# HPA (also in Git)
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: myapp-hpa
spec:
  scaleTargetRef:
    name: myapp
  minReplicas: 3
  maxReplicas: 10
```

**Timeline**:

```
10:00 - ArgoCD syncs Deployment: replicas: 3
        ArgoCD owns: spec.replicas, spec.template

10:05 - Traffic increases, CPU at 90%
        HPA updates Deployment: replicas: 7
        HPA owns: spec.replicas (transferred ownership from ArgoCD)

10:10 - ArgoCD syncs again (periodic sync)
        ArgoCD sees: I want replicas: 3, but HPA owns it
        ArgoCD detects drift but doesn't override (HPA is manager)
```

**With server-side apply**:

```yaml
# ArgoCD applies with --field-manager=argocd
# But excludes spec.replicas from management

# ArgoCD's server-side apply manifest
spec:
  # replicas: 3  ← Not specified (HPA manages this)
  template:
    spec:
      containers:
      - name: myapp
        image: myapp:1.0
```

ArgoCD manages everything except `replicas`. HPA manages only `replicas`. No conflict.

### Scenario 2: GitOps + Manual kubectl

**Setup**:

```yaml
# Git (ArgoCD syncs every 3 minutes)
spec:
  replicas: 5
  template:
    spec:
      containers:
      - image: myapp:1.0
```

**Timeline**:

```
10:00 - ArgoCD syncs: replicas: 5, image: myapp:1.0
        ArgoCD owns both fields

10:01 - Admin manually debugs: kubectl set image deployment/myapp myapp=myapp:1.0-debug
        kubectl owns: spec.template.spec.containers[0].image
        Conflicts with ArgoCD ownership

10:03 - ArgoCD syncs again
        Sees: ArgoCD wants myapp:1.0, kubectl set myapp:1.0-debug
        Conflict! ArgoCD wins (it's authoritative in this setup)
        Overwrites image back to myapp:1.0
```

**This is desired behavior** in GitOps: Git is source of truth. Manual changes get reverted.

**To prevent revert** (temporary debug):

```bash
# Tell ArgoCD to ignore this deployment temporarily
kubectl annotate deployment myapp argocd.argoproj.io/sync-options=IgnoreDifferences
```

---

## Idempotency Guarantees

Declarative model provides strong idempotency guarantees.

### Apply Operations

```bash
kubectl apply -f deployment.yaml --server-side

# First apply
Result: Deployment created with specified fields

# Second apply (no changes to file)
Result: No changes to cluster (already matches)

# Third apply (still no changes)
Result: Still no changes

# Fourth apply (changed image in file)
Result: Only image field updated
```

**Guarantee**: Applying the same manifest N times produces the same result as applying it once.

### Delete Operations

```bash
kubectl delete -f deployment.yaml

# First delete
Result: Deployment deleted

# Second delete (already deleted)
Result: Error - not found

# Idempotent delete
kubectl delete -f deployment.yaml --ignore-not-found

# Second delete (with flag)
Result: Success (no-op if already deleted)
```

**Guarantee**: Deleting N times has the same effect as deleting once.

---

## GitOps Implications

Server-side apply is fundamental to modern GitOps.

### Git as Source of Truth

```
┌──────────────┐
│  Git Repo    │  ← Single source of truth
│  (manifests) │
└──────┬───────┘
       │
       │ GitOps controller watches
       ↓
┌──────────────┐
│  Cluster     │  ← Actual state
│  (etcd)      │
└──────────────┘

Controller's job: Make Cluster match Git
```

**Without server-side apply**:
- Hard to track what GitOps controller manages vs. other actors
- Manual changes might persist or might get overwritten unpredictably
- Drift detection is fuzzy

**With server-side apply**:
- GitOps controller is a field manager
- Clearly defined ownership of each field
- Conflicts are explicit (can be configured to auto-resolve)
- Drift detection is precise

### ArgoCD Server-Side Apply

ArgoCD (starting v2.5) uses server-side apply by default:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: myapp
spec:
  syncPolicy:
    syncOptions:
    - ServerSideApply=true
```

**Benefits**:

1. **Handles large manifests**: No annotation size limit
2. **Better CRD support**: Custom resources with complex schemas
3. **Precise drift detection**: Field-level tracking
4. **Respects other managers**: Won't override HPA, VPA, etc.

---

## The Desired State Contract

Declarative model establishes a contract:

```
You (developer/operator): Declare desired state in Git
Kubernetes (controllers):  Ensure actual state matches desired state
```

**You are responsible for**:
- Writing correct manifests
- Committing to Git
- Reviewing changes (pull requests)

**Kubernetes is responsible for**:
- Reading desired state
- Comparing to actual state
- Converging actual → desired
- Self-healing when drift occurs

This separation of concerns is powerful. You think about "what," Kubernetes handles "how."

---

## Interview Deep Dive

**Question**: "Explain the difference between `kubectl create` and `kubectl apply`."

**Shallow answer**: "Apply updates if exists, create fails if exists."

**Deep answer**:
"`kubectl create` is imperative - it executes an action (create this resource). If the resource exists, it fails because the action can't be performed twice. This makes it non-idempotent.

`kubectl apply` is declarative - it ensures the cluster state matches the manifest. If the resource doesn't exist, it creates it. If it exists, it updates fields to match the manifest. Applying the same manifest multiple times is idempotent.

With server-side apply (`--server-side`), Kubernetes tracks field-level ownership. This allows multiple actors (GitOps controllers, HPA, manual users) to manage different fields of the same resource without conflict. The API server maintains `managedFields` metadata tracking which manager owns which field, enabling structured merge diff and precise drift detection.

This is why declarative management is superior for continuous deployment - automation can safely retry applies, Git becomes the auditable source of truth, and multi-actor scenarios (GitOps + autoscaling) work without conflicts."

**Question**: "How do you handle configuration drift in Kubernetes?"

**Deep answer**:
"Configuration drift is when actual cluster state diverges from desired state in Git. Kubernetes' declarative model makes drift detection straightforward.

With server-side apply, each field has an owner (field manager). A GitOps controller like ArgoCD continuously compares Git (desired) to the cluster (actual). If a field that ArgoCD manages differs, it's detected as drift.

For fields managed by other actors - like HPA managing `replicas` - ArgoCD can be configured to ignore those fields (ignoreDifferences) or transfer ownership (force apply). The key is defining clear ownership boundaries.

Detection happens automatically on every sync interval (e.g., every 3 minutes). Remediation can be automatic (ArgoCD overwrites cluster with Git) or manual (ArgoCD alerts, human decides).

The level-triggered reconciliation loop ensures drift is continuously corrected. Unlike imperative systems where drift accumulates, declarative systems continuously converge to desired state."

---

## Key Takeaways

1. **Desired vs. Actual**: The fundamental separation that enables declarative management
2. **Idempotency**: Applying the same state multiple times is safe
3. **Client-side apply**: Annotation-based three-way merge (legacy)
4. **Server-side apply**: Field-level ownership tracking (modern)
5. **Multi-actor coordination**: Managers own fields, conflicts are explicit
6. **GitOps foundation**: Git as source of truth, continuous drift detection
7. **Structured merge diff**: Understanding resource structure, not text

Declarative state management isn't just a nicer API. It's the foundation that makes GitOps, self-healing, drift detection, and multi-actor coordination possible. This is why Kubernetes can provide operational guarantees that imperative systems can't.
