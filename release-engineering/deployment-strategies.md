# Deployment Strategies: How to Safely Release Software

## The Fundamental Problem

You have a new version of your software. Users are currently using the old version. How do you transition from old to new without:
- Breaking user sessions
- Causing downtime
- Creating a bad user experience
- Making rollback difficult or impossible
- Risking the entire user base

Every deployment strategy is a solution to this problem, with different trade-offs.

---

## Rolling Updates: Incremental Replacement

### How It Works

Replace instances of the old version with the new version gradually, a few at a time.

```
Step 1: Old version running on all instances
[v1] [v1] [v1] [v1] [v1] [v1]

Step 2: Replace 2 instances
[v2] [v2] [v1] [v1] [v1] [v1]

Step 3: Replace 2 more
[v2] [v2] [v2] [v2] [v1] [v1]

Step 4: Replace final 2
[v2] [v2] [v2] [v2] [v2] [v2]

All instances now running v2
```

### The Process

**Configuration Example (Kubernetes)**:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  replicas: 6
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 2        # Can have 2 extra instances during rollout
      maxUnavailable: 1  # Can have 1 unavailable instance during rollout
  template:
    spec:
      containers:
      - name: myapp
        image: myapp:1.2.0  # New version
```

**What Happens**:

1. Kubernetes creates 2 new pods (v1.2.0) → Total: 8 pods (6 old + 2 new)
2. New pods become ready, pass health checks
3. Kubernetes terminates 2 old pods → Total: 6 pods (4 old + 2 new)
4. Repeat: Create 2 new, wait for ready, terminate 2 old
5. Continue until all pods are v1.2.0

### Timeline Example

```
Time    Old (v1.1.0)    New (v1.2.0)    Total    Traffic
──────────────────────────────────────────────────────────
0:00    6               0               6        100% v1.1.0
0:01    6               2 (starting)    8        100% v1.1.0
0:02    6               2 (ready)       8        75% v1.1.0, 25% v1.2.0
0:03    4               2               6        67% v1.1.0, 33% v1.2.0
0:04    4               4 (ready)       8        50% v1.1.0, 50% v1.2.0
0:05    2               4               6        33% v1.1.0, 67% v1.2.0
0:06    2               6 (ready)       8        25% v1.1.0, 75% v1.2.0
0:07    0               6               6        100% v1.2.0
```

### Pros

**Minimal Resource Overhead**: Only need a few extra instances during rollout (maxSurge)
- 6 instances → peak of 8 instances during rollout
- Much cheaper than running double infrastructure

**Gradual Rollout**: Issues affect progressively more users
- First few users see v1.2.0
- If it's broken, not everyone is affected immediately
- Can observe metrics as rollout progresses

**Zero Downtime**: Some instances always available
- maxUnavailable: 1 means at least 5 instances always running
- Users experience no service interruption

**Built-in to Orchestrators**: Kubernetes, ECS, Docker Swarm have native support
- No custom tooling required
- Well-tested, reliable implementation

### Cons

**Mixed Version State**: Old and new versions running simultaneously
- User A hits v1.1.0, User B hits v1.2.0
- If they interact, version inconsistencies possible
- Database schema must be compatible with both versions

**Gradual Problem Discovery**: Issues emerge slowly
- First 2 instances → 25% of users affected
- By the time you notice, maybe 50% rolled out
- More users impacted than with canary

**Slower Rollback**: Must roll backward instance by instance
- Not instant cutover back to old version
- Rollback takes similar time as rollout (minutes)

**Session Affinity Concerns**: Long-lived connections can break
- WebSocket connection to v1.1.0 instance
- That instance terminated
- Connection drops, user sees error

### When to Use Rolling Updates

✅ **Stateless services**: Each request independent, no session state
✅ **Backward compatible changes**: v1.2.0 can coexist with v1.1.0
✅ **Resource-constrained**: Can't afford double infrastructure
✅ **Frequent deployments**: Multiple per day, need fast turnaround
✅ **Standard web applications**: HTTP request/response pattern

❌ **Don't use** when you need instant rollback or versions can't coexist

---

## Blue/Green Deployment: Atomic Cutover

### How It Works

Run two complete environments. Switch traffic from old (blue) to new (green) all at once.

```
Step 1: Blue environment serving traffic
[Blue Environment]  ← 100% traffic
  [v1] [v1] [v1] [v1] [v1] [v1]

[Green Environment] ← 0% traffic
  (empty)

Step 2: Deploy to Green
[Blue Environment]  ← 100% traffic
  [v1] [v1] [v1] [v1] [v1] [v1]

[Green Environment] ← 0% traffic (testing)
  [v2] [v2] [v2] [v2] [v2] [v2]

Step 3: Switch traffic to Green
[Blue Environment]  ← 0% traffic (standby)
  [v1] [v1] [v1] [v1] [v1] [v1]

[Green Environment] ← 100% traffic
  [v2] [v2] [v2] [v2] [v2] [v2]

Step 4 (optional): Decommission Blue
[Green Environment] ← 100% traffic
  [v2] [v2] [v2] [v2] [v2] [v2]
```

### The Process

**Infrastructure Example (AWS ALB)**:

```yaml
# Blue target group (currently active)
BlueTargetGroup:
  - Instance 1 (v1.1.0)
  - Instance 2 (v1.1.0)
  - Instance 3 (v1.1.0)

# Green target group (new version)
GreenTargetGroup:
  - Instance 4 (v1.2.0)
  - Instance 5 (v1.2.0)
  - Instance 6 (v1.2.0)

# Load balancer listener
Listener:
  DefaultAction: Forward to BlueTargetGroup  # 100% to blue

# After cutover
Listener:
  DefaultAction: Forward to GreenTargetGroup  # 100% to green
```

**Deployment Steps**:

1. **Deploy to Green**: Create new environment with v1.2.0
2. **Test Green**: Smoke tests, validation, internal testing
3. **Cutover**: Change load balancer to route to Green
4. **Monitor**: Watch metrics for issues
5. **Keep Blue**: Standby for rollback (hours to days)
6. **Decommission Blue**: Once confident, tear down old environment

### Timeline Example

```
Time    Action                              Blue     Green    Traffic
────────────────────────────────────────────────────────────────────────
0:00    Green environment deployed          v1.1.0   v1.2.0   100% Blue
0:05    Testing Green internally            v1.1.0   v1.2.0   100% Blue
0:10    Smoke tests pass                    v1.1.0   v1.2.0   100% Blue
0:15    Switch load balancer to Green       v1.1.0   v1.2.0   100% Green
0:15    Monitor metrics                     v1.1.0   v1.2.0   100% Green
2:00    Metrics good, keep both running     v1.1.0   v1.2.0   100% Green
24:00   Decommission Blue (optional)        -        v1.2.0   100% Green
```

### Pros

**Instant Cutover**: Traffic switches from Blue to Green in seconds
- Load balancer config change
- All users on new version simultaneously
- No gradual migration

**Instant Rollback**: Switch back to Blue if issues detected
- Same load balancer config change
- Back to old version in seconds
- Blue environment still running and warm

**Full Testing Before Cutover**: Green environment can be tested in isolation
- Run full smoke tests
- Internal users test first
- Can even route small percentage for validation

**No Mixed Versions**: Either all users on Blue or all on Green
- No version inconsistency issues
- No "works for some users, not others" problems

**Zero Downtime**: Both environments can serve traffic
- Cutover is just routing change
- No instance termination during cutover

### Cons

**Double Infrastructure Cost**: Running two full environments
- Need 12 instances total (6 Blue + 6 Green)
- 2x compute, memory, database connections
- Expensive, especially at scale

**Resource Intensive**: Databases, caches, external services
- Do you duplicate databases? (expensive, complex)
- Or share databases? (complicates rollback)
- External service quotas (2x API calls during testing)

**Stateful Services Complexity**: Database migrations are challenging
- Schema must work for both Blue and Green
- Can't do breaking schema changes easily
- Dual-write patterns or backward compatibility required

**All-or-Nothing Risk**: Everyone switches at once
- If Green has a bug, everyone sees it immediately
- No gradual discovery like rolling update
- (Can mitigate with canary first, then Blue/Green)

### When to Use Blue/Green

✅ **Need instant rollback**: Regulatory, high-stakes deployments
✅ **Infrequent releases**: Cost of double infrastructure justified by infrequency
✅ **Monolithic applications**: Large, complex systems where rolling update risky
✅ **Critical services**: Downtime unacceptable, rollback must be instant
✅ **Pre-production testing**: Can test Green thoroughly before cutover

❌ **Don't use** when infrastructure cost is prohibitive or deploying many times per day

### Blue/Green Variants

**DNS Cutover**:
```
blue.example.com  → Old version
green.example.com → New version

www.example.com CNAME → blue.example.com   # Before
www.example.com CNAME → green.example.com  # After cutover
```

**Cons**: DNS caching delays (TTL), gradual rather than instant

**Database Blue/Green** (AWS RDS):
```
Blue DB  ← Blue App
Green DB ← Green App (with read replica of Blue)

Cutover:
1. Promote Green DB to primary
2. Switch app to Green
3. Blue DB becomes read replica
```

**Container Orchestrator** (Kubernetes):
```yaml
# Blue deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-blue
spec:
  replicas: 6

# Green deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-green
spec:
  replicas: 6

# Service switches between them
apiVersion: v1
kind: Service
metadata:
  name: myapp
spec:
  selector:
    app: myapp
    version: blue   # Change to 'green' for cutover
```

---

## Canary Releases: Progressive Rollout with Monitoring

### How It Works

Deploy new version to a small subset of instances/traffic first. Monitor metrics. Gradually increase if healthy.

```
Step 1: Canary to 10%
[v1] [v1] [v1] [v1] [v1] [v1] [v1] [v1] [v1] [v2]
                                               ↑ canary
90% traffic → v1
10% traffic → v2

Step 2: Monitor metrics for 15 minutes
Error rate good? Latency good? → Proceed

Step 3: Canary to 25%
[v1] [v1] [v1] [v1] [v1] [v1] [v1] [v2] [v2] [v2]
75% traffic → v1
25% traffic → v2

Step 4: Monitor again → Proceed

Step 5: Canary to 50%
[v1] [v1] [v1] [v1] [v1] [v2] [v2] [v2] [v2] [v2]
50% traffic → v1
50% traffic → v2

Step 6: Canary to 100%
[v2] [v2] [v2] [v2] [v2] [v2] [v2] [v2] [v2] [v2]
100% traffic → v2
```

### The Process

**Manual Canary (Kubernetes)**:

```bash
# Deploy canary with 1 replica (10% of 10 total)
kubectl apply -f deployment-canary.yaml  # 1 replica, v1.2.0

# Monitor for 15 minutes
watch 'kubectl top pods'
# Check error rates in metrics dashboard

# If good, scale canary up
kubectl scale deployment myapp-canary --replicas=3  # 30%

# Monitor again

# If good, replace main deployment
kubectl set image deployment/myapp myapp=myapp:1.2.0
kubectl delete deployment myapp-canary
```

**Automated Canary (Flagger + Istio)**:

```yaml
apiVersion: flagger.app/v1beta1
kind: Canary
metadata:
  name: myapp
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: myapp
  service:
    port: 80
  analysis:
    interval: 1m        # Check every minute
    threshold: 5        # Allow 5 failed checks
    maxWeight: 50       # Max 50% traffic to canary
    stepWeight: 10      # Increase by 10% each step
    metrics:
    - name: request-success-rate
      thresholdRange:
        min: 99         # Must maintain 99% success rate
    - name: request-duration
      thresholdRange:
        max: 500        # Max 500ms latency
```

**What Happens**:
1. Flagger creates canary deployment (v1.2.0)
2. Routes 10% traffic to canary
3. Waits 1 minute, checks metrics
4. If success rate > 99% and latency < 500ms → increase to 20%
5. Repeat until 50% or failure detected
6. If failure: Rollback automatically
7. If success: Promote canary to primary

### Progressive Rollout Timeline

```
Time    Canary %    Primary (v1)    Canary (v2)    Decision
────────────────────────────────────────────────────────────────
0:00    0%          10 instances    0              Deploy canary
0:01    10%         9               1              Check metrics
0:02    10%         9               1              ✅ Healthy → proceed
0:03    25%         8               2              Check metrics
0:04    25%         8               2              ✅ Healthy → proceed
0:05    50%         5               5              Check metrics
0:06    50%         5               5              ✅ Healthy → proceed
0:07    100%        0               10             Promote canary
```

**With Failure**:
```
Time    Canary %    Primary (v1)    Canary (v2)    Decision
────────────────────────────────────────────────────────────────
0:00    0%          10 instances    0              Deploy canary
0:01    10%         9               1              Check metrics
0:02    10%         9               1              ❌ Error rate spike!
0:03    0%          10              0              Rollback canary
```

### Pros

**Early Problem Detection**: Issues affect small percentage first
- 10% of users see the problem
- Not 100% (Blue/Green) or 50% (Rolling Update midpoint)
- Minimal blast radius

**Data-Driven Decisions**: Automated based on metrics
- Not "looks good to me"
- Actual error rates, latency, saturation
- Statistical significance

**Automatic Rollback**: No human intervention needed
- Metrics degrade → automatic rollback
- Can happen at 2am without waking anyone
- Fast response (minutes, not hours)

**Gradual Risk Increase**: Each step validates before proceeding
- 10% healthy → try 25%
- 25% healthy → try 50%
- Any step fails → stop and rollback

**Production Testing**: Real user traffic, real data
- Not synthetic tests
- Real load patterns
- Real edge cases

### Cons

**Complexity**: Requires sophisticated infrastructure
- Service mesh (Istio, Linkerd) or smart load balancer
- Metrics aggregation (Prometheus, Datadog)
- Automation tooling (Flagger, Argo Rollouts)

**Metrics Dependency**: Only as good as your metrics
- If you're not measuring the right thing, canary won't catch it
- Need comprehensive observability
- False positives can block good releases

**Slower Rollout**: Takes longer than Blue/Green
- Blue/Green: instant (seconds)
- Canary: gradual (15-30 minutes)
- Trade-off: safety vs. speed

**Small Sample Sizes**: 10% traffic might not catch rare bugs
- Bug occurs in 1% of requests
- 10% canary → only 0.1% of total users see it
- Might not be statistically significant
- Need sufficient traffic volume

### When to Use Canary

✅ **High traffic services**: Enough volume for statistical significance
✅ **Mature observability**: Comprehensive metrics, alerting
✅ **Risk-averse deployments**: Prefer gradual rollout to all-at-once
✅ **Automated operations**: Want hands-off deployments
✅ **Frequent releases**: Multiple per day, automation justified

❌ **Don't use** for low-traffic services (insufficient data) or without good metrics

### Canary Variants

**Percentage-Based Canary** (via load balancer weights):
```nginx
upstream myapp {
    server v1.myapp:80 weight=90;  # 90% to v1
    server v2.myapp:80 weight=10;  # 10% to v2 (canary)
}
```

**User-Based Canary** (via feature flag):
```javascript
if (user.isInternalEmployee || user.isEarlyAdopter) {
  return canaryVersion();  // New version
}
return stableVersion();    // Old version
```

**Geographic Canary**:
```
us-east-1:  100% v1.1.0 (stable)
us-west-2:  50% v1.1.0, 50% v1.2.0 (canary region)
eu-west-1:  100% v1.1.0 (stable)

If us-west-2 healthy → roll out to other regions
```

---

## Shadow Deployment: Running Both Versions in Parallel

### How It Works

Route real traffic to both old and new versions. Users only see old version's response. Compare responses for validation.

```
                    ┌─────────────┐
Real Request  ─────→│ Load        │
                    │ Balancer    │
                    └──────┬──────┘
                           │
              ┌────────────┴────────────┐
              │                         │
              ↓                         ↓
     ┌─────────────────┐       ┌─────────────────┐
     │  Primary (v1)   │       │  Shadow (v2)    │
     │                 │       │                 │
     │  Returns resp   │       │  Returns resp   │
     └────────┬────────┘       └────────┬────────┘
              │                         │
              │                         ↓
              │                   ┌──────────┐
              │                   │ Compare  │
              │                   │ & Log    │
              │                   └──────────┘
              ↓
        User receives
        response from v1
```

### The Process

**Implementation (with request mirroring)**:

```yaml
# Nginx config
location /api {
    proxy_pass http://primary-v1;

    # Mirror traffic to shadow
    mirror /mirror;
    mirror_request_body on;
}

location /mirror {
    internal;
    proxy_pass http://shadow-v2;
    proxy_set_header X-Shadow-Request "true";
}
```

**Traffic Flow**:
1. User request arrives
2. Primary (v1) processes request → response sent to user
3. Request duplicated to shadow (v2) → response logged, not sent to user
4. Compare v1 and v2 responses → log differences

### Comparison Logic

```python
# Pseudo-code for comparison service
def compare_shadow_to_primary(primary_response, shadow_response):
    differences = []

    # Compare status codes
    if primary_response.status != shadow_response.status:
        differences.append(f"Status: {primary_response.status} vs {shadow_response.status}")

    # Compare response times
    latency_diff = abs(primary_response.time - shadow_response.time)
    if latency_diff > 100:  # ms
        differences.append(f"Latency diff: {latency_diff}ms")

    # Compare response bodies (if deterministic)
    if primary_response.body != shadow_response.body:
        differences.append("Response body mismatch")

    # Log for analysis
    if differences:
        log.warning(f"Shadow divergence: {differences}")
        metrics.increment("shadow.divergence")
    else:
        metrics.increment("shadow.match")
```

### Pros

**Zero User Impact**: Shadow doesn't affect users
- Primary always serves users
- Shadow errors don't propagate
- Can fail spectacularly without consequences

**Real Traffic Validation**: Not synthetic tests
- Real user requests
- Real load patterns
- Real edge cases and data

**Performance Comparison**: Measure latency differences
- Is v2 faster or slower?
- Resource usage (CPU, memory)
- Database query patterns

**Gradual Confidence Building**: Run shadow for days/weeks
- Accumulate data over time
- Catch rare edge cases
- Statistical confidence before cutover

### Cons

**Double Resource Usage**: Running both versions
- 2x compute for request processing
- 2x database queries (unless read-only)
- 2x external API calls (unless mocked)

**Write Operations Complexity**: Can't shadow writes safely
- Duplicate POST/PUT/DELETE would duplicate data
- Solutions:
  - Only shadow read operations
  - Use read-replica for shadow
  - Mock external writes

**Non-Deterministic Responses**: Hard to compare
- Timestamps in responses differ
- Random IDs generated
- Need smart comparison logic to ignore expected differences

**Analysis Overhead**: Someone must review the data
- Automated comparison only catches some issues
- Need humans to analyze divergences
- Time investment required

### When to Use Shadow Deployment

✅ **Major architectural changes**: Complete rewrites, new tech stack
✅ **Performance validation**: Need to prove new version is faster
✅ **Risk-averse migrations**: Want maximum confidence before cutover
✅ **Read-heavy services**: Can safely duplicate read traffic
✅ **Long validation periods**: Can afford to run both for weeks

❌ **Don't use** for write-heavy services or when resource cost prohibitive

### Shadow Deployment Example

**Scenario**: Rewriting legacy PHP API in Go

```
Week 1: Deploy Go version as shadow
- 100% traffic to PHP (primary)
- 100% traffic mirrored to Go (shadow)
- Compare responses

Week 2: Analysis
- 95% response match
- 5% divergence (mostly timestamp formatting)
- Go is 3x faster

Week 3: Fix divergences
- Update Go to match PHP behavior exactly
- Re-deploy shadow

Week 4: Validation
- 99.9% match
- Remaining differences acceptable (whitespace, etc.)

Week 5: Cutover
- Switch primary to Go
- Keep PHP as shadow (inverted)
- Monitor for issues

Week 6: Decommission
- Go primary healthy
- Shut down PHP
```

---

## A/B Testing: Experimentation vs. Deployment

### How It Works

Route different users to different versions to measure business outcomes.

```
        ┌─────────────┐
User ──→│  Bucketing  │
        │  Algorithm  │
        └──────┬──────┘
               │
       ┌───────┴────────┐
       │                │
       ↓                ↓
  ┌─────────┐      ┌─────────┐
  │ A: v1   │      │ B: v2   │
  │ Control │      │ Variant │
  └─────────┘      └─────────┘
       │                │
       └────────┬───────┘
                ↓
         Measure Outcomes:
         - Conversion rate
         - Revenue per user
         - Engagement metrics
```

### The Distinction: A/B Testing ≠ Deployment Strategy

**Deployment Strategy**: Technical decision (is the system working?)
- Metrics: Error rates, latency, CPU usage
- Goal: Safely deploy new code
- Decision: Deploy or rollback

**A/B Testing**: Product decision (which version is better?)
- Metrics: Conversion, revenue, engagement
- Goal: Choose better product experience
- Decision: Ship A or B permanently

### A/B Testing Process

**Setup**:
```javascript
// User bucketing
function getBucket(userId) {
  const hash = murmurhash(userId);
  return (hash % 100) < 50 ? 'A' : 'B';  // 50/50 split
}

// Feature serving
function getCheckoutFlow(user) {
  const bucket = getBucket(user.id);

  if (bucket === 'A') {
    return oneStepCheckout();    // Control
  } else {
    return twoStepCheckout();    // Variant
  }
}

// Metrics tracking
trackEvent('checkout_started', {
  userId: user.id,
  bucket: bucket,
  timestamp: now()
});
```

**Analysis (after 2 weeks)**:
```
Variant A (one-step checkout):
- 10,000 users
- 1,200 completed purchases
- Conversion rate: 12%
- Revenue per user: $45

Variant B (two-step checkout):
- 10,000 users
- 1,400 completed purchases
- Conversion rate: 14%  ← 16.7% higher!
- Revenue per user: $52 ← 15.6% higher!

Decision: Ship variant B to everyone
```

### Pros

**Data-Driven Product Decisions**: Not opinions, actual user behavior
- Measure real business outcomes
- Statistical significance
- Remove guesswork

**Risk Mitigation**: Bad changes only affect 50% (or less)
- Not everyone sees the worse version
- Can stop experiment if dramatically bad

**Incremental Improvement**: Continuous experimentation culture
- Test many small changes
- Compound improvements over time

### Cons

**Requires Significant Traffic**: Need statistical significance
- Small changes need large sample sizes
- Low-traffic features can't be A/B tested effectively

**Long Duration**: Must run until significant
- Sometimes weeks or months
- Slows product velocity

**Complexity**: Bucketing, tracking, analysis infrastructure
- Need experimentation platform
- Data pipeline for metrics
- Statistical expertise

**Not for Technical Validation**: Measures business metrics, not system health
- Won't catch "v2 uses 2x CPU"
- Won't catch "v2 has occasional 500 errors"
- Still need deployment strategy (canary, etc.) for technical validation

### A/B Testing vs. Canary

**Different Goals**:

| Aspect | Canary Deployment | A/B Testing |
|--------|-------------------|-------------|
| **Goal** | Technical validation | Product validation |
| **Metrics** | Error rate, latency, CPU | Conversion, revenue, engagement |
| **Duration** | Minutes to hours | Days to weeks |
| **Decision** | Deploy or rollback | Ship A or B |
| **Automation** | Automated rollback | Manual analysis |
| **Traffic Split** | Gradual (10% → 25% → 50%) | Fixed (50/50 or 90/10) |

**Can Combine Them**:
1. Canary deploy v2 (technical validation)
2. If canary succeeds, run A/B test (product validation)
3. If A/B shows improvement, ship to 100%

---

## Comparison Matrix

| Strategy | Resource Cost | Rollback Speed | Blast Radius | Complexity | Best For |
|----------|---------------|----------------|--------------|------------|----------|
| **Rolling Update** | Low (minimal extra) | Medium (gradual) | Medium (gradual impact) | Low | Frequent deploys, stateless services |
| **Blue/Green** | High (2x infra) | Instant (seconds) | High (all users at once) | Medium | Critical services, infrequent releases |
| **Canary** | Low-Medium | Fast (automated) | Low (small % first) | High | High-traffic, mature observability |
| **Shadow** | High (2x processing) | N/A (no cutover) | None (users unaffected) | High | Major migrations, performance validation |
| **A/B Test** | Medium | N/A (experiment) | Medium (50% see variant) | High | Product experiments, business metrics |

---

## Choosing the Right Strategy

### Decision Tree

**Question 1: Can you afford double infrastructure?**
- No → Rolling Update or Canary
- Yes → Continue

**Question 2: Need instant rollback?**
- Yes → Blue/Green
- No → Continue

**Question 3: Have mature observability + high traffic?**
- Yes → Canary (automated, data-driven)
- No → Blue/Green or Rolling Update

**Question 4: Major architectural change?**
- Yes → Shadow (validate before cutover)
- No → Use deployment strategy above

**Question 5: Measuring business metrics?**
- Yes → A/B Test (after technical deployment succeeds)
- No → Use deployment strategy above

### Common Combinations

**Startup (limited resources, high velocity)**:
- Rolling Updates for most deploys
- Occasional Blue/Green for critical releases

**Mature Tech Company (high traffic, mature ops)**:
- Canary for all production deploys (automated)
- Shadow for major architecture changes
- A/B tests for product experiments

**Enterprise (risk-averse, infrequent releases)**:
- Blue/Green for production
- Extensive testing in staging first

**E-commerce (high stakes, business metrics)**:
1. Canary deploy (technical validation)
2. A/B test (business metric validation)
3. Full rollout

---

## Infrastructure Requirements

### For Rolling Updates
- Container orchestrator (Kubernetes, ECS, Docker Swarm)
- Health checks (readiness, liveness probes)
- Load balancer with dynamic instance discovery

### For Blue/Green
- Two complete environments (blue and green)
- Load balancer or DNS with instant cutover capability
- Deployment automation to manage both environments

### For Canary
- Service mesh (Istio, Linkerd, App Mesh) or smart load balancer
- Metrics aggregation (Prometheus, Datadog, New Relic)
- Automation tooling (Flagger, Argo Rollouts, Spinnaker)
- Alerting system for automated rollback

### For Shadow
- Request mirroring capability (nginx, envoy)
- Response comparison service
- Separate infrastructure for shadow (can share databases if read-only)
- Logging and analysis tools

### For A/B Testing
- User bucketing system (consistent hashing)
- Feature flag platform
- Metrics pipeline (event tracking → data warehouse)
- Statistical analysis tools

---

Deployment strategies are about managing risk. The right strategy depends on your traffic volume, infrastructure budget, operational maturity, and risk tolerance. Most mature organizations use a combination: canary for standard deploys, blue/green for critical releases, shadow for major migrations, and A/B tests for product decisions.
