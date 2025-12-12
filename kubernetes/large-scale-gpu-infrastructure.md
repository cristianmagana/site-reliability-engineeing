# Large-Scale GPU Cluster Infrastructure - Senior Engineering Perspectives

## Context & Philosophy

Operating Kubernetes at scale with GPU workloads introduces constraints that don't exist in typical web service deployments. This isn't about knowing more kubectl commands—it's about understanding the failure modes, blast radius economics, and distributed systems theory that inform every decision.

These questions explore the intersection of:
- **Distributed systems theory**: How do we maintain consistency guarantees across failure domains?
- **Resource economics**: GPUs cost $10-40K each. Downtime is measured in thousands of dollars per minute.
- **Risk quantification**: Progressive delivery, blast radius containment, error budgets
- **Security models**: Secrets propagation in untrusted networks, zero-trust architectures

---

## 1. CI/CD for 1,000+ GPU Nodes Without Downtime

### The Fundamental Problem

At this scale, you're not just deploying software—you're managing a state transition across a distributed system where:
- **Partial failures are guaranteed** (MTBF of individual nodes means multiple failures during any deployment)
- **Rollback cost is asymmetric** (rolling back GPU jobs may lose days of computation)
- **Validation windows are expensive** (can't afford to "soak" on GPUs like you would on cheap web servers)
- **The cost of being wrong is measured in petaflops-hours**

Traditional CI/CD assumes stateless workloads on commodity hardware. GPU clusters violate both assumptions.

### First Principles: What Does "Safe" Actually Mean?

Safety in this context means:

1. **Preserve running work**: Long-running GPU jobs (days/weeks) cannot be interrupted
2. **Bounded blast radius**: A bad deployment should affect <1% of capacity
3. **Fast detection**: Detect failures before they cascade (minutes, not hours)
4. **Automated rollback**: Manual intervention at 2am is not a strategy
5. **Zero-downtime guarantee**: There's always capacity available for new job submissions

This maps to the **error budget** concept: if your SLO is 99.9% uptime, you have ~43 minutes/month to spend. A botched deployment that takes down 50% of capacity for 30 minutes consumes most of your quarterly budget.

### Architecture: Multi-Tier Progressive Delivery

#### Tier 1: Canary Nodes (1-2% of fleet, ~10-20 nodes)

**Purpose**: Catch obvious failures before they propagate

**Strategy**:
- Dedicated node pool with `gpu-canary=true` taint
- Deploy system components (kubelet, CNI, device plugins) here first
- Run **synthetic GPU workloads** that exercise the full stack:
  - CUDA kernel execution (catch driver incompatibilities)
  - Multi-GPU communication (NCCL tests for interconnect issues)
  - Filesystem I/O patterns (detect storage regressions)
  - Network throughput tests (catch CNI issues)

**Why synthetic workloads?** Real user jobs are too variable. You need deterministic tests with known baselines. A 10% performance regression in NCCL all-reduce latency might be invisible in a training job's metrics but catastrophic for certain workloads.

**Validation criteria** (automated, no humans in loop):
```yaml
# Example health check criteria
slo:
  cuda_kernel_latency_p99: < 1.5ms  # Baseline + 5%
  nccl_allreduce_bandwidth: > 95% of baseline
  pod_startup_time_p95: < 30s
  node_not_ready_duration: 0s
  oom_kills: 0
  gpu_utilization_synthetic: > 98%  # Should be saturated
```

**Hold time**: 2-4 hours. Why? GPU issues often manifest under thermal load. A cold GPU might pass, but fail after 30 minutes of sustained compute.

#### Tier 2: Rolling Update with Node Cordoning (not draining)

**The key insight**: Don't drain GPU nodes—cordon them and wait for natural job completion.

**Why this matters**:
- Draining interrupts jobs (violates preserve-work principle)
- GPU jobs are typically batch/fault-tolerant, but interruption still wastes days of compute
- Jobs already have completion hooks—leverage them

**Strategy**:
```bash
# Cordon prevents NEW pods, allows existing to complete
kubectl cordon <node>

# Wait for natural completion with timeout
# Most GPU jobs: hours to days
# Set appropriate per-workload timeouts
wait_until_empty --timeout=24h --node=<node>

# Only after empty: upgrade and reboot
upgrade_node_components.sh
systemctl reboot

# Uncordon when healthy
kubectl uncordon <node>
```

**Concurrency control**: Update max 5% of nodes simultaneously (~50 nodes in a 1000-node cluster)

**Why 5%?**
- Small enough blast radius if something goes catastrophically wrong
- Large enough to maintain reasonable update velocity (1000 nodes / 50 parallel = 20 waves)
- Aligns with most organizations' error budgets (losing 5% capacity for an hour is recoverable)

#### Tier 3: Workload-Aware Scheduling During Updates

**Problem**: Not all GPUs are equal. A node running a 3-week hyperparameter sweep should update later than a node running 1-hour jobs.

**Solution**: Priority-based update ordering

```yaml
# Node annotation strategy
metadata:
  annotations:
    update-priority: "low"  # Jobs here take days
    workload-class: "long-running-research"
    avg-job-duration-hours: "72"
```

**Update orchestration**:
1. Sort nodes by `update-priority` and `avg-job-duration`
2. Update short-job nodes first (velocity)
3. Update long-job nodes during maintenance windows (control)

**Rationale**: The economics of interruption scale with job duration. Interrupting a 3-week job loses more value than waiting an extra day to update that node.

### Blast Radius Containment: Failure Domain Isolation

Distribute canary and early-wave nodes across:
- **Availability zones** (catch zone-specific issues: network, power)
- **Rack diversity** (catch ToR switch firmware interactions)
- **GPU SKU diversity** (A100 vs H100 may have different driver bugs)
- **Workload diversity** (inference, training, batch rendering)

**Why?** Failures are rarely uniform. A CUDA driver bug might only affect specific GPU models under specific workload patterns. If all your canary nodes are identical A100s running training workloads, you'll miss the H100 inference bug that takes down production.

### Automated Rollback: The Hard Part

**Detection**: What signals indicate a bad deployment?

1. **System metrics** (easy):
   - Node NotReady duration
   - Kubelet/containerd restart loops
   - OOM kills, kernel panics

2. **Workload metrics** (hard but critical):
   - GPU job success rate drop (compare to baseline)
   - Job duration increase (performance regression)
   - GPU error rate increase (Xid errors, memory ECC events)
   - Power/thermal anomalies (might indicate driver issues)

**Rollback strategy**:
```yaml
# Version pinning for all components
node-image: gpu-ubuntu-22.04-v1.2.3
nvidia-driver: 535.129.03
cuda-toolkit: 12.2.2
container-runtime: containerd-1.7.8-nvidia
cni-plugin: calico-3.26.1

# Atomic rollback: revert all or none
rollback:
  target-version: v1.2.2  # Known good version
  strategy: immediate  # Don't wait for job completion
  reason: "GPU Xid errors spiked 1000x"
```

**Trade-off**: Immediate rollback interrupts jobs, but that's acceptable if the alternative is cascading failures.

### CI Pipeline Design

```yaml
# Conceptual pipeline stages
stages:
  1_build:
    - Build node images (immutable, versioned)
    - Build GPU operator bundles
    - Container runtime packages
    - Sign artifacts (supply chain security)

  2_test:
    - Unit tests (individual components)
    - Integration tests (node bringup in isolated cluster)
    - GPU workload simulation (synthetic benchmarks)
    - Soak testing (4-8 hours under load)

  3_canary:
    - Deploy to 1% of prod fleet
    - Run synthetic workloads
    - Monitor for 4 hours
    - Auto-promote or rollback based on SLOs

  4_progressive:
    - Rolling update with rate limiting
    - Workload-aware scheduling
    - Continuous health validation
    - Emergency stop on anomaly detection

  5_completion:
    - Verify all nodes updated
    - Archive deployment metadata
    - Update fleet baseline metrics
```

### Senior-Level Trade-offs Discussion

**Option 1: Blue/Green at the cluster level**
- **Pro**: True zero-downtime, instant rollback
- **Con**: Requires 2x infrastructure (unacceptable for GPU economics)
- **When to use**: Control plane updates, not node updates

**Option 2: Immutable node images vs in-place updates**
- **Pro (immutable)**: Reproducible, atomic, easy rollback
- **Con (immutable)**: Requires node replacement, longer update windows
- **Decision**: Immutable is worth it—GPU nodes should be cattle, not pets

**Option 3: Update during maintenance windows vs continuous rolling**
- **Pro (windows)**: Predictable, can drain jobs with notice
- **Con (windows)**: Slower security patching, accumulates risk
- **Decision**: Hybrid—security patches continuous, major upgrades in windows

### Connection to Business Value

**Risk quantification example**:
- 1000 nodes × $20K/node × 50% utilization × $2/GPU-hour = $1M/hour of productive capacity
- A botched deployment taking down 50% of fleet for 1 hour = $500K lost
- Investing in progressive delivery infrastructure ($200K engineering, $50K canary hardware) pays for itself if it prevents a single major incident

**Error budget impact**:
- Without progressive delivery: ~3-4 major incidents/year (based on industry data)
- With progressive delivery: <1 major incident/year
- Difference: 99.0% vs 99.9% availability = 10x improvement in reliability

---

## 2. Secrets Management Across Multi-Region GPU Clusters

### The Fundamental Problem

Secrets in distributed systems are hard because:
- **Secrets are the skeleton key**: One leaked credential can compromise the entire fleet
- **Distribution is required**: Every node/pod needs credentials to function
- **Rotation is essential**: Static secrets are compromised secrets (assume breach)
- **Auditability is non-negotiable**: "Who accessed what, when?" is a compliance requirement
- **Geography introduces latency**: Multi-region means CAP theorem trade-offs in secret delivery

GPU clusters add specific constraints:
- **Long-running jobs**: Credential rotation can't interrupt a 3-week training run
- **Multi-tenancy**: Research teams should never see each other's secrets
- **Compliance**: Many GPU workloads involve PII/PHI (healthcare ML, etc.)

### First Principles: Zero Trust Architecture

**Assumption**: The network is hostile, nodes are compromised, humans make mistakes.

**Core principles**:
1. **No ambient authority**: Nothing works by default (no default credentials)
2. **Least privilege**: Every workload gets the minimum permissions needed
3. **Short-lived credentials**: Secrets expire quickly (hours, not months)
4. **Identity-based access**: Workloads authenticate via cryptographic identity, not shared secrets
5. **Audit everything**: Every secret access is logged immutably

### Architecture: Multi-Layer Defense

#### Layer 1: Secret Storage - External Secret Backends

**Never use Kubernetes Secrets as the source of truth.** They're just a caching/distribution layer.

**Options analysis**:

| Backend | Pros | Cons | Best For |
|---------|------|------|----------|
| HashiCorp Vault | Industry standard, dynamic secrets, excellent audit | Complex ops, single point of failure if not HA | Enterprise, high compliance |
| AWS Secrets Manager | Managed, regional HA, IAM integration | AWS-only, regional isolation | AWS-native deployments |
| GCP Secret Manager | Managed, global replication, VPC-SC integration | GCP-only | GCP-native deployments |
| Azure Key Vault | Managed, HSM-backed, RBAC | Azure-only | Azure-native deployments |

**Decision framework**:
- **Multi-cloud or on-prem**: Vault (portability)
- **Single cloud, compliance-heavy**: Cloud-native (managed HA, auditing, HSM)
- **Hybrid**: Vault with cloud KMS for root key (best of both)

**Why not Kubernetes Secrets alone?**
- Stored in etcd (if etcd is compromised, all secrets leak)
- No automatic rotation
- No audit trail of access
- No encryption at rest by default (without additional config)
- No secret versioning/rollback

#### Layer 2: Secret Injection - External Secrets Operator

**Pattern**: Sync secrets from external backend into Kubernetes Secrets (or directly into pods)

```yaml
# Example: External Secret referencing Vault
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: gpu-job-credentials
  namespace: ml-training
spec:
  # Refresh every 5 minutes
  refreshInterval: 5m

  secretStoreRef:
    name: vault-backend
    kind: ClusterSecretStore

  target:
    name: gpu-job-secrets  # K8s Secret to create
    creationPolicy: Owner

  data:
    - secretKey: db-password
      remoteRef:
        key: ml-training/database
        property: password

    - secretKey: api-token
      remoteRef:
        key: ml-training/api-credentials
        property: token
```

**Multi-region strategy**:

```yaml
# ClusterSecretStore per region
apiVersion: external-secrets.io/v1beta1
kind: ClusterSecretStore
metadata:
  name: vault-us-west
spec:
  provider:
    vault:
      server: "https://vault-us-west.internal"
      path: "secrets"
      version: "v2"
      auth:
        kubernetes:
          mountPath: "kubernetes-us-west"
          role: "gpu-cluster"
          # ServiceAccount token for auth
          serviceAccountRef:
            name: external-secrets
```

**Why this works**:
- Secrets stay in regional Vault instances (latency optimization)
- Vault handles replication/consistency (you don't)
- K8s clusters authenticate via ServiceAccount tokens (cryptographic identity)
- Secrets are synced on-demand (lazy propagation)

#### Layer 3: Workload Identity - Service Account Token Volume Projection

**Problem**: How do pods prove their identity to Vault/cloud KMS?

**Solution**: Projected ServiceAccount tokens (bound to pod lifecycle)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: gpu-training-job
spec:
  serviceAccountName: ml-trainer

  containers:
  - name: trainer
    image: ml-training:v2

    volumeMounts:
    - name: vault-token
      mountPath: /var/run/secrets/vault
      readOnly: true

  volumes:
  - name: vault-token
    projected:
      sources:
      - serviceAccountToken:
          path: token
          expirationSeconds: 3600  # 1 hour
          audience: vault  # Bound to Vault
```

**Why this is critical**:
- Token is **bound to pod**: Dies when pod dies (no leaked credentials)
- Token is **audience-bound**: Can't be used against other services
- Token is **time-limited**: Expires after 1 hour (short-lived)
- Token is **cryptographically verifiable**: Vault can validate without calling K8s API

**Vault authentication flow**:
```
1. Pod starts with projected SA token
2. App reads token from /var/run/secrets/vault/token
3. App calls Vault: "Here's my token, give me secrets"
4. Vault validates token signature (no network call to K8s)
5. Vault checks RBAC: "Does sa:ml-trainer get access to ml-training/*?"
6. Vault returns short-lived dynamic credentials
7. App uses credentials, they expire in 1 hour
8. App refreshes before expiry (continuous rotation)
```

#### Layer 4: Dynamic Secrets - Never Hardcode

**Static secret**: Password stored in Vault, used for months
**Dynamic secret**: Vault generates unique credentials per-request, auto-expires

**Example: Dynamic database credentials**:

```yaml
# Vault database config (one-time setup)
$ vault write database/config/ml-postgres \
    plugin_name=postgresql-database-plugin \
    connection_url="postgresql://{{username}}:{{password}}@postgres:5432/mldata" \
    allowed_roles="ml-trainer" \
    username="vault-admin" \
    password="admin-static-cred"

# Vault role: what permissions do generated creds get?
$ vault write database/roles/ml-trainer \
    db_name=ml-postgres \
    creation_statements="CREATE ROLE \"{{name}}\" WITH LOGIN PASSWORD '{{password}}' VALID UNTIL '{{expiration}}'; \
        GRANT SELECT, INSERT ON training_data TO \"{{name}}\";" \
    default_ttl="1h" \
    max_ttl="24h"
```

**Pod requests credentials**:
```bash
# App authenticates to Vault with SA token
$ vault login -method=kubernetes role=ml-trainer token=<projected-token>

# App requests dynamic DB credentials
$ vault read database/creds/ml-trainer

# Vault response (unique credentials, auto-expire in 1h)
{
  "username": "v-kubern-ml-train-abc123",
  "password": "A1a-randomly-generated-password",
  "ttl": "1h"
}
```

**Why dynamic secrets matter**:
- **Blast radius**: Leaked credential only valid for 1 hour, single workload
- **Auditability**: Vault logs show exactly which pod got which credential
- **No rotation pain**: Credentials auto-expire, no need to coordinate rotation
- **Compliance**: Many frameworks (PCI-DSS, HIPAA) require credential rotation

#### Layer 5: Multi-Region Replication Strategy

**Challenge**: Secrets must be available in all regions, but consistency is expensive.

**Architecture: Regional Vault clusters with asynchronous replication**

```
                    Vault Primary (us-west)
                            |
                    [Raft consensus]
                            |
          +----------------+----------------+
          |                                 |
    Vault Secondary              Vault Secondary
      (eu-west)                      (ap-south)
   [Raft consensus]              [Raft consensus]
```

**Replication model**:
- **Synchronous within region**: Raft ensures consistency (3-5 nodes per region)
- **Asynchronous between regions**: Eventual consistency (acceptable for secrets)
- **Active-active reads**: Each region serves local requests (latency optimization)
- **Active-passive writes**: Primary region handles writes, replicates to secondaries

**Why this trade-off?**
- **Latency**: Local reads are <10ms, cross-region would be 100-300ms
- **Availability**: Region failure doesn't take down other regions
- **Consistency**: Secrets rarely change (eventual consistency is fine)
- **CAP theorem**: We choose AP (availability + partition tolerance) over strong consistency

**Failure scenario**:
```
1. Primary region (us-west) goes down
2. Secondary (eu-west) promoted to primary (manual or automated)
3. New secrets written to eu-west
4. us-west comes back, syncs from eu-west
5. Eventually consistent state reached
```

**Recovery time objective (RTO)**: ~5 minutes (time to promote secondary)
**Recovery point objective (RPO)**: ~30 seconds (replication lag)

#### Layer 6: Enforcement - No Hardcoded Secrets

**Problem**: How do you prevent developers from hardcoding secrets?

**Strategy 1: Admission controller (preventive)**

```yaml
# OPA Gatekeeper policy: Reject pods with hardcoded secrets
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sProhibitEnvSecrets
metadata:
  name: no-hardcoded-secrets
spec:
  match:
    kinds:
      - apiGroups: [""]
        kinds: ["Pod"]
  parameters:
    # Reject if env var name suggests it's a secret
    prohibitedEnvPatterns:
      - ".*PASSWORD.*"
      - ".*SECRET.*"
      - ".*TOKEN.*"
      - ".*KEY.*"
    # Only allow secrets from ExternalSecrets
    allowedSecretSources:
      - "external-secrets.io/*"
```

**Strategy 2: Secret scanning in CI (detective)**

```yaml
# GitLab CI example
pre-commit:
  script:
    - gitleaks detect --source . --verbose
    - trufflehog filesystem .
  rules:
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'
```

**Strategy 3: Runtime detection (detective)**

```bash
# Falco rule: Alert if pod mounts non-external secret
- rule: Hardcoded Secret Mounted
  desc: Pod mounted a K8s secret not managed by ExternalSecrets
  condition: >
    k8s.pod.volume.secret.name exists and
    not k8s.pod.volume.secret.owner == "external-secrets"
  output: "Hardcoded secret detected (pod=%k8s.pod.name secret=%k8s.pod.volume.secret.name)"
  priority: WARNING
```

### Senior-Level Trade-offs Discussion

**1. Centralized (Vault) vs Distributed (cloud KMS per region)**

| Aspect | Centralized Vault | Distributed Cloud KMS |
|--------|-------------------|----------------------|
| Complexity | Higher (you operate it) | Lower (managed service) |
| Cost | Cheaper at scale | Expensive per secret |
| Portability | Cloud-agnostic | Cloud-locked |
| Latency | Higher (centralized) | Lower (regional) |
| Audit | Unified view | Per-cloud silos |

**Decision**: Vault for multi-cloud/hybrid, Cloud KMS for cloud-native single-cloud.

**2. Dynamic secrets vs Static secrets with rotation**

| Aspect | Dynamic | Static + Rotation |
|--------|---------|-------------------|
| Security | Best (unique per-use) | Good (if rotation frequent) |
| Complexity | Higher (backend integration) | Lower (just rotate) |
| Compatibility | Requires Vault-aware apps | Works everywhere |
| Auditability | Excellent (per-access) | Moderate (per-rotation) |

**Decision**: Dynamic for new apps, static-with-rotation for legacy.

**3. Secrets in environment variables vs mounted files**

| Aspect | Env Vars | Files |
|--------|----------|-------|
| Ease of use | Easier (standard) | Requires code change |
| Security | Leaked in dumps | Not in dumps |
| Rotation | Requires pod restart | Can watch file changes |
| K8s native | Yes | Yes |

**Decision**: Files for high-security (DB creds, API keys), env vars for low-security config.

### Connection to Business Value

**Compliance impact**:
- SOC 2, ISO 27001, PCI-DSS all require:
  - Secrets encryption at rest and in transit ✓ (Vault TLS + storage encryption)
  - Access auditing ✓ (Vault audit logs)
  - Credential rotation ✓ (dynamic secrets)
  - Least privilege ✓ (RBAC policies)

**Risk quantification**:
- **Probability of secret leak**: Industry average ~30% of breaches involve credential theft
- **Cost of breach**: $4.45M average (IBM 2023), can be $100M+ for GPU/ML companies with IP theft
- **ROI of secrets management**: $500K investment (eng + infra) vs potential $50M+ loss = 100:1 return

**Example incident**:
```
Without proper secrets management:
- Developer hardcodes AWS key in training script
- Script pushed to public GitHub (happens more than you think)
- Attacker spins up 1000 p4d.24xlarge instances ($32/hour × 1000 = $32K/hour)
- Not noticed for 48 hours = $1.5M AWS bill
- Plus: reputation damage, compliance violations

With secrets management:
- No long-lived AWS keys (IAM roles for service accounts)
- Admission controller prevents hardcoded secrets
- Secret scanning in CI catches it before merge
- Incident prevented
```

---

## 3. Kubernetes Affinity/Anti-Affinity for Mixed Workloads

### The Fundamental Problem

GPU clusters run heterogeneous workloads with competing requirements:
- **Compute-heavy** (ML training): Wants GPUs, high CPU, fast local storage
- **System services** (monitoring, logging): Wants CPU, needs to run on every node
- **Control plane**: Wants dedicated resources, can't tolerate eviction
- **Batch jobs**: Opportunistic, can tolerate eviction
- **Interactive notebooks**: Low latency, needs dedicated resources

**Naive scheduling leads to**:
- System services starved of CPU (evicted by GPU jobs)
- GPU nodes underutilized (system services steal CPU)
- Noisy neighbor problems (batch jobs impact notebooks)
- Cascading failures (control plane evicted during load spike)

**This is a resource allocation problem under constraints**—essentially a bin packing variant with heterogeneous items and multi-dimensional constraints.

### First Principles: Scheduling as a Constraint Satisfaction Problem

Kubernetes scheduler is solving:
```
Maximize: Resource utilization × workload priority
Subject to:
  - Resource constraints (CPU, memory, GPU)
  - Affinity constraints (where workload wants to run)
  - Anti-affinity constraints (where workload must NOT run)
  - Taints/tolerations (node restrictions)
  - Pod disruption budgets (HA requirements)
```

**Key insight**: Affinity rules are **soft preferences**, anti-affinity can be **hard requirements**. Use them accordingly.

### Architecture: Node Pool Segmentation + Scheduling Policies

#### Strategy 1: Node Pools with Taints (Hard Isolation)

**Purpose**: Ensure critical workloads never compete for resources

```yaml
# GPU worker nodes: Only GPU workloads allowed
---
apiVersion: v1
kind: Node
metadata:
  name: gpu-worker-001
  labels:
    node-type: gpu-worker
    gpu-model: a100
    workload-class: compute
spec:
  taints:
  - key: nvidia.com/gpu
    value: "true"
    effect: NoSchedule  # Only pods with toleration can schedule
  - key: workload-class
    value: compute-only
    effect: NoSchedule
```

```yaml
# System/control plane nodes: No GPU workloads
---
apiVersion: v1
kind: Node
metadata:
  name: system-001
  labels:
    node-type: system
    workload-class: infrastructure
spec:
  taints:
  - key: workload-class
    value: infrastructure-only
    effect: NoSchedule
```

```yaml
# GPU workload: Must tolerate GPU taint
---
apiVersion: v1
kind: Pod
metadata:
  name: ml-training
spec:
  tolerations:
  - key: nvidia.com/gpu
    operator: Equal
    value: "true"
    effect: NoSchedule
  - key: workload-class
    operator: Equal
    value: compute-only
    effect: NoSchedule

  nodeSelector:
    node-type: gpu-worker  # Ensure it lands on GPU nodes

  containers:
  - name: trainer
    resources:
      limits:
        nvidia.com/gpu: 8  # Request 8 GPUs
```

**Why taints + tolerations?**
- **Default-deny**: Workloads can't schedule unless they explicitly tolerate
- **Prevents accidents**: Developer can't accidentally schedule on wrong node
- **Enforces resource discipline**: GPU nodes only run GPU workloads

#### Strategy 2: DaemonSets for System Services (Every Node)

**Problem**: Monitoring/logging must run on every node, including GPU nodes

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: node-exporter
  namespace: monitoring
spec:
  selector:
    matchLabels:
      app: node-exporter
  template:
    spec:
      # Tolerate all taints (run everywhere)
      tolerations:
      - operator: Exists
        effect: NoSchedule
      - operator: Exists
        effect: NoExecute

      # Anti-affinity: Don't consume GPU resources
      affinity:
        nodeAffinity:
          # Prefer non-GPU nodes, but can run on GPU if needed
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            preference:
              matchExpressions:
              - key: node-type
                operator: NotIn
                values: [gpu-worker]

      # Resource limits: Prevent system services from starving workloads
      containers:
      - name: node-exporter
        resources:
          requests:
            cpu: 100m      # Guaranteed minimum
            memory: 128Mi
          limits:
            cpu: 200m      # Can't use more than this
            memory: 256Mi

      # Critical priority: Don't evict during resource pressure
      priorityClassName: system-node-critical
```

**Key decisions**:
1. **Tolerate all taints**: System services must run everywhere
2. **Resource limits**: Cap CPU/memory to prevent starving GPU jobs
3. **Priority class**: Mark as critical to prevent eviction
4. **Prefer non-GPU nodes**: But can run on GPU if no alternative (graceful degradation)

#### Strategy 3: Pod Anti-Affinity for HA (Spread Workloads)

**Problem**: Multiple replicas of a service shouldn't all die if one node fails

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: inference-api
spec:
  replicas: 10
  template:
    spec:
      affinity:
        podAntiAffinity:
          # Hard requirement: No two replicas on same node
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: app
                operator: In
                values: [inference-api]
            topologyKey: kubernetes.io/hostname

          # Soft preference: Spread across availability zones
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchExpressions:
                - key: app
                  operator: In
                  values: [inference-api]
              topologyKey: topology.kubernetes.io/zone
```

**Topology keys explained**:
- `kubernetes.io/hostname`: Spread across nodes
- `topology.kubernetes.io/zone`: Spread across availability zones
- `topology.kubernetes.io/region`: Spread across regions
- Custom labels: `rack=rack-1`, `pdu=pdu-A` (spread across power domains)

**Why required vs preferred?**
- **Required**: Scheduler will NOT place pod if constraint can't be met (risk: pod pending forever)
- **Preferred**: Scheduler tries, but will violate if needed (better for availability)

**Decision framework**:
```
Use required when:
  - Security isolation (PCI workloads vs non-PCI)
  - Blast radius containment (control plane replicas)
  - Resource conflicts (two GPU jobs needing same GPU)

Use preferred when:
  - HA optimization (spread across zones)
  - Performance hints (co-locate with cache)
  - Cost optimization (prefer cheaper nodes)
```

#### Strategy 4: Pod Affinity for Data Locality

**Problem**: ML training job should run close to data to minimize network traffic

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: distributed-training
spec:
  affinity:
    # Prefer nodes in same zone as dataset cache
    nodeAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        preference:
          matchExpressions:
          - key: topology.kubernetes.io/zone
            operator: In
            values: [us-west-2a]  # Where data lives

    # Co-locate with dataset cache pods
    podAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 50
        podAffinityTerm:
          labelSelector:
            matchExpressions:
            - key: app
              operator: In
              values: [dataset-cache]
          topologyKey: kubernetes.io/hostname
```

**Trade-off**: Data locality vs resource availability
- **Strict locality**: May wait for nodes in preferred zone
- **Relaxed locality**: Schedule anywhere, tolerate slower data access
- **Decision**: Use `preferred` (not `required`) unless data transfer cost is prohibitive

#### Strategy 5: Priority Classes (Preemption)

**Problem**: During resource contention, which workloads get evicted?

```yaml
# Priority class definitions
---
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: system-critical
value: 1000000000  # Highest
globalDefault: false
description: "Critical system services, never evict"

---
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: production-workloads
value: 100000
globalDefault: false
description: "Production GPU jobs, evict only for critical"

---
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: research-workloads
value: 10000
globalDefault: true
description: "Research jobs, can be preempted"

---
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: batch-jobs
value: 100
globalDefault: false
description: "Opportunistic batch, evict first"
```

**Preemption example**:
```
Scenario:
  - All GPU nodes full with batch-jobs (priority=100)
  - production-workload (priority=100000) submitted
  - Scheduler evicts batch-jobs to make room
  - production-workload runs, batch-jobs requeue

Result:
  - Production always gets resources
  - Batch fills unused capacity
  - Automatic resource rebalancing
```

**Why this matters at scale**:
- 1000 GPU nodes × 8 GPUs = 8000 GPUs
- If 80% allocated to production (6400 GPUs)
- Remaining 20% (1600 GPUs) available for opportunistic batch
- Batch jobs increase utilization from 80% → 95%
- Economic impact: 15% utilization increase = $5M+/year in productivity

#### Strategy 6: Topology Spread Constraints (Even Distribution)

**Problem**: Ensure workloads are evenly distributed across failure domains

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: model-inference
spec:
  replicas: 30
  template:
    spec:
      topologySpreadConstraints:
      # Max skew of 1 across zones (evenly distributed)
      - maxSkew: 1
        topologyKey: topology.kubernetes.io/zone
        whenUnsatisfiable: DoNotSchedule  # Hard requirement
        labelSelector:
          matchLabels:
            app: model-inference

      # Max skew of 2 across nodes (allow some imbalance)
      - maxSkew: 2
        topologyKey: kubernetes.io/hostname
        whenUnsatisfiable: ScheduleAnyway  # Soft preference
        labelSelector:
          matchLabels:
            app: model-inference
```

**Skew explained**:
```
30 replicas across 3 zones:
  - Perfect distribution: 10-10-10 (skew=0)
  - MaxSkew=1 allows: 9-10-11 or 10-10-10
  - MaxSkew=2 allows: 8-10-12 or 9-10-11
  - MaxSkew=5 allows: 5-10-15
```

**Why topology spread > pod anti-affinity?**
- Anti-affinity is binary (same node or not)
- Topology spread enforces **even distribution**
- Better for large replica counts (30+ pods)

### Real-World Configuration: GPU Cluster Example

```yaml
# Production ML training job
---
apiVersion: batch/v1
kind: Job
metadata:
  name: llm-training-gpt
  namespace: ml-production
spec:
  parallelism: 64  # 64 nodes × 8 GPUs = 512 GPU training
  completions: 1
  template:
    spec:
      # Priority: Production workload
      priorityClassName: production-workloads

      # Tolerations: Can run on GPU nodes
      tolerations:
      - key: nvidia.com/gpu
        operator: Equal
        value: "true"
        effect: NoSchedule
      - key: workload-class
        operator: Equal
        value: compute-only
        effect: NoSchedule

      # Node affinity: Require A100 80GB nodes
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: gpu-model
                operator: In
                values: [a100-80gb]
              - key: node-type
                operator: In
                values: [gpu-worker]

        # Pod anti-affinity: Spread across nodes for fault tolerance
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchExpressions:
                - key: job-name
                  operator: In
                  values: [llm-training-gpt]
              topologyKey: kubernetes.io/hostname

      # Topology spread: Even distribution across zones
      topologySpreadConstraints:
      - maxSkew: 2
        topologyKey: topology.kubernetes.io/zone
        whenUnsatisfiable: ScheduleAnyway
        labelSelector:
          matchLabels:
            job-name: llm-training-gpt

      containers:
      - name: trainer
        image: nvcr.io/nvidia/pytorch:23.10-py3
        resources:
          requests:
            nvidia.com/gpu: 8
            cpu: 96
            memory: 800Gi
          limits:
            nvidia.com/gpu: 8
            cpu: 96
            memory: 800Gi
```

### Senior-Level Trade-offs Discussion

**1. Taints/Tolerations vs Node Selectors**

| Aspect | Taints/Tolerations | Node Selectors |
|--------|-------------------|----------------|
| Default behavior | Deny (explicit allow) | Allow (explicit deny) |
| Flexibility | Can update taints live | Requires pod reschedule |
| Use case | Isolation, special hardware | Simple node targeting |

**Decision**: Taints for isolation (GPU nodes), selectors for hints (prefer SSD).

**2. Required vs Preferred Affinity**

| Aspect | Required | Preferred |
|--------|----------|-----------|
| Strictness | Hard constraint | Soft hint |
| Risk | Pod pending forever | May violate preference |
| Use case | Security, licensing | Performance optimization |

**Decision**: Required for compliance, preferred for everything else.

**3. Anti-Affinity vs Topology Spread**

| Aspect | Anti-Affinity | Topology Spread |
|--------|---------------|-----------------|
| Granularity | Binary (same/different) | Numeric (max skew) |
| Large deployments | Verbose (N² constraints) | Compact |
| Even distribution | No guarantee | Enforced |

**Decision**: Topology spread for replicas >10, anti-affinity for <10.

### Connection to Business Value

**Utilization optimization**:
```
Without proper affinity:
  - GPU nodes: 60% utilized (system services steal CPU)
  - System nodes: 30% utilized (oversized for peaks)
  - Wasted capacity: $3M/year

With proper affinity:
  - GPU nodes: 90% utilized (dedicated to GPU workloads)
  - System nodes: 70% utilized (right-sized)
  - Savings: $2.5M/year in infrastructure
```

**Availability impact**:
```
Without pod anti-affinity:
  - 10 replicas of inference API
  - All scheduled on 3 nodes (random placement)
  - Node failure takes down 3-4 replicas
  - SLO: 99.0% (too many single-node failures)

With pod anti-affinity:
  - 10 replicas spread across 10 nodes
  - Node failure takes down 1 replica (90% capacity remains)
  - SLO: 99.95%
```

---

## 4. etcd Backups for Petabyte-Scale Workloads

### The Fundamental Problem

etcd is the source of truth for Kubernetes. Losing etcd = losing the entire cluster state:
- All pod definitions
- All service configurations
- All secrets, configmaps
- All RBAC policies
- All custom resources

**Petabyte-scale workloads** mean:
- Thousands of nodes
- Tens of thousands of pods
- Complex stateful applications with years of accumulated config
- Recreating from scratch would take weeks/months
- Business impact of data loss: potentially catastrophic

**Backup challenges**:
- **Consistency**: etcd is distributed (3-5 nodes), backups must be consistent snapshots
- **Performance**: Backups can't impact cluster performance (etcd is latency-sensitive)
- **Retention**: How long to keep? (compliance, disaster recovery)
- **Recovery testing**: Backups you haven't tested are Schrödinger's backups
- **Scale**: etcd DB can be 8GB+ (thousands of objects)

### First Principles: Disaster Recovery Theory

**Recovery Point Objective (RPO)**: How much data can we afford to lose?
- Banking: 0 seconds (real-time replication)
- E-commerce: 5 minutes (recent orders acceptable loss)
- GPU cluster: 1 hour (can recreate recently submitted jobs)

**Recovery Time Objective (RTO)**: How long to get back online?
- Banking: <30 seconds (hot standby)
- E-commerce: <5 minutes (impact revenue)
- GPU cluster: <15 minutes (acceptable downtime)

**Backup frequency = RPO**
**Restore testing = RTO**

### Architecture: Multi-Tier Backup Strategy

#### Tier 1: Continuous etcd Snapshots

**Mechanism**: Automated snapshots every N minutes

```bash
#!/bin/bash
# Backup script running on etcd node or external runner

ETCDCTL_API=3 etcdctl snapshot save /backup/etcd-$(date +%Y%m%d-%H%M%S).db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

# Verify snapshot integrity
ETCDCTL_API=3 etcdctl snapshot status /backup/etcd-$(date +%Y%m%d-%H%M%S).db --write-out=table

# Upload to S3/GCS/Azure Blob
aws s3 cp /backup/etcd-$(date +%Y%m%d-%H%M%S).db \
  s3://k8s-etcd-backups/cluster-prod/ \
  --storage-class GLACIER_IR  # Cost optimization
```

**Frequency**: Every 5-15 minutes
- **Trade-off**: More frequent = lower RPO, higher storage cost, more etcd load
- **Decision**: 5 min for prod, 15 min for dev/staging

**Why snapshots work**:
- etcd snapshot is **point-in-time consistent** (uses Raft log index)
- Snapshot doesn't lock etcd (uses MVCC, no write blocking)
- Snapshot size ~= etcd DB size (8GB typical, 20GB for very large clusters)

#### Tier 2: Geo-Redundant Storage

**Problem**: Local backup doesn't help if datacenter burns down

**Strategy**: Multi-region replication

```yaml
# S3 bucket configuration
Resources:
  EtcdBackupBucket:
    Type: AWS::S3::Bucket
    Properties:
      BucketName: k8s-etcd-backups-prod

      # Versioning: Keep history of backups
      VersioningConfiguration:
        Status: Enabled

      # Replication: Cross-region disaster recovery
      ReplicationConfiguration:
        Role: !GetAtt ReplicationRole.Arn
        Rules:
          - Id: ReplicateAllToEU
            Status: Enabled
            Priority: 1
            Destination:
              Bucket: !GetAtt EtcdBackupBucketEU.Arn
              StorageClass: GLACIER_IR

      # Lifecycle: Retention policy
      LifecycleConfiguration:
        Rules:
          # Keep hourly backups for 7 days
          - Id: HourlyBackups
            Status: Enabled
            ExpirationInDays: 7
            Transitions:
              - TransitionInDays: 1
                StorageClass: STANDARD_IA

          # Keep daily backups for 30 days
          - Id: DailyBackups
            Status: Enabled
            ExpirationInDays: 30
            Transitions:
              - TransitionInDays: 7
                StorageClass: GLACIER_IR

          # Keep weekly backups for 1 year
          - Id: WeeklyBackups
            Status: Enabled
            ExpirationInDays: 365
            Transitions:
              - TransitionInDays: 30
                StorageClass: DEEP_ARCHIVE
```

**Cost analysis** (1000-node cluster, 8GB snapshots):
```
Snapshot frequency: Every 5 minutes = 288/day
Storage: 288 snapshots/day × 8GB = 2.3TB/day raw

With lifecycle policy:
  - 7 days × 288 snapshots = 2,016 snapshots × 8GB = ~16TB (STANDARD_IA)
  - 23 days × 1 daily = 23 × 8GB = 184GB (GLACIER_IR)
  - 48 weeks × 1 weekly = 48 × 8GB = 384GB (DEEP_ARCHIVE)

Total storage: ~17TB
Monthly cost:
  - S3 Standard-IA: 16TB × $0.0125/GB = $200
  - Glacier IR: 184GB × $0.004/GB = $0.74
  - Deep Archive: 384GB × $0.00099/GB = $0.38
  - Total: ~$201/month

Compare to:
  - Cost of cluster rebuild: 2 weeks × 10 engineers × $200/hour × 40 hours = $800K
  - ROI: $201/month prevents $800K loss = 330,000% ROI
```

#### Tier 3: Application-Level Backups (Defense in Depth)

**Insight**: etcd backup is cluster state, but doesn't capture everything

**What etcd backup misses**:
- Persistent volume data (PVCs)
- External resources (cloud load balancers, DNS)
- Custom CRDs with external dependencies
- Secrets in external secret stores (Vault)

**Solution**: Velero (Cluster + Volume Backup)

```yaml
# Velero backup schedule
apiVersion: velero.io/v1
kind: Schedule
metadata:
  name: daily-full-backup
  namespace: velero
spec:
  # Run daily at 2 AM UTC
  schedule: "0 2 * * *"

  template:
    # Include all namespaces
    includedNamespaces:
    - "*"

    # Exclude ephemeral namespaces
    excludedNamespaces:
    - kube-system
    - kube-public

    # Backup PVs
    snapshotVolumes: true

    # Default volume snapshot locations
    volumeSnapshotLocations:
    - aws-us-west-2

    # Retention: 30 days
    ttl: 720h

    # Hooks: Quiesce databases before backup
    hooks:
      resources:
      - name: postgres-backup-hook
        includedNamespaces:
        - databases
        labelSelector:
          matchLabels:
            app: postgres
        pre:
        - exec:
            command:
            - /bin/bash
            - -c
            - "pg_dump -U postgres -d mldata > /backup/pre-snapshot.sql"
        post:
        - exec:
            command:
            - /bin/bash
            - -c
            - "rm /backup/pre-snapshot.sql"
```

**Why Velero + etcd backups?**
- **etcd**: Fast restore of cluster state (minutes)
- **Velero**: Complete disaster recovery including volumes (hours)
- **Layered defense**: etcd corruption vs full cluster loss

#### Tier 4: Automated Restore Testing

**Problem**: 90% of backup failures are discovered during restore attempts

**Solution**: Monthly automated restore drills

```yaml
# Automated restore test pipeline (pseudo-code)
restore_test_pipeline:
  schedule: "0 3 1 * *"  # First day of each month, 3 AM

  steps:
    1_create_test_cluster:
      - Provision isolated K8s cluster (same version as prod)
      - No production traffic
      - Separate VPC/network

    2_fetch_latest_backup:
      - Download latest etcd snapshot from S3
      - Verify checksum/integrity
      - Download latest Velero backup

    3_restore_etcd:
      - Stop etcd on test cluster
      - Restore snapshot: etcdctl snapshot restore
      - Start etcd with restored data
      - Verify kube-apiserver connects

    4_validate_cluster_state:
      - Check all namespaces present
      - Check all deployments/statefulsets
      - Verify secrets accessible
      - Run smoke tests

    5_restore_volumes:
      - Velero restore from backup
      - Verify PVCs bind
      - Check data integrity (file count, checksums)

    6_run_integration_tests:
      - Deploy test workload
      - Verify can schedule pods
      - Check networking (pod-to-pod, pod-to-service)
      - Test DNS resolution

    7_measure_rto:
      - Track: Time from backup fetch to working cluster
      - Alert if RTO > 15 minutes

    8_cleanup:
      - Destroy test cluster
      - Archive test results
      - Send report to SRE team
```

**Why automate?**
- **Humans forget**: Manual tests skipped during busy periods
- **Drift detection**: Restore process may break due to K8s version changes
- **RTO validation**: Ensure you can actually meet 15-minute target
- **Confidence**: Know your backups work before you need them

#### Tier 5: Zero Data Loss - Continuous Replication

**Problem**: Even 5-minute backups have 5 minutes of RPO (data loss window)

**Solution**: Multi-cluster with real-time state replication (for critical workloads)

**Architecture**:
```
                      Primary Cluster (us-west)
                            |
                      [etcd cluster]
                            |
                    GitOps sync (ArgoCD)
                            |
        +-------------------+-------------------+
        |                                       |
Secondary Cluster (us-east)        Secondary Cluster (eu-west)
    [etcd cluster]                     [etcd cluster]
```

**Mechanism**: GitOps as source of truth

1. All K8s manifests stored in Git
2. ArgoCD watches Git, syncs to all clusters
3. Clusters are independently deployable
4. etcd in each cluster is independent (no cross-cluster consensus)

**Benefits**:
- **RPO = 0**: Git is source of truth, no data loss
- **RTO < 5 min**: Failover to secondary cluster (DNS change)
- **No etcd replication complexity**: Each cluster independent

**Trade-off**:
- **Cost**: 3× infrastructure (3 clusters)
- **Complexity**: Multi-cluster management
- **Use case**: Only for mission-critical workloads

**When to use**:
```
Single cluster + backups:
  - Dev/staging environments
  - Cost-sensitive workloads
  - RPO of 5-15 minutes acceptable

Multi-cluster + GitOps:
  - Production revenue-generating workloads
  - Compliance requirements (HA, DR)
  - RPO = 0 required
```

### Operational Procedures

#### Backup Verification Script

```bash
#!/bin/bash
# Run after each backup

SNAPSHOT_FILE=$1

# 1. Verify file exists and size
if [ ! -f "$SNAPSHOT_FILE" ]; then
  echo "ERROR: Snapshot file not found"
  exit 1
fi

SIZE=$(stat -f%z "$SNAPSHOT_FILE" 2>/dev/null || stat -c%s "$SNAPSHOT_FILE")
if [ "$SIZE" -lt 1000000 ]; then  # Less than 1MB is suspicious
  echo "ERROR: Snapshot file too small: $SIZE bytes"
  exit 1
fi

# 2. Verify snapshot integrity
ETCDCTL_API=3 etcdctl snapshot status "$SNAPSHOT_FILE" --write-out=json > /tmp/snapshot-status.json
if [ $? -ne 0 ]; then
  echo "ERROR: Snapshot integrity check failed"
  exit 1
fi

# 3. Extract metadata
HASH=$(jq -r '.hash' /tmp/snapshot-status.json)
REVISION=$(jq -r '.revision' /tmp/snapshot-status.json)
TOTAL_KEYS=$(jq -r '.totalKey' /tmp/snapshot-status.json)

# 4. Compare to previous backup
PREV_KEYS=$(cat /var/lib/etcd-backup/last-key-count.txt 2>/dev/null || echo "0")
KEY_DELTA=$((TOTAL_KEYS - PREV_KEYS))

if [ "$KEY_DELTA" -lt -1000 ]; then
  echo "WARNING: Significant key count drop: $KEY_DELTA keys"
  # Alert but don't fail (might be legitimate cleanup)
fi

# 5. Update baseline
echo "$TOTAL_KEYS" > /var/lib/etcd-backup/last-key-count.txt

# 6. Log success
echo "Backup verified: hash=$HASH revision=$REVISION keys=$TOTAL_KEYS delta=$KEY_DELTA"
```

#### Restore Procedure (Runbook)

```markdown
# etcd Restore Procedure

## Pre-requisites
- Latest etcd snapshot file
- Access to control plane nodes
- Downtime window approved

## Steps

### 1. Stop kube-apiserver on all control plane nodes
```bash
systemctl stop kube-apiserver
```

### 2. Stop etcd on all control plane nodes
```bash
systemctl stop etcd
```

### 3. Backup current etcd data (safety)
```bash
mv /var/lib/etcd /var/lib/etcd.backup-$(date +%Y%m%d-%H%M%S)
```

### 4. Restore snapshot on FIRST control plane node
```bash
ETCDCTL_API=3 etcdctl snapshot restore /path/to/snapshot.db \
  --name=master-1 \
  --initial-cluster=master-1=https://10.0.0.1:2380,master-2=https://10.0.0.2:2380,master-3=https://10.0.0.3:2380 \
  --initial-cluster-token=etcd-cluster-prod \
  --initial-advertise-peer-urls=https://10.0.0.1:2380 \
  --data-dir=/var/lib/etcd
```

### 5. Restore on remaining control plane nodes
(Repeat step 4 with different --name and --initial-advertise-peer-urls)

### 6. Start etcd on all nodes
```bash
systemctl start etcd
```

### 7. Verify etcd cluster health
```bash
ETCDCTL_API=3 etcdctl member list --write-out=table
ETCDCTL_API=3 etcdctl endpoint health --cluster
```

### 8. Start kube-apiserver
```bash
systemctl start kube-apiserver
```

### 9. Verify cluster state
```bash
kubectl get nodes
kubectl get pods --all-namespaces
kubectl get pvc --all-namespaces
```

### 10. Restore PVs from Velero (if needed)
```bash
velero restore create --from-backup daily-full-backup-20231201
```

## Rollback
If restore fails:
```bash
systemctl stop etcd
systemctl stop kube-apiserver
rm -rf /var/lib/etcd
mv /var/lib/etcd.backup-<timestamp> /var/lib/etcd
systemctl start etcd
systemctl start kube-apiserver
```

## Estimated RTO
- etcd restore: 5-10 minutes
- Cluster validation: 2-5 minutes
- Velero PV restore: 30-60 minutes (if needed)
Total: 15-75 minutes depending on scope
```

### Senior-Level Trade-offs Discussion

**1. etcd snapshot vs Velero vs GitOps**

| Aspect | etcd Snapshot | Velero | GitOps |
|--------|---------------|--------|--------|
| Scope | Cluster state only | State + volumes | Declarative config |
| RPO | 5-15 minutes | Hours (daily) | Real-time |
| RTO | 5-10 minutes | 30-60 minutes | 5 minutes |
| Storage cost | Low ($200/mo) | High (PV snapshots) | Minimal (Git) |
| Complexity | Low | Medium | High |

**Decision**: All three (layered defense)
- etcd: Fast recovery from etcd corruption
- Velero: Disaster recovery including stateful apps
- GitOps: Source of truth, multi-cluster sync

**2. Snapshot frequency vs etcd performance impact**

| Frequency | RPO | etcd Load | Storage Cost |
|-----------|-----|-----------|--------------|
| 1 minute | 1 min | High (can impact P99) | Very high |
| 5 minutes | 5 min | Low (negligible) | High |
| 15 minutes | 15 min | Very low | Medium |
| 1 hour | 1 hour | Negligible | Low |

**Decision**: 5 minutes for production (balanced), 15 min for dev

**3. Backup retention vs compliance requirements**

| Industry | Retention Requirement | Reasoning |
|----------|----------------------|-----------|
| Finance | 7 years | SEC regulations |
| Healthcare | 6 years | HIPAA |
| General SaaS | 30-90 days | Business continuity |
| Internal tools | 7 days | Cost optimization |

**Decision**: Consult compliance team, default to 30 days

### Connection to Business Value

**Cost of etcd data loss**:
```
Scenario: Total etcd loss, no backups
- 1000-node cluster with 5000 workloads
- Estimated rebuild time: 2-4 weeks
  - Recreate all deployments/services from memory/docs
  - Reconfigure all RBAC policies
  - Recreate all secrets/configmaps
  - Debug networking/ingress issues
  - Validate all workloads

Cost:
- SRE time: 10 engineers × 4 weeks × 40 hours × $200/hour = $3.2M
- Lost productivity: 1000 GPUs × 4 weeks × $2/GPU-hour = $1.3M
- Reputational damage: Customers lose trust
- Total: $4.5M+

Backup cost:
- Storage: $200/month × 12 = $2,400/year
- Velero infra: ~$1,000/month = $12,000/year
- Automation/testing: $50K one-time engineering
- Total: $64,400/year

ROI: $4.5M saved / $64K cost = 70:1 return
```

**Real-world incident** (anonymized):
```
Company: Large ML research lab
Incident: etcd corruption due to disk failure
Impact:
- 3-node etcd cluster
- 2 nodes failed simultaneously (same RAID controller)
- Remaining node had stale data
- No tested backups

Recovery:
- Spent 3 weeks manually recreating cluster config
- Lost 40% of historical experiment metadata
- Several teams blocked for 2+ weeks
- Estimated cost: $2M in lost productivity

Lesson: Backups you haven't tested are not backups
```

---

## Summary: Senior Engineering Mindset

These questions aren't about memorizing kubectl commands. They're about:

1. **Understanding constraints**: GPU economics, distributed systems CAP theorem, blast radius quantification
2. **Reasoning from first principles**: Why does this architecture make sense given the problem space?
3. **Quantifying trade-offs**: What do we gain/lose with each approach? What's the ROI?
4. **Connecting to business value**: How does this technical decision impact revenue, risk, compliance?
5. **Operational maturity**: It's not done until it's monitored, tested, and documented

In senior+ interviews, you're evaluated on:
- Can you design systems that account for failures?
- Can you articulate why alternative approaches wouldn't work?
- Can you estimate the business impact of technical decisions?
- Can you operate complex systems reliably at scale?

This guide is your foundation for those conversations.
