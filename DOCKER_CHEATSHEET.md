# 🐳 Docker Cheat Sheet

A practical, copy-paste-friendly Docker reference: containers, images, volumes, networks, Dockerfile, Compose, cleanup, debugging and handy one-liners.

> **How to read the placeholders**
> Anything inside `<angle_brackets>` must be replaced with your own value (and the brackets removed).
> Example: `docker stop <container_name>` → `docker stop my-app`
> Anything inside `[square_brackets]` is optional.

---

## 📑 Table of Contents

1. [Installation & Info](#1--installation--info)
2. [Images](#2--images)
3. [Running Containers](#3--running-containers)
4. [Container Lifecycle](#4--container-lifecycle)
5. [Inspecting, Logs & Debugging](#5--inspecting-logs--debugging)
6. [Exec, Attach & Copy Files](#6--exec-attach--copy-files)
7. [Ports](#7--ports)
8. [Volumes & Bind Mounts](#8--volumes--bind-mounts)
9. [Networks](#9--networks)
10. [Environment Variables](#10--environment-variables)
11. [Resource Limits](#11--resource-limits)
12. [Restart Policies & Health Checks](#12--restart-policies--health-checks)
13. [Dockerfile](#13--dockerfile)
14. [Building Images](#14--building-images)
15. [Registry: Login, Tag, Push, Pull](#15--registry-login-tag-push-pull)
16. [Docker Compose](#16--docker-compose)
17. [Cleanup & Disk Space](#17--cleanup--disk-space)
18. [Save, Load, Export, Import](#18--save-load-export-import)
19. [Security Tips](#19--security-tips)
20. [Contexts & Remote Docker](#20--contexts--remote-docker)
21. [Docker Swarm (Quick Look)](#21--docker-swarm-quick-look)
22. [Ready-to-Use Recipes](#22--ready-to-use-recipes)
23. [Powerful One-Liners & Combos](#23--powerful-one-liners--combos)
24. [Troubleshooting](#24--troubleshooting)

---

## 1. 📦 Installation & Info

```bash
# Version info
docker --version
docker version

# Detailed system info (storage driver, OS, containers count, etc.)
docker info

# Live events from the Docker daemon
docker events

# Quick health test
docker run hello-world

# Help for any command
docker <command> --help
docker run --help
```

### Linux: run Docker without `sudo`

```bash
sudo usermod -aG docker $USER
newgrp docker          # or log out & log in again
```

### Linux: manage the Docker service

```bash
sudo systemctl start docker
sudo systemctl stop docker
sudo systemctl restart docker
sudo systemctl enable docker      # start on boot
sudo systemctl status docker
```

---

## 2. 🖼️ Images

```bash
# List local images
docker images
docker image ls
docker images -a                     # include intermediate images

# Search Docker Hub
docker search <keyword>

# Pull an image
docker pull <image>
docker pull <image>:<tag>            # e.g. docker pull nginx:1.27-alpine
docker pull --platform linux/amd64 <image>:<tag>

# Remove an image
docker rmi <image_name_or_id>
docker image rm <image_name_or_id>
docker rmi -f <image_name_or_id>     # force

# Remove all unused (dangling) images
docker image prune
# Remove ALL images not used by any container
docker image prune -a

# Show image layers / build history
docker history <image>

# Inspect image metadata (JSON)
docker inspect <image>
docker image inspect <image> --format '{{.Config.Env}}'

# Tag an image
docker tag <source_image>:<tag> <new_name>:<new_tag>
```

**Example:**

```bash
docker pull postgres:16
docker tag postgres:16 myrepo/postgres:16-custom
```

---

## 3. ▶️ Running Containers

### Basic syntax

```bash
docker run [OPTIONS] <image>[:<tag>] [COMMAND] [ARGS]
```

### Most-used options

| Option | Meaning |
|---|---|
| `-d` | Run in background (detached) |
| `-it` | Interactive + TTY (get a shell) |
| `--name <name>` | Give the container a name |
| `-p <host_port>:<container_port>` | Publish a port |
| `-v <host_path_or_volume>:<container_path>` | Mount a volume / bind mount |
| `-e KEY=value` | Set an environment variable |
| `--env-file <file>` | Load env variables from file |
| `--network <network_name>` | Attach to a network |
| `--rm` | Auto-remove container when it exits |
| `--restart <policy>` | Restart policy |
| `-w <path>` | Working directory inside container |
| `-u <user>` | Run as a specific user |
| `--hostname <name>` | Set container hostname |
| `--entrypoint <cmd>` | Override the image entrypoint |

### Examples

```bash
# Run in background with a name and port mapping
docker run -d --name <container_name> -p <host_port>:<container_port> <image>:<tag>

# Real example: nginx on port 8080
docker run -d --name web -p 8080:80 nginx:alpine

# Interactive shell, delete container afterwards
docker run -it --rm ubuntu:24.04 bash

# Alpine (uses sh, not bash)
docker run -it --rm alpine sh

# Run a one-off command
docker run --rm <image> <command>
docker run --rm alpine echo "Hello Docker"

# Run with env vars, volume and network at once
docker run -d \
  --name <container_name> \
  -p <host_port>:<container_port> \
  -e <KEY>=<value> \
  -v <volume_name>:<container_path> \
  --network <network_name> \
  --restart unless-stopped \
  <image>:<tag>

# Mount current directory into container
docker run -it --rm -v "$(pwd)":/app -w /app node:20 bash

# Override entrypoint
docker run -it --rm --entrypoint sh <image>

# Create a container without starting it
docker create --name <container_name> <image>
```

---

## 4. 🔄 Container Lifecycle

```bash
# List running containers
docker ps
# List ALL containers (including stopped)
docker ps -a
# Only container IDs
docker ps -q
# Filter
docker ps -a --filter "status=exited"
docker ps --filter "name=<partial_name>"

# Start / Stop / Restart
docker start <container_name_or_id>
docker stop <container_name_or_id>
docker restart <container_name_or_id>

# Force kill
docker kill <container_name_or_id>

# Pause / Unpause
docker pause <container_name_or_id>
docker unpause <container_name_or_id>

# Rename
docker rename <old_name> <new_name>

# Remove container
docker rm <container_name_or_id>
docker rm -f <container_name_or_id>      # force (even if running)
docker rm -v <container_name_or_id>      # also remove anonymous volumes

# Wait until container stops (prints exit code)
docker wait <container_name_or_id>

# Update settings of a running container
docker update --restart unless-stopped <container_name>
docker update --memory 512m --cpus 1 <container_name>
```

---

## 5. 🔍 Inspecting, Logs & Debugging

```bash
# Logs
docker logs <container_name>
docker logs -f <container_name>                # follow (live)
docker logs --tail 100 <container_name>        # last 100 lines
docker logs -f --tail 50 <container_name>
docker logs --since 10m <container_name>       # last 10 minutes
docker logs --since "2025-01-01T10:00:00" <container_name>
docker logs -t <container_name>                # with timestamps

# Live resource usage (CPU/RAM/Network/IO)
docker stats
docker stats <container_name>
docker stats --no-stream                       # one snapshot

# Processes inside a container
docker top <container_name>

# Full details (JSON)
docker inspect <container_name>

# Filter specific fields using Go templates
docker inspect -f '{{.State.Status}}' <container_name>
docker inspect -f '{{.NetworkSettings.IPAddress}}' <container_name>
docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' <container_name>
docker inspect -f '{{json .Mounts}}' <container_name>
docker inspect -f '{{.State.ExitCode}}' <container_name>
docker inspect -f '{{.HostConfig.RestartPolicy.Name}}' <container_name>

# Show port mappings
docker port <container_name>

# Filesystem changes made in container (A=added, C=changed, D=deleted)
docker diff <container_name>
```

---

## 6. 💻 Exec, Attach & Copy Files

```bash
# Open a shell in a running container
docker exec -it <container_name> bash
docker exec -it <container_name> sh          # if bash doesn't exist (alpine)

# Run a single command
docker exec <container_name> <command>
docker exec <container_name> ls -la /app

# Exec as root / specific user
docker exec -it -u root <container_name> bash
docker exec -it -u <user> <container_name> bash

# Exec with env variable & working directory
docker exec -it -e <KEY>=<value> -w <path> <container_name> bash

# Attach to main process (detach with Ctrl+P then Ctrl+Q)
docker attach <container_name>

# Copy files: host → container
docker cp <host_path> <container_name>:<container_path>

# Copy files: container → host
docker cp <container_name>:<container_path> <host_path>

# Create an image from a container's current state
docker commit <container_name> <new_image>:<tag>
```

**Example:**

```bash
docker cp ./config.json web:/etc/app/config.json
docker cp web:/var/log/nginx/access.log ./access.log
```

---

## 7. 🌐 Ports

```bash
# host_port → container_port
-p <host_port>:<container_port>
-p 8080:80

# Bind to a specific host interface (localhost only = safer)
-p 127.0.0.1:<host_port>:<container_port>
-p 127.0.0.1:5432:5432

# UDP
-p <host_port>:<container_port>/udp

# Random host port
-p <container_port>
# Publish ALL exposed ports to random host ports
-P

# Range
-p 8000-8010:8000-8010

# See mapped ports
docker port <container_name>
```

---

## 8. 💾 Volumes & Bind Mounts

There are three ways to persist data:

| Type | Syntax | Use case |
|---|---|---|
| **Named volume** | `-v my_vol:/data` | Managed by Docker (best for databases) |
| **Bind mount** | `-v /host/path:/data` | Development, sharing host files |
| **tmpfs** | `--tmpfs /tmp` | In-memory, temporary data |

```bash
# Create / list / inspect / remove
docker volume create <volume_name>
docker volume ls
docker volume inspect <volume_name>
docker volume rm <volume_name>
docker volume prune                    # remove all unused volumes

# Use a named volume
docker run -d -v <volume_name>:<container_path> <image>

# Bind mount (absolute path needed)
docker run -d -v /absolute/host/path:<container_path> <image>
docker run -d -v "$(pwd)":<container_path> <image>          # Linux/macOS
docker run -d -v "${PWD}":<container_path> <image>          # PowerShell

# Read-only mount
docker run -d -v <volume_name>:<container_path>:ro <image>

# Modern --mount syntax (more explicit)
docker run -d \
  --mount type=volume,source=<volume_name>,target=<container_path> <image>
docker run -d \
  --mount type=bind,source=<host_path>,target=<container_path>,readonly <image>

# tmpfs
docker run -d --tmpfs /tmp:rw,size=64m <image>

# Share volumes between containers
docker run -d --volumes-from <source_container> <image>
```

### 📦 Backup & restore a volume

```bash
# Backup volume → tar.gz in current directory
docker run --rm \
  -v <volume_name>:/data \
  -v "$(pwd)":/backup \
  alpine tar czf /backup/<backup_name>.tar.gz -C /data .

# Restore from tar.gz into a volume
docker run --rm \
  -v <volume_name>:/data \
  -v "$(pwd)":/backup \
  alpine tar xzf /backup/<backup_name>.tar.gz -C /data
```

---

## 9. 🔌 Networks

### Network drivers

| Driver | Description |
|---|---|
| `bridge` | Default. Isolated network on a single host |
| `host` | Container shares the host network stack (Linux only) |
| `none` | No networking |
| `overlay` | Multi-host networking (Swarm) |
| `macvlan` | Container gets its own MAC/IP on the LAN |

```bash
# List / inspect
docker network ls
docker network inspect <network_name>

# Create
docker network create <network_name>
docker network create --driver bridge <network_name>
docker network create --subnet 172.20.0.0/16 --gateway 172.20.0.1 <network_name>

# Run a container on a network
docker run -d --name <container_name> --network <network_name> <image>

# Connect / disconnect a running container
docker network connect <network_name> <container_name>
docker network disconnect <network_name> <container_name>

# Give a container a network alias
docker run -d --network <network_name> --network-alias <alias> <image>

# Fixed IP (needs custom network with subnet)
docker run -d --network <network_name> --ip 172.20.0.10 <image>

# Host networking
docker run -d --network host <image>

# Remove
docker network rm <network_name>
docker network prune
```

> 💡 **On a user-defined network, containers can reach each other by container name (built-in DNS).**

**Example: app + database talking by name**

```bash
docker network create app-net

docker run -d --name db --network app-net \
  -e POSTGRES_PASSWORD=secret postgres:16

docker run -d --name api --network app-net -p 3000:3000 \
  -e DATABASE_URL=postgres://postgres:secret@db:5432/postgres \
  my-api:latest
```

### Reach the host machine from a container

```bash
# Docker Desktop (Mac/Windows): use host.docker.internal
# Linux: add this flag
docker run --add-host=host.docker.internal:host-gateway <image>
```

---

## 10. 🔧 Environment Variables

```bash
# Single variable
docker run -e <KEY>=<value> <image>

# Pass variable from your current shell
docker run -e <KEY> <image>

# From a file (KEY=value per line)
docker run --env-file <path_to_env_file> <image>

# Check variables inside a container
docker exec <container_name> env
docker exec <container_name> printenv <KEY>
```

`.env` example:

```env
POSTGRES_USER=admin
POSTGRES_PASSWORD=supersecret
POSTGRES_DB=mydb
```

---

## 11. ⚙️ Resource Limits

```bash
# Memory
docker run -d --memory 512m <image>
docker run -d --memory 1g --memory-swap 2g <image>

# CPU
docker run -d --cpus 1.5 <image>
docker run -d --cpuset-cpus "0,1" <image>
docker run -d --cpu-shares 512 <image>

# Limit PIDs (fork-bomb protection)
docker run -d --pids-limit 200 <image>

# Combine
docker run -d --name <container_name> --memory 256m --cpus 0.5 <image>

# Check limits/usage
docker stats <container_name>
```

---

## 12. ♻️ Restart Policies & Health Checks

### Restart policies

| Policy | Behavior |
|---|---|
| `no` | Never restart (default) |
| `on-failure[:max]` | Restart only on non-zero exit |
| `always` | Always restart (even after daemon restart) |
| `unless-stopped` | Like `always`, but not if manually stopped |

```bash
docker run -d --restart unless-stopped <image>
docker run -d --restart on-failure:5 <image>
docker update --restart unless-stopped <container_name>
```

### Health checks

```bash
docker run -d \
  --health-cmd="curl -f http://localhost:<port>/ || exit 1" \
  --health-interval=30s \
  --health-timeout=5s \
  --health-retries=3 \
  <image>

# Check health status
docker inspect -f '{{.State.Health.Status}}' <container_name>
docker ps --filter "health=unhealthy"
```

---

## 13. 📝 Dockerfile

### Instructions reference

| Instruction | Purpose |
|---|---|
| `FROM` | Base image |
| `WORKDIR` | Set working directory |
| `COPY` | Copy files from host into image |
| `ADD` | Like COPY, plus URL & tar extraction (prefer `COPY`) |
| `RUN` | Execute command at **build time** |
| `CMD` | Default command at **run time** (overridable) |
| `ENTRYPOINT` | Fixed executable at run time |
| `ENV` | Set environment variable |
| `ARG` | Build-time variable |
| `EXPOSE` | Document a port (does not publish it) |
| `VOLUME` | Declare a mount point |
| `USER` | Switch user |
| `LABEL` | Add metadata |
| `HEALTHCHECK` | Define container health test |
| `SHELL` | Change default shell |

### Template

```dockerfile
FROM <base_image>:<tag>

WORKDIR /app

COPY <source> <destination>
RUN <build_command>

ENV <KEY>=<value>
EXPOSE <port>

USER <non_root_user>

CMD ["<executable>", "<arg1>", "<arg2>"]
```

### Example: Node.js app

```dockerfile
FROM node:20-alpine

WORKDIR /app

# Copy manifests first → better layer caching
COPY package*.json ./
RUN npm ci --omit=dev

COPY . .

ENV NODE_ENV=production
EXPOSE 3000

USER node
CMD ["node", "server.js"]
```

### Example: Python app

```dockerfile
FROM python:3.12-slim

WORKDIR /app

ENV PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

EXPOSE 8000
CMD ["gunicorn", "-b", "0.0.0.0:8000", "app:app"]
```

### Example: Multi-stage build (small final image, Go)

```dockerfile
# ---- Build stage ----
FROM golang:1.23 AS builder
WORKDIR /src
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 go build -o /out/app .

# ---- Final stage ----
FROM gcr.io/distroless/static-debian12
COPY --from=builder /out/app /app
USER nonroot
ENTRYPOINT ["/app"]
```

### Example: Multi-stage (React/Vite → Nginx)

```dockerfile
FROM node:20-alpine AS build
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

FROM nginx:alpine
COPY --from=build /app/dist /usr/share/nginx/html
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

### `CMD` vs `ENTRYPOINT`

```dockerfile
# Exec form (recommended – proper signal handling)
CMD ["python", "app.py"]

# Shell form (runs inside /bin/sh -c)
CMD python app.py

# ENTRYPOINT + CMD together: CMD provides default args
ENTRYPOINT ["python", "app.py"]
CMD ["--port", "8000"]
```

### HEALTHCHECK in Dockerfile

```dockerfile
HEALTHCHECK --interval=30s --timeout=5s --retries=3 \
  CMD curl -f http://localhost:<port>/health || exit 1
```

### `.dockerignore` (always create one!)

```gitignore
.git
.gitignore
node_modules
npm-debug.log
Dockerfile
docker-compose*.yml
.env
*.md
__pycache__
*.pyc
.venv
.idea
.vscode
dist
build
```

---

## 14. 🏗️ Building Images

```bash
# Basic build (Dockerfile in current dir)
docker build -t <image_name>:<tag> .

# Custom Dockerfile path
docker build -f <path_to_dockerfile> -t <image_name>:<tag> <context_path>

# Build args
docker build --build-arg <ARG_NAME>=<value> -t <image_name>:<tag> .

# No cache
docker build --no-cache -t <image_name>:<tag> .

# Build a specific stage of a multi-stage Dockerfile
docker build --target <stage_name> -t <image_name>:<tag> .

# Multiple tags at once
docker build -t <image_name>:latest -t <image_name>:<version> .

# Show full build output
docker build --progress=plain -t <image_name>:<tag> .

# Pull latest base image while building
docker build --pull -t <image_name>:<tag> .
```

### Buildx (multi-platform builds)

```bash
# Create & use a builder
docker buildx create --name <builder_name> --use
docker buildx ls

# Build for multiple architectures and push
docker buildx build \
  --platform linux/amd64,linux/arm64 \
  -t <username>/<image_name>:<tag> \
  --push .

# Load a single-platform build into local Docker
docker buildx build --platform linux/arm64 -t <image_name>:<tag> --load .
```

---

## 15. 🚀 Registry: Login, Tag, Push, Pull

```bash
# Login / logout (Docker Hub)
docker login
docker login -u <username>
docker logout

# Login to another registry
docker login <registry_url>
docker login ghcr.io -u <github_username>

# Tag for a registry
docker tag <local_image>:<tag> <username>/<repo>:<tag>
docker tag <local_image>:<tag> <registry_url>/<username>/<repo>:<tag>

# Push / Pull
docker push <username>/<repo>:<tag>
docker pull <username>/<repo>:<tag>
```

**Example (Docker Hub):**

```bash
docker build -t myapp:1.0 .
docker tag myapp:1.0 johndoe/myapp:1.0
docker push johndoe/myapp:1.0
```

**Example (GitHub Container Registry):**

```bash
echo $GITHUB_TOKEN | docker login ghcr.io -u <github_username> --password-stdin
docker tag myapp:1.0 ghcr.io/<github_username>/myapp:1.0
docker push ghcr.io/<github_username>/myapp:1.0
```

### Run a private local registry

```bash
docker run -d -p 5000:5000 --restart always --name registry registry:2
docker tag <image> localhost:5000/<image>:<tag>
docker push localhost:5000/<image>:<tag>
```

---

## 16. 🧩 Docker Compose

> Modern Docker uses `docker compose` (with a space, v2). The old `docker-compose` (hyphen) is legacy.

### Commands

```bash
# Start everything (build if needed) in background
docker compose up -d
docker compose up -d --build              # force rebuild
docker compose up -d --force-recreate
docker compose up -d --scale <service_name>=3

# Stop & remove containers + networks
docker compose down
docker compose down -v                    # also remove volumes ⚠️
docker compose down --rmi all             # also remove images
docker compose down --remove-orphans

# Start / stop / restart without removing
docker compose start
docker compose stop
docker compose restart [<service_name>]

# Status
docker compose ps
docker compose ps -a
docker compose top

# Logs
docker compose logs
docker compose logs -f [<service_name>]
docker compose logs -f --tail 100 <service_name>

# Build / pull / push
docker compose build [<service_name>]
docker compose build --no-cache
docker compose pull
docker compose push

# Run a command in a running service
docker compose exec <service_name> <command>
docker compose exec <service_name> bash

# Run a one-off container for a service
docker compose run --rm <service_name> <command>

# Validate & print the final config
docker compose config

# Use a different file / project name / env file
docker compose -f <file.yml> up -d
docker compose -f docker-compose.yml -f docker-compose.prod.yml up -d
docker compose -p <project_name> up -d
docker compose --env-file <path_to_env> up -d

# Profiles
docker compose --profile <profile_name> up -d

# Watch mode (auto sync/rebuild on file change)
docker compose watch
```

### Template `docker-compose.yml`

```yaml
services:
  <service_name>:
    image: <image>:<tag>
    # or build from source:
    # build:
    #   context: .
    #   dockerfile: Dockerfile
    container_name: <container_name>
    ports:
      - "<host_port>:<container_port>"
    environment:
      - <KEY>=<value>
    env_file:
      - .env
    volumes:
      - <volume_name>:<container_path>
      - ./<host_path>:<container_path>
    networks:
      - <network_name>
    depends_on:
      - <other_service>
    restart: unless-stopped

volumes:
  <volume_name>:

networks:
  <network_name>:
```

### Full example: Web app + Postgres + Redis

```yaml
services:
  api:
    build: .
    container_name: api
    ports:
      - "3000:3000"
    environment:
      DATABASE_URL: postgres://admin:secret@db:5432/mydb
      REDIS_URL: redis://cache:6379
    depends_on:
      db:
        condition: service_healthy
      cache:
        condition: service_started
    networks:
      - backend
    restart: unless-stopped

  db:
    image: postgres:16-alpine
    container_name: db
    environment:
      POSTGRES_USER: admin
      POSTGRES_PASSWORD: secret
      POSTGRES_DB: mydb
    volumes:
      - pgdata:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U admin -d mydb"]
      interval: 10s
      timeout: 5s
      retries: 5
    networks:
      - backend
    restart: unless-stopped

  cache:
    image: redis:7-alpine
    container_name: cache
    command: ["redis-server", "--appendonly", "yes"]
    volumes:
      - redisdata:/data
    networks:
      - backend
    restart: unless-stopped

volumes:
  pgdata:
  redisdata:

networks:
  backend:
```

### Useful Compose keys

```yaml
services:
  app:
    image: myapp
    command: ["npm", "run", "dev"]          # override CMD
    entrypoint: ["/entrypoint.sh"]
    working_dir: /app
    user: "1000:1000"
    ports:
      - "127.0.0.1:8080:80"                 # localhost only
    expose:
      - "3000"                              # internal only, not published
    extra_hosts:
      - "host.docker.internal:host-gateway"
    deploy:
      resources:
        limits:
          cpus: "0.5"
          memory: 256M
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:3000/health"]
      interval: 30s
      timeout: 5s
      retries: 3
    profiles: ["debug"]                     # only start with --profile debug
    logging:
      driver: json-file
      options:
        max-size: "10m"
        max-file: "3"
```

---

## 17. 🧹 Cleanup & Disk Space

```bash
# Disk usage summary
docker system df
docker system df -v

# Remove stopped containers
docker container prune

# Remove dangling images
docker image prune

# Remove unused networks
docker network prune

# Remove unused volumes ⚠️ (data loss!)
docker volume prune

# Build cache
docker builder prune
docker builder prune -a

# Clean almost everything unused (stopped containers, unused networks, dangling images, build cache)
docker system prune
# Also remove ALL unused images
docker system prune -a
# Also remove volumes ⚠️
docker system prune -a --volumes

# Prune with filters
docker image prune -a --filter "until=168h"       # older than 7 days
docker container prune --filter "until=24h"
```

### Nuclear options 💣

```bash
# Stop ALL running containers
docker stop $(docker ps -q)

# Remove ALL containers
docker rm -f $(docker ps -aq)

# Remove ALL images
docker rmi -f $(docker images -aq)

# Remove ALL volumes
docker volume rm $(docker volume ls -q)
```

---

## 18. 📤 Save, Load, Export, Import

```bash
# Image → tar file (keeps layers, tags)
docker save -o <file>.tar <image>:<tag>
docker save <image>:<tag> | gzip > <file>.tar.gz

# tar file → image
docker load -i <file>.tar
docker load < <file>.tar.gz

# Container filesystem → tar (no history, no metadata)
docker export -o <file>.tar <container_name>

# tar → image (flat)
docker import <file>.tar <new_image>:<tag>

# Transfer an image to another server over SSH
docker save <image>:<tag> | gzip | ssh <user>@<server_ip> 'gunzip | docker load'
```

---

## 19. 🔐 Security Tips

```bash
# Run as non-root
docker run -u 1000:1000 <image>

# Read-only root filesystem (+ writable tmp)
docker run --read-only --tmpfs /tmp <image>

# Drop all capabilities, add only what you need
docker run --cap-drop ALL --cap-add NET_BIND_SERVICE <image>

# Prevent privilege escalation
docker run --security-opt no-new-privileges <image>

# Bind published ports to localhost only
docker run -p 127.0.0.1:<host_port>:<container_port> <image>

# Avoid this unless you REALLY know why:
# docker run --privileged <image>
# -v /var/run/docker.sock:/var/run/docker.sock   (gives full control of the host)

# Scan image for vulnerabilities
docker scout quickview <image>
docker scout cves <image>
docker scout recommendations <image>

# Use Docker secrets/files instead of passing passwords in plain env
docker run -v ./secret.txt:/run/secrets/db_pass:ro <image>
```

**Best practices**
- Use small, official base images (`alpine`, `slim`, `distroless`).
- Pin versions (`postgres:16.4`, not `postgres:latest`).
- Never bake secrets into images or commit `.env` files.
- Use multi-stage builds and `.dockerignore`.
- Run as a non-root user.

---

## 20. 🌍 Contexts & Remote Docker

```bash
# Manage contexts
docker context ls
docker context create <context_name> --docker "host=ssh://<user>@<server_ip>"
docker context use <context_name>
docker context use default
docker context rm <context_name>

# Run a single command against a remote host
docker --context <context_name> ps

# Or use the DOCKER_HOST variable
export DOCKER_HOST=ssh://<user>@<server_ip>
docker ps
```

---

## 21. 🐝 Docker Swarm (Quick Look)

```bash
docker swarm init [--advertise-addr <ip>]
docker swarm join --token <token> <manager_ip>:2377
docker swarm join-token worker

docker node ls

docker service create --name <service_name> -p <host_port>:<container_port> --replicas 3 <image>
docker service ls
docker service ps <service_name>
docker service logs -f <service_name>
docker service scale <service_name>=5
docker service update --image <image>:<new_tag> <service_name>
docker service rm <service_name>

# Deploy a stack from a compose file
docker stack deploy -c docker-compose.yml <stack_name>
docker stack ls
docker stack services <stack_name>
docker stack rm <stack_name>

docker swarm leave [--force]
```

---

## 22. 🍱 Ready-to-Use Recipes

### PostgreSQL

```bash
docker run -d --name postgres \
  -e POSTGRES_USER=<user> \
  -e POSTGRES_PASSWORD=<password> \
  -e POSTGRES_DB=<db_name> \
  -p 5432:5432 \
  -v pgdata:/var/lib/postgresql/data \
  postgres:16

# Connect with psql
docker exec -it postgres psql -U <user> -d <db_name>

# Dump & restore
docker exec postgres pg_dump -U <user> <db_name> > backup.sql
docker exec -i postgres psql -U <user> -d <db_name> < backup.sql
```

### MySQL / MariaDB

```bash
docker run -d --name mysql \
  -e MYSQL_ROOT_PASSWORD=<root_password> \
  -e MYSQL_DATABASE=<db_name> \
  -e MYSQL_USER=<user> \
  -e MYSQL_PASSWORD=<password> \
  -p 3306:3306 \
  -v mysqldata:/var/lib/mysql \
  mysql:8

docker exec -it mysql mysql -u root -p
docker exec mysql mysqldump -u root -p<root_password> <db_name> > backup.sql
```

### MongoDB

```bash
docker run -d --name mongo \
  -e MONGO_INITDB_ROOT_USERNAME=<user> \
  -e MONGO_INITDB_ROOT_PASSWORD=<password> \
  -p 27017:27017 \
  -v mongodata:/data/db \
  mongo:7

docker exec -it mongo mongosh -u <user> -p <password>
```

### Redis

```bash
docker run -d --name redis \
  -p 6379:6379 \
  -v redisdata:/data \
  redis:7-alpine redis-server --appendonly yes

docker exec -it redis redis-cli
```

### Nginx (serve a local folder)

```bash
docker run -d --name web \
  -p 8080:80 \
  -v "$(pwd)"/<folder>:/usr/share/nginx/html:ro \
  nginx:alpine
```

### RabbitMQ (with management UI)

```bash
docker run -d --name rabbitmq \
  -p 5672:5672 -p 15672:15672 \
  -e RABBITMQ_DEFAULT_USER=<user> \
  -e RABBITMQ_DEFAULT_PASS=<password> \
  rabbitmq:3-management
```

### Adminer (DB web UI)

```bash
docker run -d --name adminer -p 8081:8080 --network <network_name> adminer
```

### Portainer (Docker web UI)

```bash
docker volume create portainer_data
docker run -d --name portainer \
  -p 9443:9443 \
  --restart unless-stopped \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v portainer_data:/data \
  portainer/portainer-ce:latest
```

### Watchtower (auto-update containers)

```bash
docker run -d --name watchtower \
  -v /var/run/docker.sock:/var/run/docker.sock \
  containrrr/watchtower --interval 3600
```

### Temporary dev environments

```bash
# Node
docker run -it --rm -v "$(pwd)":/app -w /app -p 3000:3000 node:20 bash

# Python
docker run -it --rm -v "$(pwd)":/app -w /app python:3.12 bash

# Go
docker run -it --rm -v "$(pwd)":/app -w /app golang:1.23 bash

# Debian/Ubuntu playground
docker run -it --rm ubuntu:24.04 bash
```

### Debug networking (netshoot)

```bash
docker run -it --rm --network <network_name> nicolaka/netshoot
# Attach to another container's network namespace
docker run -it --rm --network container:<container_name> nicolaka/netshoot
```

---

## 23. ⚡ Powerful One-Liners & Combos

```bash
# Get container IP
docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' <container_name>

# List all containers with their IPs
docker ps -q | xargs -n1 docker inspect -f '{{.Name}} - {{range .NetworkSettings.Networks}}{{.IPAddress}} {{end}}'

# Pretty table of containers
docker ps --format "table {{.ID}}\t{{.Names}}\t{{.Image}}\t{{.Status}}\t{{.Ports}}"

# Pretty table of images
docker images --format "table {{.Repository}}\t{{.Tag}}\t{{.ID}}\t{{.Size}}"

# Only names of running containers
docker ps --format '{{.Names}}'

# Sort images by size
docker images --format "{{.Size}}\t{{.Repository}}:{{.Tag}}" | sort -h

# Show exited containers with exit code
docker ps -a --filter "status=exited" --format "table {{.Names}}\t{{.Status}}"

# Remove all exited containers
docker rm $(docker ps -aq -f status=exited)

# Remove dangling images
docker rmi $(docker images -f "dangling=true" -q)

# Remove all containers using a specific image
docker rm -f $(docker ps -aq --filter ancestor=<image>)

# Stop all containers of a compose project by label
docker stop $(docker ps -q --filter "label=com.docker.compose.project=<project_name>")

# Restart all containers
docker restart $(docker ps -q)

# Follow logs of ALL running containers (needs GNU parallel or a loop)
for c in $(docker ps -q); do docker logs -f --tail 20 $c & done

# Enter the latest created container
docker exec -it $(docker ps -lq) bash

# Get env vars of a container as clean list
docker inspect -f '{{range .Config.Env}}{{println .}}{{end}}' <container_name>

# Count containers by status
docker ps -a --format '{{.Status}}' | awk '{print $1}' | sort | uniq -c

# Show top 5 containers by memory
docker stats --no-stream --format "table {{.Name}}\t{{.MemUsage}}\t{{.CPUPerc}}" | sort -k2 -h | tail -5

# Find which container uses a port
docker ps --format '{{.Names}}: {{.Ports}}' | grep <port>

# Tail logs and filter for errors
docker logs -f <container_name> 2>&1 | grep -i error

# Run a command in every running container
for c in $(docker ps -q); do docker exec $c <command>; done

# Check the size of a container's writable layer
docker ps -s

# Get the full docker run command back from a running container (using runlike)
docker run --rm -v /var/run/docker.sock:/var/run/docker.sock assaflavie/runlike <container_name>

# Explore image layers interactively (dive)
docker run --rm -it -v /var/run/docker.sock:/var/run/docker.sock wagoodman/dive <image>:<tag>

# Quick HTTP file server for current directory
docker run --rm -p 8000:80 -v "$(pwd)":/usr/share/nginx/html:ro nginx:alpine

# Copy a file out of an image without running it
docker create --name tmp <image> && docker cp tmp:<path_in_image> . && docker rm tmp

# Auto-update: pull newest image and recreate with compose
docker compose pull && docker compose up -d
```

---

## 24. 🩺 Troubleshooting

| Problem | What to try |
|---|---|
| Container exits immediately | `docker logs <container_name>` and `docker inspect -f '{{.State.ExitCode}}' <container_name>` |
| Exit code `137` | Out-of-memory / killed (`docker inspect -f '{{.State.OOMKilled}}' <container_name>`) |
| Exit code `139` | Segmentation fault |
| Exit code `143` | Stopped with SIGTERM (normal for `docker stop`) |
| `port is already allocated` | Change host port, or find who uses it: `sudo lsof -i :<port>` / `ss -tulpn \| grep <port>` |
| `permission denied` on `/var/run/docker.sock` | Add user to docker group (see section 1) |
| Can't connect between containers | Make sure both are on the **same user-defined network**; use container name, not `localhost` |
| App unreachable from outside | App must listen on `0.0.0.0`, not `127.0.0.1` inside the container |
| Changes to code not showing | Rebuild: `docker compose up -d --build`, or use bind mounts in dev |
| Disk full | `docker system df` then `docker system prune -a` |
| Stale build cache issues | `docker build --no-cache .` / `docker builder prune` |
| Volume permissions error | Check UID/GID: `docker exec <container_name> id` and fix with `chown` or `user:` option |
| DNS problems inside container | `docker run --dns 8.8.8.8 <image>` or set `"dns"` in `/etc/docker/daemon.json` |
| Debug a crashing container | `docker run -it --rm --entrypoint sh <image>` |
| Debug a distroless/minimal container | `docker debug <container_name>` (Docker Desktop / Pro) or attach netshoot with `--network container:<name>` |

### Useful daemon config (`/etc/docker/daemon.json`)

```json
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  },
  "dns": ["8.8.8.8", "1.1.1.1"],
  "registry-mirrors": ["https://<mirror_url>"]
}
```

```bash
sudo systemctl restart docker    # apply changes
```

---

## 📌 Quick Reference Card

```text
IMAGES      docker pull | images | rmi | build | tag | push | history
CONTAINERS  docker run | ps | start | stop | restart | rm | logs | exec | cp | stats
VOLUMES     docker volume create | ls | inspect | rm | prune
NETWORKS    docker network create | ls | inspect | connect | disconnect | rm
COMPOSE     docker compose up -d | down | ps | logs -f | exec | build | pull
CLEANUP     docker system df | prune | image prune | container prune | volume prune
```

---

## 📚 Useful Links

- [Docker Docs](https://docs.docker.com/)
- [Dockerfile Reference](https://docs.docker.com/reference/dockerfile/)
- [Compose File Reference](https://docs.docker.com/reference/compose-file/)
- [Docker CLI Reference](https://docs.docker.com/reference/cli/docker/)
- [Docker Hub](https://hub.docker.com/)
- [Best Practices for Writing Dockerfiles](https://docs.docker.com/build/building/best-practices/)

---

⭐ If this cheat sheet helped you, consider giving the repo a star!
