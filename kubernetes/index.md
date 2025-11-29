# Kubernetes Bare Metal Deep Dive - Complete Guide

## Table of Contents

1. [Core Kubernetes Components](#1-core-kubernetes-components)
   - [Control Plane Components](#control-plane-components)
   - [Node Components](#node-components)
2. [Complete Pod Lifecycle - Deployment to Traffic](#2-complete-pod-lifecycle---deployment-to-traffic)
   - [Phase 1: Deployment Creation](#phase-1-deployment-creation)
   - [Phase 2: Pod Scheduling](#phase-2-pod-scheduling)
   - [Phase 3: Kubelet Takes Over](#phase-3-kubelet-takes-over)
   - [Phase 4: Service Discovery & Load Balancing](#phase-4-service-discovery--load-balancing)
   - [Phase 5: Ingress and External Traffic](#phase-5-ingress-and-external-traffic)
   - [Phase 6: Complete Traffic Flow](#phase-6-complete-traffic-flow)
3. [File System Layout - Critical Paths](#3-file-system-layout---critical-paths)
4. [Troubleshooting by Layer](#4-troubleshooting-by-layer)
   - [Layer 1: Node Health & System Resources](#layer-1-node-health--system-resources)
   - [Layer 2: Kubelet - The Node Agent](#layer-2-kubelet---the-node-agent)
   - [Layer 3: Container Runtime (containerd)](#layer-3-container-runtime-containerd)
   - [Layer 4: Networking & CNI](#layer-4-networking--cni)
   - [Layer 5: Control Plane Components](#layer-5-control-plane-components)
   - [Layer 6: Certificates](#layer-6-certificates)
   - [Layer 7: DNS & CoreDNS](#layer-7-dns--coredns)
5. [Advanced Debugging Scenarios](#5-advanced-debugging-scenarios)
6. [Monitoring Commands - Quick Reference](#6-monitoring-commands---quick-reference)
7. [Emergency Procedures](#7-emergency-procedures)

---

## 1. Core Kubernetes Components

### Control Plane Components

#### kube-apiserver
- The central management entity and only component that directly talks to etcd
- Validates and processes REST requests
- Implements the Kubernetes API (RESTful interface)
- Horizontally scalable (can run multiple instances with load balancing)
- Authentication, authorization (RBAC), and admission control happens here
- All cluster communication flows through the API server

#### etcd
- Distributed key-value store (Raft consensus algorithm)
- Single source of truth for cluster state
- Stores all cluster data: pods, services, secrets, config maps, etc.
- Uses watch mechanism for change notifications
- Critical for HA: typically run 3 or 5 nodes (odd numbers for quorum)
- On bare metal, you'd typically colocate with control plane or run dedicated etcd cluster

#### kube-scheduler
- Watches for newly created pods with no assigned node
- Selects a node based on:
  - Resource requirements (CPU, memory, ephemeral storage)
  - Affinity/anti-affinity rules
  - Taints and tolerations
  - Node selectors and labels
  - Pod topology spread constraints
  - Data locality considerations
- Two-phase process: filtering (finding feasible nodes) and scoring (ranking nodes)
- Doesn't actually place the pod—just updates the pod spec with nodeName

#### kube-controller-manager
- Runs multiple controllers in a single process:
  - **Node controller**: Monitors node health, marks nodes as NotReady
  - **Replication controller**: Maintains correct number of pods
  - **Endpoints controller**: Populates Endpoints objects (joins Services & Pods)
  - **Service Account & Token controllers**: Create default accounts and API access tokens
  - **Deployment controller**: Manages ReplicaSets
  - **StatefulSet controller**: Manages stateful applications
- Each controller watches specific resources and reconciles desired vs actual state

#### cloud-controller-manager
- On bare metal, you typically DON'T use this (it's for cloud providers)
- Or you'd use something like Metal LB controller for LoadBalancer services

### Node Components

#### kubelet
- Primary node agent running on every node
- Registers node with API server
- Watches API server for pods assigned to its node
- Manages pod lifecycle via CRI
- Reports node and pod status back to API server
- Runs liveness/readiness/startup probes
- Manages volumes and secrets
- Collects metrics (cAdvisor integration)
- On bare metal, typically runs as a systemd service

#### kube-proxy
- Network proxy running on each node
- Implements Kubernetes Service concept
- Maintains network rules (iptables, ipvs, or eBPF)
- Handles load balancing for Services
- Three modes:
  - **iptables mode** (default): Uses iptables rules for NAT
  - **ipvs mode**: Better performance, more load balancing algorithms
  - **eBPF mode** (via Cilium): Lowest latency, no iptables

#### Container Runtime
- Must implement CRI (Container Runtime Interface)
- Common runtimes:
  - **containerd**: Lightweight, most popular now
  - **CRI-O**: Kubernetes-specific, OCI-compliant
  - **Docker Engine** (via cri-dockerd shim, deprecated path)
- Responsible for pulling images, running containers, managing container lifecycle

#### Container Network Interface (CNI)
- Must implement CNI (Container Network Interface) specification
- Responsible for pod networking: IP allocation, routing, network policies
- Common CNI plugins:
  - **Calico**: BGP-based routing, supports network policies, L3/L4/L7 security
  - **Flannel**: Simple overlay network using VXLAN, easy to set up
  - **Cilium**: eBPF-based, high performance, L7 visibility and policies
  - **Weave**: Mesh networking with built-in service discovery
- CNI configuration stored in `/etc/cni/net.d/`
- CNI plugin binaries located in `/opt/cni/bin/`
- Invoked by container runtime during pod sandbox creation
- Handles: veth pair creation, IP assignment (IPAM), routing setup, policy enforcement

---

## 2. Complete Pod Lifecycle - Deployment to Traffic

### Phase 1: Deployment Creation

```bash
kubectl apply -f deployment.yaml
```

**Step 1: API Server Receives Request**
- kubectl sends POST request to kube-apiserver
- API server authenticates (client certs, bearer tokens, etc.)
- Authorization via RBAC (does user have permission?)
- Admission controllers run:
  - Mutating admission: Modifies request (inject sidecars, set defaults)
  - Validating admission: Validates request (policies, quotas)
- Deployment object written to etcd

**Step 2: Deployment Controller Detects Change**
- Deployment controller (in controller-manager) watches Deployments
- Sees new deployment via watch mechanism
- Creates a ReplicaSet based on deployment spec
- ReplicaSet written to etcd

**Step 3: ReplicaSet Controller Takes Over**
- ReplicaSet controller watches ReplicaSets
- Sees desired replica count doesn't match actual (0 pods exist)
- Creates Pod objects (doesn't create actual containers yet!)
- Pod objects written to etcd with empty `spec.nodeName`

### Phase 2: Pod Scheduling

**Step 4: Scheduler Detects Unscheduled Pod**
- Scheduler watches for pods where `spec.nodeName` is empty
- Starts scheduling algorithm:

**Filtering Phase:**
```
Available nodes: node1, node2, node3
- Check resource requests (CPU: 100m, Memory: 128Mi)
- Check taints/tolerations
- Check node selectors
- Check affinity rules
- Result: [node1, node2] pass filtering
```

**Scoring Phase:**
```
node1: score = 85 (balanced resources)
node2: score = 92 (less loaded, better spread)
Winner: node2
```

- Scheduler does NOT create the pod on the node
- Scheduler simply updates Pod object: `spec.nodeName: node2`
- Updated pod written to etcd

### Phase 3: Kubelet Takes Over

**Step 5: Kubelet Detects Pod Assignment**
- Kubelet on node2 watches API server for pods assigned to it
- Sees new pod with `spec.nodeName: node2`
- Begins pod creation process

**Step 6: Image Pulling**
```
Kubelet -> CRI (containerd) -> Image pull
```
- Kubelet calls CRI's ImageService.PullImage()
- containerd pulls image from registry
- Image layers downloaded and stored locally

**Step 7: Pod Sandbox Creation**

```
Kubelet -> CRI -> Create Pod Sandbox
```

**What's a Pod Sandbox?**
- Infrastructure container (pause container)
- Establishes Linux namespaces: network, IPC, UTS
- All containers in pod share these namespaces

**Step 8: CNI Plugin Invocation - THE CRITICAL NETWORKING PIECE**

When containerd creates the pod sandbox, it needs to set up networking:

```bash
# Kubelet calls CRI
CRI.RunPodSandbox(config)
  |
  v
# Containerd creates pause container
create_container("pause")
  |
  v
# Containerd invokes CNI plugins
execute_cni_plugin_chain()
```

**CNI Plugin Execution Details:**

```bash
# CNI config location: /etc/cni/net.d/
# Typically looks like: 10-calico.conflist or 10-flannel.conflist

# CNI plugin binary location: /opt/cni/bin/

# Containerd executes CNI plugin with:
CNI_COMMAND=ADD
CNI_CONTAINERID=abc123
CNI_NETNS=/var/run/netns/cni-xyz
CNI_IFNAME=eth0
CNI_PATH=/opt/cni/bin

# Example with Calico:
/opt/cni/bin/calico < config.json
```

**What the CNI Plugin Does (e.g., Calico):**

1. **Creates veth pair**:
   ```bash
   # In container namespace: eth0
   # In host namespace: cali1234567890
   ```

2. **Assigns IP address**:
   - Queries IPAM (IP Address Management) plugin
   - Gets IP from pod CIDR for this node (e.g., 10.244.2.15/24)
   - Configures eth0 in container with this IP

3. **Sets up routing**:
   ```bash
   # Inside pod namespace:
   ip route add default via 169.254.1.1
   
   # On host:
   ip route add 10.244.2.15/32 dev cali1234567890
   ```

4. **BGP advertisement** (Calico specific):
   - Calico node daemon advertises this route to other nodes
   - Other nodes learn: "10.244.2.15 is reachable via node2"

5. **Returns result to containerd**:
   ```json
   {
     "ips": [{"version": "4", "address": "10.244.2.15/24"}],
     "routes": [{"dst": "0.0.0.0/0", "gw": "169.254.1.1"}],
     "dns": {}
   }
   ```

**Different CNI Implementations:**

**Calico (BGP-based)**:
```
Pod IP: 10.244.2.15
Routes: Direct routing via BGP, each node knows routes to other pod CIDRs
Datapath: Native routing (no overlay), uses iptables/eBPF for policy
```

**Flannel (VXLAN overlay)**:
```
Pod IP: 10.244.2.15
Overlay: VXLAN tunnel between nodes (UDP port 8472)
Routing: Encapsulated in VXLAN, flannel.1 interface on each node
```

**Cilium (eBPF-based)**:
```
Pod IP: 10.244.2.15
Datapath: eBPF programs attached to network interfaces
Routing: Direct routing or tunneling, incredibly efficient
Policy: eBPF enforcement at L3/L4/L7
```

**Step 9: Container Creation**

After networking is configured:

```
Kubelet -> CRI -> Create Container (for each container in pod)
```

- containerd creates container from image
- Mounts volumes
- Sets environment variables
- Attaches to pod sandbox network namespace (shares IP!)
- Starts container process

**Step 10: Pod Status Update**

```
Kubelet monitors container status
  |
  v
Updates pod status in API server
  - Phase: Running
  - PodIP: 10.244.2.15
  - ContainerStatuses: Running
```

### Phase 4: Service Discovery & Load Balancing

**Step 11: Service Creation**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-app
spec:
  selector:
    app: my-app
  ports:
  - port: 80
    targetPort: 8080
```

**What happens:**

1. **Endpoints Controller** watches Services and Pods
2. Finds all pods matching selector (`app: my-app`)
3. Creates Endpoints object:
   ```yaml
   apiVersion: v1
   kind: Endpoints
   metadata:
     name: my-app
   subsets:
   - addresses:
     - ip: 10.244.2.15
       nodeName: node2
     - ip: 10.244.3.20
       nodeName: node3
     ports:
     - port: 8080
   ```

4. **kube-proxy** on EVERY node watches Services and Endpoints

**Step 12: kube-proxy Configures Load Balancing**

**iptables mode (most common):**

```bash
# On every node, kube-proxy creates iptables rules:

# KUBE-SERVICES chain (main entry point)
-A PREROUTING -j KUBE-SERVICES
-A OUTPUT -j KUBE-SERVICES

# Service ClusterIP rule
-A KUBE-SERVICES -d 10.96.0.100/32 -p tcp --dport 80 -j KUBE-SVC-XXXXX

# Load balancing across endpoints (random probability)
-A KUBE-SVC-XXXXX -m statistic --mode random --probability 0.5 -j KUBE-SEP-YYYYY
-A KUBE-SVC-XXXXX -j KUBE-SEP-ZZZZZ

# DNAT to actual pod IPs
-A KUBE-SEP-YYYYY -p tcp -j DNAT --to-destination 10.244.2.15:8080
-A KUBE-SEP-ZZZZZ -p tcp -j DNAT --to-destination 10.244.3.20:8080
```

**ipvs mode (better performance):**

```bash
# kube-proxy creates IPVS virtual server
ipvsadm -A -t 10.96.0.100:80 -s rr  # round-robin

# Adds real servers
ipvsadm -a -t 10.96.0.100:80 -r 10.244.2.15:8080 -m
ipvsadm -a -t 10.96.0.100:80 -r 10.244.3.20:8080 -m
```

### Phase 5: Ingress and External Traffic

**Step 13: Ingress Controller Deployment**

On bare metal, you need to handle Ingress differently:

**Option 1: NodePort + External LB**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: ingress-nginx
spec:
  type: NodePort
  ports:
  - port: 80
    nodePort: 30080
```

**Option 2: MetalLB (for LoadBalancer type)**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: ingress-nginx
spec:
  type: LoadBalancer
  loadBalancerIP: 192.168.1.100  # External IP
```

MetalLB runs in two modes:
- **Layer 2 mode**: Uses ARP to announce external IP from one node
- **BGP mode**: Advertises external IP via BGP to upstream routers

**Step 14: Ingress Resource**

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-app-ingress
spec:
  rules:
  - host: myapp.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: my-app
            port:
              number: 80
```

**Ingress Controller** (nginx, Traefik, etc.) watches Ingress resources and configures itself:

```nginx
# Generated nginx config
server {
  listen 80;
  server_name myapp.example.com;
  
  location / {
    proxy_pass http://my-app-backend;
  }
}

upstream my-app-backend {
  server 10.244.2.15:8080;
  server 10.244.3.20:8080;
}
```

### Phase 6: Complete Traffic Flow

**External Request Journey:**

```
Client (internet)
  |
  | DNS: myapp.example.com -> 192.168.1.100
  v
External LoadBalancer/MetalLB IP (192.168.1.100)
  |
  | ARP/BGP routing to node1
  v
Node1 Network Interface
  |
  | iptables PREROUTING -> forward to Ingress Controller pod
  v
Ingress Controller Pod (10.244.1.10:80)
  |
  | Parse Host header, match Ingress rule
  | Proxy to Service ClusterIP
  v
Service ClusterIP (10.96.0.100:80)
  |
  | iptables/ipvs DNAT transformation
  v
Pod IP (10.244.2.15:8080)
  |
  | Routed via CNI (BGP/VXLAN/etc.)
  | May cross nodes: node1 -> node2
  v
Container Process (listening on 8080)
  |
  v
Application handles request
```

**Packet transformation:**

```
# Original packet at client
SRC: 203.0.113.50:54321
DST: 192.168.1.100:80

# After MetalLB/NodePort (enters node1)
SRC: 203.0.113.50:54321
DST: 10.244.1.10:80  (Ingress controller pod)

# Ingress proxies to Service
SRC: 10.244.1.10:random
DST: 10.96.0.100:80  (Service ClusterIP)

# After kube-proxy DNAT
SRC: 10.244.1.10:random
DST: 10.244.2.15:8080  (Pod IP)

# Response path (reversed with SNAT/connection tracking)
```

---

## 3. File System Layout - Critical Paths

### Configuration Files

```bash
# Kubernetes configs
/etc/kubernetes/
├── admin.conf              # Admin kubeconfig
├── controller-manager.conf # Controller manager kubeconfig
├── kubelet.conf           # Kubelet kubeconfig
├── scheduler.conf         # Scheduler kubeconfig
├── manifests/             # Static pod manifests
│   ├── etcd.yaml
│   ├── kube-apiserver.yaml
│   ├── kube-controller-manager.yaml
│   └── kube-scheduler.yaml
└── pki/                   # ALL THE CERTIFICATES
    ├── apiserver.crt
    ├── apiserver.key
    ├── apiserver-kubelet-client.crt
    ├── apiserver-kubelet-client.key
    ├── ca.crt             # CRITICAL: Root CA
    ├── ca.key
    ├── front-proxy-ca.crt
    ├── front-proxy-ca.key
    ├── front-proxy-client.crt
    ├── front-proxy-client.key
    ├── sa.key             # Service account signing key
    ├── sa.pub
    └── etcd/
        ├── ca.crt
        ├── server.crt
        ├── server.key
        ├── peer.crt
        ├── peer.key
        ├── healthcheck-client.crt
        └── healthcheck-client.key

# CNI Configuration
/etc/cni/net.d/
├── 10-calico.conflist     # CNI config (order matters!)
└── calico-kubeconfig      # CNI plugin kubeconfig

# CNI Binaries
/opt/cni/bin/
├── calico
├── bandwidth
├── bridge
├── dhcp
├── host-local             # IPAM plugin
├── loopback
└── portmap

# Kubelet config
/var/lib/kubelet/
├── config.yaml            # Kubelet configuration
├── kubeadm-flags.env      # Kubeadm-managed flags
├── cpu_manager_state
├── memory_manager_state
├── pods/                  # Pod data
│   └── <namespace>_<pod>_<uid>/
│       ├── volumes/
│       ├── plugins/
│       └── containers/
└── pki/
    ├── kubelet-client-current.pem
    └── kubelet.crt

# Container runtime
/etc/containerd/
└── config.toml            # Containerd configuration

/var/lib/containerd/       # Containerd data

# System services
/etc/systemd/system/
├── kubelet.service
└── containerd.service

/usr/lib/systemd/system/   # Some distros use this instead
```

---

## 4. Troubleshooting by Layer

### Layer 1: Node Health & System Resources

**Check node status:**
```bash
# Quick node overview
kubectl get nodes -o wide

# Detailed node status
kubectl describe node <node-name>

# Look for:
# - Conditions: Ready, MemoryPressure, DiskPressure, PIDPressure
# - Capacity vs Allocatable
# - Allocated resources (requests/limits)
```

**Check system resources:**
```bash
# Disk pressure (common on bare metal!)
df -h
df -i  # Check inode usage too!

# Specific Kubernetes directories
du -sh /var/lib/kubelet
du -sh /var/lib/containerd
du -sh /var/log/pods

# Memory pressure
free -h
vmstat 1 5

# Check for OOM kills
dmesg | grep -i "out of memory"
dmesg | grep -i "kill"

# Check system limits
ulimit -a
cat /proc/sys/fs/file-max
cat /proc/sys/fs/file-nr

# Kubelet resource reservation
kubectl get node <node> -o jsonpath='{.status.allocatable}' | jq
```

**Cleanup commands (when disk is full):**
```bash
# Clean unused images
crictl rmi --prune

# Clean exited containers
crictl rm $(crictl ps -a -q --state=exited)

# Clean logs (careful!)
truncate -s 0 /var/log/pods/*/*/*.log

# Check what's using space
du -sh /var/lib/kubelet/pods/* | sort -h | tail -20
```

### Layer 2: Kubelet - The Node Agent

**Systemd service management:**
```bash
# Check kubelet status
systemctl status kubelet
systemctl is-active kubelet
systemctl is-enabled kubelet

# Check service file
systemctl cat kubelet

# Common kubelet service file:
cat /etc/systemd/system/kubelet.service
# Or sometimes:
cat /usr/lib/systemd/system/kubelet.service.d/10-kubeadm.conf
```

**Kubelet logs - YOUR BEST FRIEND:**
```bash
# Real-time logs
journalctl -u kubelet -f

# Last 100 lines
journalctl -u kubelet -n 100

# Since specific time
journalctl -u kubelet --since "2024-01-15 14:00:00"

# Last hour
journalctl -u kubelet --since "1 hour ago"

# With priority filtering
journalctl -u kubelet -p err  # Only errors

# Search for specific errors
journalctl -u kubelet | grep -i "error\|failed\|refused"

# Export to file for analysis
journalctl -u kubelet --since "1 hour ago" > kubelet.log
```

**Common kubelet errors and solutions:**

```bash
# Error: "Failed to get system container stats"
# Check: cgroup configuration
cat /proc/cgroups
mount | grep cgroup
# Fix: Ensure cgroup v1 or v2 is properly configured

# Error: "Unable to update cni config"
# Check CNI directory
ls -la /etc/cni/net.d/
cat /etc/cni/net.d/*.conflist
# Ensure CNI plugin binary exists
ls -la /opt/cni/bin/

# Error: "certificate has expired or is not yet valid"
# Check certificate expiration
openssl x509 -in /var/lib/kubelet/pki/kubelet-client-current.pem -noout -dates

# Error: "Failed to get node lease"
# API server connectivity issue
curl -k https://<api-server>:6443/healthz
# Check kubelet.conf
cat /etc/kubernetes/kubelet.conf
```

**Kubelet configuration:**
```bash
# Check kubelet config
cat /var/lib/kubelet/config.yaml

# Key settings to verify:
# - cgroupDriver: systemd or cgroupfs (must match containerd!)
# - clusterDNS: [10.96.0.10]  # CoreDNS service IP
# - clusterDomain: cluster.local
# - containerRuntimeEndpoint: unix:///var/run/containerd/containerd.sock

# Check runtime endpoint
crictl --runtime-endpoint unix:///var/run/containerd/containerd.sock ps
```

**Kubelet startup flags:**
```bash
# Check actual running process
ps aux | grep kubelet

# Common important flags:
# --config=/var/lib/kubelet/config.yaml
# --kubeconfig=/etc/kubernetes/kubelet.conf
# --network-plugin=cni
# --cni-bin-dir=/opt/cni/bin
# --cni-conf-dir=/etc/cni/net.d
# --pod-infra-container-image=registry.k8s.io/pause:3.9
```

### Layer 3: Container Runtime (containerd)

**Check containerd status:**
```bash
# Service status
systemctl status containerd

# Containerd logs
journalctl -u containerd -f
journalctl -u containerd --since "10 minutes ago"

# Check containerd config
cat /etc/containerd/config.toml

# Key config sections:
[plugins."io.containerd.grpc.v1.cri"]
  # systemd cgroup driver (must match kubelet!)
  [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
    SystemdCgroup = true
```

**Using crictl (Container Runtime Interface CLI):**
```bash
# List pods
crictl pods

# List containers
crictl ps -a

# Inspect container
crictl inspect <container-id>

# Container logs
crictl logs <container-id>
crictl logs --tail=100 <container-id>

# Execute in container
crictl exec -it <container-id> /bin/sh

# Check images
crictl images

# Pull image manually
crictl pull nginx:latest

# Stats
crictl stats
crictl stats <container-id>

# Pod sandbox info
crictl inspectp <pod-id>
```

**Common containerd issues:**
```bash
# Error: "failed to reserve container name"
# Stale container, cleanup:
crictl rm <container-id>

# Error: "failed to create containerd task"
# Check runtime
crictl info | jq '.config.containerd.runtimes'

# Error: "failed pulling image"
# Check image pull secrets, network, registry access
crictl pull <image> --creds username:password

# Check containerd namespaces
ctr -n k8s.io containers ls
ctr -n k8s.io images ls
```

### Layer 4: Networking & CNI

**CNI configuration troubleshooting:**

```bash
# Check CNI config files
ls -la /etc/cni/net.d/
cat /etc/cni/net.d/10-calico.conflist

# CNI config should look like:
{
  "name": "k8s-pod-network",
  "cniVersion": "0.3.1",
  "plugins": [
    {
      "type": "calico",
      "log_level": "info",
      "datastore_type": "kubernetes",
      "nodename": "node1",
      "ipam": {
        "type": "calico-ipam"
      },
      "policy": {
        "type": "k8s"
      },
      "kubernetes": {
        "kubeconfig": "/etc/cni/net.d/calico-kubeconfig"
      }
    }
  ]
}

# Verify CNI binaries
ls -la /opt/cni/bin/
# Ensure they're executable
chmod +x /opt/cni/bin/*

# Check CNI plugin logs (depends on CNI)
# Calico:
journalctl -u calico-node -f
kubectl logs -n kube-system -l k8s-app=calico-node --tail=100

# Cilium:
kubectl logs -n kube-system -l k8s-app=cilium --tail=100

# Flannel:
kubectl logs -n kube-system -l app=flannel --tail=100
```

**Network debugging:**

```bash
# Check pod networking
kubectl get pods -o wide
kubectl describe pod <pod-name>

# Check if pod got an IP
kubectl get pod <pod-name> -o jsonpath='{.status.podIP}'

# If no IP assigned:
# 1. Check CNI daemon pods
kubectl get pods -n kube-system | grep -E "calico|flannel|cilium|weave"

# 2. Check CNI daemon logs
kubectl logs -n kube-system <cni-pod-name>

# 3. Check kubelet logs for CNI errors
journalctl -u kubelet | grep -i "cni\|network"

# Common errors:
# "failed to find plugin bridge in path"
# "failed to set up sandbox"
# "error getting ClusterInformation"
```

**Testing pod network connectivity:**

```bash
# Create debug pod
kubectl run debug --image=nicolaka/netshoot -it --rm -- /bin/bash

# Inside pod:
# Test DNS
nslookup kubernetes.default.svc.cluster.local

# Test service connectivity
curl http://kubernetes.default.svc.cluster.local

# Test pod-to-pod (different nodes)
ping 10.244.2.15

# Check routes
ip route
ip addr

# Check if traffic is flowing
tcpdump -i eth0 -n

# Exit pod (it will be deleted)
exit
```

**Node network debugging:**

```bash
# Check node interfaces
ip addr show

# Look for CNI interfaces (examples):
# calixxx interfaces (Calico)
# flannel.1 interface (Flannel)
# cilium_host (Cilium)

# Check routes
ip route show

# Check iptables (if using iptables mode)
# This outputs A LOT - filter it
iptables -t nat -L KUBE-SERVICES -n | head -50
iptables -t nat -L KUBE-SERVICES -n | grep <service-name>

# Check IPVS (if using ipvs mode)
ipvsadm -Ln

# Check for connection tracking issues
conntrack -L | wc -l
cat /proc/sys/net/netfilter/nf_conntrack_max

# Common issue: conntrack table full
# Increase limit:
sysctl -w net.netfilter.nf_conntrack_max=1000000
```

**Service troubleshooting:**

```bash
# Check service
kubectl get svc <service-name> -o yaml

# Check endpoints
kubectl get endpoints <service-name>

# If endpoints are empty:
# 1. Check pod selector matches
kubectl get svc <service-name> -o jsonpath='{.spec.selector}'
kubectl get pods --show-labels | grep <label>

# 2. Check pod readiness
kubectl get pods
kubectl describe pod <pod-name>

# 3. Check readiness probe
kubectl get pod <pod-name> -o yaml | grep -A 10 readinessProbe

# Check kube-proxy
kubectl get pods -n kube-system | grep kube-proxy
kubectl logs -n kube-system kube-proxy-xxxxx

# Verify kube-proxy mode
kubectl logs -n kube-system kube-proxy-xxxxx | grep "proxy mode"

# Check if kube-proxy is syncing rules
kubectl logs -n kube-system kube-proxy-xxxxx | grep -i "sync"
```

### Layer 5: Control Plane Components

**API Server troubleshooting:**

```bash
# Check API server pod (if static pod)
kubectl get pod -n kube-system kube-apiserver-<node>
kubectl logs -n kube-system kube-apiserver-<node> --tail=100

# If kubectl doesn't work, check container directly
crictl ps | grep apiserver
crictl logs <container-id> --tail=100

# Check API server manifest
cat /etc/kubernetes/manifests/kube-apiserver.yaml

# Test API server health
curl -k https://localhost:6443/healthz
curl -k https://localhost:6443/livez
curl -k https://localhost:6443/readyz

# Check API server is listening
netstat -tlnp | grep 6443
ss -tlnp | grep 6443

# Common API server errors:
# "etcd cluster is unavailable"
# Check etcd health (see below)

# "unable to authenticate the request"
# Certificate issue (see certificate section)

# Check API server flags
kubectl get pod -n kube-system kube-apiserver-<node> -o yaml | grep command -A 50
```

**etcd troubleshooting - CRITICAL:**

```bash
# Check etcd pod
kubectl get pod -n kube-system etcd-<node>
kubectl logs -n kube-system etcd-<node> --tail=100

# If kubectl doesn't work:
crictl ps | grep etcd
crictl logs <etcd-container-id>

# Check etcd health
# Set environment variables first
export ETCDCTL_API=3
export ETCDCTL_CACERT=/etc/kubernetes/pki/etcd/ca.crt
export ETCDCTL_CERT=/etc/kubernetes/pki/etcd/server.crt
export ETCDCTL_KEY=/etc/kubernetes/pki/etcd/server.key

# Check member list
etcdctl --endpoints=https://127.0.0.1:2379 member list -w table

# Check cluster health
etcdctl --endpoints=https://127.0.0.1:2379 endpoint health -w table

# Check endpoint status (shows leader)
etcdctl --endpoints=https://127.0.0.1:2379 endpoint status -w table

# Alarm check (important!)
etcdctl --endpoints=https://127.0.0.1:2379 alarm list

# If you see "NOSPACE" alarm:
# Check etcd data size
etcdctl --endpoints=https://127.0.0.1:2379 endpoint status -w table

# Compact and defragment
etcdctl --endpoints=https://127.0.0.1:2379 compact <revision>
etcdctl --endpoints=https://127.0.0.1:2379 defrag

# Disarm alarm
etcdctl --endpoints=https://127.0.0.1:2379 alarm disarm

# Check etcd metrics
curl -k --cert /etc/kubernetes/pki/etcd/server.crt \
     --key /etc/kubernetes/pki/etcd/server.key \
     --cacert /etc/kubernetes/pki/etcd/ca.crt \
     https://127.0.0.1:2379/metrics | grep etcd_server
```

**Scheduler troubleshooting:**

```bash
# Check scheduler pod
kubectl get pod -n kube-system kube-scheduler-<node>
kubectl logs -n kube-system kube-scheduler-<node> --tail=100

# Check for pods stuck in Pending
kubectl get pods --all-namespaces | grep Pending

# Describe pending pod to see scheduler events
kubectl describe pod <pending-pod>

# Look for:
# - "0/3 nodes available: insufficient memory"
# - "0/3 nodes available: node(s) had taint"
# - "0/3 nodes available: pod has unbound PersistentVolumeClaims"

# Check scheduler logs for specific pod
kubectl logs -n kube-system kube-scheduler-<node> | grep <pod-name>
```

**Controller Manager troubleshooting:**

```bash
# Check controller manager
kubectl get pod -n kube-system kube-controller-manager-<node>
kubectl logs -n kube-system kube-controller-manager-<node> --tail=100

# Look for controller errors
kubectl logs -n kube-system kube-controller-manager-<node> | grep -i "error\|failed"

# Common issues:
# - Node controller: "failed to get node"
# - Service account controller: "failed to create token"
# - Endpoints controller: "failed to sync endpoints"

# Check leader election
kubectl logs -n kube-system kube-controller-manager-<node> | grep "leader"
```

### Layer 6: Certificates

**Certificate locations and expiration check:**

```bash
# Check ALL certificate expiration dates
kubeadm certs check-expiration

# Output shows:
# CERTIFICATE                           EXPIRES                  RESIDUAL TIME
# admin.conf                           Jan 15, 2025 12:00 UTC   364d
# apiserver                            Jan 15, 2025 12:00 UTC   364d
# apiserver-etcd-client               Jan 15, 2025 12:00 UTC   364d
# ...

# Manual certificate checking
cd /etc/kubernetes/pki

# Check each certificate
for cert in *.crt; do
    echo "=== $cert ==="
    openssl x509 -in $cert -noout -subject -issuer -dates
    echo
done

# Check specific certificate
openssl x509 -in /etc/kubernetes/pki/apiserver.crt -noout -text

# Key things to check:
# - Subject: CN (Common Name)
# - Issuer: Who signed it
# - Validity: Not Before / Not After
# - Subject Alternative Name (SAN): DNS names and IPs

# Check if certificate matches private key
openssl x509 -noout -modulus -in apiserver.crt | openssl md5
openssl rsa -noout -modulus -in apiserver.key | openssl md5
# These should match!
```

**Certificate renewal:**

```bash
# Renew all certificates
kubeadm certs renew all

# Renew specific certificate
kubeadm certs renew apiserver
kubeadm certs renew apiserver-kubelet-client

# After renewal, restart control plane pods
# They'll pick up new certificates

# For static pods (they'll restart automatically):
# Just wait or force restart:
crictl stopp $(crictl pods --name kube-apiserver -q)

# Verify new certificates
kubeadm certs check-expiration
```

**Common certificate errors:**

```bash
# Error: "x509: certificate has expired"
# Solution: Renew certificates
kubeadm certs renew all

# Error: "x509: certificate signed by unknown authority"
# Check: CA certificate
openssl x509 -in /etc/kubernetes/pki/ca.crt -noout -text

# Error: "x509: certificate is valid for X, not Y"
# SAN mismatch - check SANs:
openssl x509 -in /etc/kubernetes/pki/apiserver.crt -noout -text | grep -A1 "Subject Alternative Name"

# If API server IP or hostname changed, regenerate:
rm /etc/kubernetes/pki/apiserver.crt /etc/kubernetes/pki/apiserver.key
kubeadm init phase certs apiserver --config kubeadm-config.yaml
```

**Kubelet certificate rotation:**

```bash
# Kubelet certificates auto-rotate (if enabled)
# Check kubelet config
cat /var/lib/kubelet/config.yaml | grep -A5 rotateCertificates

# Should see:
# rotateCertificates: true
# serverTLSBootstrap: true

# Check current kubelet certificate
openssl x509 -in /var/lib/kubelet/pki/kubelet-client-current.pem -noout -dates

# Check pending CSRs (Certificate Signing Requests)
kubectl get csr

# Approve CSR if needed
kubectl certificate approve <csr-name>

# Auto-approve (in production, use proper approval mechanism)
kubectl get csr -o name | xargs kubectl certificate approve
```

**Verifying certificate trust chain:**

```bash
# Verify certificate against CA
openssl verify -CAfile /etc/kubernetes/pki/ca.crt /etc/kubernetes/pki/apiserver.crt

# Should output:
# apiserver.crt: OK

# Check etcd certificates
openssl verify -CAfile /etc/kubernetes/pki/etcd/ca.crt /etc/kubernetes/pki/etcd/server.crt
```

### Layer 7: DNS & CoreDNS

**CoreDNS troubleshooting:**

```bash
# Check CoreDNS pods
kubectl get pods -n kube-system -l k8s-app=kube-dns

# Check CoreDNS logs
kubectl logs -n kube-system -l k8s-app=kube-dns --tail=100

# Common errors:
# "plugin/loop: Loop detected"
# "no such host"
# "i/o timeout"

# Check CoreDNS configuration
kubectl get configmap -n kube-system coredns -o yaml

# Test DNS from pod
kubectl run -it --rm debug --image=busybox --restart=Never -- nslookup kubernetes.default

# Check DNS service
kubectl get svc -n kube-system kube-dns

# Verify it has endpoints
kubectl get endpoints -n kube-system kube-dns

# Check if kubelet is using correct DNS
cat /var/lib/kubelet/config.yaml | grep -A2 clusterDNS
# Should match CoreDNS service ClusterIP
```

**DNS resolution issues:**

```bash
# Test from node (bypassing CoreDNS)
nslookup kubernetes.default.svc.cluster.local <coredns-pod-ip>

# If this works but pod DNS doesn't:
# Check CNI network policies
# Check if pod can reach CoreDNS service IP

# Inside pod, check /etc/resolv.conf
kubectl exec <pod> -- cat /etc/resolv.conf

# Should look like:
# nameserver 10.96.0.10
# search default.svc.cluster.local svc.cluster.local cluster.local
# options ndots:5

# Check CoreDNS ready
kubectl get pods -n kube-system -l k8s-app=kube-dns -o wide
```

---

## 5. Advanced Debugging Scenarios

### Scenario 1: Pods stuck in ContainerCreating

```bash
# Check pod events
kubectl describe pod <pod-name>

# Common causes and checks:

# 1. CNI failure
journalctl -u kubelet | grep -i "failed to create pod sandbox"
kubectl logs -n kube-system -l k8s-app=calico-node

# 2. Image pull failure
crictl images | grep <image-name>
crictl pull <image-name>

# 3. Volume mount issues
kubectl describe pod <pod-name> | grep -A10 "Events:"
# Look for: "Unable to mount volumes"

# Check PV/PVC
kubectl get pv,pvc

# 4. Secret/ConfigMap missing
kubectl get secret <secret-name>
kubectl get configmap <configmap-name>
```

### Scenario 2: Node NotReady

```bash
# Check node status
kubectl describe node <node-name>

# Look at conditions:
kubectl get node <node-name> -o jsonpath='{.status.conditions}' | jq

# Common reasons:

# 1. Kubelet not running
systemctl status kubelet
systemctl restart kubelet

# 2. Network not ready
kubectl get pods -n kube-system | grep -E "calico|flannel|cilium"
journalctl -u kubelet | grep "network plugin"

# 3. Disk pressure
df -h
du -sh /var/lib/kubelet

# 4. Certificate issues
openssl x509 -in /var/lib/kubelet/pki/kubelet-client-current.pem -noout -dates

# 5. API server unreachable
curl -k https://<api-server>:6443/healthz
```

### Scenario 3: High API latency

```bash
# Check API server metrics
kubectl get --raw /metrics | grep apiserver_request_duration

# Check etcd latency
etcdctl --endpoints=https://127.0.0.1:2379 \
        --cacert=/etc/kubernetes/pki/etcd/ca.crt \
        --cert=/etc/kubernetes/pki/etcd/server.crt \
        --key=/etc/kubernetes/pki/etcd/server.key \
        check perf

# Check etcd database size
etcdctl --endpoints=https://127.0.0.1:2379 \
        --cacert=/etc/kubernetes/pki/etcd/ca.crt \
        --cert=/etc/kubernetes/pki/etcd/server.crt \
        --key=/etc/kubernetes/pki/etcd/server.key \
        endpoint status -w table

# If etcd DB is large (>8GB), compact:
rev=$(etcdctl --endpoints=https://127.0.0.1:2379 \
        --cacert=/etc/kubernetes/pki/etcd/ca.crt \
        --cert=/etc/kubernetes/pki/etcd/server.crt \
        --key=/etc/kubernetes/pki/etcd/server.key \
        endpoint status --write-out="json" | jq '.[0].Status.header.revision')

etcdctl --endpoints=https://127.0.0.1:2379 \
        --cacert=/etc/kubernetes/pki/etcd/ca.crt \
        --cert=/etc/kubernetes/pki/etcd/server.crt \
        --key=/etc/kubernetes/pki/etcd/server.key \
        compact $rev

etcdctl --endpoints=https://127.0.0.1:2379 \
        --cacert=/etc/kubernetes/pki/etcd/ca.crt \
        --cert=/etc/kubernetes/pki/etcd/server.crt \
        --key=/etc/kubernetes/pki/etcd/server.key \
        defrag

# Check API server load
top  # Look for kube-apiserver CPU usage
kubectl top nodes  # Requires metrics-server
```

### Scenario 4: Persistent Volume issues (Bare metal specific)

```bash
# Check PV status
kubectl get pv
kubectl describe pv <pv-name>

# Local volume issues:
# 1. Check mount point on node
ssh <node>
ls -la /mnt/disks/vol1  # Or whatever path

# 2. Check file permissions
ls -la /mnt/disks/vol1

# 3. Check if disk is mounted
mount | grep /mnt/disks/vol1
df -h | grep /mnt/disks/vol1

# 4. Check kubelet logs for mount errors
journalctl -u kubelet | grep -i "mount\|volume"

# NFS issues (common on bare metal):
# 1. Check NFS server is reachable
showmount -e <nfs-server-ip>

# 2. Test mount manually
mount -t nfs <nfs-server>:/export/path /mnt/test

# 3. Check for stale mounts
mount | grep nfs
# Unmount stale:
umount -f /mnt/stale
```

---

## 6. Monitoring Commands - Quick Reference

```bash
# Node resource usage
kubectl top nodes

# Pod resource usage
kubectl top pods --all-namespaces

# Get events (sorted by time)
kubectl get events --all-namespaces --sort-by='.lastTimestamp'

# Get events for specific resource
kubectl get events --field-selector involvedObject.name=<pod-name>

# Watch resources
kubectl get pods -w
kubectl get nodes -w

# Resource requests/limits across cluster
kubectl describe nodes | grep -A5 "Allocated resources"

# Check which pods are running on which nodes
kubectl get pods -o wide --all-namespaces

# Get all resources in namespace
kubectl get all -n <namespace>

# Check for failed pods
kubectl get pods --all-namespaces --field-selector=status.phase!=Running,status.phase!=Succeeded

# API server audit logs (if enabled)
ls -la /var/log/kubernetes/audit/
tail -f /var/log/kubernetes/audit/audit.log | jq
```

---

## 7. Emergency Procedures

### Complete node recovery

```bash
# 1. Drain node (evacuate pods)
kubectl drain <node-name> --ignore-daemonsets --delete-emptydir-data

# 2. Perform maintenance
systemctl restart kubelet
systemctl restart containerd

# 3. Uncordon node
kubectl uncordon <node-name>

# 4. Verify
kubectl get nodes
```

### Control plane recovery

```bash
# If API server is completely down:

# 1. Check static pod manifests
ls -la /etc/kubernetes/manifests/

# 2. Check kubelet is running
systemctl status kubelet
journalctl -u kubelet -f

# 3. Check containers
crictl ps | grep kube

# 4. If needed, restore from backup
# Restore /etc/kubernetes directory
# Restore etcd snapshot (requires separate procedure)

# 5. Force recreate static pods
mv /etc/kubernetes/manifests/kube-apiserver.yaml /tmp/
sleep 30
mv /tmp/kube-apiserver.yaml /etc/kubernetes/manifests/
```

### etcd Backup and Restore

```bash
# Backup etcd
ETCDCTL_API=3 etcdctl snapshot save /backup/etcd-snapshot.db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

# Verify snapshot
ETCDCTL_API=3 etcdctl snapshot status /backup/etcd-snapshot.db -w table

# Restore etcd (CAUTION: This will overwrite current data)
# 1. Stop API server
mv /etc/kubernetes/manifests/kube-apiserver.yaml /tmp/

# 2. Restore snapshot
ETCDCTL_API=3 etcdctl snapshot restore /backup/etcd-snapshot.db \
  --data-dir=/var/lib/etcd-restore

# 3. Update etcd manifest to use new data dir
# Edit /etc/kubernetes/manifests/etcd.yaml
# Change: --data-dir=/var/lib/etcd-restore

# 4. Start API server
mv /tmp/kube-apiserver.yaml /etc/kubernetes/manifests/

# 5. Verify cluster
kubectl get nodes
kubectl get pods --all-namespaces
```

## Key Takeaways for SRE

### Critical paths to memorize:
1. `/etc/kubernetes/pki/` - All certificates
2. `/etc/cni/net.d/` - CNI configuration
3. `/opt/cni/bin/` - CNI binaries
4. `/var/lib/kubelet/` - Kubelet state and config
5. `/etc/kubernetes/manifests/` - Static pod manifests

### Essential commands:
1. `journalctl -u kubelet -f` - First place to look for node issues
2. `kubectl describe node/pod` - Shows events and conditions
3. `crictl ps/logs` - When kubectl doesn't work
4. `etcdctl endpoint health` - Check etcd cluster
5. `kubeadm certs check-expiration` - Certificate management

### Common failure patterns:
1. **Pods stuck in ContainerCreating**: CNI or image pull issues
2. **Node NotReady**: Kubelet, network, or certificate problems
3. **Services not working**: Check endpoints, kube-proxy, and iptables/ipvs
4. **High latency**: Usually etcd database size or disk I/O
5. **Certificate expiration**: Use `kubeadm certs renew all`

### Bare metal specific challenges:
1. No cloud load balancers - use MetalLB or NodePort
2. No dynamic storage - use local volumes, NFS, or Ceph
3. Manual node management - no autoscaling
4. Physical network constraints - BGP, VLANs, etc.
5. Hardware failures - need proper monitoring and alerting

---

## Additional Resources

- Kubernetes Official Docs: https://kubernetes.io/docs/
- Kubernetes The Hard Way: https://github.com/kelseyhightower/kubernetes-the-hard-way
- CNI Specification: https://github.com/containernetworking/cni
- etcd Documentation: https://etcd.io/docs/
- crictl Documentation: https://github.com/kubernetes-sigs/cri-tools

---

**Document Version**: 1.0  
**Last Updated**: 2025  
**Author**: SRE Knowledge Base