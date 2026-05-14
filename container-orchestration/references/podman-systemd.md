# Podman systemd Integration: Unit Files, Quadlet, Auto-Update, and Rootless Services

## Overview

Podman's systemd integration is one of its strongest differentiators from Docker. Instead
of relying on a daemon with restart policies, Podman containers are managed as native
systemd services — enabling process supervision, dependency ordering, socket activation,
journald logging, and resource control through standard systemd mechanisms.

There are two approaches:

1. **Legacy**: `podman generate systemd` — generates traditional `.service` unit files.
2. **Modern (Recommended)**: **Quadlet** — place declarative `.container`, `.volume`, `.network`, `.kube`, or `.image` files and let the Quadlet generator produce systemd units automatically.

---

## Legacy: podman generate systemd

### Generating Unit Files

```bash
# Create and start a container
podman run -d \
  --name web \
  -p 8080:80 \
  -v web-data:/usr/share/nginx/html:Z \
  docker.io/library/nginx:stable

# Generate a unit file (--new creates container from scratch each start)
podman generate systemd --new --name web > ~/.config/systemd/user/web.service

# Without --new: unit manages the existing container (start/stop only)
podman generate systemd --name web > ~/.config/systemd/user/web.service
```

### Generated Unit File (--new)

```ini
# ~/.config/systemd/user/web.service
[Unit]
Description=Podman web.service
Documentation=man:podman-generate-systemd(1)
Wants=network-online.target
After=network-online.target
RequiresMountsFor=%t/containers

[Service]
Environment=PODMAN_SYSTEMD_UNIT=%n
Restart=on-failure
TimeoutStopSec=70
ExecStartPre=/bin/rm -f %t/%n.ctr-id
ExecStart=/usr/bin/podman run \
  --cidfile=%t/%n.ctr-id \
  --cgroups=no-conmon \
  --rm \
  --sdnotify=conmon \
  -d \
  --replace \
  --name web \
  -p 8080:80 \
  -v web-data:/usr/share/nginx/html:Z \
  docker.io/library/nginx:stable
ExecStop=/usr/bin/podman stop --ignore --cidfile=%t/%n.ctr-id
ExecStopPost=/usr/bin/podman rm -f --ignore --cidfile=%t/%n.ctr-id
Type=notify
NotifyAccess=all

[Install]
WantedBy=default.target
```

### Managing the Service

```bash
# Reload systemd (rootless)
systemctl --user daemon-reload

# Enable and start
systemctl --user enable --now web.service

# Check status
systemctl --user status web.service

# View logs
journalctl --user -u web.service -f

# Stop and disable
systemctl --user stop web.service
systemctl --user disable web.service
```

### Generating Pod Unit Files

```bash
# Create a pod with containers
podman pod create --name webapp -p 8080:80 -p 5432:5432
podman run -d --pod webapp --name frontend nginx:stable
podman run -d --pod webapp --name backend myapp:v1
podman run -d --pod webapp --name db postgres:16

# Generate unit files for the entire pod
podman generate systemd --new --name webapp --files
# Creates:
#   pod-webapp.service
#   container-frontend.service
#   container-backend.service
#   container-db.service

# Move to systemd directory
mv *.service ~/.config/systemd/user/
systemctl --user daemon-reload
systemctl --user enable --now pod-webapp.service
```

> **Note**: `podman generate systemd` is deprecated as of Podman 4.4. Migrate to Quadlet.

---

## Modern: Quadlet (Podman 4.4+)

Quadlet is a systemd generator that reads declarative container description files and
produces systemd unit files at runtime. Drop a `.container` file in the right directory
and systemd handles the rest.

### File Locations

| Context | Directory |
|---|---|
| Rootless (user) | `~/.config/containers/systemd/` |
| Rootless (user) alt | `$XDG_CONFIG_HOME/containers/systemd/` |
| Rootful (system) | `/etc/containers/systemd/` |
| Rootful (system) alt | `/usr/share/containers/systemd/` |

### Container File (.container)

```ini
# ~/.config/containers/systemd/web.container
[Unit]
Description=NGINX web server
After=network-online.target

[Container]
# Image (required)
Image=docker.io/library/nginx:stable

# Container name (optional, defaults to unit name)
ContainerName=web

# Port mapping
PublishPort=8080:80
PublishPort=8443:443

# Volumes
Volume=web-data.volume:/usr/share/nginx/html:Z
Volume=web-config.volume:/etc/nginx/conf.d:Z,ro

# Environment variables
Environment=NGINX_ENVSUBST_OUTPUT_DIR=/etc/nginx/conf.d
Environment=TZ=UTC

# Environment from file
EnvironmentFile=%h/.config/containers/web.env

# Resource limits
PodmanArgs=--memory=512m --cpus=1.0

# Health check
HealthCmd=curl -f http://localhost:80/ || exit 1
HealthInterval=30s
HealthRetries=3
HealthStartPeriod=10s
HealthTimeout=5s

# Security
SecurityLabelDisable=false
ReadOnly=true
UserNS=keep-id

# Networking
Network=webapp.network

# Auto-update (see Auto-Update section below)
AutoUpdate=registry

# Labels
Label=app=web
Label=environment=production

[Service]
Restart=always
TimeoutStartSec=120
TimeoutStopSec=30

# Systemd hardening
ProtectHome=true
ProtectSystem=strict
NoNewPrivileges=true

[Install]
WantedBy=default.target
```

### Volume File (.volume)

```ini
# ~/.config/containers/systemd/web-data.volume
[Unit]
Description=Web server data volume

[Volume]
# Named volume options
Driver=local
Label=app=web

# Optional: specify a device or host path for bind mount
# Copy=true copies image data into the volume on first use
Copy=true
```

### Network File (.network)

```ini
# ~/.config/containers/systemd/webapp.network
[Unit]
Description=Web application network

[Network]
# Network driver (bridge is default)
Driver=bridge

# Subnet configuration
Subnet=10.89.1.0/24
Gateway=10.89.1.1

# Enable DNS resolution between containers
DNS=true

# IPv6 support
IPv6=false

# Internal network (no external access)
Internal=false

Label=app=webapp
```

### Kube File (.kube)

Deploy Kubernetes YAML locally via Quadlet:

```ini
# ~/.config/containers/systemd/webapp.kube
[Unit]
Description=Web application from Kubernetes YAML
After=network-online.target

[Kube]
# Path to Kubernetes YAML file
Yaml=%h/k8s/webapp-pod.yaml

# Publish ports (override YAML)
PublishPort=8080:80

# ConfigMap files
ConfigMap=%h/k8s/app-configmap.yaml

# Auto-update
AutoUpdate=registry

# Network
Network=webapp.network

[Service]
Restart=always
TimeoutStartSec=180

[Install]
WantedBy=default.target
```

### Image File (.image)

Pre-pull images as a systemd service:

```ini
# ~/.config/containers/systemd/nginx.image
[Unit]
Description=Pull NGINX image

[Image]
Image=docker.io/library/nginx:stable

# Optional: pull policy
PodmanArgs=--tls-verify=true
```

### Activating Quadlet Units

```bash
# After placing files in ~/.config/containers/systemd/

# Reload systemd to trigger the Quadlet generator
systemctl --user daemon-reload

# The generator creates transient unit files. Verify:
systemctl --user list-unit-files 'web*'

# Enable and start
systemctl --user enable --now web.service

# Check status
systemctl --user status web.service

# View generated unit file (for debugging)
/usr/lib/systemd/user-generators/podman-user-generator \
  /tmp/quadlet-debug 2>&1
ls /tmp/quadlet-debug/

# Or use the debug command
podman system service --log-level=debug 2>&1 | head
```

### Multi-Container Application with Quadlet

```ini
# ~/.config/containers/systemd/app-network.network
[Network]
Driver=bridge
Subnet=10.89.2.0/24
DNS=true
```

```ini
# ~/.config/containers/systemd/postgres-data.volume
[Volume]
Driver=local
Label=app=myapp
Label=component=database
```

```ini
# ~/.config/containers/systemd/postgres.container
[Unit]
Description=PostgreSQL database
After=network-online.target

[Container]
Image=docker.io/library/postgres:16
ContainerName=postgres
Network=app-network.network
Volume=postgres-data.volume:/var/lib/postgresql/data:Z
Environment=POSTGRES_DB=myapp
Environment=POSTGRES_USER=appuser
Secret=postgres-password,type=env,target=POSTGRES_PASSWORD
HealthCmd=pg_isready -U appuser -d myapp
HealthInterval=10s
HealthRetries=5
HealthStartPeriod=30s
AutoUpdate=registry

[Service]
Restart=always
TimeoutStartSec=120

[Install]
WantedBy=default.target
```

```ini
# ~/.config/containers/systemd/api.container
[Unit]
Description=Application API server
After=postgres.service
Requires=postgres.service

[Container]
Image=myregistry.io/myapp-api:latest
ContainerName=api
Network=app-network.network
PublishPort=3000:3000
Environment=DATABASE_HOST=postgres
Environment=DATABASE_PORT=5432
Environment=DATABASE_NAME=myapp
Environment=DATABASE_USER=appuser
Secret=postgres-password,type=env,target=DATABASE_PASSWORD
HealthCmd=curl -f http://localhost:3000/health || exit 1
HealthInterval=15s
HealthRetries=3
AutoUpdate=registry

[Service]
Restart=always
TimeoutStartSec=60

[Install]
WantedBy=default.target
```

```ini
# ~/.config/containers/systemd/web.container
[Unit]
Description=Web frontend
After=api.service
Requires=api.service

[Container]
Image=myregistry.io/myapp-web:latest
ContainerName=web-frontend
Network=app-network.network
PublishPort=8080:80
Environment=API_URL=http://api:3000
AutoUpdate=registry

[Service]
Restart=always

[Install]
WantedBy=default.target
```

```bash
# Deploy the entire stack
systemctl --user daemon-reload
systemctl --user enable --now postgres.service api.service web.service

# Check all services
systemctl --user status postgres.service api.service web.service
```

---

## Socket Activation

systemd can listen on a socket and start the container only when a connection arrives.
This enables on-demand container startup — the container is not running until needed.

```ini
# ~/.config/containers/systemd/on-demand-web.socket
[Unit]
Description=On-demand web server socket

[Socket]
ListenStream=8080
Accept=false

[Install]
WantedBy=sockets.target
```

```ini
# ~/.config/containers/systemd/on-demand-web.container
[Unit]
Description=On-demand web server (socket-activated)

[Container]
Image=docker.io/library/nginx:stable
ContainerName=on-demand-web
# Do not publish port — systemd manages the socket
PodmanArgs=--network=host

[Service]
# Only started when the socket receives a connection
Type=notify

[Install]
# No WantedBy — started by socket activation only
```

```bash
systemctl --user daemon-reload
systemctl --user enable --now on-demand-web.socket

# Container is NOT running yet
podman ps  # empty

# First connection triggers container start
curl http://localhost:8080

# Now the container is running
podman ps  # shows on-demand-web
```

---

## Auto-Update

Podman auto-update checks registries for newer images and automatically restarts
containers when updates are available.

### Configuration

```ini
# In your .container file:
[Container]
Image=docker.io/library/nginx:stable
AutoUpdate=registry
# Options:
#   registry  — pull from registry, restart if image changed
#   local     — restart if local image tag changed (e.g., after podman build)
```

### Running Auto-Update

```bash
# Check for updates (dry run)
podman auto-update --dry-run

# Apply updates
podman auto-update

# Output shows which containers were updated:
# UNIT                 CONTAINER        IMAGE              POLICY    UPDATED
# web.service          abc123def        nginx:stable       registry  true
# api.service          def456ghi        myapp:latest       registry  false
```

### systemd Timer for Scheduled Updates

```ini
# ~/.config/systemd/user/podman-auto-update.timer
[Unit]
Description=Podman auto-update timer

[Timer]
# Run daily at 3 AM
OnCalendar=*-*-* 03:00:00
RandomizedDelaySec=900
Persistent=true

[Install]
WantedBy=timers.target
```

```ini
# ~/.config/systemd/user/podman-auto-update.service
[Unit]
Description=Podman auto-update service

[Service]
Type=oneshot
ExecStart=/usr/bin/podman auto-update
```

```bash
systemctl --user daemon-reload
systemctl --user enable --now podman-auto-update.timer

# Verify timer is active
systemctl --user list-timers podman-auto-update.timer
```

### Rollback on Failure

Podman auto-update supports automatic rollback. If the updated container fails its
health check, Podman rolls back to the previous image.

```ini
[Container]
Image=myregistry.io/myapp:latest
AutoUpdate=registry

# Health check is required for rollback to work
HealthCmd=curl -f http://localhost:3000/health || exit 1
HealthInterval=10s
HealthRetries=3
HealthStartPeriod=30s
HealthTimeout=5s
```

Rollback flow:
1. `podman auto-update` pulls new image.
2. Container restarts with new image.
3. Health check runs during `HealthStartPeriod`.
4. If health check fails after `HealthRetries`, Podman reverts to the previous image.
5. Container restarts with the old (working) image.

---

## Rootless Service Management

### Enable Lingering

By default, user systemd instances stop when the user logs out. Enable lingering to
keep services running after logout:

```bash
# Enable lingering for current user
loginctl enable-linger $USER

# Verify
loginctl show-user $USER | grep Linger
# Linger=yes

# Disable if needed
loginctl disable-linger $USER
```

### XDG Runtime Directory

Rootless Podman uses `$XDG_RUNTIME_DIR` (typically `/run/user/$UID`) for sockets and
temporary files. This directory is created at login and cleaned up at logout (unless
lingering is enabled).

```bash
# Check runtime directory
echo $XDG_RUNTIME_DIR
# /run/user/1000

# Podman socket location
ls $XDG_RUNTIME_DIR/podman/podman.sock
```

### Common Rootless Issues and Solutions

| Issue | Cause | Solution |
|---|---|---|
| Service stops after logout | Lingering not enabled | `loginctl enable-linger $USER` |
| Permission denied on bind mount | UID mapping mismatch | Use `podman unshare chown` or `UserNS=keep-id` |
| Cannot bind port < 1024 | Rootless port restriction | `sysctl net.ipv4.ip_unprivileged_port_start=80` or use port > 1024 |
| SELinux denials on volumes | Missing labels | Add `:Z` (private) or `:z` (shared) to volume mounts |
| Container not starting on boot | systemd user instance not running | Enable lingering + WantedBy=default.target |
| Slow network performance | slirp4netns overhead | Switch to pasta: `default_rootless_network_cmd = "pasta"` in containers.conf |
| DNS not resolving between containers | Not using a named network | Create a Netavark network: `podman network create mynet` |
| `podman auto-update` not running | Timer not enabled | `systemctl --user enable --now podman-auto-update.timer` |

### Production Checklist for Rootless Podman Services

```bash
# 1. Enable lingering
loginctl enable-linger $USER

# 2. Create Quadlet files
ls ~/.config/containers/systemd/
# myapp.container  myapp-data.volume  myapp.network

# 3. Reload and enable
systemctl --user daemon-reload
systemctl --user enable --now myapp.service

# 4. Verify service is running
systemctl --user status myapp.service
podman ps

# 5. Check logs
journalctl --user -u myapp.service --since today

# 6. Verify auto-update timer
systemctl --user enable --now podman-auto-update.timer
systemctl --user list-timers

# 7. Test reboot persistence
sudo reboot
# After reboot:
systemctl --user status myapp.service  # Should be active

# 8. Monitor with journald
journalctl --user -u myapp.service -f
```

---

## Migration: Docker Compose to Quadlet

### Before (docker-compose.yml)

```yaml
version: "3.9"
services:
  web:
    image: nginx:stable
    ports:
      - "8080:80"
    volumes:
      - ./html:/usr/share/nginx/html:ro
    depends_on:
      - api
    restart: always

  api:
    image: myapp:latest
    ports:
      - "3000:3000"
    environment:
      - DATABASE_URL=postgresql://user:pass@db:5432/myapp
    depends_on:
      - db
    restart: always

  db:
    image: postgres:16
    volumes:
      - pgdata:/var/lib/postgresql/data
    environment:
      - POSTGRES_PASSWORD=secretpass
    restart: always

volumes:
  pgdata:
```

### After (Quadlet files)

```ini
# app.network
[Network]
Driver=bridge
Subnet=10.89.3.0/24
DNS=true

# pgdata.volume
[Volume]
Driver=local

# db.container
[Unit]
Description=PostgreSQL
After=network-online.target
[Container]
Image=docker.io/library/postgres:16
ContainerName=db
Network=app.network
Volume=pgdata.volume:/var/lib/postgresql/data:Z
Environment=POSTGRES_PASSWORD=secretpass
AutoUpdate=registry
HealthCmd=pg_isready -U postgres
HealthInterval=10s
[Service]
Restart=always
[Install]
WantedBy=default.target

# api.container
[Unit]
Description=API server
After=db.service
Requires=db.service
[Container]
Image=myregistry.io/myapp:latest
ContainerName=api
Network=app.network
PublishPort=3000:3000
Environment=DATABASE_URL=postgresql://user:pass@db:5432/myapp
AutoUpdate=registry
[Service]
Restart=always
[Install]
WantedBy=default.target

# web.container
[Unit]
Description=Web frontend
After=api.service
Requires=api.service
[Container]
Image=docker.io/library/nginx:stable
ContainerName=web
Network=app.network
PublishPort=8080:80
Volume=%h/html:/usr/share/nginx/html:Z,ro
AutoUpdate=registry
[Service]
Restart=always
[Install]
WantedBy=default.target
```

The Quadlet approach gives you full systemd dependency ordering, journald logging,
socket activation support, and auto-update — none of which are available through
Docker Compose restart policies.

---

*Source: Walsh, M. (2023). Podman in Action. Manning Publications.*
