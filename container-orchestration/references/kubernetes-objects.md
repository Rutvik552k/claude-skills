# Kubernetes Objects: YAML Reference

This reference provides production-ready YAML examples for core Kubernetes objects
with annotations explaining each field.

---

## Pod

A Pod is the smallest deployable unit. In production, always use a Deployment rather
than creating Pods directly.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: web-app
  namespace: production
  labels:
    app: web
    version: v1
    environment: production
  annotations:
    description: "Example pod with init container and health probes"
spec:
  # Security context at pod level
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    runAsGroup: 1000
    fsGroup: 1000
    seccompProfile:
      type: RuntimeDefault

  # Service account for RBAC
  serviceAccountName: web-app-sa

  # Init container runs before main containers
  initContainers:
    - name: wait-for-db
      image: busybox:1.36
      command:
        - sh
        - -c
        - |
          until nc -z postgres-svc 5432; do
            echo "Waiting for database..."
            sleep 2
          done
      resources:
        requests:
          cpu: 50m
          memory: 32Mi
        limits:
          memory: 64Mi

  containers:
    - name: web
      image: myregistry.io/web-app:v1.2.3
      ports:
        - containerPort: 8080
          name: http
          protocol: TCP

      # Environment variables
      env:
        - name: APP_ENV
          value: "production"
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: db-credentials
              key: connection-string
        - name: LOG_LEVEL
          valueFrom:
            configMapKeyRef:
              name: app-config
              key: log-level

      # Resource management
      resources:
        requests:
          cpu: 250m
          memory: 256Mi
        limits:
          memory: 512Mi
          # CPU limits intentionally omitted to avoid throttling

      # Health probes
      startupProbe:
        httpGet:
          path: /healthz
          port: http
        failureThreshold: 30
        periodSeconds: 5

      livenessProbe:
        httpGet:
          path: /healthz
          port: http
        initialDelaySeconds: 0
        periodSeconds: 15
        timeoutSeconds: 3
        failureThreshold: 3

      readinessProbe:
        httpGet:
          path: /ready
          port: http
        initialDelaySeconds: 0
        periodSeconds: 5
        timeoutSeconds: 3
        failureThreshold: 2

      # Volume mounts
      volumeMounts:
        - name: config-volume
          mountPath: /etc/app/config
          readOnly: true
        - name: tmp
          mountPath: /tmp

      # Security context at container level
      securityContext:
        allowPrivilegeEscalation: false
        readOnlyRootFilesystem: true
        capabilities:
          drop:
            - ALL

    # Sidecar container for log shipping
    - name: log-shipper
      image: fluent/fluent-bit:2.2
      resources:
        requests:
          cpu: 50m
          memory: 64Mi
        limits:
          memory: 128Mi
      volumeMounts:
        - name: shared-logs
          mountPath: /var/log/app
          readOnly: true

  volumes:
    - name: config-volume
      configMap:
        name: app-config
    - name: tmp
      emptyDir:
        sizeLimit: 100Mi
    - name: shared-logs
      emptyDir: {}

  # Graceful shutdown
  terminationGracePeriodSeconds: 30

  # Scheduling constraints
  nodeSelector:
    kubernetes.io/os: linux
  tolerations:
    - key: "dedicated"
      operator: "Equal"
      value: "web"
      effect: "NoSchedule"
```

---

## Deployment

The standard controller for stateless applications. Manages ReplicaSets which manage Pods.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
  namespace: production
  labels:
    app: web
    version: v1
spec:
  replicas: 3
  revisionHistoryLimit: 5

  # Selector must match Pod template labels
  selector:
    matchLabels:
      app: web

  # Rolling update strategy
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1           # One extra pod during update
      maxUnavailable: 0     # Zero downtime

  template:
    metadata:
      labels:
        app: web
        version: v1
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "8080"
        prometheus.io/path: "/metrics"
    spec:
      serviceAccountName: web-app-sa
      securityContext:
        runAsNonRoot: true
        runAsUser: 1000
        fsGroup: 1000
        seccompProfile:
          type: RuntimeDefault

      # Pod anti-affinity for high availability
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
            - weight: 100
              podAffinityTerm:
                labelSelector:
                  matchLabels:
                    app: web
                topologyKey: kubernetes.io/hostname

      containers:
        - name: web
          image: myregistry.io/web-app:v1.2.3
          imagePullPolicy: IfNotPresent
          ports:
            - containerPort: 8080
              name: http

          envFrom:
            - configMapRef:
                name: app-config
            - secretRef:
                name: app-secrets

          resources:
            requests:
              cpu: 250m
              memory: 256Mi
            limits:
              memory: 512Mi

          startupProbe:
            httpGet:
              path: /healthz
              port: http
            failureThreshold: 30
            periodSeconds: 5

          livenessProbe:
            httpGet:
              path: /healthz
              port: http
            periodSeconds: 15
            failureThreshold: 3

          readinessProbe:
            httpGet:
              path: /ready
              port: http
            periodSeconds: 5
            failureThreshold: 2

          securityContext:
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem: true
            capabilities:
              drop:
                - ALL

          volumeMounts:
            - name: tmp
              mountPath: /tmp

      volumes:
        - name: tmp
          emptyDir: {}

      terminationGracePeriodSeconds: 30
```

---

## Service

### ClusterIP (Internal)

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-app-svc
  namespace: production
  labels:
    app: web
spec:
  type: ClusterIP
  selector:
    app: web            # Must match Pod labels
  ports:
    - name: http
      port: 80          # Service port (what clients connect to)
      targetPort: http  # Container port name or number (8080)
      protocol: TCP
```

### NodePort (External via node ports)

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-app-nodeport
  namespace: production
spec:
  type: NodePort
  selector:
    app: web
  ports:
    - name: http
      port: 80
      targetPort: http
      nodePort: 30080   # Static port on every node (30000-32767)
      protocol: TCP
```

### LoadBalancer (Cloud provider external LB)

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-app-lb
  namespace: production
  annotations:
    # AWS-specific annotations
    service.beta.kubernetes.io/aws-load-balancer-type: nlb
    service.beta.kubernetes.io/aws-load-balancer-scheme: internet-facing
    service.beta.kubernetes.io/aws-load-balancer-cross-zone-load-balancing-enabled: "true"
spec:
  type: LoadBalancer
  selector:
    app: web
  ports:
    - name: http
      port: 80
      targetPort: http
    - name: https
      port: 443
      targetPort: 8443
  # Restrict source IPs
  loadBalancerSourceRanges:
    - 10.0.0.0/8
    - 203.0.113.0/24
```

### Headless Service (Direct pod discovery)

```yaml
apiVersion: v1
kind: Service
metadata:
  name: db-headless
  namespace: production
spec:
  type: ClusterIP
  clusterIP: None       # Makes it headless
  selector:
    app: postgres
  ports:
    - name: postgres
      port: 5432
      targetPort: 5432
# DNS returns individual Pod IPs:
# pod-0.db-headless.production.svc.cluster.local
# pod-1.db-headless.production.svc.cluster.local
```

### ExternalName (DNS alias)

```yaml
apiVersion: v1
kind: Service
metadata:
  name: external-db
  namespace: production
spec:
  type: ExternalName
  externalName: db.example.aws.com  # CNAME target
# Resolving external-db.production.svc.cluster.local
# returns CNAME db.example.aws.com
```

---

## Ingress

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: web-app-ingress
  namespace: production
  annotations:
    # NGINX Ingress Controller annotations
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/proxy-body-size: "10m"
    nginx.ingress.kubernetes.io/rate-limit: "100"
    nginx.ingress.kubernetes.io/rate-limit-window: "1m"
    cert-manager.io/cluster-issuer: letsencrypt-prod
spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - app.example.com
        - api.example.com
      secretName: app-tls-cert

  rules:
    # Host-based routing
    - host: app.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: web-app-svc
                port:
                  name: http

          - path: /static
            pathType: Prefix
            backend:
              service:
                name: static-assets-svc
                port:
                  number: 80

    - host: api.example.com
      http:
        paths:
          - path: /v1
            pathType: Prefix
            backend:
              service:
                name: api-v1-svc
                port:
                  name: http

          - path: /v2
            pathType: Prefix
            backend:
              service:
                name: api-v2-svc
                port:
                  name: http

  # Default backend for unmatched requests
  defaultBackend:
    service:
      name: default-backend-svc
      port:
        number: 80
```

---

## ConfigMap

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: production
  labels:
    app: web
data:
  # Simple key-value pairs (injected as env vars or files)
  log-level: "info"
  max-connections: "100"
  feature-flags: "dark-mode=true,beta-api=false"

  # Multi-line configuration file
  nginx.conf: |
    server {
        listen 80;
        server_name _;

        location / {
            proxy_pass http://localhost:8080;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
        }

        location /health {
            access_log off;
            return 200 "ok\n";
        }
    }

  # Application config file
  app.yaml: |
    server:
      port: 8080
      readTimeout: 30s
      writeTimeout: 30s
    database:
      maxOpenConns: 25
      maxIdleConns: 5
      connMaxLifetime: 5m
    cache:
      ttl: 300s
      maxSize: 1000
```

### Mounting a ConfigMap

```yaml
# As environment variables
envFrom:
  - configMapRef:
      name: app-config

# Individual keys as env vars
env:
  - name: LOG_LEVEL
    valueFrom:
      configMapKeyRef:
        name: app-config
        key: log-level

# As a mounted volume (each key becomes a file)
volumes:
  - name: config
    configMap:
      name: app-config
      items:
        - key: nginx.conf
          path: nginx.conf
        - key: app.yaml
          path: app.yaml

volumeMounts:
  - name: config
    mountPath: /etc/app
    readOnly: true
```

---

## Secret

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-credentials
  namespace: production
  labels:
    app: web
type: Opaque
# Values are base64-encoded (NOT encrypted)
data:
  username: YWRtaW4=                    # echo -n "admin" | base64
  password: czNjdXIzLXBAc3N3MHJk       # echo -n "s3cur3-p@ssw0rd" | base64
  connection-string: cG9zdGdyZXNxbDovL2FkbWluOnMzY3VyMy1wQHNzdzByZEBkYjo1NDMyL215YXBw

---
# TLS Secret (for Ingress)
apiVersion: v1
kind: Secret
metadata:
  name: app-tls-cert
  namespace: production
type: kubernetes.io/tls
data:
  tls.crt: <base64-encoded-certificate>
  tls.key: <base64-encoded-private-key>

---
# Docker registry credentials
apiVersion: v1
kind: Secret
metadata:
  name: registry-creds
  namespace: production
type: kubernetes.io/dockerconfigjson
data:
  .dockerconfigjson: <base64-encoded-docker-config>
```

### Creating Secrets Imperatively

```bash
# From literal values
kubectl create secret generic db-credentials \
  --from-literal=username=admin \
  --from-literal=password='s3cur3-p@ssw0rd' \
  -n production

# From files
kubectl create secret generic tls-certs \
  --from-file=tls.crt=./server.crt \
  --from-file=tls.key=./server.key \
  -n production

# Docker registry secret
kubectl create secret docker-registry registry-creds \
  --docker-server=myregistry.io \
  --docker-username=myuser \
  --docker-password=mytoken \
  -n production
```

### Mounting Secrets

```yaml
# As environment variables (visible in kubectl describe — use with caution)
env:
  - name: DB_PASSWORD
    valueFrom:
      secretKeyRef:
        name: db-credentials
        key: password

# As mounted files (preferred for sensitive data)
volumes:
  - name: db-creds
    secret:
      secretName: db-credentials
      defaultMode: 0400    # Read-only by owner

volumeMounts:
  - name: db-creds
    mountPath: /etc/secrets/db
    readOnly: true

# Use imagePullSecrets for private registries
imagePullSecrets:
  - name: registry-creds
```

---

## NetworkPolicy

```yaml
# Default deny all ingress and egress in a namespace
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: production
spec:
  podSelector: {}          # Applies to ALL pods in namespace
  policyTypes:
    - Ingress
    - Egress

---
# Allow specific traffic for web pods
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: web-app-policy
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: web
  policyTypes:
    - Ingress
    - Egress

  ingress:
    # Allow traffic from Ingress controller
    - from:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: ingress-nginx
          podSelector:
            matchLabels:
              app.kubernetes.io/name: ingress-nginx
      ports:
        - protocol: TCP
          port: 8080

    # Allow traffic from monitoring namespace
    - from:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: monitoring
      ports:
        - protocol: TCP
          port: 8080

  egress:
    # Allow DNS resolution
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

    # Allow traffic to database
    - to:
        - podSelector:
            matchLabels:
              app: postgres
      ports:
        - protocol: TCP
          port: 5432

    # Allow traffic to external APIs (CIDR-based)
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

---

## Horizontal Pod Autoscaler

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: web-app-hpa
  namespace: production
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: web-app
  minReplicas: 3
  maxReplicas: 20

  metrics:
    # Scale on CPU utilization
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70

    # Scale on memory utilization
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 80

    # Scale on custom metric (requests per second)
    - type: Pods
      pods:
        metric:
          name: http_requests_per_second
        target:
          type: AverageValue
          averageValue: "1000"

  behavior:
    scaleUp:
      stabilizationWindowSeconds: 60
      policies:
        - type: Percent
          value: 50
          periodSeconds: 60
        - type: Pods
          value: 5
          periodSeconds: 60
      selectPolicy: Max

    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
        - type: Percent
          value: 10
          periodSeconds: 120
      selectPolicy: Min
```

---

## Namespace with Resource Quotas

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    environment: production
    team: platform

---
apiVersion: v1
kind: ResourceQuota
metadata:
  name: production-quota
  namespace: production
spec:
  hard:
    requests.cpu: "20"
    requests.memory: 40Gi
    limits.cpu: "40"
    limits.memory: 80Gi
    pods: "100"
    services: "20"
    persistentvolumeclaims: "30"

---
apiVersion: v1
kind: LimitRange
metadata:
  name: default-limits
  namespace: production
spec:
  limits:
    - type: Container
      default:
        cpu: 500m
        memory: 256Mi
      defaultRequest:
        cpu: 100m
        memory: 128Mi
      max:
        cpu: "4"
        memory: 8Gi
      min:
        cpu: 50m
        memory: 32Mi
```

---

## Quick Reference: kubectl Commands

```bash
# Apply manifests
kubectl apply -f deployment.yaml
kubectl apply -f k8s/                    # Apply all YAML in directory
kubectl apply -k overlays/production/    # Apply Kustomize overlay

# Inspect resources
kubectl get pods -n production -o wide
kubectl get svc,ing -n production
kubectl describe pod web-app-xyz -n production
kubectl get events -n production --sort-by='.lastTimestamp'

# Debugging
kubectl logs web-app-xyz -n production -c web --previous
kubectl exec -it web-app-xyz -n production -c web -- /bin/sh
kubectl port-forward svc/web-app-svc 8080:80 -n production

# Scaling and updates
kubectl scale deployment web-app --replicas=5 -n production
kubectl rollout status deployment/web-app -n production
kubectl rollout history deployment/web-app -n production
kubectl rollout undo deployment/web-app -n production
kubectl rollout undo deployment/web-app --to-revision=3 -n production

# Resource management
kubectl top pods -n production
kubectl top nodes
kubectl get resourcequota -n production
```

---

*Source: Poulton, N. (2021). The Kubernetes Book. Independently published.*
