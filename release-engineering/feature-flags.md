# Feature Flags & Runtime Configuration

## The Fundamental Problem

Traditionally, software has two knobs:
1. **What code exists** (controlled by deployment)
2. **How that code behaves** (controlled by configuration files, environment variables)

This creates a rigid contract: to change behavior, you must change code and redeploy. Feature flags add a third dimension: **which code paths execute**, controlled at runtime without redeployment.

This isn't just convenience - it fundamentally changes the economics and risk profile of software delivery.

## Why Feature Flags Exist: The Deploy/Release Decoupling

### The Traditional Model: Deploy = Release

In the classic deployment model:
```
Code merged → CI builds artifact → Artifact deployed → Users see feature
```

This coupling creates several problems:

**Deployment as a risky event**: Every deploy changes user-visible behavior. If something goes wrong, you must redeploy (rollback) or hotfix-forward. Both take time and carry risk.

**Binary decisions**: A feature is either live for everyone or no one. No gradual rollout, no easy A/B testing, no "try this for internal users first."

**Merge conflicts with velocity**: If you want to deploy Service A's bug fix but Service B's new feature isn't ready, you're stuck. This encourages long-lived feature branches, which sabotages continuous integration.

### The Feature Flag Model: Deploy ≠ Release

With feature flags:
```
Code merged → CI builds → Deployed to prod → Feature hidden behind flag
                                           → Flag enabled gradually
                                           → Users see feature incrementally
```

**Deployment becomes low-risk**: You're just placing new binary code on servers. It won't execute until the flag flips. Deployment is now a technical operation, not a business decision.

**Release becomes a configuration change**: Enabling a feature is changing a boolean value. Instant rollback (flip the flag off). No redeployment required.

**Continuous integration becomes feasible**: Merge incomplete features to main, hidden behind flags. Deploy anytime. Release when ready.

## The Theoretical Foundation: State Management

Feature flags are fundamentally about **where you store state that controls program behavior**.

### Configuration Hierarchy

Software behavior is controlled by state at different layers:

1. **Compile-time constants**: Hardcoded in source (`const MAX_CONNECTIONS = 100`)
   - Change requires recompilation and redeployment
   - Zero runtime cost
   - No flexibility

2. **Build-time configuration**: Injected during build (build flags, preprocessor directives)
   - Change requires rebuild and redeployment
   - Zero runtime cost
   - Different builds for different environments

3. **Deploy-time configuration**: Environment variables, config files
   - Change requires redeployment (or container restart)
   - Minimal runtime cost
   - Different config per environment

4. **Runtime configuration**: Feature flags, dynamic config
   - Change requires only a configuration update
   - Some runtime cost (lookup overhead)
   - Can change while process is running

Feature flags push decisions from deploy-time to runtime. This trades performance (now you have to check a flag) for flexibility (you can change behavior instantly).

### The CAP Theorem for Feature Flags

Just like distributed databases face CAP theorem constraints (Consistency, Availability, Partition tolerance - pick 2), feature flag systems face trade-offs:

**Latency vs. Consistency vs. Availability**:
- **Low latency**: Cache flag values in application memory (but risk stale data)
- **Strong consistency**: Query flag service every evaluation (but adds latency)
- **High availability**: Embed fallback defaults (but risk divergence if flag service is down)

You cannot optimize all three simultaneously. Your architecture choice depends on your use case.

## Feature Flag Patterns: Different Problems, Different Tools

Not all feature flags are created equal. They solve different problems and have different lifecycle characteristics.

### 1. Release Toggles (Release Flags)

**Purpose**: Decouple deployment from release. Hide incomplete features in production.

**Lifecycle**: Temporary. Should be removed once feature is fully released.

**Example**:
```python
if feature_flags.is_enabled("new_checkout_flow", user):
    return new_checkout_flow(cart)
else:
    return legacy_checkout_flow(cart)
```

**Characteristics**:
- Short-lived (days to weeks)
- Binary: on or off
- User-targeted (rollout to 10% of users, then 50%, then 100%)
- Should be cleaned up after full rollout

**Interview question**: "How do you prevent release flag accumulation?"
- Automated tracking (flag age, usage metrics)
- CI checks that fail if flags are too old
- Required cleanup as part of "done" definition
- Dedicated sprint for technical debt

### 2. Experiment Toggles (A/B Test Flags)

**Purpose**: Run controlled experiments. Measure impact of changes on user behavior.

**Lifecycle**: Temporary. Removed once experiment concludes and decision is made.

**Example**:
```python
if feature_flags.get_variant("checkout_button_color", user) == "variant_b":
    button_color = "green"
else:
    button_color = "blue"

# Track metrics by variant
analytics.track("checkout_completed", user, {"variant": variant})
```

**Characteristics**:
- Short-lived (duration of experiment, typically 1-4 weeks)
- Multivariate (A/B/C testing, not just boolean)
- Statistical rigor required (sample size, significance)
- Coupled with analytics

**Key insight**: This isn't just a feature toggle - it's experiment infrastructure. You need randomization, consistent assignment (same user always sees same variant), and metric tracking.

### 3. Ops Toggles (Operational Flags)

**Purpose**: Control operational behavior. Circuit breakers, kill switches, performance tuning.

**Lifecycle**: Permanent (or at least long-lived).

**Example**:
```python
# Kill switch for expensive feature
if feature_flags.is_enabled("enable_recommendations"):
    recommendations = expensive_ml_model(user)
else:
    recommendations = []  # Degrade gracefully

# Performance tuning
batch_size = feature_flags.get_int("batch_processor_size", default=100)
```

**Characteristics**:
- Long-lived or permanent
- Changed during incidents or high load
- Fast propagation critical (can't wait 5 minutes for cache invalidation)
- Often numeric, not boolean (batch sizes, timeouts, rate limits)

**This is different**: These aren't about releasing features - they're runtime control planes for your system. The flag system becomes part of your operational toolkit.

### 4. Permission Toggles (Entitlement Flags)

**Purpose**: Control access based on user attributes. Premium features, beta access, internal tools.

**Lifecycle**: Permanent. This is your authorization layer.

**Example**:
```python
if feature_flags.is_enabled("premium_analytics", user):
    return generate_advanced_report(user)
else:
    return generate_basic_report(user)
```

**Characteristics**:
- Permanent (or tied to product lifecycle)
- User-attribute based (plan tier, account age, geographic region)
- Business logic, not just technical toggle
- Often driven by external systems (billing, CRM)

**Interview nuance**: This blurs the line between feature flags and authorization. Some argue these should be in your entitlement system, not your flag system. The trade-off: flag systems are optimized for low latency and flexibility; auth systems are optimized for correctness and audit trails.

## Implementation Strategies: Where Do Flags Live?

### Option 1: Configuration Files

**How it works**: Flags stored in JSON/YAML files, deployed with application.

```json
{
  "features": {
    "new_checkout": true,
    "enable_recommendations": false
  }
}
```

**Pros**:
- Simple, no external dependencies
- Version controlled alongside code
- Zero network latency

**Cons**:
- Requires redeployment to change (defeats the purpose!)
- No user targeting
- No gradual rollouts

**When to use**: For build-time feature toggles that differ per environment (enable debug logging in staging). Not for progressive delivery.

### Option 2: Environment Variables

**How it works**: Flags stored as env vars, read on application startup.

```bash
FEATURE_NEW_CHECKOUT=true
FEATURE_RECOMMENDATIONS=false
```

**Pros**:
- Standard deployment practice
- No code changes needed
- Works with container orchestration

**Cons**:
- Requires container restart to change
- No dynamic updates
- No user targeting
- Becomes unwieldy with many flags

**When to use**: For environment-specific toggles (enable profiling, set log levels). Not for user-facing features.

### Option 3: Database

**How it works**: Flags stored in your application database, queried at runtime.

```sql
SELECT enabled FROM feature_flags WHERE flag_name = 'new_checkout';
```

**Pros**:
- Dynamic updates (change in DB, reflect immediately)
- Can support user targeting (join with user table)
- No additional infrastructure

**Cons**:
- Adds latency to every request (requires caching)
- Scales poorly (database becomes bottleneck)
- No built-in admin UI
- Couples flag state to application state

**When to use**: For small applications with low traffic. Good starting point before dedicated infrastructure.

### Option 4: Dedicated Feature Flag Service

**How it works**: External service (LaunchDarkly, Split, Unleash, Flagsmith) manages flags. Application queries via SDK.

```python
client = LaunchDarklyClient(api_key)

if client.variation("new_checkout", user, default=False):
    return new_checkout_flow()
```

**Pros**:
- Built for this purpose: targeting, rollout rules, UI, audit logs
- Fast (CDN-cached, local caching in SDK)
- Advanced features: A/B testing, gradual rollouts, kill switches
- Decoupled from application database

**Cons**:
- Additional infrastructure and cost
- External dependency (what if service is down?)
- Vendor lock-in risk

**When to use**: For production systems with progressive delivery requirements. The cost is worth it when you need reliability and flexibility.

### The Hybrid Approach: Local Cache with Remote Source of Truth

Most production systems use this pattern:

1. **Remote service** stores authoritative flag state
2. **Application SDK** caches flag values in memory
3. **Periodic sync** updates cache (every 30-60 seconds)
4. **Fallback defaults** in code if service is unreachable

This balances:
- **Low latency**: In-memory cache, no network call per check
- **Fast updates**: Flags propagate within ~1 minute
- **High availability**: Embedded defaults if service is down

**The trade-off**: Eventual consistency. Different instances might see different flag values for up to 1 minute. For most use cases, this is acceptable. For critical ops toggles (circuit breakers), you might need a push model (flag service pushes updates immediately).

## Targeting Rules: Who Sees What?

Feature flags are only useful if you can control **who** sees the feature. This is the targeting problem.

### Basic Targeting Dimensions

**1. Percentage Rollouts**:
```python
# Enable for 10% of users
if hash(user.id) % 100 < 10:
    show_new_feature()
```

Key property: **Consistent assignment**. Same user always gets same result. Use hash(user_id), not random().

**2. User Attributes**:
```python
# Enable for premium users
if user.plan_tier == "premium":
    show_feature()

# Enable for specific geographic region
if user.country in ["US", "CA", "UK"]:
    show_feature()
```

**3. Explicit Allowlist**:
```python
# Beta testers
if user.id in [123, 456, 789]:
    show_feature()

# Company employees
if user.email.endswith("@company.com"):
    show_feature()
```

**4. Ring-based Deployment**:
```
Ring 0: Internal employees (1% of traffic)
Ring 1: Beta users who opted in (5%)
Ring 2: Random 10% of users
Ring 3: Random 50% of users
Ring 4: Everyone (100%)
```

Each ring is a progressively larger blast radius. If Ring N shows issues, stop rollout.

### Complex Targeting Logic

Real flag systems support boolean logic:

```
Enable if:
  (user.plan == "premium" OR user.age_days > 30)
  AND user.country in ["US", "CA"]
  AND random_rollout(user.id) < 25%
  AND NOT user.id in blocklist
```

This becomes a rules engine. The flag service evaluates rules server-side (expensive, authoritative) or client-side (fast, but exposes logic).

### The Sticky Session Problem

For some features, you need **session consistency**: once a user sees variant A, they should keep seeing variant A for the entire session.

**Why**: Imagine an A/B test on checkout flow. User starts checkout with variant A, but mid-checkout, hash-based rollout changes and they suddenly see variant B. Confusing and breaks funnel metrics.

**Solution**: Persist variant assignment in session storage or cookie:
```python
if "checkout_variant" not in session:
    session["checkout_variant"] = determine_variant(user)
variant = session["checkout_variant"]
```

Trade-off: Now rollout percentages aren't exact (if you scale from 10% to 50%, existing sessions stay in old variant until session ends).

## Flag Evaluation: Client-Side vs. Server-Side

Where do you evaluate flag rules?

### Server-Side Evaluation

**How it works**: Application sends user context to flag service, service returns boolean:
```
App → "is 'new_checkout' enabled for user 123?" → Flag Service
Flag Service → "Yes" → App
```

**Pros**:
- Rules are private (user can't inspect)
- Complex targeting logic (rules run on server)
- Audit trail (every evaluation logged)

**Cons**:
- Network latency (unless cached)
- Requires remote service (dependency)

### Client-Side Evaluation

**How it works**: Flag service sends all rules to application, application evaluates locally:
```
Flag Service → {flag: 'new_checkout', rule: 'user.country == "US" && hash(user.id) % 100 < 50'} → App
App evaluates locally → Returns true/false
```

**Pros**:
- Zero latency (no network call per check)
- Works offline (rules cached locally)

**Cons**:
- Rules are exposed (user can inspect)
- Less dynamic (rules update on sync interval, not instantly)

**Production pattern**: Client-side evaluation for frontend (latency-critical), server-side for backend (rules should be private).

## Testing with Feature Flags: The Combinatorial Explosion

Feature flags create a new problem: **combinatorial state explosion**.

With N boolean flags, you have 2^N possible application states:
- 5 flags = 32 states
- 10 flags = 1,024 states
- 20 flags = 1,048,576 states

**You cannot test all combinations**. This is the hidden cost of feature flags.

### Strategies for Manageable Testing

**1. Test with flags in expected state**:
```python
# Test new checkout flow
@with_feature_flag("new_checkout", enabled=True)
def test_checkout():
    result = checkout_flow(cart)
    assert result.success

# Test legacy checkout flow
@with_feature_flag("new_checkout", enabled=False)
def test_legacy_checkout():
    result = checkout_flow(cart)
    assert result.success
```

Test both paths independently, but don't test all flag combinations.

**2. Flag override in test environment**:
```python
feature_flags.override("new_checkout", True)
# All tests run with new_checkout enabled
```

Ensures consistent test environment.

**3. Minimize active flags**:
The fewer active flags, the fewer combinations. This is why flag cleanup is critical. Old flags are technical debt.

**4. Acceptance tests at the edges**:
Test with flags in production-like configuration. If production is 50% rollout, test with random assignment.

### The "Dead Code" Problem

When a flag is 100% enabled, the old code path becomes dead code:

```python
if feature_flags.is_enabled("new_checkout"):  # Always true
    return new_checkout_flow()
else:
    return legacy_checkout_flow()  # This never runs
```

The old branch is still deployed, still tested, but never executed. It's dead weight.

**Solution**: Flag cleanup. Once a flag is fully rolled out and stable, remove the flag AND the old code:

```python
# After cleanup
return new_checkout_flow()
```

## Flag Lifecycle: From Creation to Cleanup

Feature flags have a lifecycle. Managing this lifecycle is key to avoiding flag rot.

### Phase 1: Creation

Developer creates flag for new feature:
```python
# Code
if feature_flags.is_enabled("new_recommendations"):
    return ml_recommendations(user)
else:
    return simple_recommendations(user)

# Flag service
create_flag(
    name="new_recommendations",
    enabled=False,  # Start disabled
    description="New ML-based recommendation engine",
    created_by="engineer@company.com",
    created_at="2025-12-06",
    expected_cleanup_date="2026-01-06"  # Set expectation
)
```

### Phase 2: Rollout

Progressive enablement:
- Day 1: Enable for internal users (Ring 0)
- Day 3: Enable for 1% of users (monitor metrics)
- Day 5: Enable for 10% (metrics look good)
- Day 7: Enable for 50%
- Day 10: Enable for 100% (fully rolled out)

**Key**: At each stage, pause and observe. If error rates spike, latency increases, or users complain, rollback (set to previous percentage).

### Phase 3: Stabilization

Flag is at 100%, feature is stable. Now it's in "soak" period:
- Monitor for 1-2 weeks
- Ensure no issues at full scale
- Gather feedback

### Phase 4: Cleanup

Flag has served its purpose. Now remove it:

1. **Remove flag check from code**:
```python
# Before
if feature_flags.is_enabled("new_recommendations"):
    return ml_recommendations(user)
else:
    return simple_recommendations(user)

# After
return ml_recommendations(user)
```

2. **Delete old code path**:
```python
# Delete this
def simple_recommendations(user):
    ...
```

3. **Archive flag in flag service**:
```
archive_flag("new_recommendations")
```

4. **Document decision**:
```
# ADR: Removed simple_recommendations in favor of ML-based system
# Date: 2026-01-06
# Reason: ML system showed 15% improvement in click-through rate
```

### Preventing Flag Rot

**Flag rot** is the accumulation of old, unused flags in your codebase. It's technical debt.

**Strategies to prevent**:
1. **Flag expiration dates**: Flags have TTL. After 90 days, CI fails if flag still exists.
2. **Automated reports**: Weekly report of flags older than X days.
3. **Required cleanup**: Flag removal is part of "done" definition for a feature.
4. **Flag usage tracking**: Instrument flag checks. If a flag hasn't been checked in 30 days, it's dead.
5. **Quarterly cleanup sprints**: Dedicated time to remove old flags.

## When NOT to Use Feature Flags

Feature flags are powerful but not free. They add complexity.

### Don't use flags for:

**1. Configuration that should be environment variables**:
```python
# Bad: This is just configuration
if feature_flags.is_enabled("use_redis_cache"):
    cache = RedisCache()
else:
    cache = MemoryCache()

# Good: Use environment variable
cache_backend = os.getenv("CACHE_BACKEND", "redis")
```

**2. Every small change**:
Flags have overhead. Not every 3-line change needs a flag. Use flags for:
- Risky changes (major refactors, new critical paths)
- User-facing features (want gradual rollout)
- Experiments (A/B tests)

Don't use flags for:
- Bug fixes (deploy and monitor)
- Internal refactors with no user impact
- Dependency updates

**3. Permanent business logic**:
```python
# Bad: This is your authorization logic, not a flag
if feature_flags.is_enabled("premium_features", user):
    return premium_dashboard()
```

This isn't a temporary toggle - it's your product structure. Flags are for temporary control. Permanent entitlement logic should be in your permission system, not your flag system.

**4. Database schema changes**:
Flags control application behavior, not schema. You can't flip a flag to add a column. Schema migrations need a different strategy (backward-compatible changes, dual-write patterns).

## The Economics of Feature Flags

Feature flags have costs and benefits. Let's quantify.

### Costs

**1. Runtime overhead**:
- Every flag check is a function call (in-memory lookup, but not free)
- 10-100 microseconds per check (depending on implementation)
- At scale, this adds up

**2. Code complexity**:
- More branches, more states
- Harder to reason about ("what does this code do?")
- Increased testing surface

**3. Technical debt**:
- Old flags that aren't cleaned up
- Dead code paths that are still deployed
- Maintenance burden

**4. Infrastructure cost**:
- Dedicated flag service (LaunchDarkly, Split) costs $$$
- Additional monitoring and observability

### Benefits

**1. Reduced deployment risk**:
- Deploy != Release, so deployments are low-risk
- Instant rollback (no redeployment)
- Gradual rollout reduces blast radius

**2. Faster time-to-market**:
- Deploy unfinished features (hidden behind flag)
- No waiting for release windows
- Continuous integration (no long-lived branches)

**3. Experimentation infrastructure**:
- A/B testing built-in
- Data-driven decisions
- Measure impact before full rollout

**4. Operational control**:
- Kill switches for expensive features
- Runtime tuning (batch sizes, timeouts)
- Degrade gracefully under load

### The Break-Even Point

Feature flags are worth it when:
- **Deployment frequency > 1/week**: Decoupling deploy from release enables velocity
- **Multiple active features in development**: Trunk-based development requires flags
- **Production incidents are expensive**: Instant rollback saves time and money
- **Gradual rollouts reduce risk**: Your error budget benefits from progressive delivery

Feature flags are overkill when:
- You deploy monthly (deployment overhead is low, flags add unnecessary complexity)
- Simple application (few features, low change rate)
- Cost of incidents is low (internal tool, low user count)

## Interview Deep Dive: Designing a Flag System

**Question**: "Design a feature flag system for a high-traffic SaaS application (10M+ requests/day). What are the key architectural decisions?"

**Your approach** (demonstrates senior-level thinking):

### 1. Requirements Gathering

"First, I'd clarify requirements:
- **Latency tolerance**: Can we accept 5ms per flag check? Or must it be sub-millisecond?
- **Consistency requirements**: Is eventual consistency (1-minute propagation) acceptable?
- **Targeting complexity**: Do we need simple boolean flags, or complex rules with user attributes?
- **Scale**: How many flags? How many evaluations per second?
- **Availability**: What happens if flag service is down - fail open (default to on) or fail closed (default to off)?"

### 2. Architecture Options

**Option A: Client-Side Evaluation with Local Cache**
```
Flag Service (Source of Truth)
    ↓ (sync every 60s)
Application In-Memory Cache → Flag Evaluation (< 1ms)
```

**Pros**: Sub-millisecond latency, works if flag service is down
**Cons**: Eventual consistency (up to 60s delay), rules are cached in app memory

**Option B: Server-Side Evaluation with CDN Cache**
```
Application → CDN (edge cache) → Flag Service
               ↑ (cache for 30s)
```

**Pros**: Faster updates (30s), rules stay on server (private)
**Cons**: Adds 10-50ms latency, depends on CDN availability

**Option C: Hybrid**
```
Flag Service → App In-Memory Cache (default cache)
             → Redis Shared Cache (for dynamic targeting)
             → Embedded Fallback Defaults (if service down)
```

"For high-traffic SaaS, I'd choose **Option A (client-side with local cache)** because:
- Latency is critical at 10M+ req/day
- 60-second propagation is acceptable for most feature flags
- Embedded defaults ensure availability"

### 3. Data Model

"I'd structure flags as:

```json
{
  "flag_id": "new_checkout",
  "enabled": true,
  "targeting_rules": [
    {"attribute": "plan_tier", "operator": "equals", "value": "premium"},
    {"attribute": "user_id", "operator": "hash_mod", "value": 50}
  ],
  "fallback_value": false,
  "created_at": "2025-12-06",
  "expires_at": "2026-03-06"
}
```

**Key decisions**:
- `fallback_value`: What to return if targeting fails or service is down
- `expires_at`: Enforces flag cleanup
- `targeting_rules`: Array of rules (supports complex AND/OR logic)"

### 4. Handling Edge Cases

"**What if flag service is down?**
- Embedded defaults in application code
- Log flag service failures (but don't alert - it shouldn't block requests)
- Degrade to default behavior

**What if a flag's targeting rules are invalid?**
- Validate rules on creation (flag service rejects invalid rules)
- If runtime evaluation fails, fall back to `fallback_value`

**How do you prevent stale cache?**
- SDK periodically syncs (every 60s)
- Flag service can push urgent updates (WebSocket or SSE)
- For critical ops flags (circuit breakers), use push model"

### 5. Monitoring & Observability

"I'd track:
- **Flag evaluation latency** (should be < 1ms in p99)
- **Cache hit rate** (should be > 99%)
- **Flag service availability** (uptime, latency)
- **Flag age** (alert if flag is > 90 days old)
- **Dead code detection** (flags that haven't been evaluated in 30 days)"

This answer shows:
- You understand trade-offs (latency vs. consistency)
- You consider failure modes (what if service is down?)
- You think about lifecycle (flag expiration, cleanup)
- You know how to scale (caching, CDN)
- You care about operability (monitoring, alerting)

## Summary: Feature Flags as State Management

Feature flags are fundamentally about **where you store state** that controls your application's behavior. Moving that state from compile-time (code) to deploy-time (env vars) to runtime (flags) trades performance for flexibility.

**Key mental models**:
1. **Deploy ≠ Release**: Flags decouple deployment (binary placement) from release (feature activation)
2. **CAP theorem applies**: You can't have low latency, strong consistency, AND high availability - pick two
3. **Flags have types**: Release toggles, experiment toggles, ops toggles, permission toggles - different lifecycles
4. **Combinatorial explosion**: N flags = 2^N states - you can't test them all, so minimize active flags
5. **Flags are technical debt**: Unless cleaned up, they rot - lifecycle management is critical

**When interviewing**, be ready to:
- Explain trade-offs between flag implementation strategies
- Design a flag system for scale and availability
- Discuss flag lifecycle and cleanup strategies
- Articulate when NOT to use flags
- Connect flags to progressive delivery and risk management

Feature flags aren't just a deployment tool - they're a fundamental shift in how you think about releasing software. They enable progressive delivery, trunk-based development, and experimentation. But they add complexity. Use them where the benefits outweigh the costs.
