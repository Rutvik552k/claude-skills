---
name: container-orchestration
description: Podman and Kubernetes container orchestration — daemonless container management, rootless containers, systemd integration, Kubernetes deployments, services, networking, and dev-to-prod workflows.
origin: ECC
---

# Container Orchestration Skill

## When to Activate

Activate this skill when the user is working on any of the following:

- Container setup and configuration (Podman, Docker, or OCI-compliant runtimes)
- Podman vs Docker comparison and migration decisions
- Kubernetes deployments, scaling, and cluster management
- Pod design patterns (sidecar, ambassador, adapter)
- Service networking (ClusterIP, NodePort, LoadBalancer, Ingress)
- Rootless containers and user namespace configuration
- systemd integration for container services (unit files, Quadlet, auto-update)
- Container image building (Containerfile, multi-stage builds, OCI layers)
- Dev-to-prod workflows bridging local Podman to Kubernetes clusters

---

## Topics Index

### 1. Container Fundamentals

#### Podman vs Docker
- **Daemonless architecture**: Podman runs containers directly via fork/exec rather than routing through a central daemon process. Each `podman run` invocation forks a child process that becomes the container runtime (conmon), eliminating the single-point-of-failure daemon model.
- **Fork/exec model**: No long-running daemon means no daemon socket to secure, no daemon crashes taking down all containers, and native integration with systemd process management.
- **CLI compatibility**: Podman implements the same CLI interface as Docker. In most cases `alias docker=podman` works. Divergences exist around Docker Compose (Podman uses `podman compose` backed by docker-compose or podman-compose), Swarm (not supported), and BuildKit-specific features.
- **Key differences**: Podman supports pods natively (shared network/PID/IPC namespaces), generates Kubernetes YAML from running containers, and supports rootless operation out of the box.

#### Rootless Containers
- **User namespaces**: Rootless containers run in a user namespace where UID 0 inside the container maps to a non-privileged UID on the host. This provides defense-in-depth — even a container escape lands in an unprivileged user context.
- **UID mapping**: Configured via `/etc/subuid` and `/etc/subgid`. Each user gets a range of subordinate UIDs (e.g., `user1:100000:65536`) that the container runtime maps into the container's UID space.
- **Limitations**: Rootless containers cannot bind to ports below 1024 (without `net.ipv4.ip_unprivileged_port_start`), have restricted network capabilities (use slirp4netns or pasta instead of CNI/Netavark bridge), and may encounter file permission issues with bind mounts.

#### OCI Images
- **Layers**: OCI images are composed of filesystem layers stacked via union mounts. Each instruction in a Containerfile creates a new layer. Layers are cached, content-addressable, and shared across images.
- **Containerfile / Dockerfile**: The build specification. Podman uses `Containerfile` by default but also reads `Dockerfile`. Syntax is identical.
- **Multi-stage builds**: Use multiple `FROM` instructions to separate build-time dependencies from runtime. Copy artifacts from builder stages to slim final images, reducing attack surface and image size.
- **Best practices**: Order instructions from least to most frequently changed (leverage cache), combine RUN commands to reduce layers, use `.containerignore`/`.dockerignore`, pin base image digests for reproducibility.

#### Container Lifecycle
- `create` -> `start` -> `running` -> `stop` -> `exited` -> `rm`
- Containers can be paused/unpaused, checkpointed (CRIU), and restored.
- Exit codes: 0 = success, 1-125 = application error, 126 = command not executable, 127 = command not found, 128+N = killed by signal N.

---

### 2. Podman Advanced

#### systemd Integration
- **Unit files**: Podman containers can be managed as systemd services. Use `podman generate systemd --new --name <container>` to create unit files (legacy approach).
- **Quadlet (Podman 4.4+)**: The modern approach — place `.container`, `.volume`, `.network`, `.kube`, or `.image` files in `~/.config/containers/systemd/` (rootless) or `/etc/containers/systemd/` (rootful). systemd's Quadlet generator converts these to transient unit files at boot.
- **Socket activation**: systemd can listen on a socket and start the container on first connection, enabling on-demand container startup.
- **Auto-update**: Tag containers with `io.containers.autoupdate=registry`, then `podman auto-update` (or a systemd timer) pulls newer images and restarts containers. Supports rollback on failure.

#### User Namespaces
- **subuid/subgid**: Each user entry in `/etc/subuid` and `/etc/subgid` defines a range of subordinate IDs. Podman reads these to set up user namespace mappings.
- **`--userns` flag**: Controls namespace behavior — `auto` (automatic mapping), `keep-id` (map host UID to same UID in container, useful for bind mounts), `host` (no user namespace).
- **`podman unshare`**: Enter the user namespace without starting a container — useful for debugging file permission issues on volumes.

#### Registries
- **registries.conf**: Located at `/etc/containers/registries.conf` or `~/.config/containers/registries.conf`. Defines search order for short names, blocked registries, and mirror configuration.
- **Mirrors**: Configure pull-through caches or alternative registry mirrors for air-gapped or high-availability environments.
- **Short-name aliasing**: Podman 4.0+ uses a short-name alias file to resolve ambiguous image names (e.g., `python` -> `docker.io/library/python`), prompting the user on first pull.

#### Podman Compose
- `podman compose` delegates to an external compose provider (docker-compose or podman-compose).
- Supports most Compose v2 features. Notable gaps: depends_on conditions, some network driver options.
- Alternative: `podman pod create` + `podman run --pod` for native pod-based multi-container setups.

#### Volume Management
- **Named volumes**: `podman volume create mydata` — stored in Podman's volume directory, managed lifecycle.
- **Bind mounts**: `-v /host/path:/container/path` — direct host filesystem access. Rootless bind mounts may need `:U` flag for UID mapping or `podman unshare chown`.
- **tmpfs**: `--tmpfs /tmp` — in-memory filesystem, useful for scratch data and secrets.
- **SELinux labels**: `:Z` (private unshared label) or `:z` (shared label) relabel bind-mount content for SELinux access. Use `:Z` for single-container access, `:z` when multiple containers share a volume.

#### Networking
- **CNI / Netavark**: Podman supports two network backends. CNI (legacy) uses plugin-based networking. Netavark (default since Podman 4.0) is a Rust-based stack with better performance, dual-stack IPv4/IPv6, and DNS via Aardvark.
- **slirp4netns**: Default rootless networking — user-mode TCP/IP stack. Higher overhead, no container-to-container communication without pods or shared networks.
- **pasta**: Newer rootless networking alternative. Uses Linux network namespaces more efficiently than slirp4netns, offering better performance and port-forwarding capabilities.
- **Rootful networking**: Uses bridge networking by default, similar to Docker. Supports macvlan, ipvlan, and other advanced drivers.

---

### 3. Kubernetes Fundamentals

#### Primer
Kubernetes is a container orchestration platform that automates deployment, scaling, and management of containerized applications. It uses a declarative model — you describe the desired state and Kubernetes reconciles the actual state to match.

#### Control Plane
- **API Server (kube-apiserver)**: The front door to Kubernetes. All communication flows through the API server — kubectl, controllers, kubelets, and other components all interact via REST calls to the API server. It handles authentication, authorization (RBAC), admission control, and persists state to etcd.
- **etcd**: Distributed key-value store holding all cluster state. The single source of truth. Runs as a 3- or 5-node cluster for HA. Back up etcd regularly — losing it means losing the cluster.
- **Scheduler (kube-scheduler)**: Watches for unscheduled Pods and assigns them to nodes based on resource requests, affinity/anti-affinity rules, taints/tolerations, and topology constraints.
- **Controller Manager (kube-controller-manager)**: Runs reconciliation loops (controllers) that watch desired state and drive actual state toward it. Includes the ReplicaSet controller, Deployment controller, Node controller, Job controller, and others.

#### Worker Nodes
- **kubelet**: The node agent. Receives PodSpecs from the API server and ensures containers are running and healthy. Reports node and pod status back to the control plane.
- **kube-proxy**: Manages network rules on nodes to implement Service abstractions. Operates in iptables, IPVS, or nftables mode. Routes traffic from Service ClusterIPs to backend Pod IPs.
- **Container runtime (containerd / CRI-O)**: Pulls images, creates and manages containers. Kubernetes uses the Container Runtime Interface (CRI) to communicate with any OCI-compliant runtime. containerd (default for most distros) and CRI-O (default for OpenShift) are the two primary options.

#### Declarative Model
- Define desired state in YAML manifests. Use `kubectl apply -f` to submit to the API server.
- The system continuously reconciles actual state toward desired state via controller loops.
- Imperative commands (`kubectl create`, `kubectl run`) exist but are discouraged for production — use declarative YAML checked into version control.

#### Kubernetes DNS (CoreDNS)
- Every Service gets a DNS entry: `<service>.<namespace>.svc.cluster.local`
- Pods can resolve Services by short name within the same namespace, or by FQDN across namespaces.
- Headless services (clusterIP: None) return Pod IPs directly, enabling direct pod-to-pod discovery.
- CoreDNS is configurable via the `coredns` ConfigMap in kube-system namespace.

---

### 4. Working with Pods

#### Pod Spec
A Pod is the smallest deployable unit in Kubernetes — one or more containers sharing network namespace (same IP, localhost communication), IPC namespace, and optionally PID namespace. Pods are ephemeral and should not be created directly in production (use Deployments).

#### Init Containers and Sidecar Containers
- **Init containers**: Run to completion before app containers start. Used for setup tasks — database migrations, config file generation, waiting for dependencies. They run sequentially; the Pod proceeds only after all init containers succeed.
- **Sidecar containers (K8s 1.28+ native)**: Long-running containers with `restartPolicy: Always` defined in the `initContainers` section. They start before and outlive the main container. Used for log shippers, service mesh proxies, and monitoring agents.

#### Resource Requests and Limits
- **Requests**: The guaranteed minimum resources a container needs. The scheduler uses requests for placement decisions.
- **Limits**: The maximum resources a container can consume. CPU limits throttle; memory limits trigger OOM kills.
- **Best practice**: Always set requests. Set memory limits equal to requests for predictable behavior. Be cautious with CPU limits (consider leaving them unset to avoid unnecessary throttling).

#### Pod Lifecycle
- **Pending**: Pod accepted but not yet scheduled or pulling images.
- **Running**: At least one container is running.
- **Succeeded**: All containers terminated successfully (exit code 0). Applies to Jobs/batch workloads.
- **Failed**: All containers terminated, at least one with non-zero exit code.
- **Unknown**: Node communication failure.

#### Multi-Container Patterns
- **Sidecar**: Augments the main container — log collection, monitoring, TLS proxy (e.g., Envoy sidecar for service mesh).
- **Ambassador**: Proxies network connections for the main container — connection pooling, protocol translation, service discovery abstraction.
- **Adapter**: Transforms the main container's output — log format normalization, metrics format conversion (e.g., Prometheus exporter adapter).

#### Health Probes
- **Liveness probe**: Detects deadlocks and stuck containers. Failure triggers a container restart. Use with caution — aggressive liveness probes cause unnecessary restarts.
- **Readiness probe**: Gates traffic to the container. Failing readiness removes the Pod from Service endpoints. Use for startup dependencies and graceful shutdown.
- **Startup probe**: Gives slow-starting containers time to initialize before liveness kicks in. Disables liveness/readiness probes until it succeeds.
- **Probe types**: httpGet, tcpSocket, exec, grpc.

#### ConfigMaps and Secrets
- **ConfigMap**: Store non-sensitive configuration data. Inject as environment variables or mount as files. Updates to mounted ConfigMaps propagate to pods (with delay).
- **Secret**: Store sensitive data (base64-encoded, not encrypted by default). Enable encryption at rest via EncryptionConfiguration. Types: Opaque, kubernetes.io/tls, kubernetes.io/dockerconfigjson, etc.
- **Best practice**: Use external secret managers (Vault, AWS Secrets Manager) with operators like External Secrets Operator for production secrets.

---

### 5. Deployments and Scaling

#### Deployment Object
A Deployment manages ReplicaSets, which manage Pods. It is the standard workload controller for stateless applications. A Deployment spec includes a Pod template and a replica count.

#### ReplicaSets
ReplicaSets ensure a specified number of Pod replicas are running at all times. Deployments create and manage ReplicaSets — you should never create ReplicaSets directly.

#### Rolling Updates and Rollbacks
- **Rolling update** (default strategy): Incrementally replaces old Pods with new ones. Controlled by `maxUnavailable` (how many Pods can be down during update) and `maxSurge` (how many extra Pods can be created).
- **Recreate strategy**: Kills all existing Pods before creating new ones. Causes downtime but avoids running two versions simultaneously.
- **Rollback**: `kubectl rollout undo deployment/<name>` reverts to the previous ReplicaSet. Use `--to-revision=N` for specific versions. Deployment keeps revision history (configurable via `revisionHistoryLimit`).
- **Rollout status**: `kubectl rollout status deployment/<name>` blocks until complete. Use in CI/CD pipelines to gate post-deployment steps.

#### Horizontal Pod Autoscaler (HPA)
- Automatically scales replica count based on observed metrics.
- **CPU/Memory**: Built-in metrics via metrics-server. `targetAverageUtilization` or `targetAverageValue`.
- **Custom metrics**: Scale on application-specific metrics (request rate, queue depth) via Prometheus adapter or KEDA.
- **Behavior tuning**: Configure `scaleUp` and `scaleDown` policies to control scaling velocity and stabilization windows to prevent flapping.

---

### 6. Services and Networking

#### ClusterIP
- Default Service type. Assigns a virtual IP reachable only within the cluster.
- kube-proxy programs iptables/IPVS rules to load-balance traffic across backend Pods.
- Used for internal communication between microservices.

#### NodePort
- Extends ClusterIP by opening a static port (30000-32767) on every node.
- External traffic reaches `<NodeIP>:<NodePort>` and is forwarded to the Service.
- Not recommended for production — use LoadBalancer or Ingress instead.

#### LoadBalancer
- Extends NodePort by provisioning a cloud provider's load balancer (AWS ELB/NLB, GCP LB, Azure LB).
- Each LoadBalancer Service gets its own external IP. Can be expensive with many services.
- On bare metal, use MetalLB to provide LoadBalancer functionality.

#### ExternalName
- Maps a Service to an external DNS name (CNAME record). No proxying or load balancing.
- Used to reference external services (databases, third-party APIs) via Kubernetes DNS.

#### Headless Services
- Set `clusterIP: None`. DNS returns Pod IPs directly instead of a virtual IP.
- Used by StatefulSets for stable network identities and by applications that need direct pod-to-pod communication.

#### Ingress
- HTTP/HTTPS routing layer that sits in front of Services.
- Supports path-based and host-based routing, TLS termination, rate limiting, and request transformation.
- Requires an Ingress Controller (NGINX, Traefik, HAProxy, AWS ALB Ingress Controller, etc.).
- Kubernetes Gateway API is the successor to Ingress, offering more expressive routing.

#### Network Policies
- Firewall rules for pod-to-pod traffic. Default: all traffic allowed.
- Define ingress and egress rules using pod selectors, namespace selectors, and CIDR blocks.
- Require a CNI plugin that supports NetworkPolicy (Calico, Cilium, Weave Net). Flannel does NOT support NetworkPolicy.

---

### 7. Podman-to-Kubernetes Bridge

#### podman generate kube
- Generates Kubernetes YAML from running Podman containers and pods.
- `podman generate kube <pod-name>` produces Pod, Service, PersistentVolumeClaim manifests.
- Useful for bootstrapping Kubernetes manifests from a working local setup.
- Limitations: generates Pod resources (not Deployments), may need manual adjustment for production.

#### podman play kube
- Deploys Kubernetes YAML locally using Podman.
- `podman play kube deployment.yaml` creates containers, pods, and volumes from K8s manifests.
- Supports ConfigMaps, Secrets (from files), PVCs, and Pod specs.
- Enables testing Kubernetes manifests locally without a cluster.

#### Dev-to-Prod Workflow
1. **Develop locally** with `podman run` or `podman compose` for rapid iteration.
2. **Generate K8s YAML** with `podman generate kube` to bootstrap manifests.
3. **Test locally** with `podman play kube` to validate manifests.
4. **Enhance manifests** — add Deployment wrappers, resource requests/limits, health probes, Services, Ingress.
5. **Deploy to cluster** with `kubectl apply -f` or a GitOps tool (Argo CD, Flux).
6. **Iterate** — use the same container images in both environments, vary configuration via ConfigMaps/Secrets and environment-specific overlays (Kustomize or Helm).

---

## Key Concepts

| Concept | Summary |
|---|---|
| Daemonless containers | Podman's fork/exec model eliminates the single-daemon bottleneck and improves security |
| Rootless containers | User namespaces map container root to unprivileged host UID, providing defense-in-depth |
| OCI standard | Open Container Initiative ensures image and runtime portability across tools |
| Declarative model | Kubernetes reconciliation loops continuously drive actual state toward desired state |
| Pod networking | All containers in a Pod share the same network namespace — communicate via localhost |
| Service abstraction | Stable virtual IP and DNS name decoupled from ephemeral Pod IPs |
| Rolling updates | Zero-downtime deployments with configurable surge and unavailability thresholds |
| Quadlet | Modern Podman-systemd integration replacing `podman generate systemd` |
| Health probes | Liveness, readiness, and startup probes enable self-healing and traffic management |
| Network Policies | Default-deny + explicit allow rules for pod-to-pod microsegmentation |

## Problem-Solving Patterns

| Problem | Pattern |
|---|---|
| Container cannot bind low ports rootless | Set `net.ipv4.ip_unprivileged_port_start=0` or use port mapping to high ports |
| Volume permission denied in rootless container | Use `podman unshare chown` or `--userns=keep-id` for bind mounts |
| Pods stuck in Pending | Check node resources (`kubectl describe node`), taints/tolerations, and PVC binding |
| Service not routing traffic | Verify selector labels match Pod labels, check readiness probes and endpoints (`kubectl get ep`) |
| CrashLoopBackOff | Inspect logs (`kubectl logs --previous`), check resource limits (OOMKilled), verify health probes |
| Image pull errors | Check registry credentials (imagePullSecrets), image name/tag, network access to registry |
| Rolling update stuck | Check Pod disruption budgets, resource availability, and failed readiness probes |
| Container networking isolation | Apply NetworkPolicy with default-deny ingress/egress, then whitelist required paths |
| Podman compose networking issues | Use `podman network create` explicitly, check Netavark/CNI backend configuration |
| Systemd container not starting on boot | Ensure Quadlet files are in correct directory, run `systemctl --user daemon-reload`, enable lingering with `loginctl enable-linger` |

## Common Pitfalls

| Pitfall | Explanation |
|---|---|
| Running as root in containers | Always use rootless Podman or set `runAsNonRoot: true` in K8s securityContext |
| No resource requests/limits | Pods without requests can starve other workloads; without limits, a single pod can OOM the node |
| Using :latest tag | Non-deterministic deployments. Pin image tags or digests for reproducibility |
| Aggressive liveness probes | Too-fast liveness probes restart containers during normal load spikes — use startup probes for slow apps |
| Ignoring NetworkPolicy | Default Kubernetes allows all pod-to-pod traffic. Implement default-deny policies per namespace |
| Secrets in environment variables | Env vars appear in `kubectl describe pod` and crash dumps. Prefer mounted Secret volumes |
| Not backing up etcd | etcd holds all cluster state. Regular snapshots are critical for disaster recovery |
| Single-replica Deployments | No high availability. Run at least 2 replicas with pod anti-affinity for production workloads |
| Skipping readiness probes | Without readiness probes, traffic routes to Pods before they can serve, causing errors during startup and deployment |
| SELinux label confusion | `:Z` is private (single container), `:z` is shared (multiple containers). Using `:Z` on shared data causes access errors |

---

## Source Material

- Walsh, M. (2023). *Podman in Action*. Manning Publications. Covers daemonless container management, rootless containers, Podman networking, systemd integration, Quadlet, volume management, and the Podman-to-Kubernetes workflow.
- Poulton, N. (2021). *The Kubernetes Book*. Independently published. Covers Kubernetes architecture, Pods, Deployments, Services, networking, storage, ConfigMaps, Secrets, RBAC, and cluster operations.
