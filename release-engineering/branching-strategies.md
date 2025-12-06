# Branching Strategies: How Code Flows Through Your Organization

## Why This Matters

Your branching strategy isn't just "how you use git." It's how your entire organization manages change, coordinates releases, and balances velocity against stability. The choice between GitFlow, Trunk-Based Development, or GitHub Flow fundamentally impacts:

- Release cadence (weekly? daily? continuously?)
- Integration risk (when do you discover conflicts?)
- CI effectiveness (are you actually integrating continuously?)
- Deployment complexity (how do you get code to production?)
- Team coordination overhead (how much process is required?)

## The Core Trade-Off

All branching strategies navigate the same fundamental tension:

**Isolation vs. Integration**

- **More isolation** (long-lived branches): Developers work independently, less conflict, but integration is deferred and becomes exponentially more complex
- **More integration** (short-lived branches, frequent merges): Conflicts surface early and are cheaper to resolve, but requires discipline and tooling

There's no perfect answer - just trade-offs optimized for different contexts.

---

## GitFlow: Traditional Enterprise Release Management

### The Model

```
main/master      ──────●─────────────●──────────────●──────────────●─────
                     v1.0.0        v1.1.0        v2.0.0        v2.0.1
                       ↑             ↑             ↑             ↑
                       │             │             │             │
release/*        ──────┴─────────────┴─────────────┴─────────────┴─────
                   release/1.0.0  release/1.1.0  release/2.0.0
                   (1.0.0-rc.1)   (1.1.0-rc.1)   (2.0.0-rc.1)
                       ↑             ↑             ↑
                       │             │             │
develop          ──────┴─────────────┴─────────────┴─────────────────────
                   1.0.0-dev.X    1.1.0-dev.X    2.0.0-dev.X    2.1.0-dev.X
                     ↑ ↑ ↑          ↑ ↑ ↑          ↑ ↑ ↑
                     │ │ │          │ │ │          │ │ │
feature/*        ────┴─┴─┴──────────┴─┴─┴──────────┴─┴─┴──────────────────
              feature/login  feature/search  feature/api-v2
              1.0.0-feat.X   1.1.0-feat.X    2.0.0-feat.X

hotfix/*                                                  ────●────
                                                      hotfix/2.0.1
                                                      (2.0.1-hotfix.1)
                                                            ↓
                                                    main + develop
```

**Version Flow Example:**

1. **v1.0.0 Release**:
   - `develop` at `1.0.0-dev.42` → create `release/1.0.0`
   - `release/1.0.0` builds tagged `1.0.0-rc.1`, `1.0.0-rc.2` (release candidates)
   - Merge to `main` → tag `v1.0.0`
   - `develop` moves to `1.1.0-dev.X` (next version)

2. **v1.1.0 Release** (minor version):
   - Features merged to `develop`: `feature/search` (1.1.0-feat.abc123)
   - `develop` at `1.1.0-dev.87` → create `release/1.1.0`
   - Stabilization: `1.1.0-rc.1`
   - Merge to `main` → tag `v1.1.0`

3. **v2.0.0 Release** (major version, breaking changes):
   - Feature with breaking change: `feature/api-v2`
   - `develop` at `2.0.0-dev.123` → create `release/2.0.0`
   - Stabilization: `2.0.0-rc.1`, `2.0.0-rc.2`
   - Merge to `main` → tag `v2.0.0`

4. **v2.0.1 Hotfix** (patch for production bug):
   - Critical bug found in `v2.0.0`
   - Create `hotfix/2.0.1` from `main` (v2.0.0)
   - Fix bug, builds tagged `2.0.1-hotfix.1`
   - Merge to `main` → tag `v2.0.1`
   - Also merge to `develop` (so fix not lost)

### Branch Types and Lifespans

**main/master** (permanent)
- Only production-ready code
- Every commit is tagged with a release version
- Direct commits forbidden
- Protected branch with required reviews

**develop** (permanent)
- Integration branch for ongoing development
- Features merge here first
- Always "next release" in progress
- Can be unstable

**feature/\*** (temporary, days to weeks)
- One feature per branch
- Created from: develop
- Merged to: develop
- Deleted after: merge to develop

**release/\*** (temporary, days to week)
- Release preparation and stabilization
- Created from: develop
- Merged to: main AND develop
- Only bug fixes, no new features
- Deleted after: released to main

**hotfix/\*** (temporary, hours to days)
- Emergency production fixes
- Created from: main
- Merged to: main AND develop
- Deleted after: fix deployed

### Typical Workflow

**Feature Development**:
```bash
# Start new feature
git checkout develop
git pull origin develop
git checkout -b feature/user-authentication

# Develop feature (days/weeks)
git add .
git commit -m "feat(auth): add OAuth2 login"

# When ready, merge to develop
git checkout develop
git pull origin develop
git merge feature/user-authentication
git push origin develop

# Delete feature branch
git branch -d feature/user-authentication
```

**Release Process**:
```bash
# Start release
git checkout develop
git checkout -b release/1.2.0

# Stabilization phase (bug fixes only)
git commit -m "fix(api): resolve race condition"
git commit -m "fix(ui): correct alignment"

# Release to production
git checkout main
git merge release/1.2.0
git tag v1.2.0
git push origin main --tags

# Merge fixes back to develop
git checkout develop
git merge release/1.2.0

# Delete release branch
git branch -d release/1.2.0
```

**Hotfix Process**:
```bash
# Critical bug in production!
git checkout main
git checkout -b hotfix/1.2.1

# Fix the bug
git commit -m "fix(payment): resolve transaction failure"

# Deploy fix
git checkout main
git merge hotfix/1.2.1
git tag v1.2.1
git push origin main --tags

# Also merge to develop so fix isn't lost
git checkout develop
git merge hotfix/1.2.1

# Delete hotfix branch
git branch -d hotfix/1.2.1
```

### Version Mapping

```bash
Branch                    → Version Pattern
─────────────────────────────────────────────────────
main                      → v1.2.3 (stable releases only)
develop                   → 1.3.0-dev.{BUILD_NUMBER}
feature/user-auth         → 1.3.0-feat-user-auth.{COMMIT_SHA}
release/1.3.0             → 1.3.0-rc.{BUILD_NUMBER}
hotfix/1.2.4              → 1.2.4-hotfix.{BUILD_NUMBER}
```

### When to Use GitFlow

**Ideal for**:
- **Scheduled release cycles**: Ship every 2 weeks, monthly, quarterly
- **Multiple versions in production**: Supporting v1.x, v2.x, v3.x simultaneously
- **Regulated environments**: Finance, healthcare (need audit trails, formal release process)
- **Large teams**: Need isolation between parallel workstreams
- **Non-continuous deployment**: Mobile apps, desktop software, embedded systems

**Characteristics**:
- Release Cycle: Weeks to months
- Stability: High (explicit stabilization phase in release branches)
- Parallel Work: Excellent (isolated feature branches)
- Complexity: High (multiple long-lived branches to manage)
- Integration Frequency: Low (features integrate to develop when complete)

### Problems with GitFlow

**Integration delay**: Features can diverge from develop for weeks. When they finally merge, conflicts are semantic, not just syntactic. "It worked in my branch" doesn't mean it works integrated.

**Not actually continuous integration**: You're running CI on feature branches, but that's continuous *building*. Integration only happens when merging to develop.

**Merge complexity**: The more branches diverge, the harder merging becomes. Release branches merging back to develop can introduce conflicts.

**Process overhead**: Managing the branch lifecycle (create release, merge to main, merge back to develop, delete) requires discipline.

---

## Trunk-Based Development (TBD): Continuous Integration Done Right

### The Model

```
main/trunk    ───●───●───●───●───●───●───●───●───●───●───●───●───●───●───●───
              commit commit commit commit commit commit commit commit commit
              abc123 def456 ghi789 jkl012 mno345 pqr678 stu901 vwx234 yza567
                │           │           │   │       │           │       │
                │           │           │   │       │           │       │
              [v1.0.0]   [v1.1.0]    [v1.2.0] │   [v2.0.0]   [v2.0.1]  │
              Tag on     Tag on      Tag on   │    Tag on    Tag on    │
              main       main        main     │    main      main      │
                                              │                        │
                                         (not tagged)            (not tagged)
                                         unreleased              unreleased

Version on main:
  Before v1.0.0:  1.0.0-dev.{BUILD}+{SHA}
  After v1.0.0:   1.1.0-dev.{BUILD}+{SHA}  (next version in development)
  After v1.1.0:   1.2.0-dev.{BUILD}+{SHA}
  After v1.2.0:   1.3.0-dev.{BUILD}+{SHA}
  After v2.0.0:   2.1.0-dev.{BUILD}+{SHA}

Feature flags ────────────────────────────────────────── (runtime toggles)
                     ↑       ↑       ↑       ↑       ↑
                     │       │       │       │       │
Short-lived      ──┐   ┌──┐   ┌──┐   ┌──┐   ┌──┐   ┌──┐
branches           └───┘  └───┘  └───┘  └───┘  └───┘  └───┘
(< 1-2 days)     feat/A feat/B feat/C feat/D feat/E feat/F
```

### Core Principles

1. **Single long-lived branch**: main/trunk. Everything else is temporary.
2. **Short-lived feature branches**: Hours to 2 days maximum. Merge quickly.
3. **Frequent integration**: Multiple merges to main per day.
4. **Feature flags**: Hide incomplete features at runtime, not in branches.
5. **Always releasable**: Main can be deployed to production at any time.
6. **Release from main**: Tag main when ready to release.

### How Tagging and Releases Work in TBD

In TBD, releases are simply **tags on main**. There's no separate release branch - you tag a commit on main when you're ready to release.

**The Flow:**

```bash
# Continuous commits to main
git commit -m "feat: add user search"        # commit abc123
git commit -m "fix: resolve pagination bug"  # commit def456
git commit -m "feat: add export feature"     # commit ghi789

# Ready to release? Tag main
git tag v1.1.0                               # Tag commit ghi789
git push origin main --tags

# Keep working
git commit -m "refactor: optimize queries"   # commit jkl012
git commit -m "feat: add filters"            # commit mno345

# Another release? Tag again
git tag v1.2.0                               # Tag commit mno345
git push origin main --tags
```

**Key Insight**: Not every commit becomes a release. You have continuous commits, but discrete releases (tags).

### Version Progression on Main

**Detailed Timeline:**

```
Commit    Version                         Action
──────────────────────────────────────────────────────────────────
abc123    1.0.0-dev.42+abc123            Working on next release
def456    1.0.0-dev.43+def456            Still developing
ghi789    1.0.0-dev.44+ghi789            Ready to release!
          ↓
          v1.0.0 (tag created)            ← RELEASE
          ↓
jkl012    1.1.0-dev.45+jkl012            Next version starts
mno345    1.1.0-dev.46+mno345            More features
pqr678    1.1.0-dev.47+pqr678            Ready to release!
          ↓
          v1.1.0 (tag created)            ← RELEASE
          ↓
stu901    1.2.0-dev.48+stu901            Next version
vwx234    1.2.0-dev.49+vwx234            Feature complete
yza567    1.2.0-dev.50+yza567            Ready to release!
          ↓
          v1.2.0 (tag created)            ← RELEASE
```

**Versioning Convention:**
- **Untagged commits**: `{NEXT_VERSION}-dev.{BUILD_NUMBER}+{COMMIT_SHA}`
- **Tagged commits**: `{VERSION}` (clean semver, no suffix)

### Release Decision Process

**When to tag a release?**

Option 1: **Time-based** (scheduled releases)
```bash
# Every Friday at 5pm
git tag v1.$(date +%U).0  # Week-based versioning
```

Option 2: **Feature-based** (when features complete)
```bash
# New feature complete and tested
git commit -m "feat: add payment processing"
git tag v1.3.0  # Tag immediately
```

Option 3: **On-demand** (release anytime)
```bash
# Customer needs urgent feature
git commit -m "feat: add custom reports"
git tag v1.2.1
# Deploy within minutes
```

### Detailed Release Workflow

**Step 1: Commit to Main**
```bash
# Developer merges feature to main
git checkout main
git pull origin main
git merge feature/user-export
git push origin main

# CI builds and tests
# Artifact: myapp:1.2.0-dev.47+abc123def
```

**Step 2: Decide to Release**
```bash
# Product owner: "Ship it!"
# Engineering: Tag main

git tag v1.2.0
git push origin main --tags
```

**Step 3: CI Detects Tag**
```yaml
# CI workflow triggers on tag
on:
  push:
    tags:
      - 'v*'

jobs:
  release:
    runs-on: ubuntu-latest
    steps:
      - name: Build release artifact
        run: |
          VERSION=${GITHUB_REF#refs/tags/}
          docker build -t myapp:${VERSION} .
          docker push myapp:${VERSION}

      - name: Update convenience tags
        run: |
          docker tag myapp:${VERSION} myapp:latest
          docker push myapp:latest

      - name: Deploy to production
        run: |
          kubectl set image deployment/myapp myapp=myapp:${VERSION}
```

**Step 4: Artifact Created**
```
myapp:1.2.0-dev.47+abc123def  (from commit before tag)
myapp:1.2.0                   (from tagged commit) ← RELEASE
myapp:latest                  (points to 1.2.0)
```

**Step 5: Continue Development**
```bash
# Immediately after tagging v1.2.0
git commit -m "feat: start next feature"

# New build version
# Artifact: myapp:1.3.0-dev.48+def456ghi
```

### Multiple Releases Per Day

TBD enables high-velocity releases:

```
9:00 AM:  Commit abc123 → merge to main
10:00 AM: Commit def456 → merge to main
11:00 AM: Tag v1.2.0 → RELEASE 1

1:00 PM:  Commit ghi789 → merge to main
2:00 PM:  Commit jkl012 → merge to main
3:00 PM:  Tag v1.2.1 → RELEASE 2

4:00 PM:  Commit mno345 → merge to main
5:00 PM:  Tag v1.2.2 → RELEASE 3

Three releases in one day, all from main
```

### Hotfix in TBD

**Traditional GitFlow**: Create hotfix branch, merge to main and develop
**TBD**: Just commit to main and tag

```bash
# Production bug discovered
git checkout main
git pull origin main

# Fix the bug
git commit -m "fix: resolve payment timeout"
git push origin main

# Tag as patch release
git tag v1.2.1  # Increment PATCH version
git push origin main --tags

# CI automatically deploys
# No special hotfix branch needed
```

**Why this works**: Main is always production-ready. A hotfix is just another commit → tag → deploy.

### Rollback in TBD

**Rollback = Deploy an older tag**

```bash
# Current production: v1.2.3 (broken)
# Previous version:   v1.2.2 (stable)

# Rollback: Just deploy the old tag
kubectl set image deployment/myapp myapp=myapp:1.2.2

# Or rollforward with a fix
git revert abc123  # Revert bad commit
git commit -m "fix: revert breaking change"
git tag v1.2.4
# Deploy v1.2.4
```

### Typical Workflow

```bash
# Start work (morning)
git checkout main
git pull origin main
git checkout -b quick-fix

# Small, focused change
git add .
git commit -m "fix(api): handle null response"

# Merge same day
git checkout main
git pull origin main
git merge quick-fix
git push origin main

# Delete branch immediately
git branch -d quick-fix

# Tag for release when ready
git tag v1.2.3
git push origin v1.2.3
```

### Feature Flags: The Key Enabler

Instead of long-lived branches, use runtime toggles:

```javascript
// Incomplete feature merged to main, but hidden
if (featureFlags.isEnabled('new-checkout-flow')) {
  return <NewCheckout />; // New code, behind flag
}
return <OldCheckout />; // Stable code path
```

**Progressive Rollout**:
```
Day 1: Merge to main, flag = OFF (0% of users)
Day 2: flag = internal-only (developers and QA)
Day 3: flag = 5% (canary users)
Day 4: flag = 25% (expanded rollout)
Day 5: flag = 100% (full release)
Week 2: Remove old code, deprecate flag
```

### Version Mapping

```bash
Context                   → Version Pattern
─────────────────────────────────────────────────────
main (untagged)           → 1.3.0-dev.{BUILD}+{SHA}
main (tagged)             → 1.3.0
feature branch            → 1.3.0-{SHORT_SHA}
```

**Key Difference from GitFlow**: No explicit alpha/beta/rc phases. Instead:
- Development versions: `-dev` suffix
- Released versions: No suffix
- Feature flags control exposure, not version numbers

### When to Use TBD

**Ideal for**:
- **Continuous deployment**: Deploy multiple times per day
- **SaaS applications**: Web apps, cloud services
- **Fast iteration**: Startup velocity, rapid experimentation
- **Strong automation**: Comprehensive tests, CI/CD maturity
- **Single production version**: Everyone on latest

**Characteristics**:
- Release Cycle: Continuous (can release anytime)
- Stability: High (main always releasable)
- Parallel Work: Moderate (small changes, quick merges)
- Complexity: Low (one main branch)
- Integration Frequency: Extremely high (multiple times per day)

### Requirements for TBD Success

**Fast, reliable CI**: If CI takes 1 hour, you can't merge multiple times per day. Target: < 10 minutes.

**Comprehensive automated testing**: You're deploying from main. Tests must catch regressions.

**Feature flag infrastructure**: Need tooling to toggle features per user/percentage/environment.

**Small batch sizes**: Changes must be incremental. No "two-week feature branches."

**Team discipline**: Everyone commits to frequent integration. No hoarding work.

### Problems with TBD

**Feature flag debt**: Flags are conditional complexity. Must have lifecycle management (creation → rollout → cleanup).

**Requires maturity**: TBD fails without fast CI, good tests, and disciplined team.

**Partial features in production**: Code for incomplete features exists in production (though gated by flags). Adds surface area.

---

## GitHub Flow: Simplified Continuous Deployment

### The Model

```
main          ───●───●───●───●───●───●───●───●───
               [v1.0] [v1.1] [v1.2] [v1.3]
                ↑  │   ↑  │   ↑  │   ↑  │
                │  ↓   │  ↓   │  ↓   │  ↓
PR branches   [PR1] [PR2] [PR3] [PR4]
```

### Core Principles

1. **Main is always deployable**: Production-ready at all times
2. **All work via Pull Requests**: No direct commits to main
3. **Deploy from main**: After PR merge, deploy
4. **Tag for releases**: Versioned releases are tags on main

### Typical Workflow

```bash
# Create branch
git checkout main
git pull origin main
git checkout -b fix-login-bug

# Make changes
git add .
git commit -m "fix(auth): resolve session timeout"
git push origin fix-login-bug

# Open PR (via web UI)
# → Code review
# → CI checks run
# → Approval required

# Merge to main
# (via web UI: squash and merge or merge commit)

# Deploy from main
# (often automated on merge)

# Tag if this is a versioned release
git tag v1.2.3
git push origin v1.2.3
```

### Version Mapping

```bash
Context                     → Version
─────────────────────────────────────────────────────
main (untagged commits)     → {NEXT_VERSION}-dev.{BUILD}
main (tagged)               → {TAG_VERSION}
PR #123                     → {NEXT_VERSION}-pr.123.{COMMIT_SHA}
```

### When to Use GitHub Flow

**Ideal for**:
- **Small to medium teams**: 2-20 developers
- **Web applications**: Can deploy continuously
- **Simple projects**: Don't need complex branching
- **Value simplicity**: Minimal process overhead

**Characteristics**:
- Release Cycle: Continuous
- Stability: High (main always deployable)
- Parallel Work: Good (via PRs)
- Complexity: Very Low (just main + PRs)
- Integration Frequency: High (PR per feature/fix)

### GitHub Flow vs TBD

**Similarities**:
- Both use single long-lived branch (main)
- Both enable continuous deployment
- Both keep main releasable

**Differences**:
- **GitHub Flow**: More structured (PRs required, code review enforced)
- **TBD**: Can commit directly to main (depends on team size/maturity)
- **GitHub Flow**: Branches live days (until PR approved)
- **TBD**: Branches live hours (merge quickly)

---

## Comparison Matrix

| Aspect | GitFlow | Trunk-Based | GitHub Flow |
|--------|---------|-------------|-------------|
| **Branch Lifetime** | Weeks/months | Hours/days | Days |
| **Long-Lived Branches** | 2+ (main, develop) | 1 (main) | 1 (main) |
| **Pre-Release Phases** | Explicit (release/\*, alpha/beta/rc) | Implicit (feature flags) | Minimal (tags) |
| **Release Frequency** | Scheduled (weeks/months) | Continuous (anytime) | Continuous |
| **Merge Complexity** | High (long-lived branches) | Low (short-lived branches) | Low |
| **Hotfix Process** | Dedicated hotfix/\* branches | Commit to main + tag | PR to main |
| **CI Integration** | Low (delayed to develop merge) | Very High (constant) | High |
| **Feature Isolation** | High (separate branches) | Low (flags) | Medium (PRs) |
| **Process Overhead** | High | Low | Medium |
| **Best For** | Enterprise, regulated | High-velocity teams | Small/medium teams |
| **Requires** | Formal process | Fast CI, automation | PR discipline |

---

## Decision Framework: Which Strategy Should You Use?

### Choose GitFlow When:

✅ Formal, scheduled release cycles (v1.0 ships June 1st)
✅ Multiple versions in production (enterprise customers on different versions)
✅ Regulatory/compliance requirements (need formal release audit trail)
✅ Large teams (50+ developers needing isolation)
✅ Non-continuous deployment (mobile apps, packaged software, embedded systems)
✅ Complex release coordination (multiple teams, dependencies)

❌ Don't use GitFlow if you want high deployment velocity or continuous delivery

### Choose Trunk-Based Development When:

✅ Continuous deployment capability (deploy multiple times per day)
✅ Fast iteration required (startup, competitive market)
✅ SaaS or web applications (control production environment)
✅ Strong automated testing (comprehensive test coverage)
✅ Mature DevOps practices (monitoring, feature flags, rollback)
✅ Single production version (everyone uses latest)

❌ Don't use TBD without fast CI (< 10 min), good tests, and team discipline

### Choose GitHub Flow When:

✅ Small to medium teams (2-20 developers)
✅ Web applications (can deploy continuously)
✅ Need simplicity (minimal process)
✅ Want enforced code review (PRs)
✅ Continuous deployment to single environment

❌ Don't use GitHub Flow for large teams (PR bottlenecks) or complex release coordination

---

## The Relationship Between Branching and CI

**GitFlow Sabotages CI**:
- Long-lived feature branches delay integration
- Running tests on feature branches is continuous *building*, not continuous *integration*
- Integration only happens when merging to develop (could be weeks later)
- By the time you integrate, semantic conflicts are expensive to fix

**TBD Enables CI**:
- Short-lived branches mean frequent integration
- Conflicts surface early when they're cheap to fix
- Fast CI is prerequisite (can't merge multiple times/day with slow CI)
- Main is always integrated, always tested

**Key Insight**: Your branching strategy determines whether you're doing actual continuous integration or just continuous building.

---

## Feature Flags: The Fourth Pillar

Feature flags transform how you think about releases:

**Traditional**: Branch → Develop → Stabilize → Deploy = Release
**With Flags**: Merge incomplete → Deploy → Enable progressively = Release

### Benefits

1. **Decouple deploy from release**: Code ships, features activate separately
2. **Progressive rollout**: 0% → internal → 5% → 100%
3. **Instant rollback**: Flip flag vs. re-deploy
4. **A/B testing**: Show different features to different users
5. **Kill switches**: Disable problematic features instantly

### Technical Debt Warning

Feature flags are conditional complexity:

```javascript
if (flag1 && flag2 && !flag3) {
  // New flow
} else if (flag1 && !flag2) {
  // Another flow
} else {
  // Old flow
}
```

This is combinatorial explosion. Must have lifecycle:
1. **Create**: Flag starts OFF
2. **Rollout**: Progressively enable
3. **Cleanup**: Remove flag and old code path

Flags that live > 6 months are technical debt.

---

## Common Questions and Considerations

**"Which branching strategy is best?"**

There is no universal "best" - only trade-offs optimized for context. Consider:
- Deployment model (continuous vs. scheduled)
- Team size and distribution
- Product type (SaaS vs. packaged software)
- Regulatory requirements
- Existing infrastructure and tooling

**"How often should teams merge to main?"**

This connects to the integration cost curve: longer divergence = exponential integration cost. Consider:
- CI speed (can't merge 10x/day with 1-hour CI)
- Test coverage (need confidence in automated checks)
- Team size (smaller teams can merge more frequently)
- Feature complexity (larger changes need more isolation)

**"How do you handle releases?"**

Options depend on branching strategy:
- **TBD**: Release from main (tag when ready)
- **GitFlow**: Release branches for stabilization
- **GitHub Flow**: Tag main after PR merge

Key decisions: Versioning automation, rollback plans, observability gates, approval processes.

---

**Remember**: There's no "best" branching strategy, only trade-offs optimized for your context. The goal is to choose the strategy that best balances your team's need for velocity, stability, and coordination.
