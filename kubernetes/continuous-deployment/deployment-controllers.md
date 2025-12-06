# Deployment Controllers: Orchestrating Updates

## The Hierarchy: Three Levels

Kubernetes doesn't deploy pods directly. There's a three-level hierarchy:

```
Deployment (high-level intent)
    ↓ manages
ReplicaSet (versioned pod template)
    ↓ manages
Pods (actual containers)
```

**Why three levels?** Each serves a distinct purpose in the deployment lifecycle.

---

## Level 1: Deployment

**What it is**: Declarative updates for Pods and ReplicaSets.

**Manifest**:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  replicas: 3
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
```

**What Deployment controller does**:
- Watches Deployment objects
- Creates/updates ReplicaSets
- Manages rolling updates
- Tracks rollout history
- Handles rollback

**Deployment is about**: Version management and update strategy.

---

## Level 2: ReplicaSet

**What it is**: Ensures a specified number of pod replicas are running.

**How it's created**:

When you create a Deployment, the Deployment controller creates a ReplicaSet automatically:

```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: myapp-5d4f7c6b9f  # Generated name: deployment-name + hash
  ownerReferences:
  - apiVersion: apps/v1
    kind: Deployment
    name: myapp
    uid: a1b2c3d4-e5f6-7890-abcd-ef1234567890
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
      pod-template-hash: 5d4f7c6b9f  # Hash of pod template
  template:
    metadata:
      labels:
        app: myapp
        pod-template-hash: 5d4f7c6b9f
    spec:
      containers:
      - name: myapp
        image: myapp:1.0.0
```

**What ReplicaSet controller does**:
- Watches ReplicaSet objects
- Creates/deletes Pods to match `spec.replicas`
- Replaces failed Pods
- Monitors Pod health

**ReplicaSet is about**: Maintaining desired replica count for a specific pod template.

---

## Level 3: Pods

**What they are**: The actual running containers.

**How they're created**:

ReplicaSet controller creates Pods:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp-5d4f7c6b9f-x7k9m  # replicaset-name + random suffix
  labels:
    app: myapp
    pod-template-hash: 5d4f7c6b9f
  ownerReferences:
  - apiVersion: apps/v1
    kind: ReplicaSet
    name: myapp-5d4f7c6b9f
spec:
  containers:
  - name: myapp
    image: myapp:1.0.0
```

**What happens next**:
- Scheduler assigns Pod to a Node
- Kubelet on that Node creates the container
- Container runtime (containerd) pulls image and starts container

---

## Why Three Levels? The Rolling Update Story

The hierarchy enables **rolling updates** without downtime.

### Scenario: Update image from v1.0.0 to v1.1.0

**Traditional approach (two levels: Deployment → Pods)**:

```
1. Deployment controller deletes old Pods
2. Creates new Pods with new image
3. Problem: Downtime between delete and create
```

**Kubernetes approach (three levels)**:

```
1. Deployment controller creates NEW ReplicaSet (v1.1.0)
2. Scales NEW ReplicaSet up (0 → 1 → 2 → 3)
3. Simultaneously scales OLD ReplicaSet down (3 → 2 → 1 → 0)
4. Old and new Pods coexist during rollout
5. Zero downtime
```

**The key insight**: ReplicaSets represent **versions**. Multiple ReplicaSets can exist simultaneously, allowing gradual traffic shift from old to new.

---

## Rolling Update Mechanics

### Step-by-Step Example

**Initial state**:

```
Deployment: myapp
  replicas: 3
  image: myapp:1.0.0

ReplicaSet: myapp-abcd1234 (old)
  replicas: 3
  image: myapp:1.0.0

Pods:
  myapp-abcd1234-aaa (v1.0.0)
  myapp-abcd1234-bbb (v1.0.0)
  myapp-abcd1234-ccc (v1.0.0)
```

**User updates Deployment**:

```bash
kubectl set image deployment/myapp myapp=myapp:1.1.0
```

**Deployment controller detects change**:

```
Old pod template hash: abcd1234
New pod template hash: xyz5678 (different because image changed)
Action: Create new ReplicaSet with hash xyz5678
```

**Step 1: Create new ReplicaSet**:

```
Deployment: myapp
  replicas: 3
  image: myapp:1.1.0  (updated)

ReplicaSet: myapp-abcd1234 (old)
  replicas: 3

ReplicaSet: myapp-xyz5678 (new) ← Created
  replicas: 0  (starts at 0)
```

**Step 2: Scale up new, scale down old (iteration 1)**:

```
maxSurge: 1        # Can have 1 extra pod (3 + 1 = 4 total)
maxUnavailable: 1  # Can have 1 unavailable pod (3 - 1 = 2 minimum)
```

```
ReplicaSet: myapp-abcd1234 (old)
  replicas: 3 → 2  (scaled down by 1)

ReplicaSet: myapp-xyz5678 (new)
  replicas: 0 → 1  (scaled up by 1)

Pods:
  myapp-abcd1234-aaa (v1.0.0) ← Terminating
  myapp-abcd1234-bbb (v1.0.0)
  myapp-abcd1234-ccc (v1.0.0)
  myapp-xyz5678-xxx (v1.1.0) ← Creating
```

**Step 3: Wait for new Pod to become Ready**:

```
Deployment controller monitors new Pod's readiness probe
Pod myapp-xyz5678-xxx: Not ready (still starting)
Action: Wait... don't proceed yet
```

**Step 4: New Pod becomes Ready**:

```
Pod myapp-xyz5678-xxx: Ready ✓
Action: Proceed to next iteration
```

**Step 5: Scale up new, scale down old (iteration 2)**:

```
ReplicaSet: myapp-abcd1234 (old)
  replicas: 2 → 1

ReplicaSet: myapp-xyz5678 (new)
  replicas: 1 → 2

Pods:
  myapp-abcd1234-bbb (v1.0.0) ← Terminating
  myapp-abcd1234-ccc (v1.0.0)
  myapp-xyz5678-xxx (v1.1.0)
  myapp-xyz5678-yyy (v1.1.0) ← Creating
```

**Step 6: Wait for readiness again**:

```
Pod myapp-xyz5678-yyy: Ready ✓
Action: Proceed
```

**Step 7: Final iteration**:

```
ReplicaSet: myapp-abcd1234 (old)
  replicas: 1 → 0  (fully scaled down)

ReplicaSet: myapp-xyz5678 (new)
  replicas: 2 → 3  (fully scaled up)

Pods:
  myapp-abcd1234-ccc (v1.0.0) ← Terminating (last old pod)
  myapp-xyz5678-xxx (v1.1.0)
  myapp-xyz5678-yyy (v1.1.0)
  myapp-xyz5678-zzz (v1.1.0) ← Creating
```

**Final state**:

```
ReplicaSet: myapp-abcd1234 (old)
  replicas: 0  (kept for rollback)

ReplicaSet: myapp-xyz5678 (new)
  replicas: 3

Pods:
  myapp-xyz5678-xxx (v1.1.0)
  myapp-xyz5678-yyy (v1.1.0)
  myapp-xyz5678-zzz (v1.1.0)
```

**Rollout complete**. Old ReplicaSet remains (scaled to 0) for rollback capability.

---

## MaxSurge and MaxUnavailable: The Math

These parameters control the rolling update speed and resource usage.

```yaml
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxSurge: 1
    maxUnavailable: 1
```

### MaxSurge

**Definition**: Maximum number of Pods that can be created above desired replicas.

**Example** (desired replicas: 10):

```
maxSurge: 25% → 10 * 0.25 = 2.5 → rounds up to 3
Maximum total pods during rollout: 10 + 3 = 13
```

**Higher maxSurge**:
- Faster rollout (more new pods created simultaneously)
- More resource usage (extra pods running)
- Use case: Plenty of cluster capacity, want speed

**Lower maxSurge**:
- Slower rollout
- Less resource usage
- Use case: Tight on resources, can afford slower rollout

**maxSurge: 0**:
- No extra pods created
- Must scale down old first, then scale up new
- Slower, but guaranteed no over-capacity

### MaxUnavailable

**Definition**: Maximum number of Pods that can be unavailable during update.

**Example** (desired replicas: 10):

```
maxUnavailable: 10% → 10 * 0.1 = 1
Minimum available pods during rollout: 10 - 1 = 9
```

**Higher maxUnavailable**:
- Faster rollout (can terminate more old pods at once)
- More risk (less capacity during rollout)
- Use case: Can tolerate temporary capacity reduction

**Lower maxUnavailable**:
- Slower rollout
- Safer (maintains more capacity)
- Use case: Can't afford capacity reduction

**maxUnavailable: 0**:
- Must scale up new first, then scale down old
- Guarantees full capacity always
- Requires resources for extra pods (maxSurge must be > 0)

### Combinations

```
maxSurge: 1, maxUnavailable: 1 (default for many workloads)
- Can have 11 pods (10 + 1 surge)
- Can have 9 available (10 - 1 unavailable)
- Balanced speed and safety

maxSurge: 0, maxUnavailable: 1
- No extra pods
- Can have 9 available
- Resource-constrained, slow rollout

maxSurge: 3, maxUnavailable: 0
- Can have 13 pods (10 + 3)
- Always 10 available (no unavailable allowed)
- Fast, safe, but requires extra capacity

maxSurge: 50%, maxUnavailable: 50%
- Can have 15 pods (10 + 5)
- Can have 5 available (10 - 5)
- Very fast, but risky (half capacity at times)
```

### Calculating Rollout Speed

**Formula**:

```
Iterations = ceil(replicas / min(maxSurge_absolute, replicas - (replicas - maxUnavailable_absolute)))

Time = Iterations × (Pod startup time + readiness probe delay)
```

**Example** (10 replicas, maxSurge: 2, maxUnavailable: 1, pod startup: 30s):

```
Each iteration:
- Scale down: 1 (limited by maxUnavailable)
- Scale up: 2 (limited by maxSurge)
- Net change: 1 pod per iteration

Iterations: 10 / 1 = 10
Time: 10 × 30s = 5 minutes (minimum)
```

---

## Deployment Strategies

### RollingUpdate (Default)

Gradual replacement, as described above.

```yaml
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxSurge: 1
    maxUnavailable: 1
```

**Use case**: Most deployments (zero downtime)

### Recreate

Delete all old Pods, then create new Pods.

```yaml
strategy:
  type: Recreate
```

**Process**:
1. Scale old ReplicaSet to 0 (all pods terminated)
2. Wait for all old pods to fully terminate
3. Create new ReplicaSet with desired replicas
4. New pods start

**Use case**:
- Application can't run multiple versions simultaneously (e.g., database schema incompatibility)
- Shared resources (file locks, singleton services)
- Cost-sensitive (can't afford extra pods during rollout)

**Trade-off**: Downtime during update (old pods deleted before new pods ready)

---

## StatefulSets: Ordered Deployments

For stateful applications (databases, message queues), use **StatefulSets** instead of Deployments.

### Key Differences from Deployments

**1. Stable Identity**

Pods have predictable names:

```
# Deployment pods (random suffix)
myapp-5d4f7c6b9f-x7k9m
myapp-5d4f7c6b9f-k3p2n

# StatefulSet pods (ordinal index)
myapp-0
myapp-1
myapp-2
```

**2. Ordered Deployment and Scaling**

```
Scaling up (0 → 3):
  Create myapp-0, wait until Ready
  Create myapp-1, wait until Ready
  Create myapp-2, wait until Ready

Scaling down (3 → 1):
  Delete myapp-2, wait until fully terminated
  Delete myapp-1, wait until fully terminated
  myapp-0 remains
```

**3. Stable Storage**

Each pod gets its own PersistentVolumeClaim:

```
myapp-0 → pvc-myapp-0
myapp-1 → pvc-myapp-1
myapp-2 → pvc-myapp-2

If myapp-1 is rescheduled to another node, it reattaches to pvc-myapp-1
```

**4. Rolling Update (Ordered)**

```
updateStrategy:
  type: RollingUpdate
  rollingUpdate:
    partition: 0  # Update pods with ordinal >= partition

Update from v1.0.0 to v1.1.0:
  Delete myapp-2 (highest ordinal), wait, create new myapp-2 (v1.1.0)
  Delete myapp-1, wait, create new myapp-1 (v1.1.0)
  Delete myapp-0, wait, create new myapp-0 (v1.1.0)
```

**Use StatefulSets for**:
- Databases (PostgreSQL, MySQL, MongoDB)
- Message queues (Kafka, RabbitMQ)
- Distributed systems needing stable network identity (Zookeeper, etcd)

---

## DaemonSets: One Pod Per Node

**What it is**: Ensures one pod runs on every node (or subset of nodes).

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: node-exporter
spec:
  selector:
    matchLabels:
      app: node-exporter
  template:
    spec:
      containers:
      - name: node-exporter
        image: prom/node-exporter:v1.0.0
```

**Behavior**:
- New node added → DaemonSet controller creates pod on it
- Node removed → Pod automatically deleted
- Update strategy: RollingUpdate (one node at a time) or OnDelete (manual)

**Use cases**:
- Log collectors (Fluentd, Filebeat)
- Monitoring agents (node-exporter, datadog-agent)
- Network plugins (CNI daemons)
- Storage daemons (Ceph, GlusterFS)

---

## ReplicaSet Revision History

Deployments keep old ReplicaSets for rollback.

```bash
kubectl get replicasets

NAME                  DESIRED   CURRENT   READY   AGE
myapp-5d4f7c6b9f      0         0         0       2h    (v1.0.0 - old)
myapp-xyz5678abc      3         3         3       30m   (v1.1.0 - current)
```

**Rollout history**:

```bash
kubectl rollout history deployment/myapp

REVISION  CHANGE-CAUSE
1         kubectl set image deployment/myapp myapp=myapp:1.0.0
2         kubectl set image deployment/myapp myapp=myapp:1.1.0
```

**Rollback**:

```bash
kubectl rollout undo deployment/myapp  # Rollback to previous revision
kubectl rollout undo deployment/myapp --to-revision=1  # Rollback to specific revision
```

**How rollback works**:
1. Find old ReplicaSet for target revision
2. Scale old ReplicaSet up (0 → 3)
3. Scale current ReplicaSet down (3 → 0)
4. Same rolling update process, just in reverse

**Revision limit**:

```yaml
spec:
  revisionHistoryLimit: 10  # Keep last 10 ReplicaSets (default)
```

Old ReplicaSets beyond this limit are garbage collected.

---

## Deployment Conditions

Deployments report their status via **conditions**.

```bash
kubectl describe deployment myapp

Conditions:
  Type           Status  Reason
  ----           ------  ------
  Available      True    MinimumReplicasAvailable
  Progressing    True    NewReplicaSetAvailable
```

### Condition Types

**Available**:
- True: Minimum replicas are available (considers minReadySeconds)
- False: Not enough replicas available

**Progressing**:
- True: Deployment is progressing (creating new pods, scaling)
- False: Deployment stuck (e.g., image pull error, insufficient resources)

**ReplicaFailure**:
- True: Failed to create/delete pods (quota exceeded, node unavailable)

### ProgressDeadlineSeconds

```yaml
spec:
  progressDeadlineSeconds: 600  # 10 minutes
```

If Deployment doesn't progress within this time, `Progressing` condition becomes False (reason: ProgressDeadlineExceeded).

**Use case**: Automated rollback - if deployment stuck > 10 minutes, trigger rollback.

---

## Interview Deep Dive

**Question**: "Why does Kubernetes have Deployment, ReplicaSet, and Pod instead of just Deployment and Pod?"

**Shallow answer**: "To manage replicas."

**Deep answer**:
"The three-level hierarchy enables zero-downtime rolling updates. When you update a Deployment's pod template, the Deployment controller creates a new ReplicaSet with the new template. It then gradually scales up the new ReplicaSet while scaling down the old one, respecting maxSurge and maxUnavailable constraints.

Having ReplicaSets as an intermediate layer allows multiple versions of pods to coexist during rollout - the old ReplicaSet manages v1.0.0 pods, the new ReplicaSet manages v1.1.0 pods. Traffic shifts gradually from old to new.

Additionally, old ReplicaSets are kept (scaled to 0) for rollback. If the new version has issues, kubectl rollout undo simply reverses the scaling - scale old ReplicaSet back up, scale new ReplicaSet back down. No redeployment needed.

Without ReplicaSets, you'd have to delete all old pods before creating new ones (downtime) or implement complex custom logic to manage multiple pod versions (reinventing ReplicaSets)."

**Question**: "How do you ensure zero downtime during deployments?"

**Deep answer**:
"Zero downtime requires:

1. **Readiness probes**: New pods must pass readiness before receiving traffic. Deployment controller waits for readiness before proceeding with rollout.

2. **maxUnavailable: 0**: Ensures minimum desired replicas are always available. New pods must be ready before old pods terminate.

3. **Graceful shutdown**: Old pods receive SIGTERM, have terminationGracePeriodSeconds to finish requests, then SIGKILL. Service stops sending new traffic immediately (removed from endpoints).

4. **Sufficient maxSurge**: Allows creating new pods before terminating old pods. Prevents capacity reduction during rollout.

5. **Rolling update strategy**: Gradual replacement, not all-at-once Recreate.

The interplay between readiness probes, Service endpoints, and Deployment controller ensures traffic only goes to ready pods, and old pods finish their requests before termination."

---

## Key Takeaways

1. **Three-level hierarchy**: Deployment manages ReplicaSets, ReplicaSets manage Pods - enables rolling updates
2. **Rolling updates**: Create new ReplicaSet, scale up new while scaling down old
3. **maxSurge/maxUnavailable**: Control rollout speed, resource usage, and safety
4. **StatefulSets**: Ordered deployments with stable identity and storage for stateful apps
5. **DaemonSets**: One pod per node for system-level services
6. **Revision history**: Old ReplicaSets kept for instant rollback
7. **Deployment conditions**: Available, Progressing, ReplicaFailure signal deployment health

Deployment controllers abstract the complexity of safe, zero-downtime updates. Understanding the mechanics - the hierarchy, the reconciliation logic, the math behind maxSurge/maxUnavailable - is essential for debugging deployments and designing robust update strategies.
