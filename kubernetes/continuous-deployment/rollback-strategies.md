# Rollback Strategies: When Things Go Wrong

## The Rollback Imperative

No matter how careful you are, bad deployments happen. The difference between a minor incident and a major outage is how quickly you can revert to a known-good state.

**The question isn't "will we need to rollback?" It's "how fast can we rollback?"**

---

## Kubernetes Native Rollback: kubectl rollout undo

### How It Works

Kubernetes keeps old ReplicaSets (scaled to 0) for rollback.

**Current state**:
```bash
kubectl get replicasets

NAME                DESIRED   CURRENT   READY   AGE
myapp-abc123def     0         0         0       2h    # Old (v1.0.0)
myapp-xyz789ghi     6         6         6       30m   # Current (v1.1.0)
```

**Rollback**:
```bash
kubectl rollout undo deployment/myapp
```

**What happens**:
1. Deployment controller finds previous ReplicaSet (myapp-abc123def)
2. Scales up old ReplicaSet (0 → 6)
3. Scales down current ReplicaSet (6 → 0)
4. Same rolling update process, just reversed

**Result**:
```bash
kubectl get replicasets

NAME                DESIRED   CURRENT   READY   AGE
myapp-abc123def     6         6         6       2h    # Restored
myapp-xyz789ghi     0         0         0       30m   # Rolled back
```

**Timeline**:
```
0:00 - kubectl rollout undo deployment/myapp
0:01 - Old ReplicaSet: 0 → 2 (maxSurge: 2)
0:02 - Old pods become ready
0:03 - Current ReplicaSet: 6 → 4
0:04 - Old ReplicaSet: 2 → 4
0:05 - Old pods become ready
0:06 - Current ReplicaSet: 4 → 2
0:07 - Old ReplicaSet: 4 → 6
0:08 - Current ReplicaSet: 2 → 0
0:09 - Rollback complete
```

Same duration as rollout (respects maxSurge and maxUnavailable).

### Rollout History

```bash
kubectl rollout history deployment/myapp

REVISION  CHANGE-CAUSE
1         Initial deployment (v1.0.0)
2         Update image to v1.1.0
3         Update image to v1.2.0
```

**Rollback to specific revision**:
```bash
kubectl rollout undo deployment/myapp --to-revision=1
```

Goes back to revision 1 (v1.0.0), skipping revision 2.

### Revision History Limit

```yaml
spec:
  revisionHistoryLimit: 10  # Keep last 10 ReplicaSets (default)
```

**Why limit?**
- Old ReplicaSets consume etcd space
- kubectl get rs clutter
- Too many = slower API queries

**Trade-off**:
- Higher limit: Can rollback further back
- Lower limit: Less resource usage

**Recommendation**: 10 (default) is reasonable. Can rollback 10 versions.

**Garbage collection**:
```bash
# Current ReplicaSet + 10 old ones = 11 total
# 11th oldest is deleted when 12th is created
```

---

## GitOps Rollback: Reverting Git

### Git Revert vs. Git Reset

**Scenario**: Deployed bad version via GitOps (ArgoCD/Flux).

**Option 1: git revert** (safe, creates new commit):

```bash
# Bad commit
git log

commit def456  (HEAD -> main)
  Update image to v1.2.0  # Bad version

commit abc123
  Update image to v1.1.0  # Good version

# Revert bad commit
git revert def456

commit ghi789  (HEAD -> main)
  Revert "Update image to v1.2.0"  # New commit undoing def456

commit def456
  Update image to v1.2.0

commit abc123
  Update image to v1.1.0
```

**What happens**:
- New commit (ghi789) reverses changes from def456
- Image tag changes: v1.2.0 → v1.1.0
- GitOps controller detects commit ghi789
- Applies change: Deployment rolls back

**Pros**:
- Preserves history (bad commit still visible)
- Safe for shared branches (main/master)
- Clear audit trail

**Cons**:
- Creates extra commit (clutter)

**Option 2: git reset** (destructive, rewrites history):

```bash
# Reset to good commit
git reset --hard abc123

git log

commit abc123  (HEAD -> main)
  Update image to v1.1.0  # Good version

# Bad commit (def456) is gone from history
```

```bash
# Force push (required after reset)
git push origin main --force
```

**Pros**:
- Clean history (bad commit deleted)
- No extra commits

**Cons**:
- **Rewrites history** (dangerous on shared branches)
- If others pulled bad commit, their history diverges
- Can break GitOps sync if controller cached commit

**Recommendation**:
- Use `git revert` on main/master (shared branches)
- Use `git reset` only on feature branches (single developer)

### GitOps Rollback Process

**ArgoCD**:

```bash
# Option 1: Sync to previous revision
argocd app sync myapp --revision abc123

# Option 2: Rollback via UI
# ArgoCD UI → Applications → myapp → History → Select previous revision → Rollback

# Option 3: Git revert (preferred)
git revert def456
git push
# ArgoCD auto-syncs (if automated sync enabled)
```

**Flux**:

```bash
# Option 1: Suspend reconciliation, manual apply
flux suspend kustomization myapp-prod
kubectl apply -f old-manifest.yaml

# Option 2: Git revert (preferred)
git revert def456
git push
# Flux auto-syncs

# Option 3: Update Kustomization to point to specific commit
kubectl edit kustomization myapp-prod
# Change spec.sourceRef.ref.commit to abc123
```

---

## Automated Rollback Triggers

### Trigger 1: Error Rate Threshold

**Rule**: If error rate > 5% for 5 consecutive minutes, rollback.

**Implementation** (Flagger):

```yaml
apiVersion: flagger.app/v1beta1
kind: Canary
metadata:
  name: myapp
spec:
  analysis:
    threshold: 5  # Allow 5 failed checks before rollback
    metrics:
    - name: request-success-rate
      thresholdRange:
        min: 95  # Success rate must be >= 95%
      interval: 1m
```

**How it works**:

```
Minute 1: Success rate = 96% ✓ (>= 95%)
Minute 2: Success rate = 94% ✗ (< 95%) - Fail count: 1
Minute 3: Success rate = 93% ✗ - Fail count: 2
Minute 4: Success rate = 92% ✗ - Fail count: 3
Minute 5: Success rate = 91% ✗ - Fail count: 4
Minute 6: Success rate = 90% ✗ - Fail count: 5

Action: ROLLBACK (5 consecutive failures)
  Set canary weight to 0%
  Scale canary ReplicaSet to 0
  Alert: "Canary failed - rolled back"
```

### Trigger 2: Latency SLO Violation

**Rule**: If P99 latency > 500ms, rollback.

```yaml
metrics:
- name: request-duration
  thresholdRange:
    max: 500  # P99 latency must be <= 500ms
  interval: 1m
```

### Trigger 3: Deployment Stuck

**Rule**: If deployment doesn't progress for 10 minutes, rollback.

**Kubernetes native** (progressDeadlineSeconds):

```yaml
spec:
  progressDeadlineSeconds: 600  # 10 minutes
```

**What happens**:

```
0:00 - Deployment starts rolling out
0:05 - New pods stuck in ImagePullBackOff (can't pull image)
...
0:10 - 10 minutes elapsed, no progress

Deployment condition:
  type: Progressing
  status: "False"
  reason: ProgressDeadlineExceeded
```

**Automated rollback** (external controller watching condition):

```python
# Pseudo-code
if deployment.status.conditions.progressing == False:
    if deployment.status.conditions.reason == "ProgressDeadlineExceeded":
        kubectl.rollout.undo(deployment)
        alert("Deployment stuck, rolled back")
```

### Trigger 4: Pod Crash Loop

**Rule**: If > 50% of pods restarting, rollback.

**Prometheus alert**:

```yaml
- alert: HighPodRestartRate
  expr: |
    (
      rate(kube_pod_container_status_restarts_total{deployment="myapp"}[5m]) > 0.1
    ) * 60  # More than 0.1 restarts/second = 6 restarts/minute
  for: 5m
  labels:
    severity: critical
  annotations:
    summary: "Pods crashing frequently, consider rollback"
```

**Webhook to rollback**:

```bash
# Alert receiver triggers webhook
POST /rollback/myapp
→ kubectl rollout undo deployment/myapp
```

---

## Rollback vs. Roll-Forward Decision Framework

Not every issue requires rollback. Sometimes rolling forward (fixing and redeploying) is better.

### When to Rollback

✅ **Rollback if**:
- **Immediate user impact**: Users experiencing errors/outages now
- **Unknown root cause**: Don't know what's wrong yet
- **Can't fix quickly**: Fix requires investigation/development (>30 min)
- **High blast radius**: Affecting large percentage of users
- **Degrading rapidly**: Situation getting worse

**Example**: Deployed v1.2.0, error rate jumped from 0.1% to 10%. Users complaining. Don't know why.

**Action**: Rollback to v1.1.0 immediately (5 minutes). Investigate v1.2.0 offline.

### When to Roll-Forward

✅ **Roll-forward if**:
- **Root cause known**: You know exactly what's wrong
- **Fix is trivial**: One-line config change
- **Can deploy quickly**: Fix + deploy < rollback time
- **Rollback has consequences**: Database migration can't be reverted

**Example**: Deployed v1.2.0, realized environment variable missing. Users see errors.

**Action**: Add environment variable, deploy v1.2.1 (5 minutes). Faster than rollback + investigating.

### Decision Tree

```
Issue detected
    ↓
Do you know root cause?
    ├── No → ROLLBACK
    └── Yes
        ↓
    Can you fix in < 15 minutes?
        ├── No → ROLLBACK
        └── Yes
            ↓
        Is rollback risky (DB migration, etc.)?
            ├── Yes → ROLL-FORWARD (fix and deploy)
            └── No
                ↓
            Is fix trivial and tested?
                ├── Yes → ROLL-FORWARD
                └── No → ROLLBACK (investigate offline)
```

### Example Scenarios

**Scenario 1: Image tag typo**

```
Deployed: image: myapp:1.2.0
Reality: Tag 1.2.0 doesn't exist in registry
Result: ImagePullBackOff, all pods stuck

Root cause: Obvious (typo in manifest)
Fix: Change to image: myapp:1.2.1 (correct tag)
Time to fix: 2 minutes

Decision: ROLL-FORWARD (fix typo, redeploy)
```

**Scenario 2: Memory leak**

```
Deployed: v1.2.0
Result: Pods gradually using more memory, OOMKilled after 30 minutes

Root cause: Unknown (memory leak somewhere)
Fix: Need profiling, investigation (hours)
Time to fix: Unknown

Decision: ROLLBACK to v1.1.0 (investigate offline)
```

**Scenario 3: Database migration breaking app**

```
Deployed: v1.2.0 with DB migration (added NOT NULL column)
Result: Old queries failing (column doesn't allow NULL)

Root cause: Known (migration incompatible)
Fix: Add default value to migration
Time to fix: 10 minutes

Decision: ROLL-FORWARD
Reason: Rollback requires reverting DB migration (risky, data loss)
        Roll-forward faster and safer
```

---

## Database Migration Rollback Challenges

Database migrations complicate rollback.

### The Problem

```
State 1: App v1.0.0, Database schema v1
State 2: App v1.1.0, Database schema v2 (migration ran)

Rollback app: v1.1.0 → v1.0.0
Problem: v1.0.0 app expects schema v1, but DB is schema v2
```

**If migration is backward-incompatible**: Old app breaks.

### Strategy 1: Backward-Compatible Migrations

**Make migrations non-breaking**:

```sql
-- BAD: Breaking migration
ALTER TABLE users DROP COLUMN email;

-- App v1.0.0 expects email column
-- After migration, v1.0.0 breaks: "column email does not exist"
-- Rollback app → broken

-- GOOD: Non-breaking migration
-- Step 1: Deploy v1.1.0 that doesn't use email column (but tolerates it)
-- Step 2: Wait days/weeks
-- Step 3: Run migration to drop column (once v1.0.0 is long gone)
ALTER TABLE users DROP COLUMN email;
```

**Additive migrations are safe**:

```sql
-- Safe: Adding column (with default)
ALTER TABLE users ADD COLUMN phone VARCHAR(20) DEFAULT '';

-- v1.0.0 (old app): Ignores phone column (doesn't select it)
-- v1.1.0 (new app): Uses phone column
-- Rollback app: v1.1.0 → v1.0.0 → Still works (column exists, unused)
```

**Safe migrations**:
- Add column (with default or nullable)
- Add table (old app doesn't query it)
- Add index (invisible to app)

**Unsafe migrations** (require careful planning):
- Drop column (old app expects it)
- Rename column (old app uses old name)
- Change column type (old app assumes old type)
- Add NOT NULL constraint (old app doesn't set it)

### Strategy 2: Dual-Write Pattern

**For breaking changes, support both old and new schema**:

```
Phase 1: Deploy v1.0.5 (supports both schemas)
  - Writes to old and new columns
  - Reads from old column (primary)

Phase 2: Run migration
  - Add new column
  - Backfill data from old to new column

Phase 3: Deploy v1.1.0 (reads from new column)
  - Still writes to both (for rollback safety)

Phase 4: Monitor for 1 week
  - If stable, proceed

Phase 5: Deploy v1.2.0 (drops old column support)
  - Stops writing to old column

Phase 6: Run cleanup migration
  - Drop old column
```

**Rollback safe at every phase**:
- Phase 1 → Phase 0: Easy (app didn't change schema)
- Phase 3 → Phase 2: Safe (app still writes to old column)
- Phase 4 → Phase 3: Safe (old column still exists)

### Strategy 3: Blue/Green Databases (Expensive)

**Use separate databases for blue/green**:

```
Blue:
  - App v1.0.0
  - Database v1 (schema v1)

Green:
  - App v1.1.0
  - Database v2 (schema v2, migrated from replica of v1)

Cutover: Switch traffic from blue to green
Rollback: Switch traffic back to blue (database untouched)
```

**Pros**: Perfect rollback (DB unchanged)
**Cons**: Expensive (2x databases), complex (data sync during cutover)

---

## Partial Rollback Strategies

Sometimes you don't want to rollback everything.

### Per-Region Rollback

**Scenario**: Deployed to 3 regions (us-east, us-west, eu-west). us-east has issues.

**Action**: Rollback only us-east, leave others on new version.

```bash
# Rollback us-east
kubectl --context us-east rollout undo deployment/myapp

# Leave us-west and eu-west on new version
```

**Use case**: Region-specific issues (network, data center, configuration).

### Per-Environment Rollback

**Scenario**: Deployed to staging and prod. Staging has issues.

**Action**: Rollback staging, pause prod rollout.

```yaml
# ArgoCD: Disable auto-sync for prod
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: myapp-prod
spec:
  syncPolicy:
    automated: {}  # Disable automated sync

# Manual: Hold prod deployment
# Rollback staging, investigate
```

**Use case**: Catch issues in staging before they hit prod.

### Feature Flag Rollback (Instant)

**Scenario**: New checkout flow causing issues. Deployed code, enabled via feature flag.

**Action**: Disable feature flag (seconds), code still deployed.

```bash
# Disable flag via API
curl -X PATCH https://feature-flags.example.com/flags/new-checkout \
  -d '{"enabled": false}'

# All users instantly see old checkout flow
# Code for new checkout still deployed, just not used
```

**Use case**: Fastest rollback (no deployment), can re-enable if false alarm.

---

## Interview Deep Dive

**Question**: "How do you rollback a Kubernetes deployment?"

**Shallow answer**: "kubectl rollout undo."

**Deep answer**:
"The rollback mechanism depends on deployment strategy and what triggered the need.

For native Kubernetes deployments, `kubectl rollout undo deployment/myapp` is the built-in mechanism. Kubernetes maintains old ReplicaSets scaled to 0. Undo reverses the rolling update - scales up the previous ReplicaSet while scaling down the current one. This respects the same maxSurge and maxUnavailable constraints, taking the same time as the initial rollout. You can rollback to specific revisions with `--to-revision=N`, but only within the revisionHistoryLimit (default 10).

For GitOps deployments, rollback happens through Git. The safest approach is `git revert <bad-commit>` which creates a new commit undoing the change. This preserves history and works well on shared branches. The GitOps controller (ArgoCD, Flux) detects the revert commit and reconciles the cluster to the previous state. Alternatively, you can use ArgoCD's UI to rollback to a previous synced state.

For progressive delivery with Flagger or Argo Rollouts, rollback can be automated based on metrics. If canary error rate exceeds the threshold, the controller automatically sets canary weight to 0%, effectively rolling back without manual intervention.

The rollback vs. roll-forward decision is critical. If root cause is known and the fix is trivial, rolling forward (deploying a fix) is often faster than rollback plus investigation. But if impact is severe, cause is unknown, or fix time is uncertain, rollback immediately to restore service, then investigate offline.

Database migrations add complexity - you can't always rollback schema changes. The solution is backward-compatible migrations or dual-write patterns where the app supports both old and new schema during transition, making rollback safe at any phase."

**Question**: "What makes a deployment difficult to rollback?"

**Deep answer**:
"Several factors can make rollback risky or impossible.

First, database migrations that are backward-incompatible. If the new version drops a column or changes a type, rolling back the application code doesn't undo the schema change. The old app code expects the old schema, but the database has the new schema. This breaks the rolled-back app. The solution is making migrations additive or using dual-write patterns.

Second, stateful changes that can't be reverted. If the new version processed user data in a way that's irreversible - deleted records, transformed data formats, sent notifications - rollback doesn't undo those actions. You've created persistent effects.

Third, external service integrations. If the new version called external APIs (sent emails, created resources, posted webhooks), rollback doesn't retract those calls. You may have triggered irreversible external state changes.

Fourth, distributed systems with version dependencies. If Service A (v2) calls Service B with a new API that Service B (v1) doesn't support, rolling back A to v1 doesn't help if B is already upgraded. You have a dependency ordering problem.

Fifth, long-lived ReplicaSet retention. If you set revisionHistoryLimit too low and rolled out several versions, the version you want to rollback to might have been garbage collected. You can still redeploy from source, but you've lost the instant rollback capability.

The way to mitigate these: Test rollback procedures in staging, design for backward compatibility, use feature flags to decouple code deployment from feature activation (can disable feature without redeploying), and maintain higher revisionHistoryLimit for critical services."

---

## Key Takeaways

1. **kubectl rollout undo**: Scales up old ReplicaSet, scales down current ReplicaSet
2. **Revision history**: Keep last N ReplicaSets, configurable via revisionHistoryLimit
3. **GitOps rollback**: git revert (safe, preserves history) vs. git reset (destructive, rewrites history)
4. **Automated triggers**: Error rate thresholds, latency SLOs, deployment stuck, crash loops
5. **Rollback vs. roll-forward**: Known + fixable quickly = roll-forward, unknown + high impact = rollback
6. **Database migrations**: Backward-compatible changes, dual-write pattern, blue/green databases
7. **Partial rollback**: Per-region, per-environment, feature flag (instant)

Rollback is insurance - you hope you never need it, but when you do, speed matters. Design deployments for easy rollback: keep old ReplicaSets, make migrations backward-compatible, use feature flags for instant disable. The best rollback is the one you never have to execute because you caught the issue in canary first.
