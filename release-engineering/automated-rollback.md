# Automated Rollback & Circuit Breakers

## The Fundamental Tension

When you deploy new code to production, you face a dilemma:

**Deploy conservatively**: Slow rollouts, extensive manual testing, humans watching dashboards. Safe, but slow. You ship features weeks late.

**Deploy aggressively**: Fast rollouts, automated gates, minimal human intervention. Fast, but risky. You might ship broken code to all users.

Automated rollback is the mechanism that lets you deploy aggressively while maintaining safety. It's the safety net that enables velocity.

The theory: **Deploy fast, detect problems faster, rollback fastest.**

## What Is Automated Rollback?

**Definition**: A system that detects degraded service health during a deployment and automatically reverts to the previous known-good version, without human intervention.

**The contract**:
1. You deploy a new version (canary, then gradual rollout)
2. Automated systems monitor health metrics
3. If metrics exceed failure thresholds → Automatic rollback
4. If metrics stay healthy → Continue rollout

**This enables**: High deployment velocity without proportional increase in risk.

## Why Automated Rollback Is Hard

You might think: "Just rollback if error rate increases." But in practice, it's much harder.

### Problem 1: Defining "Degraded"

What counts as degraded?
- Error rate goes from 0.1% to 0.2% → Is that bad enough to rollback?
- Latency p99 goes from 200ms to 250ms → 25% increase, but is it user-impacting?
- CPU utilization increases from 40% to 60% → Higher resource usage, but same user experience

There's no universal threshold. It's context-dependent:
- For a payment service, 0.1% error increase = thousands of failed transactions = rollback immediately
- For a recommendation widget, 0.1% error increase = some users don't see suggestions = maybe tolerable

**The challenge**: Codify business judgment into algorithmic thresholds.

### Problem 2: False Positives

Rolling back a good deployment is costly:
- Delays feature launch (business impact)
- Wastes engineering time (investigate false alarm)
- Erodes trust in automation (engineers override automation)

**Sources of false positives**:
- **Natural variance**: Metrics fluctuate. Error rate is 0.1% ± 0.05%. A spike to 0.15% might be noise, not a real problem.
- **External factors**: Downstream service is slow. Both old and new versions suffer, but new version gets blamed.
- **Cold start effects**: New version has empty caches. First 5 minutes look worse, then normalize.
- **Sample size**: With only 100 requests, 1 error = 1% error rate. Not statistically significant.

**The challenge**: Distinguish signal from noise.

### Problem 3: The Blast Radius vs. Detection Time Trade-off

**Small canary (1% of traffic)**:
- Pro: Small blast radius if deployment is bad
- Con: Takes longer to accumulate enough data to detect problems (low sample size)

**Large canary (50% of traffic)**:
- Pro: Faster detection (high sample size)
- Con: Large blast radius if deployment is bad

**Example**:
- At 1% canary with 10,000 RPS total → 100 RPS to canary
- If error rate is 1%, you see 1 error/second → Need 5+ minutes to confirm it's statistically significant
- Meanwhile, you've affected 300 requests

**The challenge**: Balance detection speed with blast radius.

### Problem 4: Rollback Isn't Always Possible

Some changes are hard to rollback:
- **Database schema changes**: If new version writes data in new format, old version can't read it
- **External API calls**: If new version published data to external system, rolling back doesn't undo that
- **Multi-service dependencies**: Service A depends on new feature in Service B. Rolling back A breaks it.

**The challenge**: Rollback requires forward/backward compatibility.

## Health Checks: The Foundation of Automated Rollback

Before you can rollback automatically, you need to know if a service is healthy. That's what health checks do.

### The Three Types of Health Checks (Kubernetes Model)

#### 1. Liveness Probe

**Question**: Is this instance alive (or should we kill it)?

**Purpose**: Detect when an instance is in a broken state that won't self-recover (deadlock, infinite loop, crashed but zombie process).

**Action on failure**: Kill the instance and restart it.

**Example**:
```yaml
livenessProbe:
  httpGet:
    path: /healthz
    port: 8080
  initialDelaySeconds: 30  # Wait 30s after start before checking
  periodSeconds: 10  # Check every 10s
  failureThreshold: 3  # Fail 3 times in a row before killing
```

**What the endpoint checks**:
```python
@app.route('/healthz')
def liveness():
    # Very basic check: Is the process running?
    # Don't check dependencies (DB, cache) - those are readiness, not liveness
    return {'status': 'ok'}, 200
```

**Key principle**: Liveness checks should be **cheap and fast**. Don't check external dependencies. If your DB is down, that's not a liveness issue (killing your app won't fix it).

**Anti-pattern**:
```python
# BAD: Liveness check depends on database
@app.route('/healthz')
def liveness():
    if database.is_connected():
        return {'status': 'ok'}, 200
    else:
        return {'status': 'error'}, 500
```

If DB is down, all instances fail liveness, Kubernetes kills them all, they restart, DB is still down, they fail again → death spiral.

#### 2. Readiness Probe

**Question**: Is this instance ready to serve traffic?

**Purpose**: Detect when an instance is temporarily unable to handle requests (dependencies down, warming up caches, processing backlog).

**Action on failure**: Remove from load balancer (stop sending traffic), but don't kill it. Re-add when it becomes ready.

**Example**:
```yaml
readinessProbe:
  httpGet:
    path: /ready
    port: 8080
  initialDelaySeconds: 10
  periodSeconds: 5
  failureThreshold: 2
```

**What the endpoint checks**:
```python
@app.route('/ready')
def readiness():
    # Check dependencies
    if not database.is_connected():
        return {'status': 'not ready', 'reason': 'database unavailable'}, 503

    if not cache.is_connected():
        return {'status': 'not ready', 'reason': 'cache unavailable'}, 503

    # Check if instance is overloaded
    if request_queue_depth() > 1000:
        return {'status': 'not ready', 'reason': 'queue overloaded'}, 503

    return {'status': 'ready'}, 200
```

**Key principle**: Readiness checks can be expensive. They check if the instance can handle requests right now.

**Use case**: During deployment, new instances might need to warm up (load caches, establish connection pools). Readiness probe keeps them out of rotation until ready.

#### 3. Startup Probe

**Question**: Has this instance finished starting up?

**Purpose**: Give slow-starting applications more time before liveness checks start.

**Problem solved**: Some applications take 60+ seconds to start (load large datasets, warm JVM). If liveness probe starts immediately, it kills the instance before it finishes starting.

**Example**:
```yaml
startupProbe:
  httpGet:
    path: /healthz
    port: 8080
  initialDelaySeconds: 0
  periodSeconds: 10
  failureThreshold: 30  # Allow up to 300s (5 minutes) to start
```

**Behavior**: While startup probe is failing, liveness probe is disabled. Once startup probe succeeds, liveness probe takes over.

### Using Health Checks for Rollback Decisions

**Scenario**: You deploy a canary. Some canary instances fail readiness checks.

**Interpretation**:
- If canary instances never become ready (readiness fails for 5+ minutes) → Deployment is broken, rollback
- If canary instances become ready after 2 minutes (warming up) → Deployment is fine, continue
- If baseline instances also fail readiness (DB is down) → External issue, don't blame canary

**The algorithm**:
```
Deploy canary
Wait for startup probes to pass (instance is running)
Wait for readiness probes to pass (instance is ready)
If readiness fails after 5 minutes → Rollback (deployment is broken)
If readiness passes → Start monitoring metrics (error rate, latency)
```

## Circuit Breaker Pattern: Fail Fast, Prevent Cascading Failures

The circuit breaker pattern is borrowed from electrical engineering. Just like a circuit breaker in your home trips when there's too much current (preventing fire), a software circuit breaker stops requests to a failing service (preventing cascading failures).

### The Problem: Cascading Failures

**Scenario**:
```
Service A → Service B → Service C (database)
```

Service C (database) becomes slow (overloaded). Service B's requests to C timeout (take 30 seconds). Service A's requests to B timeout. Now all three services are degraded.

**Why it cascades**:
- Service B has 100 worker threads
- Each request to C takes 30s (instead of 50ms)
- Workers are blocked waiting for C
- After 100 slow requests, all workers are blocked
- New requests to B queue up (or are rejected)
- Service B is now effectively down, even though it's "running"

This is a **thread exhaustion** failure mode. One slow dependency brings down the entire call chain.

### The Circuit Breaker Solution

**Idea**: If Service C is failing, stop sending requests to it. Fail fast (return error immediately) instead of waiting for timeout.

**States**:

1. **Closed** (normal operation):
   - Requests pass through to Service C
   - Track failure rate
   - If failure rate exceeds threshold → Open circuit

2. **Open** (failing fast):
   - Requests to Service C are blocked
   - Return error immediately (don't wait for timeout)
   - After timeout period (e.g., 30s) → Transition to Half-Open

3. **Half-Open** (testing recovery):
   - Allow a limited number of requests through (e.g., 10)
   - If they succeed → Close circuit (Service C is healthy again)
   - If they fail → Open circuit again (Service C still unhealthy)

**State diagram**:
```
       Closed
         ↓ (failures > threshold)
       Open
         ↓ (timeout expires)
      Half-Open
       ↙     ↘
    Closed   Open
 (success)  (failure)
```

### Implementation Example

```python
class CircuitBreaker:
    def __init__(self, failure_threshold=5, timeout=30):
        self.failure_threshold = failure_threshold
        self.timeout = timeout
        self.failure_count = 0
        self.state = 'closed'
        self.last_failure_time = None

    def call(self, func):
        if self.state == 'open':
            # Check if timeout has expired
            if time.time() - self.last_failure_time > self.timeout:
                self.state = 'half-open'
                self.failure_count = 0
            else:
                # Circuit is open, fail fast
                raise CircuitBreakerOpen("Service unavailable")

        try:
            result = func()
            # Success
            if self.state == 'half-open':
                self.state = 'closed'  # Service recovered
            self.failure_count = 0
            return result
        except Exception as e:
            # Failure
            self.failure_count += 1
            self.last_failure_time = time.time()

            if self.failure_count >= self.failure_threshold:
                self.state = 'open'

            raise e

# Usage
db_circuit_breaker = CircuitBreaker(failure_threshold=5, timeout=30)

def get_user_profile(user_id):
    try:
        return db_circuit_breaker.call(lambda: database.query(f"SELECT * FROM users WHERE id = {user_id}"))
    except CircuitBreakerOpen:
        # Fail fast, return cached data or error
        return get_cached_profile(user_id)
```

### Circuit Breakers for Rollback

**Use case**: New version of Service A calls Service B in a new way (higher QPS, different parameters). This overloads Service B.

**Without circuit breaker**: Service A keeps hammering Service B, both services degrade.

**With circuit breaker**: Circuit breaker detects Service B is failing, opens circuit, Service A fails fast (returns errors, but doesn't hang). Service B can recover.

**Rollback trigger**: If circuit breaker opens during canary deployment, that's a signal that the new version is causing problems. Automated rollback.

**The algorithm**:
```
Deploy canary
Monitor circuit breaker state for canary instances
If circuit breaker opens within first 10 minutes → Rollback (new version is overloading dependencies)
If circuit breaker stays closed → Continue rollout
```

### Bulkhead Pattern: Isolating Failures

Related to circuit breakers: **bulkheads** isolate failures to prevent total system collapse.

**Analogy**: A ship has bulkheads (watertight compartments). If one compartment floods, others stay dry. The ship doesn't sink.

**In software**: Separate thread pools for different dependencies.

**Without bulkheads**:
```
Service A has one thread pool (100 threads)
Calls to Service B and Service C both use this pool
Service C becomes slow (30s per request)
All 100 threads get blocked on Service C
Now Service A can't call Service B either (no free threads)
```

**With bulkheads**:
```
Service A has two thread pools:
- Pool 1: 50 threads for Service B
- Pool 2: 50 threads for Service C

Service C becomes slow
Pool 2's 50 threads get blocked
Pool 1 is unaffected, Service A can still call Service B
```

**Trade-off**: Lower resource utilization (threads are pre-allocated, can't share). But better isolation (one slow dependency doesn't bring down everything).

## Automated Rollback Triggers: When to Rollback

You've defined health checks, circuit breakers, and observability metrics. Now: when do you actually trigger a rollback?

### Trigger 1: Error Rate Threshold

**Metric**: `(failed_requests / total_requests) * 100`

**Threshold examples**:
- **Absolute threshold**: Error rate > 1%
- **Relative threshold**: Canary error rate > baseline error rate * 2 (doubling)
- **Delta threshold**: Canary error rate > baseline + 0.5% (absolute increase)

**Example rule**:
```
IF canary_error_rate > baseline_error_rate * 1.5
AND canary_sample_size > 1000 requests
AND duration > 5 minutes
THEN rollback
```

**Nuances**:
- **Sample size**: Don't rollback based on 10 requests (too noisy)
- **Duration**: Don't rollback on a 1-minute spike (could be transient)
- **Error severity**: Weight 5xx errors higher than 4xx (server errors vs. client errors)

### Trigger 2: Latency SLO Violation

**Metric**: p95 or p99 latency

**Threshold examples**:
- **Absolute**: p99 latency > 500ms
- **Relative**: Canary p99 > baseline p99 * 1.3 (30% slower)
- **SLO-based**: Canary p99 > SLO target (e.g., 200ms)

**Example rule**:
```
IF canary_p99_latency > 500ms
OR canary_p99_latency > baseline_p99_latency * 1.5
THEN rollback
```

**Nuances**:
- **Warm-up period**: Ignore first 5 minutes (caches are cold)
- **Traffic patterns**: Latency varies by time of day (compare same time windows)
- **Success vs. error latency**: Separate them (errors might fail fast, skewing latency down)

### Trigger 3: Saturation Limits

**Metric**: CPU, memory, disk I/O, network

**Threshold examples**:
- **Absolute**: CPU > 90%
- **Relative**: Canary CPU > baseline CPU * 2 (using twice as much)

**Example rule**:
```
IF canary_cpu_utilization > 80%
OR canary_memory_utilization > 85%
THEN rollback
```

**Nuances**:
- **Efficiency vs. resource usage**: Higher CPU might be okay if latency/errors are the same (new version is less efficient but functionally fine)
- **Autoscaling**: If autoscaler adds instances, that's a signal canary is more resource-intensive
- **Cost implications**: Higher resource usage = higher cloud costs

### Trigger 4: Health Check Failures

**Metric**: Readiness/liveness probe failure rate

**Threshold examples**:
- **Absolute**: > 10% of canary instances failing readiness
- **Duration**: Readiness fails for > 5 minutes

**Example rule**:
```
IF (failing_canary_instances / total_canary_instances) > 0.1
AND duration > 5 minutes
THEN rollback
```

**Nuances**:
- **Startup time**: Allow time for instances to start (don't rollback during first 2 minutes)
- **External dependencies**: If baseline instances also fail, it's not the deployment (DB is down)

### Trigger 5: Circuit Breaker Opens

**Metric**: Circuit breaker state changes from closed to open

**Threshold**: Any circuit breaker opening in canary instances

**Example rule**:
```
IF circuit_breaker_state == 'open' in canary
AND circuit_breaker_state == 'closed' in baseline
THEN rollback
```

**Interpretation**: Canary is overloading a dependency that baseline isn't. New version is problematic.

### Combining Triggers: Multi-Signal Rollback

Don't rely on a single signal. Combine multiple:

**Conservative (AND logic - all must be true)**:
```
IF canary_error_rate > threshold
AND canary_latency > threshold
AND duration > 10 minutes
THEN rollback
```

**Aggressive (OR logic - any can trigger)**:
```
IF canary_error_rate > threshold
OR canary_latency > threshold
OR circuit_breaker_opens
THEN rollback
```

**Weighted scoring**:
```
score = 0
if canary_error_rate > baseline * 2: score += 10
if canary_latency_p99 > baseline * 1.5: score += 5
if canary_cpu > baseline * 1.5: score += 3
if circuit_breaker_opens: score += 8

if score >= 15: rollback immediately
elif score >= 10: pause and alert
else: continue
```

**Best practice**: Use a combination. Critical signals (error rate, health checks) can trigger solo. Less critical signals (CPU usage) require additional confirmation.

## Rollback Mechanisms: How to Rollback

You've decided to rollback. Now: how do you actually do it?

### Mechanism 1: Traffic Shifting (Immediate)

**How it works**: Redirect traffic away from canary instances to baseline instances.

**Implementation**:
- **Load balancer weight change**: Set canary weight to 0%
- **Service mesh traffic routing**: Update routing rules (Istio, Linkerd)
- **DNS cutover**: Change DNS to point to old version (slow, DNS caching)

**Example (Kubernetes with Istio)**:
```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: checkout-service
spec:
  hosts:
  - checkout-service
  http:
  - route:
    - destination:
        host: checkout-service
        subset: v1.2.2  # Baseline
      weight: 100  # Rollback: Send 100% to old version
    - destination:
        host: checkout-service
        subset: v1.2.3  # Canary
      weight: 0  # Rollback: Send 0% to new version
```

**Characteristics**:
- **Speed**: Instant (seconds)
- **Complete**: All traffic immediately goes to old version
- **Reversible**: Can shift back if rollback was a mistake

**Use when**: You need immediate rollback (production outage).

### Mechanism 2: Scale Down Canary (Gradual)

**How it works**: Reduce canary instances to 0, leave baseline instances running.

**Implementation**:
- **Kubernetes**: Scale canary deployment to 0 replicas
- **Auto-scaling groups**: Set canary ASG desired count to 0

**Example (Kubernetes)**:
```bash
kubectl scale deployment checkout-service-canary --replicas=0
```

**Characteristics**:
- **Speed**: Moderate (minutes, as instances drain)
- **Graceful**: Existing connections complete before shutdown
- **Reversible**: Scale back up to retry

**Use when**: Canary is degraded but not critical (you can wait a few minutes for graceful shutdown).

### Mechanism 3: Version Pinning (Configuration Change)

**How it works**: Update deployment configuration to pin to old version, re-deploy.

**Implementation**:
- **Kubernetes**: Update image tag in deployment spec
- **VM-based**: Update configuration management (Ansible, Chef) to deploy old version

**Example (Kubernetes)**:
```yaml
# Before (canary rollout)
image: myapp:1.2.3

# After (rollback)
image: myapp:1.2.2
```

**Characteristics**:
- **Speed**: Slow (minutes to hours, depending on deployment pipeline)
- **Complete**: Old version becomes new desired state
- **Permanent**: Rollback is locked in (requires new deploy to move forward)

**Use when**: You want to ensure old version stays deployed (not just traffic shift, but actual instances).

### Mechanism 4: Feature Flag Toggle (Code Already Deployed)

**How it works**: New code is deployed, but behind a feature flag. Rollback = disable flag.

**Implementation**:
- **Feature flag service**: Set flag to `enabled=false`
- **Application**: Checks flag, falls back to old code path

**Example**:
```python
# Code (both old and new paths deployed)
if feature_flags.is_enabled("new_checkout_flow"):
    return new_checkout_flow()  # New version
else:
    return legacy_checkout_flow()  # Old version (rollback)

# Rollback: Disable flag
feature_flags.set("new_checkout_flow", enabled=False)
```

**Characteristics**:
- **Speed**: Instant (seconds, flag propagation)
- **Granular**: Can rollback for specific users (e.g., only premium users)
- **Zero downtime**: No redeployment needed

**Use when**: New code is already deployed, you're just toggling behavior. This is the fastest rollback mechanism.

### Comparison: Rollback Mechanisms

| Mechanism | Speed | Granularity | Reversibility | Use Case |
|-----------|-------|-------------|---------------|----------|
| Traffic shift | Instant | All traffic | Easy | Production outage, need immediate rollback |
| Scale down | Minutes | Instance-level | Easy | Gradual rollback, non-critical |
| Version pin | Slow (deploy) | Version-level | Requires deploy | Permanent rollback, lock in old version |
| Feature flag | Instant | User-level | Instant | Code already deployed, toggle behavior |

**Best practice**: Use feature flags for rapid rollback, traffic shift for emergency, version pinning for long-term stability.

## The Decision Algorithm: Rollback vs. Roll-Forward

When deployment issues occur, you have two options:

### Option 1: Rollback

**Definition**: Revert to the previous known-good version.

**Pros**:
- **Fast**: Get back to working state quickly
- **Safe**: Known-good version (has been running in production)
- **Low cognitive load**: Don't need to debug under pressure

**Cons**:
- **Delays feature**: Have to fix and re-deploy later
- **Doesn't fix root cause**: Issue will reoccur if you don't fix it

**When to choose rollback**:
- Error rate is high (> 1%, affecting many users)
- Root cause is unknown (don't know what's wrong)
- Fix will take time (can't patch in 10 minutes)
- Feature is non-critical (can wait for proper fix)

### Option 2: Roll-Forward (Hotfix)

**Definition**: Fix the issue in a new version, deploy that.

**Pros**:
- **Preserves progress**: Feature stays deployed (if partially working)
- **Fixes root cause**: Issue is resolved, not just deferred

**Cons**:
- **Slow**: Have to write code, test, deploy (20+ minutes)
- **Risky**: Hotfix might introduce new bugs (fixing under pressure)
- **Cognitive load**: Debugging in production is stressful

**When to choose roll-forward**:
- Error rate is low (< 0.5%, affecting few users)
- Root cause is known (clear bug, simple fix)
- Fix is simple (one-line change)
- Rollback is problematic (e.g., database schema changed, can't rollback)

### The Decision Matrix

```
                     Fix is simple     Fix is complex
                     ↓                 ↓
Error rate high  →   Roll-forward      Rollback
Error rate low   →   Roll-forward      Rollback
```

**Special case: Can't rollback** (database migration, external API change):
- Must roll-forward, even if complex
- This is why backward compatibility is critical (makes rollback possible)

### Example Decision Process

**Scenario**: Canary deployment shows 0.3% error rate (baseline: 0.1%). Errors are "database timeout."

**Analysis**:
1. **Severity**: 0.3% is low, but 3x baseline → Moderate severity
2. **Root cause**: Database timeouts → Suggests new version has slow query or higher QPS to DB
3. **Fix complexity**: Need to identify query, optimize, test → Complex (hours)
4. **Rollback feasibility**: No database schema changes → Rollback is safe

**Decision**: Rollback. Error rate is 3x baseline, fix is complex, rollback is safe.

**Post-rollback**:
1. Investigate in staging (why database timeouts?)
2. Fix issue (optimize query, add caching)
3. Test in staging
4. Re-deploy canary

## False Positive Prevention: The Hardest Problem

Automated rollback is only useful if it's accurate. Too many false positives → Engineers disable automation.

### Strategy 1: Statistical Significance Testing

Don't compare point estimates (canary error rate vs. baseline error rate). Compare distributions.

**Method**: Chi-squared test for error rates
```
Baseline: 1000 requests, 1 error (0.1% error rate)
Canary: 100 requests, 0 errors (0% error rate)

Is canary better? Or just lucky?
→ Run chi-squared test: p-value = 0.85 (not significant)
→ Canary sample size is too small, don't make decision yet
```

**Method**: T-test for latency
```
Baseline p99: 200ms ± 20ms (mean ± std dev)
Canary p99: 220ms

Is canary worse? Or within variance?
→ Run t-test: p-value = 0.12 (not significant at α=0.05)
→ Difference is within normal variance, don't rollback
```

**Threshold**: Only rollback if p-value < 0.05 (95% confidence that difference is real).

### Strategy 2: Minimum Sample Size

Don't make decisions on small sample sizes.

**Rule**: Require at least 1000 requests before comparing metrics.

**Example**:
```
Canary deployed, sees 50 requests in first minute
3 errors → 6% error rate
→ Don't rollback yet (sample size too small)

After 10 minutes, canary sees 1200 requests
15 errors → 1.25% error rate (baseline: 0.1%)
→ Now sample size is sufficient, and error rate is significantly higher
→ Rollback
```

### Strategy 3: Multiple Observation Windows

Don't rollback on a single 1-minute spike. Require sustained degradation.

**Rule**: Error rate must exceed threshold in 3 consecutive 5-minute windows.

**Example**:
```
Window 1 (10:00-10:05): Canary error rate 0.3% (baseline: 0.1%) ← Above threshold
Window 2 (10:05-10:10): Canary error rate 0.1% (baseline: 0.1%) ← Back to normal
→ Don't rollback (spike was transient)

Alternative:
Window 1: 0.3% error rate
Window 2: 0.4% error rate
Window 3: 0.35% error rate
→ Sustained degradation, rollback
```

### Strategy 4: Warm-Up Period

Ignore metrics during the first 5 minutes (cold caches, JIT warmup, connection pools).

**Rule**: Start monitoring at t+5 minutes after deploy.

**Example**:
```
Deploy canary at 10:00
Canary latency at 10:02: 800ms (baseline: 200ms) ← Looks bad
→ Don't rollback yet (warm-up period)

Canary latency at 10:07: 220ms (baseline: 200ms) ← Within variance
→ Canary is healthy after warm-up
```

### Strategy 5: Correlate with External Factors

Check if baseline is also degraded (indicates external issue, not canary).

**Rule**: Only rollback if canary is significantly worse than baseline AND baseline is healthy.

**Example**:
```
Canary error rate: 2%
Baseline error rate: 1.8%
→ Both are degraded, suggests database is down (external issue)
→ Don't rollback canary (it's not at fault)

Alternative:
Canary error rate: 2%
Baseline error rate: 0.1%
→ Only canary is degraded
→ Rollback (canary is the problem)
```

### Strategy 6: Human-in-the-Loop for Borderline Cases

When metrics are borderline (not clearly bad, but not clearly good), alert a human.

**Rule**:
- If error rate > 2x baseline → Auto-rollback (clearly bad)
- If error rate > 1.5x baseline → Alert human, pause rollout
- If error rate < 1.5x baseline → Continue (clearly good)

**Example**:
```
Canary error rate: 0.15% (baseline: 0.1%)
→ 1.5x baseline (borderline)
→ Pause rollout, alert on-call engineer
→ Engineer reviews logs, sees errors are expected (stricter validation)
→ Engineer approves rollout
```

## Human Override: Manual Controls

Automation is powerful, but humans need override capability.

### Break-Glass Rollback

**Purpose**: Emergency manual rollback when automation fails or is too slow.

**Implementation**:
- **Runbook**: "To rollback, run: `kubectl rollout undo deployment/checkout-service`"
- **Button in UI**: Rollback button in deployment dashboard (Spinnaker, Argo CD)
- **Chat ops**: Slack command `/rollback checkout-service`

**Example**:
```
10:30 AM: Automated rollback detects error rate spike, initiates rollback
10:31 AM: Rollback is in progress (draining canary instances)
10:32 AM: On-call engineer sees user reports of checkout failures
10:33 AM: Engineer clicks "Emergency Rollback" button
→ Immediately sets canary traffic to 0% (doesn't wait for graceful drain)
```

**Use when**: Automation is too slow, every second counts.

### Manual Approval Gates

**Purpose**: Require human approval before proceeding with rollout (for high-risk changes).

**Implementation**:
- **Deployment pipeline**: After canary is healthy for 30 minutes, pause and wait for approval
- **Slack notification**: "Canary is healthy. Approve to continue to 50%?"
- **Engineer reviews metrics, approves (or rejects)**

**Example (Argo Rollouts)**:
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: checkout-service
spec:
  strategy:
    canary:
      steps:
      - setWeight: 10
      - pause: {duration: 10m}  # Automated pause
      - pause: {}  # Wait for manual approval
      - setWeight: 50
```

**Use when**: High-risk deployments (payment systems, auth systems), want human review before full rollout.

### Disabling Automated Rollback

**Purpose**: Temporarily disable automation (e.g., during scheduled maintenance, known external issues).

**Implementation**:
- **Feature flag**: `automated_rollback_enabled=false`
- **Annotation in deployment**: `autoscaling.alpha.kubernetes.io/rollback-disabled: "true"`

**Example**:
```
# During database maintenance
SET automated_rollback_enabled = false
DEPLOY canary (will see higher error rates due to DB maintenance)
WAIT for DB maintenance to complete
RE-ENABLE automated_rollback_enabled
```

**Use when**: External factors will cause false positives (scheduled downtime, known issues).

## Post-Rollback: Incident Response and Learning

Rollback isn't the end - it's the beginning of learning.

### Immediate: Incident Response

**1. Verify rollback success**
- Error rate returns to baseline
- Latency returns to normal
- Health checks pass

**2. Communicate**
- Update status page: "Issue resolved, service restored"
- Notify stakeholders: "Deployed v1.2.3, encountered errors, rolled back to v1.2.2"
- Internal chat: "Rollback complete, investigating root cause"

**3. Preserve evidence**
- Save logs from canary instances (before they're deleted)
- Export metrics/traces for time window
- Capture error messages

### Short-term: Root Cause Analysis

**1. Reproduce in staging**
- Deploy the problematic version to staging
- Attempt to reproduce the issue
- If you can't reproduce, what's different in production?

**2. Analyze logs and traces**
- What errors occurred?
- Where in the code did they happen?
- What was different about failing requests?

**3. Identify root cause**
- Code bug (logic error, null pointer, etc.)?
- Performance issue (slow query, memory leak)?
- Dependency issue (new version overloaded downstream service)?
- Configuration issue (wrong environment variable)?

### Long-term: Process Improvement

**1. Why didn't we catch this earlier?**
- Missing test coverage?
- Staging environment doesn't match production?
- Load testing inadequate?

**2. Could we have detected it faster?**
- Missing observability (need more metrics, logs, traces)?
- Rollback threshold too permissive (should trigger sooner)?
- Sample size too small (should wait longer for statistical significance)?

**3. Could we have prevented it?**
- Better code review?
- More thorough testing?
- Canary should start smaller (1% instead of 10%)?

**4. Update runbooks**
- Document what went wrong
- Document how we fixed it
- Update rollback procedures if needed

### The Blameless Postmortem

**Key principle**: Focus on systems, not individuals.

**Bad**: "Engineer X deployed broken code."
**Good**: "Deployment passed all tests in staging, but failed in production due to data pattern that only exists in production. We need production-like data in staging."

**Template**:
```
# Incident: Rollback of v1.2.3 due to 3x error rate increase

## Timeline
- 10:30 AM: Deployed v1.2.3 canary (10% traffic)
- 10:35 AM: Error rate increased from 0.1% to 0.3%
- 10:40 AM: Automated rollback triggered
- 10:45 AM: Rollback complete, error rate returned to 0.1%

## Root Cause
New version introduced a database query without an index, causing timeouts under load.

## Impact
- 0.2% of users affected (approximately 500 users over 10 minutes)
- No data loss
- Service degraded but not unavailable

## Action Items
1. Add database query performance tests to CI (Owner: Alice, Due: 2025-12-15)
2. Add index to production database (Owner: Bob, Due: 2025-12-10)
3. Increase staging database size to match production (Owner: Charlie, Due: 2025-12-20)
4. Re-deploy v1.2.3 with fix (Owner: Alice, Due: 2025-12-12)

## Lessons Learned
- Staging database is smaller than production, didn't catch performance issue
- Missing query performance monitoring in observability
```

## Interview Deep Dive: Designing an Automated Rollback System

**Question**: "Design an automated rollback system for a high-traffic e-commerce site. What are the key components and decision points?"

**Your approach** (demonstrates senior-level thinking):

### 1. Define Success Criteria

"First, I'd clarify what 'successful deployment' means:

**For e-commerce, critical metrics**:
- **Checkout completion rate**: % of users who complete purchase
- **Payment success rate**: % of payment transactions that succeed
- **Product page load time**: Latency p95
- **Search result accuracy**: Click-through rate on search results

**Thresholds**:
- Checkout completion rate: Must stay above 95% (if it drops, revenue impact)
- Payment success rate: Must stay above 99.5% (failed payments = lost revenue + angry customers)
- Product page load time p95: Must stay under 1 second (slow pages = abandoned carts)

These are more specific than generic 'error rate' - they're tied to business outcomes."

### 2. Define Rollout Strategy

"I'd use a **progressive canary rollout**:

**Stage 1: Internal (1% traffic, employees only)**
- Duration: 30 minutes
- Purpose: Catch obvious bugs with real usage
- Rollback threshold: Any error in critical path

**Stage 2: Small canary (5% traffic)**
- Duration: 1 hour
- Purpose: Detect issues at low scale
- Rollback threshold: Checkout completion rate < 95% OR payment success rate < 99.5%

**Stage 3: Large canary (25% traffic)**
- Duration: 2 hours
- Purpose: Detect issues at higher scale (caching, DB load)
- Rollback threshold: Same as stage 2

**Stage 4: Full rollout (100%)**
- Duration: Indefinite
- Continued monitoring"

### 3. Define Health Checks

"I'd implement **three layers of health checks**:

**Liveness probe** (basic aliveness):
```python
@app.route('/healthz')
def liveness():
    # Just check if process is alive
    return {'status': 'ok'}, 200
```

**Readiness probe** (can serve traffic):
```python
@app.route('/ready')
def readiness():
    if not database.ping(): return 503
    if not cache.ping(): return 503
    if not payment_gateway.ping(): return 503
    return {'status': 'ready'}, 200
```

**Deep health check** (business logic):
```python
@app.route('/health/deep')
def deep_health():
    # Try to fetch a product (tests DB, cache, application logic)
    try:
        product = get_product(id=1)
        if not product: return 503
    except: return 503

    # Try to add to cart (tests session management)
    try:
        add_to_cart(product_id=1, user_id='health_check_user')
    except: return 503

    return {'status': 'healthy'}, 200
```

The deep health check runs periodically (every 60s) and alerts if it fails, but doesn't block traffic (only readiness probe does that)."

### 4. Define Rollback Triggers

"I'd use **multi-signal rollback** with weighted scoring:

**Critical signals** (auto-rollback immediately):
- Payment success rate < 99.5%
- Checkout completion rate < 95%
- Readiness probe failure rate > 10%

**Warning signals** (alert, don't auto-rollback):
- Product page latency p95 > 1.5s (but < 2s)
- Search latency p95 > 500ms (but < 1s)
- CPU utilization > 80%

**Scoring algorithm**:
```python
score = 0
if payment_success_rate < 99.5: score += 100  # Critical
if checkout_completion_rate < 95: score += 100  # Critical
if readiness_failure_rate > 0.1: score += 50
if page_latency_p95 > 2000: score += 30
if search_latency_p95 > 1000: score += 20
if cpu_utilization > 0.9: score += 10

if score >= 100: auto_rollback()
elif score >= 50: pause_and_alert()
else: continue_rollout()
```"

### 5. Define Rollback Mechanism

"For e-commerce, I'd use **feature flags + traffic shifting**:

**Feature flags** (for new features):
- New recommendation algorithm → Behind flag, instant disable
- New checkout UI → Behind flag, instant rollback to old UI

**Traffic shifting** (for infrastructure changes):
- Updated payment service → Shift traffic from v2 to v1
- New database query optimizer → Route traffic to old version

**Combined approach**:
```
Deploy v1.2.3 (new checkout UI behind flag)
Enable flag for 5% of users
Monitor metrics
If metrics degrade:
  1. Disable feature flag (instant, affects in-flight users)
  2. Shift traffic to old version (gradual, affects new requests)
  3. Scale down v1.2.3 instances (cleanup)
```"

### 6. False Positive Prevention

"To avoid unnecessary rollbacks:

**Minimum sample size**: Don't rollback until canary sees 10,000 requests (at 5% canary with 100K RPS, that's ~10 minutes).

**Warm-up period**: Ignore first 5 minutes of metrics (caches warming up).

**Statistical significance**: Use chi-squared test for success rates (p-value < 0.05 to rollback).

**External dependency check**: If baseline AND canary both degrade, it's external (database down, payment gateway slow). Don't blame canary.

**Human override**: If metrics are borderline (e.g., checkout rate is 94.8%, just below 95% threshold), pause and alert on-call engineer for decision."

### 7. Monitoring and Alerting

"I'd set up **three levels of alerts**:

**P0 (page on-call immediately)**:
- Automated rollback triggered
- Payment success rate < 99%
- Checkout down (5xx rate > 5%)

**P1 (alert, but don't page)**:
- Rollout paused (metrics borderline)
- Latency degraded (but users still completing checkouts)

**P2 (informational)**:
- Canary promoted to next stage
- Rollout completed successfully

**Dashboards**:
- Real-time canary vs. baseline comparison
- Business metrics (revenue, conversion rate)
- Technical metrics (latency, error rate, saturation)"

**This demonstrates**:
- You understand business context (e-commerce = revenue impact)
- You define metrics tied to outcomes (not just generic error rate)
- You balance velocity with safety (progressive rollout)
- You prevent false positives (statistical rigor)
- You have layered defenses (health checks, metrics, feature flags)
- You communicate effectively (alerting strategy)

## Summary: Automated Rollback as a Safety Net

Automated rollback is the mechanism that enables high-velocity deployment. Without it, you must deploy conservatively (slow rollouts, extensive manual testing). With it, you can deploy aggressively (fast rollouts, automated gates) while maintaining safety.

**Key mental models**:
1. **Deploy fast, detect faster, rollback fastest**: The three speeds of progressive delivery
2. **Health checks are contracts**: Liveness (is it alive?), Readiness (can it serve traffic?), Startup (has it finished starting?)
3. **Circuit breakers prevent cascades**: Fail fast when dependencies are down, prevent thread exhaustion
4. **Rollback vs. roll-forward**: Fast vs. fixes root cause - context-dependent decision
5. **False positives kill automation**: Over-sensitive rollback → Engineers disable it → Automation fails

**When interviewing**, be ready to:
- Design an automated rollback system (triggers, mechanisms, false positive prevention)
- Explain circuit breaker pattern (why it exists, how it prevents cascading failures)
- Discuss rollback vs. roll-forward decision matrix
- Debug a false positive rollback (what went wrong? how to prevent?)
- Connect automated rollback to business outcomes (velocity, reliability, error budgets)

Automated rollback isn't perfect - you'll have false positives, false negatives, and edge cases. But it's better than the alternative: manual deployments, slow rollouts, and production incidents. The goal isn't zero risk - it's optimal velocity within acceptable risk. Automated rollback is how you achieve that balance.
