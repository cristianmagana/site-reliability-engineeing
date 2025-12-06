# Health Checks and Probes: The Foundation of Safe Deployments

## The Problem Without Health Checks

Traditional deployment: Deploy new version, hope it works.

```bash
# Traditional
deploy_new_version()
send_traffic()  # Immediately
# Hope the application is ready...
```

**What can go wrong**:
- Application still starting up (loading config, warming caches)
- Database connection not established yet
- Dependency services not reachable
- Application crashes immediately after start

**Result**: Users get errors (503, timeout) until application is actually ready.

**Kubernetes solution**: Don't send traffic until application explicitly signals it's ready.

---

## Three Types of Probes

Kubernetes has three distinct probe types, each serving a different purpose.

### Liveness Probe: "Is the Application Alive?"

**Question answered**: Should Kubernetes restart this container?

**Example scenario**: Application has a deadlock. Process is running, but not responding.

```yaml
livenessProbe:
  httpGet:
    path: /healthz
    port: 8080
  initialDelaySeconds: 30
  periodSeconds: 10
  timeoutSeconds: 5
  failureThreshold: 3
```

**Behavior**:
- Kubernetes calls `GET /healthz` every 10 seconds
- If response is 2xx/3xx: Healthy
- If response is 4xx/5xx or timeout: Unhealthy
- After 3 consecutive failures (failureThreshold): Restart container

**Use case**: Detect and recover from application deadlocks, infinite loops, unrecoverable errors.

### Readiness Probe: "Is the Application Ready for Traffic?"

**Question answered**: Should Kubernetes send traffic to this pod?

**Example scenario**: Application is starting up, loading 2GB of data into memory. Process is alive, but can't serve requests yet.

```yaml
readinessProbe:
  httpGet:
    path: /ready
    port: 8080
  initialDelaySeconds: 10
  periodSeconds: 5
  timeoutSeconds: 3
  failureThreshold: 2
```

**Behavior**:
- Kubernetes calls `GET /ready` every 5 seconds
- If healthy: Add pod IP to Service endpoints (receives traffic)
- If unhealthy: Remove pod IP from Service endpoints (no traffic)
- Container is NOT restarted (just removed from load balancer)

**Use case**:
- Application startup (loading data, warming caches)
- Temporary unavailability (overloaded, waiting for dependency)
- Graceful shutdown (draining connections)

### Startup Probe: "Is the Application Still Starting Up?"

**Question answered**: Should Kubernetes wait longer before checking liveness?

**Example scenario**: Application takes 2 minutes to start (loading large ML model). Liveness probe would kill it prematurely.

```yaml
startupProbe:
  httpGet:
    path: /startup
    port: 8080
  initialDelaySeconds: 0
  periodSeconds: 10
  failureThreshold: 30  # 30 * 10s = 5 minutes max startup time
```

**Behavior**:
- Runs BEFORE liveness and readiness
- Kubernetes calls `GET /startup` every 10 seconds
- Once startup succeeds: Liveness and readiness probes begin
- If startup fails 30 times (5 minutes): Restart container

**Use case**:
- Slow-starting applications (loading ML models, large datasets)
- Legacy applications with unpredictable startup times

---

## The Interaction: How Probes Work Together

```
Pod starts
    ↓
Startup probe runs (if configured)
    ↓ (every 10s until success or failure)
Startup succeeds
    ↓
Liveness probe starts (every 10s)
Readiness probe starts (every 5s)
    ↓
Both running continuously

If liveness fails → Container restarts
If readiness fails → Removed from Service (no restart)
```

**Example timeline**:

```
Time    Event
─────────────────────────────────────────────────────
0:00    Container starts
0:00    Startup probe: Check /startup (initialDelay: 0)
0:10    Startup probe: Fail (still loading)
0:20    Startup probe: Fail (still loading)
0:30    Startup probe: Success (loaded)
0:30    Liveness probe: Start checking /healthz
0:30    Readiness probe: Start checking /ready
0:35    Readiness probe: Success → Pod added to Service
        (Traffic begins)
0:40    Liveness probe: Success
0:40    Readiness probe: Success
...
2:00    Readiness probe: Fail (temporary overload)
        → Pod removed from Service (no new traffic)
2:05    Readiness probe: Fail (still overloaded)
2:10    Readiness probe: Success (recovered)
        → Pod added back to Service
```

Pod was never restarted (liveness still passing). Just temporarily removed from load balancer.

---

## Probe Configuration Parameters

### initialDelaySeconds

**What it is**: How long to wait after container starts before first probe.

```yaml
initialDelaySeconds: 30
```

**Too low**: Probe fails during startup → container restarted unnecessarily
**Too high**: Slow to detect issues, slow to start serving traffic

**Recommendation**:
- Liveness: Set to expected startup time (or use startup probe instead)
- Readiness: Set low (5-10s) if using startup probe
- Startup: 0 (start checking immediately)

### periodSeconds

**What it is**: How often to run the probe.

```yaml
periodSeconds: 10
```

**Too low**: Excessive load on application (probes every second)
**Too high**: Slow to detect failures

**Recommendation**:
- Liveness: 10-30s (doesn't need to be frequent)
- Readiness: 5-10s (needs to be responsive for traffic routing)
- Startup: 10s (balance between fast detection and load)

### timeoutSeconds

**What it is**: How long to wait for probe response before considering it failed.

```yaml
timeoutSeconds: 5
```

**Too low**: False negatives (probe times out even when healthy)
**Too high**: Slow failure detection

**Recommendation**: 3-10s depending on application response time

### failureThreshold

**What it is**: How many consecutive failures before taking action.

```yaml
failureThreshold: 3
```

**Too low (1)**: Single transient failure causes restart/removal
**Too high (10)**: Slow to react to real failures

**Recommendation**:
- Liveness: 3-5 (avoid premature restarts)
- Readiness: 1-3 (quick removal from load balancer acceptable)
- Startup: High (30+) to allow long startup

### successThreshold

**What it is**: How many consecutive successes before considering healthy.

```yaml
successThreshold: 1  # Default and only valid value for liveness/startup
```

**Note**: Only configurable for readiness probe. Liveness and startup require `successThreshold: 1`.

**For readiness**: Setting to 2-3 prevents flapping (pod removed and added rapidly).

---

## Probe Types

### HTTP GET

Most common. Kubernetes sends HTTP GET request.

```yaml
httpGet:
  path: /healthz
  port: 8080
  httpHeaders:
  - name: X-Custom-Header
    value: Awesome
  scheme: HTTP  # or HTTPS
```

**Response codes**:
- 200-399: Success
- Others: Failure

**Use case**: HTTP/REST services (most applications)

### TCP Socket

Kubernetes attempts to open TCP connection.

```yaml
tcpSocket:
  port: 5432
```

**Success**: Connection established
**Failure**: Connection refused or timeout

**Use case**: Non-HTTP services (databases, message queues)

### Exec (Command)

Kubernetes runs a command in the container.

```yaml
exec:
  command:
  - cat
  - /tmp/healthy
```

**Success**: Exit code 0
**Failure**: Non-zero exit code

**Use case**: Custom health checks, legacy applications

### gRPC

Kubernetes calls a gRPC health check service.

```yaml
grpc:
  port: 9090
  service: liveness  # Optional service name
```

**Requires**: Application implements gRPC Health Checking Protocol

**Use case**: gRPC microservices

---

## Probe Design Patterns

### Pattern 1: Separate Endpoints for Liveness and Readiness

```go
// Liveness: Check if application is fundamentally broken
http.HandleFunc("/healthz", func(w http.ResponseWriter, r *http.Request) {
    // Minimal check: Is the process alive and not deadlocked?
    w.WriteHeader(200)
    w.Write([]byte("ok"))
})

// Readiness: Check if application can handle traffic
http.HandleFunc("/ready", func(w http.ResponseWriter, r *http.Request) {
    // Check dependencies
    if !database.Ping() {
        w.WriteHeader(503)
        w.Write([]byte("database unavailable"))
        return
    }

    if !cache.Ping() {
        w.WriteHeader(503)
        w.Write([]byte("cache unavailable"))
        return
    }

    // Check if overloaded
    if activeConnections > maxConnections {
        w.WriteHeader(503)
        w.Write([]byte("overloaded"))
        return
    }

    w.WriteHeader(200)
    w.Write([]byte("ready"))
})
```

**Why separate**:
- Liveness failures → restart (drastic)
- Readiness failures → remove from LB (graceful)

**Liveness should check**: Application process fundamentals
**Readiness should check**: Dependencies, capacity, temporary conditions

### Pattern 2: Don't Check Dependencies in Liveness

```go
// BAD: Liveness checks database
http.HandleFunc("/healthz", func(w http.ResponseWriter, r *http.Request) {
    if !database.Ping() {
        w.WriteHeader(503)  // ← BAD
        return
    }
    w.WriteHeader(200)
})
```

**Problem**: Database goes down → liveness fails → all pods restart → pods still can't reach database → infinite restart loop.

**Cascading failure**: One service's failure causes all dependent services to restart.

```go
// GOOD: Liveness checks only itself
http.HandleFunc("/healthz", func(w http.ResponseWriter, r *http.Request) {
    // Only check if application process is healthy
    w.WriteHeader(200)
})

// Readiness checks dependencies
http.HandleFunc("/ready", func(w http.ResponseWriter, r *http.Request) {
    if !database.Ping() {
        w.WriteHeader(503)  // ← OK for readiness
        return
    }
    w.WriteHeader(200)
})
```

**Result**: Database down → readiness fails → pods removed from LB → no new traffic → pods wait for database to recover → no restarts.

### Pattern 3: Startup Probe for Slow-Starting Applications

```yaml
# Application takes 3 minutes to load ML model

# Without startup probe: Liveness must wait 3 minutes
livenessProbe:
  httpGet:
    path: /healthz
    port: 8080
  initialDelaySeconds: 180  # 3 minutes
  periodSeconds: 10

# Problem: If application crashes at 2:30, liveness doesn't detect until 3:00
```

```yaml
# With startup probe: Liveness can be aggressive after startup
startupProbe:
  httpGet:
    path: /healthz
    port: 8080
  periodSeconds: 10
  failureThreshold: 30  # 30 * 10s = 5 minutes max

livenessProbe:
  httpGet:
    path: /healthz
    port: 8080
  periodSeconds: 10
  failureThreshold: 3  # Only 30s after startup

# Startup probe gives 5 minutes for initial load
# After startup succeeds, liveness is aggressive (30s failure detection)
```

### Pattern 4: Graceful Shutdown with Readiness

```go
// Signal handler for graceful shutdown
sigChan := make(chan os.Signal, 1)
signal.Notify(sigChan, syscall.SIGTERM)

go func() {
    <-sigChan

    // 1. Fail readiness probe immediately
    readinessLock.Lock()
    isReady = false
    readinessLock.Unlock()

    // 2. Wait for traffic to drain (removed from endpoints)
    time.Sleep(10 * time.Second)

    // 3. Close server
    server.Shutdown(context.Background())
}()
```

**Process**:
1. Pod receives SIGTERM (kubectl delete, rolling update)
2. Readiness probe starts failing immediately
3. Kubernetes removes pod from Service endpoints (no new traffic)
4. Existing connections finish (grace period)
5. Server shuts down cleanly

---

## Integration with Rolling Updates

Readiness probes are critical for zero-downtime deployments.

### Rolling Update Flow

```
Deployment controller creates new pod (v1.1.0)
    ↓
Pod starts, readiness probe runs
    ↓
Readiness probe fails (still starting)
    ↓ (every 5s)
Readiness probe succeeds
    ↓
Pod marked Ready
    ↓
Deployment controller sees new pod Ready
    ↓
Terminates old pod (v1.0.0)
    ↓
Old pod receives SIGTERM
    ↓
Old pod's readiness probe fails (graceful shutdown)
    ↓
Old pod removed from Service
    ↓
Old pod terminates
```

**Without readiness probes**:
- New pod gets traffic immediately (might not be ready) → errors
- Old pod terminates immediately (might have in-flight requests) → errors

**With readiness probes**:
- New pod waits until ready before getting traffic
- Old pod drains traffic before terminating

### MinReadySeconds

```yaml
spec:
  minReadySeconds: 30
```

**What it does**: Pod must be ready for this duration before considered "available."

**Use case**: Prevent flaky pods (passes readiness, crashes 10 seconds later) from being counted.

**Effect on rolling update**:
```
New pod ready at 0:10
minReadySeconds: 30
Pod considered available at 0:40
Deployment controller waits until 0:40 before continuing rollout
```

Slows rollout but increases safety.

---

## Common Anti-Patterns

### Anti-Pattern 1: Liveness = Readiness

```yaml
# BAD: Same probe for liveness and readiness
livenessProbe:
  httpGet:
    path: /health  # ← Checks dependencies
    port: 8080

readinessProbe:
  httpGet:
    path: /health  # ← Same endpoint
    port: 8080
```

**Problem**: Database unavailable → both fail → pod removed from Service AND restarted → restart doesn't fix database → infinite restart loop.

**Fix**: Separate endpoints, liveness checks only self.

### Anti-Pattern 2: Expensive Probes

```go
// BAD: Liveness probe does expensive operation
http.HandleFunc("/healthz", func(w http.ResponseWriter, r *http.Request) {
    // Full database query
    rows := db.Query("SELECT COUNT(*) FROM users")  // ← SLOW

    // Heavy computation
    result := complexAlgorithm()  // ← EXPENSIVE

    w.WriteHeader(200)
})
```

**Problem**: Probe runs every 10s → 6 expensive ops/minute → application under constant load from health checks.

**Fix**: Probes should be lightweight (< 100ms).

```go
// GOOD: Liveness is cheap
http.HandleFunc("/healthz", func(w http.ResponseWriter, r *http.Request) {
    w.WriteHeader(200)  // Just alive check
})
```

### Anti-Pattern 3: No Startup Probe for Slow Apps

```yaml
# BAD: Slow app without startup probe
livenessProbe:
  httpGet:
    path: /healthz
    port: 8080
  initialDelaySeconds: 10  # ← Too low for 2-minute startup
  periodSeconds: 10
  failureThreshold: 3

# Application takes 2 minutes to start
# Liveness starts checking at 0:10
# Fails at 0:10, 0:20, 0:30 (3 failures)
# Container restarted at 0:30 (before it could finish starting!)
```

**Fix**: Use startup probe or increase initialDelaySeconds.

### Anti-Pattern 4: Ignoring Probe Results

```yaml
# Probe configured but application doesn't implement endpoint
readinessProbe:
  httpGet:
    path: /ready
    port: 8080
```

```
Application doesn't have /ready endpoint
→ Probe gets 404
→ Readiness always fails
→ Pod never receives traffic
```

**Fix**: Implement the probe endpoint!

---

## Readiness Gates: Custom Readiness

For advanced scenarios, you can define custom readiness conditions.

```yaml
spec:
  readinessGates:
  - conditionType: "example.com/feature-flag-enabled"
```

**Use case**:
- Don't send traffic until external system approves
- Progressive delivery: External controller sets readiness based on metrics
- Multi-cluster deployments: Don't send traffic until DNS propagated

**External controller** sets the condition:

```go
pod.Status.Conditions = append(pod.Status.Conditions, v1.PodCondition{
    Type:   "example.com/feature-flag-enabled",
    Status: v1.ConditionTrue,
})
```

Pod is not considered Ready until all readiness gates are True.

---

## Interview Deep Dive

**Question**: "What's the difference between liveness and readiness probes?"

**Shallow answer**: "Liveness restarts the pod, readiness removes from service."

**Deep answer**:
"Liveness and readiness serve fundamentally different purposes and have different failure semantics.

Liveness answers 'Is the application process fundamentally broken?' - deadlocked, in an infinite loop, corrupted state that can't self-heal. Liveness failures trigger a restart because the assumption is the problem is unrecoverable without restarting. The check should only verify the process itself, not dependencies. Checking dependencies in liveness causes cascading failures - if a database goes down, all dependent services restart, which doesn't fix the database and just creates more load.

Readiness answers 'Can the application handle traffic right now?' - has it finished starting, are its dependencies available, is it overloaded? Readiness failures just remove the pod from Service endpoints - no restart. This is appropriate for temporary conditions that might resolve (database comes back, load decreases, caches warm up).

The key design principle: liveness should be a narrow check (is MY process healthy?), readiness can be broader (am I and my dependencies ready?). Violating this causes problems - checking dependencies in liveness creates restart storms, not having separate readiness means traffic hits starting/overloaded pods."

**Question**: "How do probes integrate with rolling updates?"

**Deep answer**:
"Readiness probes are the mechanism that enables zero-downtime rolling updates. When the Deployment controller scales up a new ReplicaSet, it creates new pods but doesn't immediately terminate old pods. It waits for new pods' readiness probes to succeed before marking them as Ready.

Only after a new pod is Ready does the controller proceed - it terminates an old pod, respecting maxUnavailable constraints. The old pod receives SIGTERM and ideally fails its readiness probe immediately as part of graceful shutdown. This removes it from Service endpoints before termination, preventing new traffic from hitting a terminating pod.

The critical timing is: New pod ready → old pod terminates → old pod removed from endpoints → old pod stops. Without readiness, you'd have: new pod starts → gets traffic immediately (might fail) → old pod terminates (in-flight requests fail). The probe provides the synchronization point - new pod must prove it's ready before old pod can be terminated.

Additionally, minReadySeconds adds a soak period - pod must be ready for N seconds before considered available. This prevents flaky pods that pass readiness but crash shortly after from being considered healthy."

---

## Key Takeaways

1. **Three probe types**: Liveness (restart?), Readiness (traffic?), Startup (still starting?)
2. **Separate endpoints**: Liveness checks self, readiness checks dependencies
3. **Avoid cascading failures**: Don't check dependencies in liveness
4. **Startup probe for slow apps**: Allows aggressive liveness after startup completes
5. **Graceful shutdown**: Fail readiness on SIGTERM, drain traffic before exit
6. **Integration with rolling updates**: Readiness gates traffic, enables zero-downtime
7. **Lightweight probes**: Health checks run frequently, keep them fast

Health checks are the contract between your application and Kubernetes. Without them, Kubernetes is blind - it can't tell if your app is ready, if it's broken, if it needs restart. With well-designed probes, Kubernetes can make intelligent decisions about traffic routing and failure recovery, enabling truly self-healing deployments.
