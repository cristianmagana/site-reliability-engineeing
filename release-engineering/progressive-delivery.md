# Progressive Delivery: Decoupling Deploy from Release

## The Fundamental Insight

**Traditional thinking**: Deploy = Release
- You deploy code → users immediately see it
- Binary decision: all users or no users
- High risk (everyone sees new version at once)

**Progressive delivery thinking**: Deploy ≠ Release
- You deploy code → users don't see it yet (feature flags OFF)
- Gradual decision: 0% → 1% → 5% → 25% → 100% of users
- Low risk (controlled, observable exposure)

```
Traditional:
Deploy v1.2.0  ────→  100% of users see v1.2.0 immediately

Progressive Delivery:
Deploy v1.2.0  ────→  0% see v1.2.0 (flag OFF)
               ────→  1% see v1.2.0 (internal users)
               ────→  5% see v1.2.0 (canary)
               ────→  25% see v1.2.0 (early adopters)
               ────→  100% see v1.2.0 (full release)
```

---

## Deploy vs. Release: The Core Distinction

### Deploy (Binary Placement)

**What it means**: Getting code onto servers
- Docker image pushed to registry
- Kubernetes deployment updated
- Instances running new version
- Code is **present** but not necessarily **active**

**Example**:
```bash
# Deploy
docker build -t myapp:1.2.0 .
docker push myapp:1.2.0
kubectl set image deployment/myapp myapp=myapp:1.2.0

# Code now running on servers, BUT...
# Users might not see it yet (feature flags can hide it)
```

### Release (Traffic Exposure)

**What it means**: Letting users experience the new version
- Feature flag enabled
- Traffic routed to new version
- Users see new functionality
- Code is **active** and **visible**

**Example**:
```javascript
// New checkout flow deployed but hidden
if (featureFlags.isEnabled('new-checkout-flow', user)) {
  return newCheckoutFlow();  // Only if flag enabled for this user
}
return oldCheckoutFlow();     // Most users still see this
```

### The Decoupling

```
Day 1 (Monday):
  Deploy: v1.2.0 deployed to production
  Release: 0% of users see it (flag = OFF for everyone)

Day 2 (Tuesday):
  Deploy: No change (still v1.2.0)
  Release: 1% of users see it (flag = ON for internal employees)

Day 3 (Wednesday):
  Deploy: No change
  Release: 5% of users see it (flag = ON for 5%)

Day 4 (Thursday):
  Deploy: No change
  Release: 25% of users see it

Day 5 (Friday):
  Deploy: No change
  Release: 100% of users see it (full release)

Week 2:
  Deploy: No change
  Release: Still 100%
  Code cleanup: Remove flag, delete old code path
```

**Key Point**: One deploy, five releases. Deploy is a one-time technical operation. Release is a gradual, business-driven process.

---

## Ring-Based Deployment

### The Concept

Organize users into concentric "rings" of increasing size and decreasing risk tolerance.

```
    ┌─────────────────────────────────────────┐
    │         Ring 5: Everyone                │
    │                                         │
    │   ┌──────────────────────────────────┐  │
    │   │    Ring 4: All Customers         │  │
    │   │                                  │  │
    │   │  ┌────────────────────────────┐  │  │
    │   │  │  Ring 3: Early Adopters    │  │  │
    │   │  │                            │  │  │
    │   │  │  ┌──────────────────────┐  │  │  │
    │   │  │  │  Ring 2: Canary      │  │  │  │
    │   │  │  │  (Beta Users)        │  │  │  │
    │   │  │  │                      │  │  │  │
    │   │  │  │  ┌────────────────┐  │  │  │  │
    │   │  │  │  │ Ring 1: Team   │  │  │  │  │
    │   │  │  │  │ (Internal)     │  │  │  │  │
    │   │  │  │  └────────────────┘  │  │  │  │
    │   │  │  └──────────────────────┘  │  │  │
    │   │  └────────────────────────────┘  │  │
    │   └──────────────────────────────────┘  │
    └─────────────────────────────────────────┘
```

### Ring Definitions

**Ring 0: Developers**
- Who: Engineering team who built the feature
- Size: 10-50 people
- When: Immediately after deploy
- Risk tolerance: High (they built it, can debug immediately)
- Monitoring: Manual testing, local debugging

**Ring 1: Internal Employees**
- Who: Entire company (non-engineering)
- Size: 100-1,000 people
- When: 6-24 hours after Ring 0
- Risk tolerance: Medium-high (internal users, forgiving)
- Monitoring: Bug reports via Slack, internal support tickets

**Ring 2: Canary Users**
- Who: Selected external users (beta program, early adopters)
- Size: 1-5% of user base
- When: 1-2 days after Ring 1
- Risk tolerance: Medium (opted into beta, expect occasional issues)
- Monitoring: Automated metrics (error rates, latency), support tickets

**Ring 3: Early Majority**
- Who: General users, but not all
- Size: 25-50% of user base
- When: 2-3 days after Ring 2 (if metrics good)
- Risk tolerance: Low (regular users expecting stable experience)
- Monitoring: Real-time dashboards, alerts, customer satisfaction

**Ring 4: Everyone**
- Who: 100% of users
- Size: All remaining users
- When: 3-7 days after Ring 3 (if no issues)
- Risk tolerance: Very low (mainstream users)
- Monitoring: Ongoing monitoring, long-term metric trends

### Ring Progression Example

**Feature: New search algorithm**

```
Day 1 (Monday 9am):
  Ring 0 (Developers): 10 engineers
  - Enable flag for engineering@company.com
  - Manual testing, debugging
  - Observations: Works well, performance good

Day 1 (Monday 5pm):
  Ring 1 (Internal): 500 employees
  - Enable flag for @company.com email domain
  - Slack channel for feedback
  - Observations: 2 edge cases reported, fixed same day

Day 2 (Tuesday 10am):
  Ring 2 (Canary): 5% of users (50,000 users)
  - Enable flag for 5% via random sampling
  - Monitor dashboards every hour
  - Metrics:
    - Search latency: P50=100ms (same as old), P99=300ms (same)
    - Error rate: 0.1% (acceptable)
    - Search success rate: 85% (up from 80%! ✅)

Day 3 (Wednesday 10am):
  Ring 3 (Early Majority): 25% of users (250,000 users)
  - Increase flag to 25%
  - Automated monitoring
  - Metrics still healthy

Day 5 (Friday 10am):
  Ring 4 (Everyone): 100% of users (1,000,000 users)
  - Enable flag for everyone
  - Continue monitoring

Week 2:
  - No issues, metrics stable
  - Remove feature flag code
  - Delete old search algorithm
```

### Benefits of Rings

**Graduated Risk**: Problems discovered in smaller, more forgiving rings
- Internal employees find bugs before customers
- Canary users are opted-in, expect issues
- By the time it reaches everyone, it's been battle-tested

**Early Feedback**: Internal users provide rich feedback
- Engineers can reproduce issues immediately
- Non-technical employees find UX problems
- Real-world usage patterns emerge

**Escape Hatches**: Can stop at any ring
- Ring 2 shows high error rate → stop, rollback, don't proceed to Ring 3
- Don't risk 100% of users when 5% already shows issues

**Organizational Alignment**: Different stakeholders care about different rings
- Engineering: Ring 0 (does it work?)
- Product: Ring 1-2 (is UX good?)
- Executive: Ring 3-4 (business metrics)

---

## Percentage-Based Rollouts

### How It Works

Gradually increase the percentage of users who see the new version.

```
0% ────→ 1% ────→ 5% ────→ 10% ────→ 25% ────→ 50% ────→ 100%
   15min   30min   1hr      2hr      4hr      overnight
```

### Implementation

**Consistent Hashing** (user always sees same version):

```javascript
function shouldShowNewFeature(userId) {
  const hash = murmurhash3(userId + "new-checkout-flow");
  const bucket = hash % 100;  // 0-99
  const rolloutPercentage = getFeatureFlagPercentage();

  return bucket < rolloutPercentage;
}

// Example:
// User A, hash % 100 = 5
//   - When rollout = 1%  → false (5 >= 1)
//   - When rollout = 10% → true (5 < 10)
//   - User A starts seeing feature at 10% rollout

// User B, hash % 100 = 87
//   - When rollout = 50% → false (87 >= 50)
//   - When rollout = 90% → true (87 < 90)
//   - User B starts seeing feature at 90% rollout
```

**Why consistent hashing?** User always sees same version. Prevents flickering:
- Bad: User sees old checkout, refreshes, sees new checkout, refreshes, sees old checkout
- Good: User sees old checkout until rollout reaches them, then always sees new checkout

### Rollout Schedule Examples

**Aggressive Rollout** (high confidence):
```
Time      Percentage    Users (of 1M)    Decision Gate
────────────────────────────────────────────────────────
0:00      0%            0                Deploy
0:15      1%            10,000           Monitor metrics
0:30      5%            50,000           Error rate good?
1:00      10%           100,000          Latency good?
2:00      25%           250,000          Business metrics?
4:00      50%           500,000          Support tickets?
8:00      100%          1,000,000        Full release
```

**Conservative Rollout** (new feature, higher risk):
```
Day       Percentage    Users            Decision Gate
────────────────────────────────────────────────────────
Day 1     0%            0                Deploy, internal testing
Day 2     0.1%          1,000            Initial canary
Day 3     1%            10,000           Metrics review
Day 4     5%            50,000           Business review
Day 7     10%           100,000          1 week soak test
Day 10    25%           250,000          Expand gradually
Day 14    50%           500,000          Mid-point check
Day 21    100%          1,000,000        Full release (3 weeks)
```

**Emergency Rollout** (critical bug fix):
```
Time      Percentage    Users            Decision Gate
────────────────────────────────────────────────────────
0:00      0%            0                Deploy fix
0:05      10%           100,000          Quick smoke test
0:15      50%           500,000          Error rate check
0:30      100%          1,000,000        Full release (30 min)
```

### Monitoring During Rollout

**Dashboard showing real-time comparison**:

```
New Version (25% of users)               Old Version (75% of users)
──────────────────────────────────────────────────────────────────
Error Rate:    0.15%  ⚠️  (+0.05%)        Error Rate:    0.10%
P50 Latency:   120ms  ✅  (same)          P50 Latency:   120ms
P99 Latency:   450ms  ⚠️  (+50ms)         P99 Latency:   400ms
Requests/sec:  2,500                      Requests/sec:  7,500
Success Rate:  99.85%                     Success Rate:  99.90%

Decision: ⚠️  WARNING - Slight degradation
Action: Pause rollout, investigate P99 latency increase
```

**Automated Decision Logic**:

```python
def should_continue_rollout(new_metrics, old_metrics):
    # Error rate threshold
    if new_metrics.error_rate > old_metrics.error_rate * 1.1:  # 10% increase
        return ROLLBACK, "Error rate increased by >10%"

    # Latency threshold
    if new_metrics.p99_latency > old_metrics.p99_latency * 1.2:  # 20% increase
        return PAUSE, "P99 latency increased by >20%"

    # Absolute error threshold
    if new_metrics.error_rate > 0.5:  # 0.5%
        return ROLLBACK, "Absolute error rate too high"

    # All good
    return PROCEED, "Metrics healthy"
```

---

## User Segmentation Strategies

### By User Attributes

**Geographic**:
```javascript
if (user.country === 'US' && rolloutConfig.regions.includes('US')) {
  return newVersion();
}
```

Use case: Roll out to US first, then EU, then Asia
- Different legal requirements per region
- Different usage patterns
- Can limit blast radius to one region

**By Plan Tier**:
```javascript
if (user.plan === 'enterprise' && rolloutConfig.tiers.includes('enterprise')) {
  return newVersion();
}
```

Use case: Enterprise customers first (more forgiving, better relationship)
- OR: Free users first (less revenue risk)

**By Account Age**:
```javascript
if (user.accountAge > 365 && rolloutConfig.includes('veteran_users')) {
  return newVersion();
}
```

Use case: Long-time users are more forgiving of changes
- OR: New users won't notice the change (no old habits)

**By Activity Level**:
```javascript
if (user.requestsLastWeek < 10 && rolloutConfig.includes('low_activity')) {
  return newVersion();
}
```

Use case: Low-activity users have less revenue impact

### By Risk Tolerance

**Opt-In Beta Users**:
```javascript
if (user.betaProgram === true) {
  return newVersion();  // Always show to beta users
}
```

**Power Users** (high engagement, more likely to report bugs):
```javascript
if (user.requestsPerDay > 100) {
  return newVersion();
}
```

**Internal Accounts** (can tolerate issues):
```javascript
if (user.email.endsWith('@company.com')) {
  return newVersion();
}
```

### Segmentation Example

**Feature: New pricing model (high risk)**

```
Week 1: Internal employees only
  - Segment: email.endsWith('@company.com')
  - Users: 500
  - Goal: Find obvious bugs

Week 2: Enterprise customers (best relationship)
  - Segment: plan === 'enterprise'
  - Users: 10,000
  - Goal: Get feedback from high-touch customers

Week 3: Beta opt-in users (volunteered for early access)
  - Segment: betaProgram === true
  - Users: 50,000
  - Goal: Broader feedback, real usage

Week 4: Long-time customers (loyal, forgiving)
  - Segment: accountAge > 730 (2 years)
  - Users: 200,000
  - Goal: Expand to stable, established users

Week 5: Everyone else
  - Segment: all remaining users
  - Users: 740,000
  - Goal: Full rollout
```

---

## Gradual Traffic Shifting

### Load Balancer Weighted Routing

**Nginx example**:

```nginx
upstream myapp {
    # Old version (v1.1.0)
    server v1-instance-1:80 weight=75;
    server v1-instance-2:80 weight=75;
    server v1-instance-3:80 weight=75;

    # New version (v1.2.0)
    server v2-instance-1:80 weight=25;
    server v2-instance-2:80 weight=25;
}

# Result: 75% traffic to v1, 25% traffic to v2
```

**Progressive shift**:

```
Hour 0:  weight=95 (v1), weight=5 (v2)   →  95% / 5%
Hour 1:  weight=90 (v1), weight=10 (v2)  →  90% / 10%
Hour 2:  weight=75 (v1), weight=25 (v2)  →  75% / 25%
Hour 4:  weight=50 (v1), weight=50 (v2)  →  50% / 50%
Hour 8:  weight=25 (v1), weight=75 (v2)  →  25% / 75%
Hour 12: weight=0 (v1),  weight=100 (v2) →  0% / 100%
```

### Service Mesh Traffic Shifting

**Istio example**:

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: myapp
spec:
  hosts:
  - myapp.example.com
  http:
  - match:
    - uri:
        prefix: /
    route:
    - destination:
        host: myapp
        subset: v1
      weight: 75    # 75% to v1.1.0
    - destination:
        host: myapp
        subset: v2
      weight: 25    # 25% to v1.2.0
```

**Gradual automation with Flagger**:

```yaml
apiVersion: flagger.app/v1beta1
kind: Canary
metadata:
  name: myapp
spec:
  analysis:
    interval: 5m
    threshold: 5
    maxWeight: 50
    stepWeight: 5    # Increase by 5% every interval

# What happens:
# 0:00  - 0% to v2 (deploy)
# 0:05  - 5% to v2 (metrics good, increase)
# 0:10  - 10% to v2
# 0:15  - 15% to v2
# ...
# 0:50  - 50% to v2 (maxWeight reached)
```

### Sticky Sessions

**Problem**: User request routed to v1, next request to v2
- Different session state
- Different cached data
- Inconsistent experience

**Solution**: Session affinity based on user ID

```nginx
upstream myapp {
    ip_hash;  # Route same IP to same server

    server v1:80 weight=75;
    server v2:80 weight=25;
}
```

**Or cookie-based**:

```nginx
map $cookie_version $backend {
    "v2"    v2-backend;
    default v1-backend;
}

server {
    location / {
        proxy_pass http://$backend;
    }
}
```

Once user sees v2, they always see v2 (even if rollout percentage decreases).

---

## Kill Switches: Instant Rollback Without Re-Deploy

### The Concept

Feature flag that can be flipped OFF immediately to disable a feature without deploying new code.

```
Normal state:
  Feature flag = ON for 100% users
  All users see new checkout flow

Problem detected (e.g., payment processor issues):
  Flip flag to OFF (takes 5 seconds)
  All users immediately see old checkout flow
  No deployment needed, no waiting for CI/CD
```

### Implementation

**Feature flag service** (centralized config):

```javascript
// Frontend fetches flags every 30 seconds
setInterval(async () => {
  const flags = await fetch('/api/feature-flags').then(r => r.json());
  window.featureFlags = flags;
}, 30000);

// Check flag before using feature
function getCheckoutFlow() {
  if (window.featureFlags['new-checkout-flow']) {
    return newCheckoutFlow();
  }
  return oldCheckoutFlow();
}
```

**Kill switch dashboard**:

```
Feature: new-checkout-flow
Status: 🟢 ENABLED (100% of users)

[  KILL SWITCH  ]  ← Big red button

If clicked:
  - Flag set to OFF immediately
  - All servers fetch new config within 30s
  - Users see old checkout flow
  - New deploy NOT required
```

### When to Use Kill Switches

**Payment Processing Issues**:
```
12:00 PM: New checkout deployed
12:05 PM: User reports: "Payment button doesn't work"
12:06 PM: Engineering investigates, confirms bug in new checkout
12:07 PM: [KILL SWITCH] Disable new-checkout-flow flag
12:08 PM: All users see old checkout (working)
12:30 PM: Fix deployed, tested
1:00 PM: Re-enable new-checkout-flow flag
```

**External Service Outage**:
```
New feature: Show weather in user dashboard (calls external weather API)

Weather API goes down → dashboards break
[KILL SWITCH] Disable weather-widget flag
Dashboards work again (without weather)
Weather API comes back → re-enable
```

**Performance Degradation**:
```
New feature: AI-powered recommendations

Rollout to 50% users
Database CPU spikes to 90% (AI queries expensive)
[KILL SWITCH] Disable AI recommendations
Database CPU drops to 30%
Optimize queries, re-enable gradually
```

### Kill Switch vs. Rollback

| Aspect | Kill Switch (Flag OFF) | Rollback (Re-Deploy) |
|--------|------------------------|----------------------|
| **Speed** | Instant (< 1 min) | Slow (5-20 min) |
| **Scope** | Feature-specific | All changes in deploy |
| **Code** | New code still running | Old code deployed |
| **Risk** | Low (just config change) | Medium (full deploy) |
| **Use Case** | Feature issues | Systemic issues |

**Example**:

Deploy contains:
- New checkout flow (feature flagged)
- Bug fix for search
- Performance improvement for API

**Scenario 1**: Checkout has issue
→ Use kill switch (disable checkout flag)
→ Search fix and API improvement still active

**Scenario 2**: API improvement causes outage
→ Use rollback (re-deploy old version)
→ Lose search fix and checkout improvement too

---

## Rollout Automation: Metrics-Driven Progression

### Automated Decision-Making

Instead of manual "looks good, proceed," automate based on metrics.

**Configuration**:

```yaml
rollout:
  feature: new-recommendation-engine
  stages:
    - percentage: 5
      duration: 30m
      success_criteria:
        - metric: error_rate
          threshold: < 0.5%
        - metric: p99_latency
          threshold: < 500ms
        - metric: recommendation_ctr
          threshold: > 10%

    - percentage: 25
      duration: 2h
      success_criteria:
        - metric: error_rate
          threshold: < 0.3%
        - metric: revenue_per_user
          threshold: >= baseline * 0.95  # No more than 5% drop

    - percentage: 100
      duration: infinity
```

**Automated Process**:

```
1. Deploy new recommendation engine (flag OFF)
2. Enable flag for 5% of users
3. Monitor for 30 minutes
4. Check: error_rate < 0.5%? ✅
   Check: p99_latency < 500ms? ✅
   Check: recommendation_ctr > 10%? ✅
5. Auto-promote to 25%
6. Monitor for 2 hours
7. Check: error_rate < 0.3%? ✅
   Check: revenue_per_user drop < 5%? ✅
8. Auto-promote to 100%
9. Monitor ongoing
```

**If any check fails**:

```
3. Check: error_rate < 0.5%? ❌ (actual: 0.7%)
4. Auto-rollback to 0%
5. Alert engineering team
6. Investigation required before retry
```

### Rollout Orchestration Tools

**Flagger** (Kubernetes-native):
- Integrates with Istio, Linkerd, App Mesh
- Progressive traffic shifting
- Automated rollback on metric degradation
- Webhooks for notifications

**LaunchDarkly** (SaaS feature flag platform):
- Centralized flag management
- User segmentation
- Gradual rollouts
- Kill switches
- A/B testing integration

**Split.io** (Feature delivery platform):
- Feature flags with metrics
- Real-time monitoring
- User targeting
- Impact analysis

**Harness** (Continuous delivery platform):
- Deployment verification
- Multi-cloud support
- Automated rollback
- Approval workflows

---

## Progressive Delivery Anti-Patterns

### Anti-Pattern 1: Deploy Without Flags

```
❌ Bad:
Deploy v1.2.0 → Everyone sees it immediately → Issues affect 100%

✅ Good:
Deploy v1.2.0 with flag OFF → Enable for 5% → Monitor → Expand
```

### Anti-Pattern 2: Long-Lived Feature Flags

```
❌ Bad:
Create flag "new-checkout" in 2023
Roll out gradually over weeks
Flag reaches 100%
Flag stays in code forever (never cleaned up)
6 months later: Code has 50 flags, nobody knows which are still relevant

✅ Good:
Create flag "new-checkout-2024-01"
Roll out over weeks
Reach 100%
2 weeks later: Remove flag from code, delete old code path
```

**Flag Lifecycle**:
```
Create → Roll out → Reach 100% → Monitor for stability → Clean up (remove flag)
```

### Anti-Pattern 3: Ignoring Metrics

```
❌ Bad:
Enable for 5% → "Looks fine" (didn't check metrics) → Enable for 100%
Actually: 5% had 2x error rate, but sample size too small to notice manually

✅ Good:
Enable for 5% → Check dashboard:
  - Error rate: 0.3% (new) vs 0.15% (old) = 2x increase ⚠️
  - Investigate before expanding
```

### Anti-Pattern 4: No Rollback Plan

```
❌ Bad:
Enable feature for 100% → Issues arise → "How do we disable this?"
Scramble to deploy old version → Takes 30 minutes → Users impacted

✅ Good:
Enable feature with kill switch → Issues arise → Flip switch (30 seconds)
```

### Anti-Pattern 5: Forgetting Database Compatibility

```
❌ Bad:
v1.2.0 changes database schema (removes column)
Deploy v1.2.0 with flag OFF
Enable flag for 50%
50% users use new code (no column)
50% users use old code (expects column) → ERROR!

✅ Good:
Step 1: Deploy v1.2.0 that writes to both old and new schema (dual-write)
Step 2: Enable feature flag gradually
Step 3: 100% on new code
Step 4: Deploy v1.3.0 that removes old schema support
Step 5: Run migration to drop old column
```

---

## Progressive Delivery Checklist

Before rolling out a feature:

- [ ] **Feature flag implemented**: Can enable/disable at runtime
- [ ] **Kill switch accessible**: Dashboard or CLI to disable instantly
- [ ] **Metrics defined**: What determines success/failure?
- [ ] **Monitoring dashboard**: Real-time comparison of new vs. old
- [ ] **Rollout plan**: 0% → 1% → 5% → 25% → 100% with time intervals
- [ ] **Success criteria**: Automated thresholds for each stage
- [ ] **Rollback plan**: How to disable if issues arise
- [ ] **Database compatibility**: New code works with old schema, old code works with new schema
- [ ] **User segmentation**: Who sees it first? (Internal → canary → everyone)
- [ ] **Cleanup plan**: When to remove flag code (after 100% + 2 weeks stable)

---

Progressive delivery transforms releases from risky, binary events into gradual, observable, controllable processes. Deploy the code, but control who sees it and when. Monitor every step. Rollback instantly if needed. Only proceed when data proves it's safe. This is how modern, high-velocity teams ship safely.
