# Podman vs Docker: Feature Comparison and Migration Guide

## Architecture Comparison

| Feature | Docker | Podman |
|---|---|---|
| Architecture | Client-server (daemon) | Daemonless (fork/exec) |
| Root requirement | Daemon runs as root by default | Rootless by default |
| Process model | All containers are children of dockerd | Each container is a direct child process (via conmon) |
| Socket | `/var/run/docker.sock` | Per-user socket at `$XDG_RUNTIME_DIR/podman/podman.sock` |
| Compose | Docker Compose (native) | `podman compose` (delegates to docker-compose or podman-compose) |
| Pod support | No native pod concept | Native pod support (`podman pod create`) |
| Swarm | Docker Swarm built-in | Not supported (use Kubernetes) |
| Build tool | BuildKit (default) | Buildah (integrated) |
| Image format | OCI / Docker v2 | OCI / Docker v2 |
| systemd integration | Limited (needs wrapper scripts) | Native (Quadlet, socket activation) |
| Kubernetes YAML | Not supported | `podman generate kube` / `podman play kube` |
| Init system in containers | tini (--init) | catatonit (--init) |
| Auto-update | Not built-in | `podman auto-update` with label-based policy |

## CLI Equivalencies

Most Docker commands work identically with Podman. The following table documents the
complete CLI mapping:

### Container Lifecycle

```bash
# Docker                              # Podman (identical)
docker run -d --name web nginx        podman run -d --name web nginx
docker stop web                       podman stop web
docker start web                      podman start web
docker restart web                    podman restart web
docker rm web                         podman rm web
docker rm -f web                      podman rm -f web
docker ps                             podman ps
docker ps -a                          podman ps -a
docker logs web                       podman logs web
docker logs -f web                    podman logs -f web
docker exec -it web bash              podman exec -it web bash
docker inspect web                    podman inspect web
docker top web                        podman top web
docker stats                          podman stats
```

### Image Management

```bash
# Docker                              # Podman (identical)
docker build -t myapp .               podman build -t myapp .
docker images                         podman images
docker pull nginx:1.25                podman pull nginx:1.25
docker push registry.io/myapp:v1     podman push registry.io/myapp:v1
docker tag myapp:latest myapp:v2     podman tag myapp:latest myapp:v2
docker rmi myapp:v1                  podman rmi myapp:v1
docker image prune                    podman image prune
docker system prune -a                podman system prune -a
```

### Volume and Network

```bash
# Docker                              # Podman (identical)
docker volume create mydata           podman volume create mydata
docker volume ls                      podman volume ls
docker volume inspect mydata          podman volume inspect mydata
docker volume rm mydata               podman volume rm mydata
docker network create mynet           podman network create mynet
docker network ls                     podman network ls
docker network inspect mynet          podman network inspect mynet
docker network rm mynet               podman network rm mynet
```

### Podman-Only Commands

```bash
# Pod management (no Docker equivalent)
podman pod create --name mypod -p 8080:80
podman run --pod mypod --name web nginx
podman run --pod mypod --name sidecar busybox
podman pod ps
podman pod stop mypod
podman pod rm mypod

# Kubernetes YAML generation
podman generate kube mypod > pod.yaml
podman play kube pod.yaml

# systemd unit generation (legacy)
podman generate systemd --new --name web

# Auto-update
podman auto-update

# User namespace utilities
podman unshare ls -la /path/to/volume
podman unshare chown 1000:1000 /path/to/volume

# System migration
podman system migrate
podman system connection add remote-host ssh://user@host/run/podman/podman.sock
```

## Migration Guide: Docker to Podman

### Step 1: Install Podman

```bash
# Fedora / RHEL / CentOS Stream
sudo dnf install podman podman-compose

# Ubuntu / Debian
sudo apt install podman podman-compose

# macOS (via Homebrew)
brew install podman
podman machine init
podman machine start

# Windows (via WSL2 or Podman Desktop)
winget install RedHat.Podman-Desktop
```

### Step 2: Set Up the Alias (Optional)

```bash
# Add to ~/.bashrc or ~/.zshrc
alias docker=podman

# Or install podman-docker package (provides /usr/bin/docker -> podman symlink)
sudo dnf install podman-docker   # Fedora/RHEL
sudo apt install podman-docker   # Debian/Ubuntu
```

### Step 3: Migrate Docker Compose Files

```yaml
# docker-compose.yml works with podman compose
# Example: web application with database
version: "3.9"
services:
  web:
    image: myapp:latest
    ports:
      - "8080:80"
    environment:
      - DATABASE_URL=postgresql://db:5432/myapp
    depends_on:
      - db
    restart: unless-stopped

  db:
    image: postgres:16
    volumes:
      - pgdata:/var/lib/postgresql/data
    environment:
      - POSTGRES_DB=myapp
      - POSTGRES_USER=appuser
      - POSTGRES_PASSWORD_FILE=/run/secrets/db_password
    secrets:
      - db_password

volumes:
  pgdata:

secrets:
  db_password:
    file: ./secrets/db_password.txt
```

```bash
# Run with Podman
podman compose up -d
podman compose ps
podman compose logs -f web
podman compose down
```

### Step 4: Migrate Dockerfiles to Containerfiles

```dockerfile
# Containerfile (also works as Dockerfile)
# Multi-stage build example
FROM node:20-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production
COPY . .
RUN npm run build

FROM node:20-alpine
RUN addgroup -g 1001 appgroup && \
    adduser -u 1001 -G appgroup -s /bin/sh -D appuser
WORKDIR /app
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/node_modules ./node_modules
COPY --from=builder /app/package.json ./
USER appuser
EXPOSE 3000
CMD ["node", "dist/server.js"]
```

```bash
# Build with Podman (identical to Docker)
podman build -t myapp:v1 .
podman build -t myapp:v1 -f Containerfile .
```

### Step 5: Migrate Volumes

```bash
# Export Docker volumes
docker run --rm -v mydata:/data -v $(pwd):/backup busybox \
  tar czf /backup/mydata.tar.gz -C /data .

# Import into Podman
podman volume create mydata
podman run --rm -v mydata:/data -v $(pwd):/backup:Z busybox \
  tar xzf /backup/mydata.tar.gz -C /data
```

### Step 6: Migrate to systemd Management

```bash
# Old way: docker with restart policies
# docker run -d --restart=always --name web nginx

# Podman way: Quadlet unit file
# ~/.config/containers/systemd/web.container
```

```ini
# ~/.config/containers/systemd/web.container
[Unit]
Description=Web server container
After=network-online.target

[Container]
Image=docker.io/library/nginx:stable
PublishPort=8080:80
Volume=web-data.volume:/usr/share/nginx/html:Z
AutoUpdate=registry

[Service]
Restart=always
TimeoutStartSec=120

[Install]
WantedBy=default.target
```

```bash
# Activate the Quadlet service
systemctl --user daemon-reload
systemctl --user enable --now web
systemctl --user status web
journalctl --user -u web -f
```

## When to Choose Podman Over Docker

### Choose Podman When:

1. **Security is a priority**: Rootless by default, no privileged daemon socket to protect.
2. **Running on RHEL/Fedora**: Podman is the default container tool, fully supported by Red Hat.
3. **systemd integration matters**: Quadlet provides first-class systemd management without wrapper scripts.
4. **Kubernetes is the target**: `podman generate kube` and `podman play kube` bridge local dev to K8s.
5. **Multi-container pods**: Native pod support for containers sharing namespaces.
6. **CI/CD pipelines**: No daemon means simpler container-in-container setups (no DinD or socket mounting).
7. **Compliance requirements**: No root daemon means smaller attack surface for auditors.

### Choose Docker When:

1. **Docker Swarm**: Podman has no Swarm equivalent (use Kubernetes instead).
2. **BuildKit features**: Advanced build features (cache mounts, SSH forwarding) work best with Docker BuildKit.
3. **Docker Desktop GUI**: Docker Desktop provides a polished GUI experience (though Podman Desktop is catching up).
4. **Existing toolchain**: CI/CD pipelines, IDE plugins, and third-party tools with tight Docker integration.
5. **Team familiarity**: If the entire team knows Docker and migration cost outweighs benefits.

### Compatibility Notes

```bash
# Test compatibility with your existing workflow
podman info                     # Check Podman version and configuration
podman system info              # Detailed system information
podman version                  # Version and API compatibility

# Verify Docker API compatibility (for tools that use the Docker API)
systemctl --user enable --now podman.socket
export DOCKER_HOST=unix://$XDG_RUNTIME_DIR/podman/podman.sock

# Test with Docker Compose using Podman socket
docker-compose --host unix://$XDG_RUNTIME_DIR/podman/podman.sock up -d
```

## Performance Comparison

| Metric | Docker | Podman |
|---|---|---|
| Container startup time | ~300ms (via daemon) | ~250ms (direct fork/exec) |
| Memory overhead (daemon) | ~50-100MB (dockerd) | 0 (no daemon) |
| Memory per container | Similar | Similar |
| Image pull speed | Similar | Similar |
| Build speed | BuildKit is faster for complex builds | Buildah comparable for standard builds |
| Rootless overhead | ~5-10% (userns) | ~5-10% (userns, default) |
| Network (rootless) | slirp4netns (if rootless Docker) | slirp4netns or pasta |

## Common Migration Issues

| Issue | Solution |
|---|---|
| `DOCKER_HOST` not set | Export `DOCKER_HOST=unix://$XDG_RUNTIME_DIR/podman/podman.sock` |
| Docker Compose version mismatch | Install `docker-compose` v2 or use `podman-compose` |
| `/var/run/docker.sock` not found | Enable Podman socket: `systemctl --user enable --now podman.socket` |
| Volume permissions in rootless | Use `:Z` SELinux label or `podman unshare chown` |
| Network isolation differences | Create explicit networks with `podman network create` |
| BuildKit syntax not recognized | Use `BUILDAH_FORMAT=docker` or standard Dockerfile syntax |
| Container restart after reboot | Use Quadlet/systemd instead of `--restart=always` (needs loginctl enable-linger) |
| Bind mount UID mismatch | Use `--userns=keep-id` to map host UID into container |

---

*Source: Walsh, M. (2023). Podman in Action. Manning Publications.*
