# GitOps Workflows: Git as Source of Truth

## The GitOps Principles

GitOps is an operational model where Git is the single source of truth for declarative infrastructure and applications.

**The four principles**:

1. **Declarative**: System described declaratively (Kubernetes manifests in Git)
2. **Versioned and immutable**: Git provides version control and immutability
3. **Pulled automatically**: Software agents pull desired state from Git
4. **Continuously reconciled**: Agents ensure actual state matches Git

---

## Push vs. Pull: The Fundamental Difference

### Traditional CI/CD (Push-Based)

```
Developer commits code
    ↓
CI/CD pipeline triggered
    ↓
Pipeline builds Docker image
    ↓
Pipeline runs: kubectl apply -f deployment.yaml
    ↓
Cluster updated
```

**Characteristics**:
- CI/CD system has write access to production cluster
- Credentials stored in CI/CD (security risk)
- Deployment happens during pipeline execution
- If pipeline fails mid-deploy, cluster in unknown state
- Audit trail is CI/CD logs

**Security model**: CI/CD is trusted, has cluster admin access

### GitOps (Pull-Based)

```
Developer commits code
    ↓
CI/CD pipeline triggered
    ↓
Pipeline builds Docker image
    ↓
Pipeline commits updated manifest to Git (image tag change)
    ↓
GitOps controller (in cluster) detects Git change
    ↓
Controller pulls manifest from Git
    ↓
Controller applies manifest to cluster
    ↓
Cluster updated
```

**Characteristics**:
- CI/CD has NO write access to cluster
- GitOps controller runs inside cluster (secure)
- Deployment is continuous (controller always syncing)
- If controller crashes, it resumes when restarted
- Audit trail is Git history

**Security model**: Only GitOps controller (inside cluster) has cluster access. CI/CD can only push to Git.

---

## Why Pull-Based is Superior for CD

### Security

**Push-based**:
- CI/CD has cluster credentials
- If CI/CD compromised, cluster compromised
- Credentials in CI/CD secrets (must be rotated, managed)

**Pull-based (GitOps)**:
- CI/CD has NO cluster credentials
- If CI/CD compromised, can only push to Git (which is reviewed)
- Only controller (inside cluster) has cluster access
- No credentials to leak

### Auditability

**Push-based**:
- Audit trail: CI/CD logs ("At 14:32, pipeline ran kubectl apply")
- Logs can be lost, incomplete
- Hard to answer: "What was deployed on Jan 15?"

**Pull-based (GitOps)**:
- Audit trail: Git history
- `git log` shows exact state at any time
- `git diff` shows what changed
- `git revert` for rollback
- Immutable, signed commits

### Drift Detection

**Push-based**:
- Manual `kubectl edit` changes cluster
- No automatic detection
- Cluster drifts from intended state

**Pull-based (GitOps)**:
- Controller continuously compares Git (desired) to cluster (actual)
- Detects drift automatically
- Can auto-remediate (reapply from Git) or alert

**Example**:

```
10:00 - GitOps controller syncs: replicas: 5 (from Git)
10:15 - Admin manually runs: kubectl scale deployment myapp --replicas=10
10:16 - Controller detects drift: Git says 5, cluster has 10
10:17 - Controller re-applies: replicas: 5 (Git wins)
```

Manual change reverted automatically.

### Disaster Recovery

**Push-based**:
- Cluster destroyed
- Must re-run all deployment pipelines
- Hope pipelines are in good state

**Pull-based (GitOps)**:
- Cluster destroyed
- Recreate cluster, install GitOps controller
- Point controller at Git repo
- Controller rebuilds entire cluster from Git
- Back to desired state

**Git is the disaster recovery plan.**

---

## ArgoCD: Declarative GitOps

### Architecture

```
                 ┌──────────────────┐
                 │   Git Repo       │
                 │   (manifests)    │
                 └────────┬─────────┘
                          │
                          │ (polls every 3 minutes)
                          ↓
                 ┌──────────────────┐
                 │  ArgoCD Server   │ (in cluster)
                 └────────┬─────────┘
                          │
                          │ (applies)
                          ↓
                 ┌──────────────────┐
                 │  Kubernetes API  │
                 └──────────────────┘
```

**Components**:
- **Application Controller**: Monitors Git, syncs to cluster
- **API Server**: Web UI, CLI, webhook receiver
- **Repo Server**: Renders manifests (Helm, Kustomize)
- **Redis**: Caching layer

### Application CRD

ArgoCD introduces an `Application` CRD:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: myapp
  namespace: argocd
spec:
  # Source: Where manifests are
  source:
    repoURL: https://github.com/myorg/myapp
    targetRevision: main  # Branch or tag
    path: k8s/prod       # Path to manifests

  # Destination: Where to deploy
  destination:
    server: https://kubernetes.default.svc
    namespace: production

  # Sync policy
  syncPolicy:
    automated:
      prune: true       # Delete resources not in Git
      selfHeal: true    # Revert manual changes
    syncOptions:
    - CreateNamespace=true
```

**What this does**:
- ArgoCD watches `github.com/myorg/myapp` (main branch, `k8s/prod/` directory)
- Renders manifests (if Helm/Kustomize)
- Applies to production namespace
- Automatically syncs on Git changes
- Deletes resources removed from Git (`prune`)
- Reverts manual cluster changes (`selfHeal`)

### Sync Strategies

**Automated sync**:

```yaml
syncPolicy:
  automated:
    prune: true
    selfHeal: true
```

**Behavior**:
- Git changes → Auto-applied within 3 minutes
- Manual cluster changes → Auto-reverted
- Resources deleted from Git → Auto-deleted from cluster

**Manual sync**:

```yaml
syncPolicy: {}  # No automated
```

**Behavior**:
- Git changes → ArgoCD detects, shows diff
- Human reviews diff
- Human clicks "Sync" button or runs `argocd app sync myapp`
- Changes applied

**When to use automated**: Dev, staging, non-critical environments
**When to use manual**: Production (want human approval)

### Health Assessment

ArgoCD checks resource health beyond just creation.

**Healthy states**:

```
Deployment: Desired replicas == Ready replicas
Service: Endpoints exist
Ingress: LoadBalancer IP assigned
Job: Completed successfully
```

**Degraded states**:

```
Deployment: Replicas not ready (ImagePullError, CrashLoopBackOff)
Job: Failed
PersistentVolumeClaim: Pending (no volume available)
```

**Application status**:

```bash
argocd app get myapp

Health Status: Healthy
Sync Status: Synced
```

### Sync Waves: Ordered Deployment

Deploy resources in specific order.

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: database-config
  annotations:
    argocd.argoproj.io/sync-wave: "0"  # Deploy first

---
apiVersion: batch/v1
kind: Job
metadata:
  name: database-migration
  annotations:
    argocd.argoproj.io/sync-wave: "1"  # Deploy second (after ConfigMap)

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
  annotations:
    argocd.argoproj.io/sync-wave: "2"  # Deploy third (after migration)
```

**Order**:
1. Wave 0: ConfigMap (database config)
2. Wait for Wave 0 to be healthy
3. Wave 1: Job (database migration)
4. Wait for Wave 1 to complete
5. Wave 2: Deployment (application)

**Use case**: Database migrations before application deployment.

### Sync Hooks: Pre/Post Actions

Run jobs at specific sync phases.

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: pre-sync-backup
  annotations:
    argocd.argoproj.io/hook: PreSync  # Run before sync
    argocd.argoproj.io/hook-delete-policy: BeforeHookCreation
spec:
  template:
    spec:
      containers:
      - name: backup
        image: backup-tool
        command: ["backup-database.sh"]
      restartPolicy: Never
```

**Hook phases**:
- `PreSync`: Before sync starts (backups, validation)
- `Sync`: During sync (rarely used)
- `PostSync`: After sync completes (smoke tests, notifications)
- `SyncFail`: If sync fails (rollback, alerts)

---

## Flux: GitOps Toolkit

Flux is a toolkit for building GitOps workflows. More modular than ArgoCD.

### Architecture

```
                 ┌──────────────────┐
                 │   Git Repo       │
                 └────────┬─────────┘
                          │
                ┌─────────┴─────────┐
                │                   │
                ↓                   ↓
        ┌───────────────┐   ┌───────────────┐
        │Source         │   │Kustomize      │
        │Controller     │   │Controller     │
        └───────────────┘   └───────┬───────┘
                                    │
                                    ↓
                            ┌───────────────┐
                            │Helm           │
                            │Controller     │
                            └───────┬───────┘
                                    │
                                    ↓
                            ┌───────────────┐
                            │Notification   │
                            │Controller     │
                            └───────────────┘
```

**Controllers** (modular):
- **Source Controller**: Watches Git repos, Helm repos, S3 buckets
- **Kustomize Controller**: Applies Kustomize overlays
- **Helm Controller**: Deploys Helm releases
- **Notification Controller**: Sends alerts (Slack, etc.)
- **Image Automation Controller**: Updates image tags in Git

### GitRepository Resource

```yaml
apiVersion: source.toolkit.fluxcd.io/v1
kind: GitRepository
metadata:
  name: myapp-repo
  namespace: flux-system
spec:
  interval: 1m  # Check Git every 1 minute
  url: https://github.com/myorg/myapp
  ref:
    branch: main
  secretRef:
    name: git-credentials
```

Source Controller watches this Git repo.

### Kustomization Resource

```yaml
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: myapp-prod
  namespace: flux-system
spec:
  interval: 5m  # Reconcile every 5 minutes
  sourceRef:
    kind: GitRepository
    name: myapp-repo
  path: ./k8s/overlays/prod  # Path in repo
  prune: true               # Delete resources not in Git
  targetNamespace: production
```

Kustomize Controller applies manifests from `k8s/overlays/prod`.

### HelmRelease Resource

```yaml
apiVersion: helm.toolkit.fluxcd.io/v2beta1
kind: HelmRelease
metadata:
  name: myapp
  namespace: production
spec:
  interval: 5m
  chart:
    spec:
      chart: ./charts/myapp
      sourceRef:
        kind: GitRepository
        name: myapp-repo
  values:
    replicaCount: 5
    image:
      tag: 1.2.3
```

Helm Controller deploys Helm chart.

### Image Automation

Automatically update image tags in Git when new images are built.

```yaml
apiVersion: image.toolkit.fluxcd.io/v1beta1
kind: ImageRepository
metadata:
  name: myapp
spec:
  image: docker.io/myorg/myapp
  interval: 1m  # Scan registry every minute

---
apiVersion: image.toolkit.fluxcd.io/v1beta1
kind: ImagePolicy
metadata:
  name: myapp
spec:
  imageRepositoryRef:
    name: myapp
  policy:
    semver:
      range: 1.x.x  # Only versions matching 1.x.x

---
apiVersion: image.toolkit.fluxcd.io/v1beta1
kind: ImageUpdateAutomation
metadata:
  name: myapp
spec:
  sourceRef:
    kind: GitRepository
    name: myapp-repo
  git:
    commit:
      author:
        name: Flux
        email: flux@example.com
      messageTemplate: "Update image to {{range .Updated.Images}}{{.}}{{end}}"
  update:
    path: ./k8s/overlays/prod
    strategy: Setters  # Update image tags in YAML
```

**What happens**:
1. New image pushed: `docker.io/myorg/myapp:1.2.4`
2. ImageRepository detects new tag
3. ImagePolicy checks: 1.2.4 matches `1.x.x` ✓
4. ImageUpdateAutomation updates `k8s/overlays/prod/deployment.yaml`: `image: myapp:1.2.3` → `image: myapp:1.2.4`
5. Commits change to Git
6. GitRepository detects commit
7. Kustomization applies updated manifest
8. Deployment rolls out v1.2.4

**Fully automated** CI/CD: CI builds image → Flux updates Git → Flux deploys.

---

## Multi-Environment Strategies

### Strategy 1: Directories per Environment

```
git-repo/
├── base/
│   ├── deployment.yaml
│   └── service.yaml
└── overlays/
    ├── dev/
    │   └── kustomization.yaml
    ├── staging/
    │   └── kustomization.yaml
    └── prod/
        └── kustomization.yaml
```

**ArgoCD Applications**:

```yaml
# Dev app
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: myapp-dev
spec:
  source:
    repoURL: https://github.com/myorg/myapp
    path: overlays/dev

---
# Prod app
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: myapp-prod
spec:
  source:
    repoURL: https://github.com/myorg/myapp
    path: overlays/prod
```

**Promotion**: Copy/update prod overlay, commit to main branch.

### Strategy 2: Branches per Environment

```
branches/
├── dev
├── staging
└── prod
```

**ArgoCD Applications**:

```yaml
# Dev app
spec:
  source:
    repoURL: https://github.com/myorg/myapp
    targetRevision: dev  # dev branch

---
# Prod app
spec:
  source:
    repoURL: https://github.com/myorg/myapp
    targetRevision: prod  # prod branch
```

**Promotion**: Merge dev → staging → prod.

**Pros**: Clear separation, merge commits show promotion
**Cons**: Branch drift, merge conflicts

### Strategy 3: Separate Repos per Environment

```
repos/
├── myapp-dev (git.com/myorg/myapp-dev)
├── myapp-staging (git.com/myorg/myapp-staging)
└── myapp-prod (git.com/myorg/myapp-prod)
```

**ArgoCD Applications** point to different repos.

**Pros**: Complete isolation, different permissions per repo
**Cons**: Duplication, harder to track changes across environments

### Strategy 4: Mono-Repo with App-of-Apps

```
git-repo/
├── apps/
│   ├── app1/
│   ├── app2/
│   └── app3/
└── environments/
    ├── dev/
    │   └── apps.yaml  (references all apps)
    ├── staging/
    │   └── apps.yaml
    └── prod/
        └── apps.yaml
```

**Root app** (app-of-apps pattern):

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: prod-apps
spec:
  source:
    repoURL: https://github.com/myorg/infrastructure
    path: environments/prod
```

**environments/prod/apps.yaml**:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: app1-prod
spec:
  source:
    repoURL: https://github.com/myorg/infrastructure
    path: apps/app1/overlays/prod

---
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: app2-prod
spec:
  source:
    repoURL: https://github.com/myorg/infrastructure
    path: apps/app2/overlays/prod
```

**One root Application** deploys all apps for an environment.

---

## Secret Management

Git is public/versioned. Secrets can't be in plaintext.

### Approach 1: Sealed Secrets

**Concept**: Encrypt secrets in Git, controller decrypts in cluster.

```bash
# Create SealedSecret (encrypted)
kubectl create secret generic mysecret --from-literal=password=hunter2 --dry-run=client -o yaml | \
  kubeseal -o yaml > mysealedsecret.yaml

# Commit mysealedsecret.yaml to Git (encrypted, safe)
git add mysealedsecret.yaml
git commit -m "Add encrypted secret"
```

**mysealedsecret.yaml** (safe to commit):

```yaml
apiVersion: bitnami.com/v1alpha1
kind: SealedSecret
metadata:
  name: mysecret
spec:
  encryptedData:
    password: AgBy3i4OJSWK+PiTySYZZA9rO43cGDEq...  # Encrypted
```

**Sealed Secrets Controller** (in cluster):
- Watches SealedSecret resources
- Decrypts using private key (only exists in cluster)
- Creates Secret resource
- Application uses Secret normally

**Security**: Private key never leaves cluster. Only controller can decrypt.

### Approach 2: External Secrets Operator

**Concept**: Sync secrets from external secret manager (AWS Secrets Manager, Vault, etc.).

```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: database-secret
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: aws-secrets-manager
    kind: SecretStore
  target:
    name: database-secret  # Kubernetes Secret to create
  data:
  - secretKey: password
    remoteRef:
      key: prod/database  # Key in AWS Secrets Manager
      property: password
```

**External Secrets Operator**:
- Watches ExternalSecret resources
- Fetches value from AWS Secrets Manager
- Creates Kubernetes Secret
- Refreshes every 1 hour

**Security**: Secrets stored in AWS (IAM controlled), not in Git.

### Approach 3: SOPS (Secrets OPerationS)

**Concept**: Encrypt specific fields in YAML files.

```yaml
# secret.yaml (encrypted)
apiVersion: v1
kind: Secret
metadata:
  name: mysecret
stringData:
  password: ENC[AES256_GCM,data:xxx,iv:yyy,tag:zzz]  # Encrypted value
  username: admin  # Plaintext (not sensitive)

sops:
  kms:
  - arn: arn:aws:kms:us-east-1:123456789:key/abc123  # KMS key used for encryption
```

**Decryption**: ArgoCD/Flux plugin decrypts during sync using KMS.

---

## Drift Detection and Remediation

### Detecting Drift

**ArgoCD**:

```bash
argocd app get myapp

Sync Status: OutOfSync

GROUP  KIND        NAME   STATUS
apps   Deployment  myapp  OutOfSync  # replicas: Git=5, Cluster=10
```

**Flux**:

```bash
flux get kustomizations

NAME       READY   MESSAGE
myapp-prod False   kustomization path './k8s/overlays/prod' not found
```

### Remediation Options

**1. Auto-remediation** (revert manual changes):

```yaml
# ArgoCD
syncPolicy:
  automated:
    selfHeal: true

# Flux
spec:
  prune: true
  force: true  # Force apply even if conflict
```

**2. Alert only** (manual remediation):

```yaml
# ArgoCD
syncPolicy: {}  # No automated

# Flux notification
apiVersion: notification.toolkit.fluxcd.io/v1beta1
kind: Alert
metadata:
  name: drift-alert
spec:
  eventSources:
  - kind: Kustomization
    name: myapp-prod
  eventSeverity: error
  providerRef:
    name: slack
```

Sends alert to Slack: "Drift detected in myapp-prod."

---

## Interview Deep Dive

**Question**: "What's the difference between push-based and pull-based deployment?"

**Shallow answer**: "Pull-based uses Git as source of truth."

**Deep answer**:
"Push-based and pull-based differ fundamentally in security model, auditability, and operational resilience.

In push-based CI/CD, the pipeline has write access to the production cluster. When code is committed, CI builds an artifact and directly executes kubectl apply against the cluster. This requires storing cluster credentials in CI, creating a security risk - if CI is compromised, the attacker has cluster admin access. The audit trail is CI logs, which can be incomplete if the pipeline fails mid-deployment.

In pull-based GitOps, CI has no cluster access. CI builds the artifact and commits an updated manifest to Git (changing the image tag). A GitOps controller running inside the cluster continuously polls Git, detects the change, pulls the manifest, and applies it. Only the controller has cluster access, and it runs within the cluster's security perimeter. Cluster credentials never leave the cluster.

This provides superior security (no credentials in CI), better auditability (Git history is the complete deployment record), and drift detection (controller continuously reconciles cluster state to Git, reverting manual changes). The trade-off is operational complexity - you need to run and maintain the GitOps controller. But for production Kubernetes, the security and operational benefits typically outweigh the complexity."

**Question**: "ArgoCD vs. Flux - when would you choose each?"

**Deep answer**:
"Both are GitOps controllers but have different philosophies and ecosystems.

ArgoCD is batteries-included - it's a single installation with a built-in UI, CLI, and comprehensive Application CRD. It has strong health assessment, understanding resource-specific health (Deployments, Jobs, etc.) beyond just creation. The UI is excellent for visualization and troubleshooting. It supports multiple config management tools (Helm, Kustomize, Jsonnet) natively. ArgoCD is better when you want an all-in-one solution with minimal assembly required.

Flux is a modular toolkit - separate controllers for Git sources, Kustomize, Helm, notifications, image automation. It's more Kubernetes-native, using standard CRDs and relying on external UIs (Weave GitOps). Flux excels at image automation - automatically updating image tags in Git when new images are pushed, enabling fully automated CI/CD. It's better when you want composability and fine-grained control.

Choose ArgoCD if: You want a unified UI, need advanced RBAC with projects/teams, prefer a single tool that does everything, have multiple clusters to manage centrally.

Choose Flux if: You're already using Kustomize/Helm and want minimal abstraction, need image automation, prefer modular components, want a lighter weight solution, are comfortable with Kubernetes-native patterns.

Both handle the core GitOps workflow well - the choice often comes down to organizational preference for integrated vs. modular tools."

---

## Key Takeaways

1. **GitOps principles**: Declarative, versioned, pulled automatically, continuously reconciled
2. **Pull-based > Push-based**: Better security (no CI cluster access), auditability (Git history), drift detection
3. **ArgoCD**: Batteries-included, Application CRD, excellent UI, multi-cluster management
4. **Flux**: Modular toolkit, image automation, Kubernetes-native, composable controllers
5. **Multi-environment**: Directories, branches, or separate repos - each has trade-offs
6. **Secret management**: Sealed Secrets (encrypt in Git), External Secrets (sync from vault), SOPS (encrypt fields)
7. **Drift detection**: Auto-remediation (revert changes) or alert-only (manual review)

GitOps makes Git the single source of truth for your cluster state. Every change goes through Git (versioned, reviewed, auditable). Controllers continuously reconcile cluster to Git, preventing drift. This operational model provides security, auditability, and disaster recovery that traditional CI/CD cannot match.
