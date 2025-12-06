# Release Orchestration & Coordination

## The Coordination Problem

In a monolithic application, deployment is simple: Build one artifact, deploy to servers, done.

In a distributed system with 50 microservices, deployment becomes a coordination problem:
- Service A depends on Service B's new API
- Service C's database migration must run before deploying Service D
- Service E and Service F can't both deploy simultaneously (they share a dependency)
- Service G's deployment broke Service H (unexpected downstream impact)

This is the **release orchestration problem**: How do you safely deploy changes across multiple interdependent services?

The fundamental tension: **Service autonomy vs. coordinated safety**.

**Autonomy**: Each team deploys independently, at their own pace. Fast, but risky (might break other services).

**Coordination**: Centralized orchestration, carefully ordered deployments. Safe, but slow (bottleneck, overhead).

There's no perfect solution - only trade-offs.

## The Theory: Distributed Systems Deployment

When you deploy a change in a distributed system, you're changing the **distributed state** of the system. Unlike a monolith (where all code changes atomically), a distributed system has no global transaction - services update independently.

### The Two Generals Problem (and why it matters)

Classic distributed systems problem: Two armies on opposite hills need to coordinate an attack. They send messengers, but messengers might be captured. How can they reach consensus?

**Answer**: They can't. There's no protocol that guarantees consensus with unreliable communication.

**Relevance to deployment**: When you deploy Service A and Service B, there's a period where they're running different versions. You can't atomically update both. This creates **transient incompatibility windows**.

**Example**:
```
Service A v1 → Calls Service B v1 (works)
Service B deploys v2 (API changed)
Service A v1 → Calls Service B v2 (breaks!)
Service A deploys v2 (updated to use new API)
Service A v2 → Calls Service B v2 (works)
```

During the window where A is v1 and B is v2, the system is broken.

**Solution**: Backward compatibility. Service B v2 must still accept v1 API calls.

### The CAP Theorem for Deployments

Just like distributed databases face CAP theorem (Consistency, Availability, Partition tolerance), distributed deployments face similar trade-offs:

**Deployment velocity vs. System consistency vs. Service autonomy**:
- **High velocity**: Deploy fast (but might cause transient inconsistencies)
- **Strong consistency**: Coordinated deployments (but slow, requires synchronization)
- **Full autonomy**: Each service deploys independently (but might break other services)

You can't maximize all three. Your choice depends on your system's constraints.

**Example 1: E-commerce (consistency matters)**:
- Payment service and Order service must stay compatible
- Choose: Coordination + consistency (sacrifice some autonomy)
- Use: Coordinated deployments, backward compatibility requirements

**Example 2: Social media feed (velocity matters)**:
- Feed service can tolerate transient issues (users refresh)
- Choose: Velocity + autonomy (sacrifice strong consistency)
- Use: Independent deployments, graceful degradation

## Service Dependencies: The Deployment DAG

Before you can orchestrate deployments, you need to understand dependencies.

### Mapping the Dependency Graph

**Services can depend on each other in several ways**:

**1. API dependency**: Service A calls Service B's API
```
Order Service → Payment Service (API call)
```

**2. Data dependency**: Service A reads Service B's database
```
Analytics Service → User Database (read replica)
```

**3. Event dependency**: Service A publishes events, Service B consumes them
```
Order Service → [Event Queue] → Shipping Service
```

**4. Shared dependency**: Both services depend on a common library or service
```
Service A → Shared Auth Library v1.2.3
Service B → Shared Auth Library v1.2.3
```

### Directed Acyclic Graph (DAG) of Deployments

You can model service dependencies as a **DAG**:

```
         Frontend
         /      \
    Order       User
    Service     Service
      |           |
    Payment     Auth
    Service     Service
         \      /
         Database
```

**Deployment order** (topological sort of DAG):
1. Database (leaf node, no dependencies)
2. Auth Service, Payment Service (depend only on Database)
3. Order Service, User Service (depend on Auth/Payment)
4. Frontend (depends on Order/User)

**Rule**: Deploy dependencies before dependents.

**Why**: If you deploy Frontend first (which expects new Order Service API), but Order Service is still old version, Frontend breaks.

### Circular Dependencies: The Coordination Nightmare

**Problem**: Service A depends on Service B, Service B depends on Service A.

```
Order Service ←→ Inventory Service
```

Order Service calls Inventory to check stock.
Inventory calls Order to reserve items.

**This breaks the DAG**. You can't deploy one before the other.

**Solutions**:

**1. Break the cycle** (architectural change):
Extract shared logic to a third service:
```
Order Service → Stock Service
Inventory Service → Stock Service
```

**2. Backward compatibility** (deployment strategy):
- Deploy Order Service v2 (can call both old and new Inventory API)
- Deploy Inventory Service v2 (can call both old and new Order API)
- Now both can be deployed independently

**3. Feature flags** (runtime decoupling):
- Deploy both services with new code, behind flags
- Enable flags after both are deployed

**Best practice**: Avoid circular dependencies. They indicate tight coupling.

## Database Migrations: The Hardest Coordination Problem

Database migrations are uniquely hard because:
1. **Schema changes are stateful** (can't easily rollback)
2. **Database is shared** (multiple services might depend on same schema)
3. **Migrations take time** (especially for large tables)
4. **Zero-downtime is required** (can't take database offline)

### The Naive Approach (and why it fails)

**Naive strategy**:
1. Take database offline
2. Run migration (add column, change data type, etc.)
3. Deploy new application code
4. Bring database online

**Problems**:
- **Downtime**: Database is offline during migration (unacceptable for 24/7 services)
- **Rollback is impossible**: Can't undo schema change (data might have been written in new format)

### The Expand/Contract Pattern (Zero-Downtime Migrations)

**Principle**: Make schema changes in backward-compatible steps.

**Three phases**:

#### Phase 1: Expand (Add, Don't Remove)

Add new schema elements (columns, tables) without removing old ones.

**Example**: Renaming column `name` to `full_name`

**Expand**:
```sql
-- Add new column
ALTER TABLE users ADD COLUMN full_name VARCHAR(255);

-- Backfill data (copy name → full_name)
UPDATE users SET full_name = name WHERE full_name IS NULL;
```

**Application code** (v1.1):
- **Reads**: Read from `name` (old column, still works)
- **Writes**: Write to BOTH `name` and `full_name` (dual-write)

**State after Phase 1**:
- Database has both `name` and `full_name` columns
- Old app code (v1.0) still works (reads `name`)
- New app code (v1.1) writes to both

**Deploy**: v1.1 to all instances. No breaking changes.

#### Phase 2: Migrate (Switch Reads)

Switch application to read from new column.

**Application code** (v1.2):
- **Reads**: Read from `full_name` (new column)
- **Writes**: Write to BOTH `name` and `full_name` (still dual-write)

**State after Phase 2**:
- All reads use `full_name`
- Still writing to both (for backward compatibility)

**Deploy**: v1.2 to all instances. Old column (`name`) is no longer read.

#### Phase 3: Contract (Remove Old)

Remove old schema elements.

**Verify**: No application code reads from `name` column (might take days/weeks to be sure).

**Contract**:
```sql
-- Remove old column
ALTER TABLE users DROP COLUMN name;
```

**Application code** (v1.3):
- **Reads**: Read from `full_name`
- **Writes**: Write ONLY to `full_name` (no more dual-write)

**State after Phase 3**:
- Only `full_name` exists
- Migration complete

### Why This Works

**At every step, both old and new application versions can run**:
- Phase 1: Old app (v1.0) reads `name`, new app (v1.1) reads `name` or `full_name`
- Phase 2: New app (v1.2) reads `full_name`, data is there (backfilled)
- Phase 3: Only new app (v1.3) is running

**You can rollback** (to some extent):
- After Phase 1: Rollback to v1.0 (reads `name`, which is still being written)
- After Phase 2: Rollback to v1.1 (reads `name`, which is still being written)
- After Phase 3: Can't rollback (old column is gone)

**Key insight**: Never remove old schema until you're sure no code references it.

### Complex Migration: Changing Data Types

**Problem**: Change `user_id` from INT to UUID.

**Can't do**:
```sql
-- This breaks during migration (data type incompatible)
ALTER TABLE users ALTER COLUMN user_id TYPE UUID;
```

**Expand/Contract approach**:

**Phase 1: Add new column**
```sql
ALTER TABLE users ADD COLUMN user_uuid UUID;

-- Generate UUIDs for existing users
UPDATE users SET user_uuid = uuid_generate_v4() WHERE user_uuid IS NULL;
```

**Phase 2: Dual-write**
App writes to both `user_id` and `user_uuid`.

**Phase 3: Migrate foreign keys**
All tables that reference `user_id` need a `user_uuid` column too. Repeat expand/contract for each.

**Phase 4: Switch reads**
App reads from `user_uuid` instead of `user_id`.

**Phase 5: Contract**
Drop `user_id` column (and all foreign keys).

**Timeline**: This might take weeks or months for a large, critical table.

### The Dual-Write Consistency Problem

**Issue**: When writing to two columns/tables simultaneously, writes might partially fail.

**Example**:
```python
# Dual-write
user.name = "Alice"
user.full_name = "Alice"
db.save(user)  # What if this fails after writing `name` but before `full_name`?
```

**Solutions**:

**1. Transactional writes** (if same database):
```python
with db.transaction():
    user.name = "Alice"
    user.full_name = "Alice"
    db.save(user)
```
Atomically writes both or neither.

**2. Eventual consistency** (if different databases):
Accept that data might be briefly inconsistent. Use background jobs to reconcile.

**3. Write-ahead log**:
Write to queue first, then update both columns from queue (guarantees order).

### Dealing with Migrations in Microservices

**Problem**: Each service has its own database. How do you coordinate schema changes across services?

**Example**:
```
Order Service DB: Has `user_id` (INT)
User Service DB: Changing `user_id` from INT to UUID
```

Order Service's `user_id` foreign key will break once User Service switches to UUID.

**Solutions**:

**1. Add a new field, deprecate old one**:
User Service returns both:
```json
{
  "user_id": 12345,  // Old (deprecated)
  "user_uuid": "a1b2c3d4-..."  // New
}
```

Order Service updates to use `user_uuid` when available, falls back to `user_id`.

**2. API versioning** (discussed below):
User Service v2 returns UUIDs, v1 returns INTs. Order Service calls v1 until ready to migrate.

**3. Event schema evolution**:
If using event-driven architecture, publish events with both fields during migration.

**Key principle**: Never make breaking changes to APIs or data formats without a migration path.

## API Versioning: Managing Breaking Changes

When you change an API, you risk breaking clients. API versioning is how you manage this.

### What Is a Breaking Change?

**Breaking changes**:
- Removing an endpoint
- Removing a field from response
- Adding a required field to request
- Changing field type (string → int)
- Changing endpoint URL

**Non-breaking changes**:
- Adding a new endpoint
- Adding an optional field to request
- Adding a field to response (clients should ignore unknown fields)
- Deprecating an endpoint (but still supporting it)

### API Versioning Strategies

#### Strategy 1: URI Versioning

**How it works**: Version is in the URL path.

**Example**:
```
/v1/users
/v2/users
```

**Pros**:
- Simple, explicit
- Easy to route (different endpoints)
- Can deploy v1 and v2 simultaneously (different handlers)

**Cons**:
- URL changes (clients must update)
- Versioning entire API, not individual resources

**When to use**: Public APIs, major breaking changes.

#### Strategy 2: Header Versioning

**How it works**: Version is in HTTP header.

**Example**:
```
GET /users
Header: Accept: application/vnd.myapi.v2+json
```

**Pros**:
- URL stays the same
- Can version specific resources

**Cons**:
- Less visible (version is in header, not URL)
- Requires clients to send correct header

**When to use**: Internal APIs, gradual migration.

#### Strategy 3: Query Parameter Versioning

**How it works**: Version is a query parameter.

**Example**:
```
/users?version=2
```

**Pros**:
- Simple to implement
- URL is mostly stable

**Cons**:
- Can be forgotten (clients might not pass version)
- Clutters query parameters

**When to use**: Quick fixes, temporary versioning.

#### Strategy 4: Content Negotiation

**How it works**: Client specifies desired response format, server returns compatible version.

**Example**:
```
GET /users
Header: Accept: application/json; version=2
```

**Pros**:
- RESTful (uses HTTP standards)
- Flexible (clients can request specific version)

**Cons**:
- Complex to implement
- Less discoverable

**When to use**: APIs with complex versioning needs.

### The Two-Version Rule

**Principle**: Support at most two versions simultaneously (current and previous).

**Why**: Supporting 3+ versions becomes maintenance nightmare.

**Lifecycle**:
1. Launch v1 (only version)
2. Launch v2, support v1 and v2 (two versions)
3. Deprecate v1 (announce sunset date)
4. Sunset v1 after grace period (e.g., 6 months)
5. Now only v2 is supported

**During v1 sunset**:
- Log clients still using v1 (identify who needs to migrate)
- Send deprecation warnings in API responses
- Contact major clients (help them migrate)
- After grace period, return 410 Gone for v1 endpoints

### Semantic Versioning for APIs

**Format**: `MAJOR.MINOR.PATCH`

**Increment**:
- **MAJOR**: Breaking changes (remove endpoint, change response format)
- **MINOR**: Non-breaking additions (add new endpoint, add optional field)
- **PATCH**: Bug fixes (no API changes)

**Example**:
```
v1.0.0: Initial release
v1.1.0: Added /users/search endpoint (non-breaking)
v1.1.1: Fixed bug in /users/search (no API change)
v2.0.0: Removed /users/delete endpoint (breaking)
```

**In URL versioning**: Only include MAJOR version (`/v1/`, `/v2/`). MINOR and PATCH don't change URL.

### Deprecation Timeline

**Best practice**: Give clients **at least 6 months** to migrate from deprecated API.

**Example timeline**:
- **2025-12-01**: Launch v2, announce v1 deprecation (sunset date: 2026-06-01)
- **2026-03-01**: Start returning deprecation warnings in v1 responses
- **2026-05-01**: Email all clients still using v1 (final warning)
- **2026-06-01**: Sunset v1 (return 410 Gone)

**Deprecation header**:
```
HTTP/1.1 200 OK
Deprecation: true
Sunset: Sat, 01 Jun 2026 00:00:00 GMT
Link: <https://api.example.com/v2/users>; rel="successor-version"
```

Clients can programmatically detect deprecation and migrate.

## Coordinated Rollouts: Deploying Multiple Services

Sometimes you need to deploy multiple services together (e.g., Service A's change depends on Service B's new feature).

### Option 1: Sequential Deployment (Safe, Slow)

**How it works**: Deploy services one at a time, in dependency order.

**Example**:
```
1. Deploy Database migration
2. Wait for migration to complete (might take hours)
3. Deploy Service B (uses new DB schema)
4. Wait for Service B to be healthy
5. Deploy Service A (calls Service B's new API)
6. Wait for Service A to be healthy
```

**Pros**:
- Safe (each step is validated before next)
- Easy to rollback (rollback one service at a time)

**Cons**:
- Slow (hours to deploy all services)
- Requires coordination (manual or scripted)

**When to use**: High-risk changes, database migrations, breaking changes.

### Option 2: Parallel Deployment with Backward Compatibility (Fast, Requires Planning)

**How it works**: Deploy all services simultaneously, rely on backward compatibility.

**Requirements**:
- Service B v2 must support both old and new API (backward compatible)
- Service A v2 can call Service B v1 or v2 (forward compatible)

**Example**:
```
1. Deploy Service A v2 and Service B v2 simultaneously
2. Service A v2 might talk to Service B v1 instances (backward compatible)
3. Service B v2 might receive calls from Service A v1 instances (forward compatible)
4. Eventually all instances update, system converges to v2
```

**Pros**:
- Fast (minutes instead of hours)
- No coordination overhead

**Cons**:
- Requires careful API design (backward/forward compatibility)
- Harder to debug (version mismatch issues)

**When to use**: Non-breaking changes, mature APIs with compatibility guarantees.

### Option 3: Feature Flags (Fastest, Most Complex)

**How it works**: Deploy all services with new code, behind flags. Enable flags after deployment.

**Example**:
```
1. Deploy Service A v2 (new code behind flag "new_feature", disabled)
2. Deploy Service B v2 (new code behind flag "new_feature", disabled)
3. Both deployed, but behaving like v1 (flags disabled)
4. Enable "new_feature" flag
5. Both services activate new code simultaneously
```

**Pros**:
- Instant activation (toggle flag, no redeployment)
- Easy rollback (disable flag)
- Can enable for specific users first (gradual rollout)

**Cons**:
- Requires feature flag infrastructure
- Both code paths must exist (new and old)
- Flag coordination (if flag states diverge, services are incompatible)

**When to use**: Coordinated feature launches, high-velocity teams.

## Release Trains: Scheduled Deployment Windows

**Concept**: Instead of deploying whenever code is ready, deploy on a **fixed schedule** (e.g., every Tuesday at 10am).

**Example**:
```
Week 1: Code developed Monday-Friday, deployed Tuesday (Week 2)
Week 2: Code developed Monday-Friday, deployed Tuesday (Week 3)
```

### Pros of Release Trains

**1. Predictability**:
- Teams know when deployments happen
- Operations team can plan (on-call coverage, monitoring)

**2. Batching**:
- Multiple changes deploy together (reduces number of deployments)

**3. Coordination**:
- All services deploy at the same time (easier to coordinate)

**4. Risk management**:
- Avoid deploying on Fridays (if it breaks, you work weekends)
- Avoid deploying before holidays

### Cons of Release Trains

**1. Slow**:
- Features sit in staging for days before deploying
- High cost of delay

**2. Large batch size**:
- Many changes deploy together (harder to identify cause of issues)

**3. Merge conflicts**:
- Features must merge to release branch before train leaves
- Encourages long-lived feature branches

**4. Blocked by one bad change**:
- If one team's change is broken, entire train is delayed

### When to Use Release Trains

**Good fit**:
- Enterprise software (customers expect infrequent updates)
- Mobile apps (app store approval process takes days anyway)
- Regulated industries (change control requires approval)

**Bad fit**:
- SaaS applications (continuous deployment is better)
- High-velocity startups (releases train slows down iteration)

### Hybrid Approach: Train for Major, Continuous for Minor

**Scheduled releases** for major features (user-facing, high risk).
**Continuous deployment** for bug fixes, small improvements.

**Example**:
```
Tuesday train: Deploy major features (new UI, new billing system)
Continuous: Deploy bug fixes anytime (don't wait for train)
```

## Deployment Locks: Preventing Conflicting Deployments

**Problem**: Two teams try to deploy to the same service simultaneously.

**Example**:
```
10:00 AM: Team A deploys Service X v1.2.3 (50% canary)
10:05 AM: Team B deploys Service X v1.2.4 (doesn't know Team A is deploying)
```

**Result**: Confusion. Which version is in production? If Team A's canary fails, which version do we rollback to?

### Solution: Deployment Locks

**How it works**: Only one deployment can be in progress for a service at a time.

**Mechanism**:
1. Team A starts deployment, acquires lock on Service X
2. Team B tries to deploy, blocked (lock is held)
3. Team B waits (or cancels their deployment)
4. Team A finishes, releases lock
5. Team B can now deploy

**Implementation**:

**Option 1: Database lock**:
```sql
-- Acquire lock
INSERT INTO deployment_locks (service_name, locked_by, locked_at)
VALUES ('service-x', 'team-a', NOW())
ON CONFLICT DO NOTHING;

-- Check if lock acquired
SELECT locked_by FROM deployment_locks WHERE service_name = 'service-x';

-- Release lock
DELETE FROM deployment_locks WHERE service_name = 'service-x' AND locked_by = 'team-a';
```

**Option 2: Distributed lock (Redis, etcd)**:
```python
lock = redis_client.lock("deployment:service-x", timeout=3600)
if lock.acquire(blocking=False):
    try:
        deploy_service()
    finally:
        lock.release()
else:
    print("Deployment already in progress")
```

**Option 3: CI/CD tool enforces**:
GitLab CI, GitHub Actions, Spinnaker can enforce "one deployment at a time."

### Lock Timeout

**Problem**: Lock is acquired, but deployment fails midway (pipeline crashes, engineer's laptop dies). Lock is never released.

**Solution**: Locks have TTL (time-to-live). After 1 hour, lock expires.

**Trade-off**: If deployment takes longer than TTL, lock expires while deployment is still running.

**Best practice**: Set TTL to 2x typical deployment duration (if deployments usually take 30 minutes, TTL = 60 minutes).

## Rollback Coordination: Cascading Rollbacks

**Problem**: You rollback Service A (it's broken). But Service B depends on Service A's old version. Now Service B is broken too.

### Example: Cascading Failure

```
Order Service v2 (calls Payment Service v2's new API)
Payment Service v2 (new API)

Payment Service v2 has bugs → Rollback to v1
Payment Service v1 (old API, doesn't support new fields)
Order Service v2 calls Payment v1 → Fails (API incompatible)
→ Must rollback Order Service to v1 too
```

### Solution 1: Backward Compatibility (Prevent the Problem)

**Prevent cascading rollbacks** by ensuring backward compatibility.

**Payment Service v2**:
- Supports both old API (for Order v1) and new API (for Order v2)

**Now**:
- Rollback Payment v2 → v1: Order Service v2 calls old API (still works, degraded but functional)

### Solution 2: Dependency-Aware Rollback

**How it works**: Rollback system knows dependency graph, automatically rolls back dependents.

**Example**:
```
Rollback Payment Service v2 → v1
System detects: Order Service v2 depends on Payment Service v2
System automatically: Rollback Order Service v2 → v1
```

**Implementation**:
Store dependency metadata:
```yaml
services:
  order-service:
    version: v2
    depends_on:
      - payment-service:v2

  payment-service:
    version: v2
```

Rollback tool:
```python
def rollback_service(service, version):
    # Rollback this service
    deploy(service, version)

    # Find dependents
    dependents = get_dependents(service)
    for dependent in dependents:
        # Rollback dependents too (recursive)
        rollback_service(dependent, get_compatible_version(dependent, version))
```

**Pros**: Automatic, prevents broken state.
**Cons**: Might rollback more services than necessary (blast radius).

### Solution 3: Manual Coordination

**How it works**: Rollback Payment Service, alert Order Service team.

**Process**:
1. Rollback Payment Service v2 → v1
2. Post in Slack: "Payment Service rolled back to v1. Order Service might be affected."
3. Order Service team investigates
4. If Order Service is broken, they rollback too

**Pros**: Human judgment (might not need to rollback Order Service).
**Cons**: Manual, slow, requires coordination.

**Best practice**: Combine automated alerts with human decision-making.

## Communication & Coordination Overhead

Coordinated deployments require communication. This has costs.

### Internal Communication

**Who needs to know about deployments?**

**1. Other engineering teams**:
- Teams whose services depend on yours
- Teams who share infrastructure (database, queue)

**2. On-call engineers**:
- Might need to respond if deployment causes incidents

**3. Customer support**:
- Users might report issues, support needs context

**4. Product/business teams**:
- New features are launching

### External Communication

**Who outside the company needs to know?**

**1. Customers (for user-facing changes)**:
- New features (marketing opportunity)
- Breaking changes (API deprecations, need to migrate)
- Downtime (scheduled maintenance)

**2. Partners (for API changes)**:
- API versioning
- Deprecation timelines

### Communication Channels

**Status pages** (e.g., status.example.com):
- Current system status (operational, degraded, outage)
- Scheduled maintenance windows
- Incident history

**Internal chat** (Slack, Teams):
- "#deployments" channel: Automated notifications of all deployments
- "#incidents" channel: Deployment issues, rollbacks

**Email**:
- Major version announcements
- API deprecation notices (sent to all API users)

**In-app notifications**:
- "New feature available" banners
- "Update required" alerts (for mobile apps)

### The Cost of Coordination

**Coordination overhead grows with number of services**.

**Formula**: Coordination cost ≈ O(N²), where N = number of services.

**Why**: Each service might depend on multiple others. As N grows, dependency graph explodes.

**Example**:
- 5 services: ~10 possible dependencies
- 50 services: ~1,225 possible dependencies
- 500 services: ~124,750 possible dependencies

**This is why microservices can become a coordination nightmare.**

### Strategies to Reduce Coordination Overhead

**1. Loose coupling**:
- Services communicate via events (async, no tight dependency)
- Contract testing (ensure API compatibility)

**2. API stability**:
- Backward compatibility (no breaking changes)
- Semantic versioning (clear expectations)

**3. Service ownership**:
- Each team owns their services end-to-end
- Teams responsible for not breaking others

**4. Automated testing**:
- Integration tests across services
- Smoke tests after deployment

**5. Feature flags**:
- Deploy code, enable features later (decouple deploy from release)

## Interview Deep Dive: Orchestrating a Multi-Service Deployment

**Question**: "You need to deploy a new feature that spans 5 microservices and requires a database migration. Walk me through your deployment strategy."

**Your approach** (demonstrates senior-level thinking):

### 1. Understand the Dependencies

"First, I'd map out the dependency graph:

**Services involved**:
- Frontend (UI for new feature)
- API Gateway (routes requests)
- Order Service (business logic)
- Inventory Service (checks stock)
- Database (schema change required)

**Dependency DAG**:
```
Frontend → API Gateway → Order Service → Database
                       → Inventory Service → Database
```

**Deployment order** (topological sort):
1. Database (leaf node)
2. Order Service, Inventory Service (depend on DB)
3. API Gateway (depends on Order/Inventory)
4. Frontend (depends on API Gateway)"

### 2. Plan the Database Migration

"The database migration is the riskiest part. I'd use the **expand/contract pattern**:

**Phase 1: Expand** (add new columns, don't remove old):
```sql
ALTER TABLE orders ADD COLUMN priority VARCHAR(20);
ALTER TABLE inventory ADD COLUMN reserved_quantity INT DEFAULT 0;
```

**Deploy Order Service v1.1** (dual-write):
- Writes to both old fields and new `priority` field
- Reads from old fields (for backward compatibility)

**Deploy Inventory Service v1.1** (dual-write):
- Writes to both `available_quantity` and `reserved_quantity`

**Why dual-write**: Old services (v1.0) can still run, reading old fields. New services (v1.1) write to both.

**Phase 2: Migrate** (switch reads):
**Deploy Order Service v1.2**:
- Reads from `priority` field
- Still writes to both (for rollback safety)

**Deploy Inventory Service v1.2**:
- Reads from `reserved_quantity`

**Phase 3: Contract** (remove old, weeks later):
After v1.2 is stable for 2+ weeks, drop old columns."

### 3. Coordinate the Rollout

"I'd use a **staged rollout with feature flags**:

**Stage 1: Deploy all services with feature flag disabled**
- Deploy Database migration (expand)
- Deploy Order Service v1.1 (behind flag `new_priority_feature`)
- Deploy Inventory Service v1.1 (behind flag `new_priority_feature`)
- Deploy API Gateway v1.1 (behind flag)
- Deploy Frontend v1.1 (behind flag)

**State after Stage 1**: All new code is deployed, but behaves like old version (flags disabled).

**Stage 2: Enable feature flag for internal users**
- Enable `new_priority_feature` for employees only
- Test end-to-end (place test orders, verify priority works)
- Monitor metrics (error rate, latency)

**Stage 3: Gradual rollout to users**
- Enable for 1% of users
- Monitor for 1 hour (error rate, business metrics)
- Increase to 10%, then 50%, then 100%

**Stage 4: Clean up**
- After 100% rollout is stable for 2 weeks, remove feature flag
- After 4 weeks, run database contract phase (drop old columns)"

### 4. Define Rollback Strategy

"For each stage, I'd define rollback plans:

**If Stage 1 fails** (deployment issues):
- Rollback each service individually (standard rollback)
- Database migration (expand) is safe to leave (doesn't break old code)

**If Stage 2 fails** (internal testing finds bugs):
- Disable feature flag (instant rollback, no redeployment)
- Fix bug, re-enable flag

**If Stage 3 fails** (user-facing issues):
- Disable feature flag for affected users (granular rollback)
- Investigate, fix, re-enable

**If we can't fix**:
- Disable flag entirely
- Rollback all services to previous version
- Rollback database migration (drop new columns, safe because dual-write preserved old data)"

### 5. Monitoring and Alerts

"I'd set up comprehensive monitoring:

**Business metrics**:
- Order completion rate (should not decrease)
- Revenue (should not decrease)
- Feature usage (how many users using priority orders?)

**Technical metrics**:
- Error rate (by service, by version)
- Latency (p95, p99)
- Database query performance (new columns might slow down queries)

**Alerts**:
- P0: Order completion rate < 95% → Auto-rollback
- P0: Error rate > 1% → Auto-rollback
- P1: Latency p99 > 1.5x baseline → Pause rollout, investigate
- P2: Feature usage tracking (informational)

**Dashboard**: Real-time view of rollout progress:
- % of users with feature enabled
- Error rate (canary vs. baseline)
- Business metrics (orders/hour)"

### 6. Communication Plan

"I'd communicate proactively:

**Before deployment**:
- Email to engineering teams: 'Deploying priority orders feature Tuesday 10am'
- Slack announcement: 'Database migration scheduled, expect brief slowdown'
- Status page: 'Scheduled maintenance window 10am-12pm'

**During deployment**:
- Slack updates: 'Stage 1 complete, all services deployed'
- Status page: 'Maintenance in progress'

**After deployment**:
- Slack: 'Rollout complete, 100% of users on new feature'
- Email to stakeholders: 'Priority orders feature is live'
- Status page: 'All systems operational'

**If issues occur**:
- Slack: 'Rollback in progress due to elevated error rate'
- Status page: 'Investigating elevated error rates'
- Postmortem: Blameless review, document lessons learned"

### 7. Handling Edge Cases

"Some things that could go wrong:

**Database migration takes 6 hours** (large table):
- Run migration during low-traffic window (2am)
- Use online schema change tool (pt-online-schema-change for MySQL)
- Communicate extended maintenance window

**Circular dependency discovered** (Order Service calls Inventory, Inventory calls Order):
- Break the cycle (introduce message queue, make calls async)
- Or use feature flags (both services deployed, enabled together)

**One service fails to deploy**:
- Don't block entire rollout (feature flag keeps it disabled)
- Fix failing service, redeploy
- Enable flag once all services healthy

**Rollback needed after 2 weeks**:
- Database contract phase not started yet (still dual-writing)
- Disable feature flag (instant)
- Rollback services (standard process)
- Safe because old columns still exist"

**This demonstrates**:
- You understand dependencies (DAG, deployment order)
- You know database migration patterns (expand/contract)
- You use feature flags for coordination (decouple deploy from release)
- You plan for rollback at every stage
- You communicate proactively
- You monitor business + technical metrics
- You think through edge cases

## Summary: Orchestration Is Distributed Systems Engineering

Release orchestration is fundamentally a **distributed systems problem**. You're managing state changes across multiple independent services that communicate over unreliable networks.

**Key mental models**:
1. **Service autonomy vs. coordination**: Fast iteration vs. safe deployment - you can't maximize both
2. **DAG of dependencies**: Deploy dependencies before dependents
3. **Expand/contract pattern**: Zero-downtime database migrations through backward compatibility
4. **API versioning**: Manage breaking changes with deprecation timelines
5. **Feature flags for coordination**: Deploy all services, enable features together
6. **Communication cost scales as O(N²)**: More services = exponentially more coordination

**When interviewing**, be ready to:
- Map service dependencies (draw the DAG)
- Design a multi-service deployment strategy (orchestration, rollback, monitoring)
- Explain database migration patterns (why expand/contract, not naive approach)
- Discuss API versioning strategies (when to use each)
- Articulate trade-offs (release trains vs. continuous deployment, coordination vs. autonomy)
- Connect to distributed systems theory (Two Generals, CAP theorem)

The goal of release orchestration isn't to eliminate coordination - it's to **minimize unnecessary coordination** while maintaining safety. Use backward compatibility to decouple services. Use feature flags to decouple deployment from release. Use automated testing to catch integration issues early. And when coordination is truly needed (breaking changes, complex migrations), orchestrate carefully with clear rollback plans.

Microservices give you deployment independence, but that independence comes at the cost of coordination complexity. Good release orchestration is about finding the right balance for your organization's velocity and risk tolerance.
