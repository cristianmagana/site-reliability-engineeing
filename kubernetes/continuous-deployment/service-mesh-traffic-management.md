# Service Mesh Traffic Management: Fine-Grained Control for CD

## Why kube-proxy Isn't Enough

Kubernetes' built-in networking (kube-proxy) provides basic load balancing, but it has fundamental limitations for advanced continuous deployment.

### What kube-proxy Provides

**Layer 4 (TCP/UDP) load balancing**:

```
Client → Service (ClusterIP) → kube-proxy (iptables/ipvs) → Random pod
```

**Traffic distribution**: Random selection among healthy pods.

**Example** (Service with 3 pods):
```
Request 1 → Pod A (33% probability)
Request 2 → Pod C (33% probability)
Request 3 → Pod A (33% probability)
Request 4 → Pod B (33% probability)
```

Roughly equal distribution over time, but no guarantees per request.

### What kube-proxy Cannot Do

**1. Percentage-based traffic splitting**

Want 95% to stable, 5% to canary? With kube-proxy:
```
Stable: 19 pods
Canary: 1 pod
Total: 20 pods

Result: ~95/5 split, but requires 20 pods
```

Can't do 5% with fewer pods. Can't do 2%, 7%, or any arbitrary percentage.

**2. Header-based routing**

Want to route internal employees to canary?
```
if request.headers["X-User-Group"] == "internal":
    route to canary
else:
    route to stable
```

kube-proxy operates at Layer 4 (TCP). It doesn't see HTTP headers (Layer 7).

**3. Retry policies**

Request fails? kube-proxy doesn't retry. Application must handle retries.

**4. Circuit breaking**

Canary pods failing? kube-proxy keeps sending traffic. No automatic circuit breaking.

**5. Timeouts**

Want to timeout requests after 5 seconds? Application must implement it.

**6. Traffic mirroring**

Want to send duplicate traffic to canary for testing without affecting responses? Can't do it.

---

## Enter Service Mesh: Layer 7 Traffic Management

A service mesh adds a **sidecar proxy** to every pod. All traffic goes through the proxy.

```
Before (kube-proxy only):
  Client → Service → kube-proxy → Pod A

With service mesh:
  Client → Service → Sidecar Proxy → (rules: routing, retries, timeouts) → Pod A
```

**The sidecar proxy**:
- Intercepts all inbound and outbound traffic
- Operates at Layer 7 (HTTP, gRPC)
- Enforces routing rules, retries, timeouts
- Collects metrics (latency, error rates)
- Managed by control plane

### Service Mesh Architecture

```
                 ┌──────────────────┐
                 │  Control Plane   │
                 │  (Istiod, etc.)  │
                 └────────┬─────────┘
                          │ (config)
         ┌────────────────┼────────────────┐
         ↓                ↓                ↓
    ┌─────────┐      ┌─────────┐      ┌─────────┐
    │  Pod A  │      │  Pod B  │      │  Pod C  │
    │┌───────┐│      │┌───────┐│      │┌───────┐│
    ││Sidecar││      ││Sidecar││      ││Sidecar││
    ││ Proxy ││      ││ Proxy ││      ││ Proxy ││
    │└───┬───┘│      │└───┬───┘│      │└───┬───┘│
    │    │App │      │    │App │      │    │App │
    └────┼────┘      └────┼────┘      └────┼────┘
         └────────────────┴────────────────┘
                 (data plane - proxies)
```

**Control plane**: Configures proxies (routing rules, policies)
**Data plane**: Proxies handle actual traffic

---

## Istio: Feature-Rich Service Mesh

### Core Concepts

**VirtualService**: Routing rules (where traffic goes)
**DestinationRule**: Policies for traffic to a destination (load balancing, circuit breaking)
**Gateway**: Ingress/egress traffic management

### VirtualService: Traffic Routing

**Basic routing**:

```yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: myapp
spec:
  hosts:
  - myapp.default.svc.cluster.local
  http:
  - route:
    - destination:
        host: myapp
        subset: stable
```

All traffic to `myapp` goes to `stable` subset.

**Canary: Percentage-based routing**:

```yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: myapp
spec:
  hosts:
  - myapp
  http:
  - route:
    - destination:
        host: myapp
        subset: stable
      weight: 95  # 95% to stable

    - destination:
        host: myapp
        subset: canary
      weight: 5   # 5% to canary
```

**Result**: Exactly 5% of requests to canary, regardless of pod count.

**Header-based routing**:

```yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: myapp
spec:
  hosts:
  - myapp
  http:
  # Route 1: Internal employees → canary
  - match:
    - headers:
        x-user-group:
          exact: "internal"
    route:
    - destination:
        host: myapp
        subset: canary

  # Route 2: Beta users → canary
  - match:
    - headers:
        x-user-type:
          exact: "beta"
    route:
    - destination:
        host: myapp
        subset: canary

  # Route 3: Everyone else → stable
  - route:
    - destination:
        host: myapp
        subset: stable
```

**Specific user canary**:

```yaml
http:
- match:
  - headers:
      user-id:
        exact: "user-12345"
  route:
  - destination:
      host: myapp
      subset: canary
```

User 12345 always gets canary. Others get stable.

**URI-based routing**:

```yaml
http:
- match:
  - uri:
      prefix: "/api/v2"
  route:
  - destination:
      host: myapp
      subset: v2

- match:
  - uri:
      prefix: "/api/v1"
  route:
  - destination:
      host: myapp
      subset: v1
```

Route based on API version in path.

### DestinationRule: Traffic Policies

**Define subsets** (versions):

```yaml
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: myapp
spec:
  host: myapp
  subsets:
  - name: stable
    labels:
      version: stable

  - name: canary
    labels:
      version: canary
```

Subsets correspond to pod labels.

**Load balancing**:

```yaml
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: myapp
spec:
  host: myapp
  trafficPolicy:
    loadBalancer:
      simple: LEAST_REQUEST  # or ROUND_ROBIN, RANDOM, PASSTHROUGH

  subsets:
  - name: stable
    labels:
      version: stable
    trafficPolicy:
      loadBalancer:
        consistentHash:
          httpHeaderName: "user-id"  # Sticky sessions by user-id
```

**Options**:
- `ROUND_ROBIN`: Cycle through pods
- `LEAST_REQUEST`: Send to pod with fewest active requests
- `RANDOM`: Random selection
- `PASSTHROUGH`: No load balancing
- `consistentHash`: Sticky sessions (by header, cookie, IP)

**Circuit breaking**:

```yaml
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: myapp
spec:
  host: myapp
  trafficPolicy:
    connectionPool:
      tcp:
        maxConnections: 100      # Max TCP connections per pod
      http:
        http1MaxPendingRequests: 10   # Max pending requests
        http2MaxRequests: 100          # Max concurrent requests
        maxRequestsPerConnection: 2    # Max requests per connection

    outlierDetection:
      consecutiveErrors: 5       # Eject pod after 5 errors
      interval: 30s              # Check every 30s
      baseEjectionTime: 30s      # Eject for at least 30s
      maxEjectionPercent: 50     # Max 50% of pods can be ejected
```

**How circuit breaking works**:

```
Request to Pod A: Error
Request to Pod A: Error
Request to Pod A: Error
Request to Pod A: Error
Request to Pod A: Error (5th consecutive error)

Action: Pod A ejected from load balancer for 30 seconds

Subsequent requests: Only go to Pod B, Pod C (healthy pods)

After 30 seconds: Pod A re-added to pool, given another chance
```

**Retry policy**:

```yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: myapp
spec:
  hosts:
  - myapp
  http:
  - route:
    - destination:
        host: myapp
    retries:
      attempts: 3              # Retry up to 3 times
      perTryTimeout: 2s        # Timeout each attempt after 2s
      retryOn: 5xx,reset,connect-failure  # Retry on these conditions
```

**Timeout**:

```yaml
http:
- route:
  - destination:
      host: myapp
  timeout: 10s  # Request times out after 10s
```

**Fault injection** (for testing):

```yaml
http:
- fault:
    delay:
      percentage:
        value: 10.0     # 10% of requests
      fixedDelay: 5s    # Delayed by 5 seconds

    abort:
      percentage:
        value: 5.0      # 5% of requests
      httpStatus: 503   # Fail with 503
  route:
  - destination:
      host: myapp
```

Use case: Test how application handles slow dependencies or failures.

---

## Linkerd: Lightweight Service Mesh

Linkerd is simpler and lighter than Istio. Focused on performance and ease of use.

### Key Differences from Istio

**Simplicity**: Fewer CRDs, less configuration complexity
**Performance**: Rust-based proxy (vs. Envoy in Istio), lower latency and resource usage
**Automatic features**: mTLS, retries, timeouts work out-of-the-box

### Traffic Splitting with Linkerd

**SMI (Service Mesh Interface)** standard:

```yaml
apiVersion: split.smi-spec.io/v1alpha1
kind: TrafficSplit
metadata:
  name: myapp-split
spec:
  service: myapp          # Root service
  backends:
  - service: myapp-stable
    weight: 900           # 90% (900/1000)

  - service: myapp-canary
    weight: 100           # 10% (100/1000)
```

**Services**:

```yaml
# Root service (used by clients)
apiVersion: v1
kind: Service
metadata:
  name: myapp
spec:
  selector:
    app: myapp

---
# Stable backend
apiVersion: v1
kind: Service
metadata:
  name: myapp-stable
spec:
  selector:
    app: myapp
    version: stable

---
# Canary backend
apiVersion: v1
kind: Service
metadata:
  name: myapp-canary
spec:
  selector:
    app: myapp
    version: canary
```

**How it works**:

```
Client requests myapp service
  ↓
Linkerd proxy intercepts
  ↓
Checks TrafficSplit
  ↓
90% → myapp-stable service
10% → myapp-canary service
```

### Linkerd Automatic Features

**Retries**: Automatic retry on failure (configurable via annotation)

```yaml
metadata:
  annotations:
    retry.linkerd.io/http: "5xx"  # Retry on 5xx errors
    retry.linkerd.io/limit: "3"   # Max 3 retries
```

**Timeouts**:

```yaml
metadata:
  annotations:
    timeout.linkerd.io/request: "10s"
```

**mTLS**: Automatic mutual TLS between services (encrypted, authenticated)

---

## Progressive Delivery Integration

### Canary Deployment with Istio

**Initial state**:

```yaml
# 100% to stable
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: myapp
spec:
  hosts:
  - myapp
  http:
  - route:
    - destination:
        host: myapp
        subset: stable
      weight: 100
```

**Canary starts** (Flagger or manual):

```yaml
# 5% to canary
http:
- route:
  - destination:
      host: myapp
      subset: stable
    weight: 95

  - destination:
      host: myapp
      subset: canary
    weight: 5
```

**Gradual increase**:

```
Iteration 1: 5% canary
  Check metrics → Pass

Iteration 2: 10% canary
  Check metrics → Pass

Iteration 3: 25% canary
  Check metrics → Pass

Iteration 4: 50% canary
  Check metrics → Pass

Promotion: 100% canary
  Update stable deployment to canary version
  Set weight: 100% stable (which is now the new version)
```

### Blue/Green with Istio

**Both versions running**:

```yaml
http:
# All traffic to blue
- route:
  - destination:
      host: myapp
      subset: blue
    weight: 100

  - destination:
      host: myapp
      subset: green
    weight: 0
```

**Cutover** (instant):

```yaml
# All traffic to green
- route:
  - destination:
      host: myapp
      subset: blue
    weight: 0

  - destination:
      host: myapp
      subset: green
    weight: 100
```

Change takes effect immediately (seconds).

---

## Observability Benefits

Service meshes provide rich metrics automatically.

### Request-Level Metrics

**Automatically collected**:
- Request count
- Request duration (latency)
- Error rates
- Traffic volume

**Without application changes**: Sidecar proxy captures everything.

### Golden Signals (Google SRE)

**Latency**: Request duration (P50, P99)
**Traffic**: Requests per second
**Errors**: Error rate (5xx responses)
**Saturation**: Resource utilization (connections, queue depth)

**Prometheus metrics** (example from Istio):

```promql
# Request rate
rate(istio_requests_total{destination_service="myapp"}[1m])

# Error rate
rate(istio_requests_total{destination_service="myapp",response_code=~"5.."}[1m])
/
rate(istio_requests_total{destination_service="myapp"}[1m])

# P99 latency
histogram_quantile(0.99,
  rate(istio_request_duration_milliseconds_bucket{destination_service="myapp"}[1m])
)
```

### Distributed Tracing

Service mesh integrates with tracing systems (Jaeger, Zipkin).

**Automatic trace propagation**:

```
Request: Client → Service A → Service B → Service C

Trace:
  Span 1: Client → Service A (50ms)
    Span 2: Service A → Service B (30ms)
      Span 3: Service B → Service C (20ms)

Total latency: 50ms
  Service A processing: 20ms (50 - 30)
  Service B processing: 10ms (30 - 20)
  Service C processing: 20ms
```

See exactly where time is spent across services.

---

## When to Use a Service Mesh

### Use a Service Mesh If:

✅ **Need fine-grained traffic control** (5% canary, header-based routing)
✅ **Automated canary/progressive delivery** (Flagger, Argo Rollouts)
✅ **Multi-service architecture** (many microservices communicating)
✅ **Need resilience patterns** (retries, timeouts, circuit breaking)
✅ **Security requirements** (mTLS between services)
✅ **Centralized observability** (metrics, tracing across all services)

### Don't Use a Service Mesh If:

❌ **Simple monolith** (single service, no microservices)
❌ **Resource-constrained** (sidecar proxies add CPU/memory overhead)
❌ **Operational complexity concerns** (service mesh adds moving parts)
❌ **Native K8s is sufficient** (basic rolling updates work fine)

### Cost-Benefit Analysis

**Costs**:
- **Resource overhead**: Sidecar per pod (100-200MB memory, ~0.1 CPU per proxy)
- **Operational complexity**: New components to manage, learn, debug
- **Latency**: Small increase (1-5ms per hop due to proxy)

**Benefits**:
- **Fine-grained control**: Precise traffic splitting, advanced routing
- **Resilience**: Automatic retries, circuit breaking, timeouts
- **Observability**: Metrics and tracing without code changes
- **Security**: mTLS without application changes

**When it's worth it**: Multi-service systems (10+ services) with advanced deployment needs.

---

## Istio vs. Linkerd Comparison

| Feature | Istio | Linkerd |
|---------|-------|---------|
| **Proxy** | Envoy (C++) | Linkerd2-proxy (Rust) |
| **Performance** | Good | Excellent (lower latency, less memory) |
| **Features** | Comprehensive | Focused essentials |
| **Complexity** | High (many CRDs) | Low (simpler) |
| **mTLS** | Manual configuration | Automatic |
| **Traffic splitting** | VirtualService | SMI TrafficSplit |
| **Fault injection** | ✅ | ❌ |
| **External services** | ✅ (ServiceEntry) | Limited |
| **Multi-cluster** | ✅ | ✅ |
| **Adoption** | Higher (CNCF graduated) | Growing (CNCF graduated) |

**Choose Istio if**: Need advanced features (fault injection, complex routing, multi-cluster federation)

**Choose Linkerd if**: Want simplicity, performance, out-of-the-box security

---

## Interview Deep Dive

**Question**: "Why use a service mesh instead of kube-proxy?"

**Shallow answer**: "Service mesh gives more control."

**Deep answer**:
"kube-proxy operates at Layer 4 (TCP/UDP), providing basic load balancing by randomly distributing connections to pod IPs. It's sufficient for simple use cases but has fundamental limitations for advanced continuous deployment.

First, percentage-based traffic splitting requires manipulating replica counts. To achieve 5% canary with kube-proxy, you need 1 canary pod and 19 stable pods - 20 total. You can't do arbitrary percentages without adding pods.

Second, kube-proxy can't route based on HTTP headers, cookies, or request attributes because it doesn't parse Layer 7 protocols. You can't route internal users to canary or implement A/B testing based on user segments.

Third, there's no built-in retry, timeout, or circuit breaking. If a request fails, the application must handle it. If a pod is failing, kube-proxy continues sending traffic.

Service mesh adds a sidecar proxy to every pod, intercepting traffic at Layer 7. This enables percentage-based routing independent of replicas (5% to canary with 3 pods each), header-based routing (route beta users to canary), and automatic retries, timeouts, circuit breaking. The mesh also provides comprehensive observability - request-level metrics, latency percentiles, error rates, distributed tracing - without application code changes.

The trade-off is complexity and resource overhead - each sidecar adds ~100-200MB memory and introduces small latency. It's justified for microservices architectures needing progressive delivery and resilience patterns."

**Question**: "How does circuit breaking work in a service mesh?"

**Deep answer**:
"Circuit breaking in service mesh prevents cascading failures by temporarily removing unhealthy pods from the load balancer pool based on error rates.

The sidecar proxy tracks errors per backend pod. When a pod exceeds a consecutive error threshold - say 5 errors in a row - the proxy ejects it from the pool for a base ejection time, like 30 seconds. During ejection, no traffic goes to that pod. This prevents the 'thundering herd' problem where all requests pile onto a failing instance.

There's also connection pool limits - max concurrent connections, max pending requests. If a pod hits these limits, the circuit breaks immediately rather than queueing indefinitely. This caps the blast radius.

The key parameters are consecutive errors (how many failures before ejection), interval (how often to check), base ejection time (how long to eject), and max ejection percent (can't eject more than 50% of pods, preventing total failure if all pods are struggling).

After the ejection time expires, the pod is gradually reintroduced - maybe 10% of its normal traffic initially. If it succeeds, traffic increases. If it still fails, it's ejected again.

This is superior to application-level circuit breakers because it's centralized - the mesh handles it for all services uniformly - and it's based on actual traffic patterns, not synthetic health checks."

---

## Key Takeaways

1. **kube-proxy limitations**: Layer 4 only, no percentage splits, no header routing, no retries
2. **Service mesh adds Layer 7**: HTTP-aware routing, fine-grained traffic control
3. **Istio**: Feature-rich, VirtualServices for routing, DestinationRules for policies
4. **Linkerd**: Lightweight, automatic mTLS, SMI TrafficSplit
5. **Progressive delivery**: Enables automated canary (Flagger, Argo Rollouts)
6. **Observability**: Automatic metrics, distributed tracing, no code changes
7. **Trade-offs**: Resource overhead, complexity vs. advanced capabilities

Service mesh is infrastructure for advanced continuous deployment. It separates traffic management from application code, providing centralized control over routing, resilience, and observability. Essential for progressive delivery at scale, but adds operational complexity - use when benefits outweigh costs.
