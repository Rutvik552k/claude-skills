# Container and Kubernetes Networking Deep Dive

## Container Networking Fundamentals

### Linux Network Namespaces

Every container runs in its own network namespace — an isolated network stack with its own
interfaces, routing tables, iptables rules, and port space. The container runtime connects
these namespaces to the host or to each other using virtual ethernet pairs (veth pairs),
bridges, and routing rules.

```
┌─────────────────────────────────────────────────┐
│  Host Network Namespace                         │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐     │
│  │ eth0     │  │ podman0  │  │ cni0     │     │
│  │ (host)   │  │ (bridge) │  │ (bridge) │     │
│  └──────────┘  └─────┬────┘  └─────┬────┘     │
│                      │             │            │
│              ┌───────┴─────────────┴──────┐    │
│              │    veth pairs              │    │
│              └───────┬─────────────┬──────┘    │
│                      │             │            │
│  ┌───────────────────┴──┐  ┌──────┴──────────┐ │
│  │ Container NS 1       │  │ Container NS 2  │ │
│  │ eth0: 10.88.0.2      │  │ eth0: 10.88.0.3 │ │
│  └──────────────────────┘  └─────────────────┘ │
└─────────────────────────────────────────────────┘
```

### CNI (Container Network Interface)

CNI is a specification and plugin framework for configuring container network interfaces.

- **How it works**: The container runtime calls a CNI plugin binary with JSON configuration. The plugin sets up networking for the container (creates veth pair, assigns IP, sets routes).
- **Plugin types**: Bridge, macvlan, ipvlan, host-local IPAM, DHCP IPAM, flannel, calico, cilium.
- **Configuration location**: `/etc/cni/net.d/` for rootful, `~/.config/cni/net.d/` for rootless.

```json
{
  "cniVersion": "1.0.0",
  "name": "podman-bridge",
  "type": "bridge",
  "bridge": "cni-podman0",
  "isGateway": true,
  "ipMasq": true,
  "hairpinMode": true,
  "ipam": {
    "type": "host-local",
    "routes": [{ "dst": "0.0.0.0/0" }],
    "ranges": [
      [{ "subnet": "10.88.0.0/16", "gateway": "10.88.0.1" }]
    ]
  }
}
```

### Netavark (Podman 4.0+ Default)

Netavark is Podman's Rust-based network stack that replaces CNI. It provides:

- **Better performance**: Single binary instead of chaining multiple CNI plugins.
- **Built-in DNS**: Aardvark-dns provides container name resolution without external plugins.
- **Dual-stack**: Native IPv4 and IPv6 support.
- **Simplified configuration**: Network configs stored in `$HOME/.local/share/containers/storage/networks/` (rootless).

```bash
# Create a network with Netavark
podman network create --driver bridge --subnet 10.89.0.0/24 --gateway 10.89.0.1 mynet

# Inspect network
podman network inspect mynet

# Connect container to network
podman run -d --name web --network mynet nginx

# Container DNS resolution (Aardvark)
podman run --rm --network mynet busybox nslookup web
# Returns: 10.89.0.2
```

Network configuration file (`~/.local/share/containers/storage/networks/mynet.json`):

```json
{
  "name": "mynet",
  "id": "a1b2c3d4...",
  "driver": "bridge",
  "network_interface": "podman1",
  "subnets": [
    {
      "subnet": "10.89.0.0/24",
      "gateway": "10.89.0.1"
    }
  ],
  "ipv6_enabled": false,
  "internal": false,
  "dns_enabled": true,
  "ipam_options": {
    "driver": "host-local"
  }
}
```

### Rootless Container Networking

Rootless containers cannot create real bridge interfaces or modify iptables rules (these
require CAP_NET_ADMIN). Instead, they use userspace network stacks.

#### slirp4netns

The original rootless networking solution. Creates a TAP device in the container's network
namespace and routes traffic through a user-space TCP/IP stack (derived from QEMU's slirp).

```bash
# Default rootless networking (slirp4netns)
podman run -d -p 8080:80 --name web nginx
# Container gets 10.0.2.100, traffic NATed through slirp4netns

# Port forwarding works but adds overhead
# No container-to-container DNS by default
```

**Characteristics:**
- ~10-15% network throughput overhead versus rootful networking
- Each container gets its own slirp4netns process
- Default container IP: 10.0.2.100
- No direct container-to-container communication (use pods or networks)

#### pasta (Podman 5.0+ recommended)

A newer rootless networking backend that uses Linux network namespaces more efficiently.

```bash
# Use pasta explicitly
podman run -d --network pasta -p 8080:80 --name web nginx

# Or set as default in containers.conf
# [network]
# default_rootless_network_cmd = "pasta"
```

**Advantages over slirp4netns:**
- Lower overhead (near-native performance for many workloads)
- Uses the host's network stack more directly
- Better port-forwarding performance
- Supports binding to specific addresses

### Pod Networking in Podman

Containers in a Podman pod share a network namespace — exactly like Kubernetes pods.

```bash
# Create a pod with port mappings
podman pod create --name webapp -p 8080:80 -p 5432:5432

# Add containers — they share the pod's network namespace
podman run -d --pod webapp --name frontend nginx
podman run -d --pod webapp --name backend myapp:latest
podman run -d --pod webapp --name db postgres:16

# frontend, backend, and db all share the same IP
# They communicate via localhost:
# backend connects to localhost:5432 for database
# frontend proxies to localhost:8080 for backend
```

```
┌─ Pod: webapp ──────────────────────────────┐
│  Shared Network Namespace (10.89.0.5)      │
│  ┌──────────┐ ┌─────────┐ ┌────────────┐ │
│  │ frontend │ │ backend │ │ db         │ │
│  │ :80      │ │ :3000   │ │ :5432      │ │
│  └──────────┘ └─────────┘ └────────────┘ │
│  All containers reach each other via      │
│  localhost on their respective ports      │
└────────────────────────────────────────────┘
```

---

## Kubernetes Networking

### The Kubernetes Networking Model

Kubernetes imposes three fundamental networking rules:

1. **Every Pod gets its own IP address** — no NAT between pods.
2. **All Pods can communicate with all other Pods** without NAT (unless restricted by NetworkPolicy).
3. **Agents on a node can communicate with all Pods on that node**.

These rules are implemented by the CNI plugin installed in the cluster (Calico, Cilium, Flannel, Weave Net, etc.).

### kube-proxy

kube-proxy runs on every node and implements the Service abstraction by programming
network rules that redirect Service ClusterIP traffic to backend Pod IPs.

#### iptables Mode (Default)

```
Client Pod → ClusterIP (10.96.0.10:80) → iptables DNAT → Backend Pod (10.244.1.5:8080)

iptables chains:
KUBE-SERVICES → KUBE-SVC-XXXX (per Service) → KUBE-SEP-YYYY (per endpoint)

# Random load balancing via iptables probability rules:
-A KUBE-SVC-XXXX -m statistic --mode random --probability 0.333 -j KUBE-SEP-001
-A KUBE-SVC-XXXX -m statistic --mode random --probability 0.500 -j KUBE-SEP-002
-A KUBE-SVC-XXXX -j KUBE-SEP-003
```

**Limitations:**
- O(n) rule evaluation with large numbers of Services
- No connection draining during endpoint changes
- Random distribution, not true round-robin

#### IPVS Mode

```bash
# Enable IPVS mode in kube-proxy config
apiVersion: kubeproxy.config.k8s.io/v1alpha1
kind: KubeProxyConfiguration
mode: "ipvs"
ipvs:
  scheduler: "rr"  # round-robin, lc (least-conn), sh (source-hash)
```

**Advantages:**
- O(1) connection routing via hash table
- Multiple load-balancing algorithms (round-robin, least connections, source hashing)
- Better performance at scale (1000+ Services)
- Connection draining support

### Service Types In Depth

#### ClusterIP Internals

```
┌─ Cluster ──────────────────────────────────────────┐
│                                                     │
│  Client Pod ──→ 10.96.0.10:80 (ClusterIP)         │
│                      │                              │
│               kube-proxy rules                      │
│                      │                              │
│           ┌──────────┼──────────┐                  │
│           ▼          ▼          ▼                   │
│     Pod 10.244.1.5  Pod 10.244.2.3  Pod 10.244.3.7│
│     :8080           :8080           :8080          │
│     (Node 1)        (Node 2)        (Node 3)      │
└─────────────────────────────────────────────────────┘
```

#### NodePort Internals

```
External Client ──→ NodeIP:30080
                        │
                   kube-proxy
                        │
                   ClusterIP:80
                        │
                 ┌──────┼──────┐
                 ▼      ▼      ▼
              Pod 1   Pod 2   Pod 3
```

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-nodeport
spec:
  type: NodePort
  selector:
    app: web
  ports:
    - port: 80
      targetPort: 8080
      nodePort: 30080
  # Preserve client source IP (only route to local pods)
  externalTrafficPolicy: Local
```

`externalTrafficPolicy: Local` avoids an extra hop but may cause uneven distribution
if pods are not evenly spread across nodes.

#### LoadBalancer Internals

```
Internet ──→ Cloud LB (external IP)
                  │
             ┌────┼────┐
             ▼    ▼    ▼
          Node1 Node2 Node3 (NodePort)
             │    │    │
          kube-proxy rules
             │    │    │
          Pod endpoints
```

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-lb
  annotations:
    # AWS NLB with TLS termination
    service.beta.kubernetes.io/aws-load-balancer-type: "nlb"
    service.beta.kubernetes.io/aws-load-balancer-ssl-cert: "arn:aws:acm:us-east-1:123456789:certificate/abc-123"
    service.beta.kubernetes.io/aws-load-balancer-ssl-ports: "443"
    service.beta.kubernetes.io/aws-load-balancer-backend-protocol: "http"
spec:
  type: LoadBalancer
  externalTrafficPolicy: Local
  selector:
    app: web
  ports:
    - name: https
      port: 443
      targetPort: 8080
```

### Ingress Deep Dive

Ingress provides HTTP/HTTPS routing at Layer 7, consolidating multiple Services behind
a single external endpoint.

```
Internet
    │
    ▼
┌─────────────────────────────────┐
│  Ingress Controller (NGINX)     │
│  (Deployment + LoadBalancer)    │
│                                 │
│  Rules:                         │
│  app.example.com/* → web-svc    │
│  api.example.com/v1/* → api-svc │
│  *.example.com/docs/* → docs-svc│
└────────┬───────────┬────────────┘
         │           │
    ┌────▼───┐ ┌────▼────┐
    │web-svc │ │ api-svc │
    │(3 pods)│ │(5 pods) │
    └────────┘ └─────────┘
```

#### NGINX Ingress Controller Configuration

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: main-ingress
  annotations:
    # TLS
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/force-ssl-redirect: "true"

    # Rate limiting
    nginx.ingress.kubernetes.io/limit-rps: "50"
    nginx.ingress.kubernetes.io/limit-burst-multiplier: "5"

    # Timeouts
    nginx.ingress.kubernetes.io/proxy-connect-timeout: "5"
    nginx.ingress.kubernetes.io/proxy-read-timeout: "60"
    nginx.ingress.kubernetes.io/proxy-send-timeout: "60"

    # CORS
    nginx.ingress.kubernetes.io/enable-cors: "true"
    nginx.ingress.kubernetes.io/cors-allow-origin: "https://app.example.com"
    nginx.ingress.kubernetes.io/cors-allow-methods: "GET, POST, PUT, DELETE, OPTIONS"

    # Request/response transformation
    nginx.ingress.kubernetes.io/configuration-snippet: |
      more_set_headers "X-Frame-Options: DENY";
      more_set_headers "X-Content-Type-Options: nosniff";
      more_set_headers "Strict-Transport-Security: max-age=31536000";

    # cert-manager integration
    cert-manager.io/cluster-issuer: "letsencrypt-prod"

spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - app.example.com
        - api.example.com
      secretName: example-tls

  rules:
    - host: app.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: frontend-svc
                port:
                  number: 80

    - host: api.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: api-svc
                port:
                  number: 80
```

### Network Policies Deep Dive

Network Policies are Kubernetes-native firewall rules. They require a CNI plugin that
supports them (Calico, Cilium, Weave Net — NOT Flannel).

#### Default Deny All

The first step in any namespace security setup:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: production
spec:
  podSelector: {}       # Empty selector = all pods
  policyTypes:
    - Ingress
    - Egress
# After applying: all pods in 'production' namespace cannot
# send or receive any traffic until explicitly allowed
```

#### Allow DNS (Required After Default Deny)

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-dns
  namespace: production
spec:
  podSelector: {}
  policyTypes:
    - Egress
  egress:
    - to:
        - namespaceSelector: {}
          podSelector:
            matchLabels:
              k8s-app: kube-dns
      ports:
        - protocol: UDP
          port: 53
        - protocol: TCP
          port: 53
```

#### Microservice Communication Policy

```yaml
# Allow frontend -> backend
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: backend-allow-frontend
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
    - Ingress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: frontend
      ports:
        - protocol: TCP
          port: 8080

---
# Allow backend -> database
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: database-allow-backend
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: postgres
  policyTypes:
    - Ingress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: backend
      ports:
        - protocol: TCP
          port: 5432

---
# Allow monitoring namespace to scrape all pods
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-monitoring-scrape
  namespace: production
spec:
  podSelector: {}
  policyTypes:
    - Ingress
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: monitoring
      ports:
        - protocol: TCP
          port: 9090     # Prometheus metrics port
```

#### Egress to External Services

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: backend-egress
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
    - Egress
  egress:
    # Allow DNS
    - to:
        - namespaceSelector: {}
          podSelector:
            matchLabels:
              k8s-app: kube-dns
      ports:
        - protocol: UDP
          port: 53

    # Allow database access
    - to:
        - podSelector:
            matchLabels:
              app: postgres
      ports:
        - protocol: TCP
          port: 5432

    # Allow external HTTPS (e.g., third-party APIs)
    - to:
        - ipBlock:
            cidr: 0.0.0.0/0
            except:
              - 10.0.0.0/8
              - 172.16.0.0/12
              - 192.168.0.0/16
      ports:
        - protocol: TCP
          port: 443
```

### Debugging Network Issues

```bash
# Check Service endpoints
kubectl get endpoints web-svc -n production
# If empty: selector labels don't match any pod labels

# Check if pods are ready (unready pods are removed from endpoints)
kubectl get pods -n production -l app=web -o wide

# Test DNS resolution from inside the cluster
kubectl run debug --rm -it --image=busybox:1.36 -- nslookup web-svc.production.svc.cluster.local

# Test connectivity from a debug pod
kubectl run debug --rm -it --image=nicolaka/netshoot -- bash
# Inside the pod:
curl -v http://web-svc.production.svc.cluster.local:80
tcpdump -i eth0 port 80
nmap -sT -p 80 web-svc.production.svc.cluster.local

# Check kube-proxy mode and rules
kubectl get configmap kube-proxy -n kube-system -o yaml
# iptables mode: check rules on a node
iptables-save | grep web-svc
# IPVS mode: check virtual servers
ipvsadm -Ln

# Check NetworkPolicy is enforced
kubectl get networkpolicy -n production
kubectl describe networkpolicy web-app-policy -n production

# Verify CNI plugin supports NetworkPolicy
kubectl get pods -n kube-system | grep -E 'calico|cilium|weave'
```

### Service Mesh Comparison (Networking Layer)

| Feature | No Mesh | Istio | Linkerd | Cilium |
|---|---|---|---|---|
| mTLS | Manual | Automatic | Automatic | Automatic |
| Traffic splitting | Not native | VirtualService | TrafficSplit | CiliumEnvoyConfig |
| Observability | Basic metrics | Full L7 metrics | L7 metrics (lightweight) | L3/L4/L7 via eBPF |
| Sidecar | None | Envoy sidecar | linkerd2-proxy | No sidecar (eBPF) |
| Resource overhead | None | High (~128MB/sidecar) | Low (~25MB/sidecar) | Minimal (kernel) |
| NetworkPolicy | K8s native | K8s + Istio AuthorizationPolicy | K8s native | K8s + CiliumNetworkPolicy |

---

*Sources: Walsh, M. (2023). Podman in Action. Manning Publications. Poulton, N. (2021). The Kubernetes Book. Independently published.*
