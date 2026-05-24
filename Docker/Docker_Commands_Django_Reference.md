# 🐳 Complete Docker Commands Reference
### For Python & Django Backend Developers — From Zero to Production

> **How to read this guide:**
> Every command includes: **What it does**, **Why you use it**, **Syntax breakdown**, **Real Django examples**, and **Common flags**. Combined/chained commands are marked with 🔗.

---

## 📋 Table of Contents

| Section | Topic |
|---------|-------|
| [1. Image Commands](#1-image-commands) | Build, pull, push, list, remove images |
| [2. Container Lifecycle Commands](#2-container-lifecycle-commands) | Run, start, stop, restart, remove containers |
| [3. Container Inspection & Debugging](#3-container-inspection--debugging) | Logs, exec, inspect, stats, diff |
| [4. Volume Commands](#4-volume-commands) | Create, list, inspect, remove volumes |
| [5. Network Commands](#5-network-commands) | Create, connect, inspect networks |
| [6. Docker Compose Commands](#6-docker-compose-commands) | Full Compose workflow |
| [7. Registry & Distribution Commands](#7-registry--distribution-commands) | Login, push, pull, tag |
| [8. System & Cleanup Commands](#8-system--cleanup-commands) | Prune, disk usage, info |
| [9. 🔗 Combined & Chained Commands](#9--combined--chained-commands) | Power-user workflows |
| [10. Django-Specific Command Workflows](#10-django-specific-command-workflows) | Day-to-day Django + Docker |

---

## 1. Image Commands

Images are the **blueprints** your containers are built from. Think of them as Python classes — you define them once, instantiate (run) them many times.

---

### `docker build`

**What it does:** Reads your `Dockerfile` and builds a Docker image layer by layer.

**Why you use it:** Every time you write or update a `Dockerfile` for your Django project, you run this to create the image. Without building, there's nothing to run.

```bash
# ── Basic Syntax ─────────────────────────────────────────────────────────
docker build [OPTIONS] PATH | URL

# ── Most Common Usage ────────────────────────────────────────────────────

# Build from Dockerfile in current directory, tag as 'myapp'
docker build -t myapp .

# Build and tag with version
docker build -t myapp:v1.0.0 .

# Build with multiple tags at once
docker build -t myapp:v1.0.0 -t myapp:latest .

# Build from a specific Dockerfile (e.g., different file for dev/prod)
docker build -f Dockerfile.prod -t myapp:prod .
docker build -f Dockerfile.dev  -t myapp:dev  .

# Build with build-time arguments (ARG in Dockerfile)
docker build --build-arg PYTHON_VERSION=3.11 -t myapp .

# Force rebuild from scratch — bypass all cached layers
docker build --no-cache -t myapp .

# Pull the latest version of the base image before building
docker build --pull -t myapp .

# Combine: no-cache + pull (guaranteed freshest build)
docker build --no-cache --pull -t myapp .

# Build and show verbose output (useful for debugging build failures)
docker build --progress=plain -t myapp .

# Build targeting a specific stage in a multi-stage Dockerfile
docker build --target builder -t myapp:builder .
docker build --target production -t myapp:prod .

# Build with a secret (never baked into image — requires BuildKit)
DOCKER_BUILDKIT=1 docker build \
  --secret id=github_token,env=GITHUB_TOKEN \
  -t myapp .
```

**Key Flags:**
| Flag | Short | Description |
|------|-------|-------------|
| `--tag` | `-t` | Name and optional tag (`name:tag`) |
| `--file` | `-f` | Specify Dockerfile path |
| `--no-cache` | | Don't use cached layers |
| `--pull` | | Always pull newer base image |
| `--build-arg` | | Set build-time variables |
| `--target` | | Build a specific stage |
| `--progress` | | Output type: `auto`, `plain`, `tty` |

**Django Example — Production build:**
```bash
# Build your production Django image
docker build \
  -f Dockerfile.prod \
  --build-arg DJANGO_ENV=production \
  --no-cache \
  -t mycompany/django-app:v2.1.0 \
  -t mycompany/django-app:latest \
  .
```

---

### `docker images`

**What it does:** Lists all Docker images stored on your local machine.

**Why you use it:** To see what images you have, check their sizes, find old/dangling images, and verify a build succeeded.

```bash
# ── Basic Syntax ─────────────────────────────────────────────────────────
docker images [OPTIONS] [REPOSITORY[:TAG]]

# ── Most Common Usage ────────────────────────────────────────────────────

# List all images
docker images

# List images for a specific repo
docker images myapp

# List all images including dangling (untagged) ones
docker images -a

# Show only dangling images (untagged — safe to delete)
docker images -f dangling=true

# Show only image IDs (useful for scripting)
docker images -q

# Show digests (full SHA hash)
docker images --digests

# Custom format — name, tag, and size only
docker images --format "table {{.Repository}}\t{{.Tag}}\t{{.Size}}"

# Filter by label
docker images -f label=env=production

# Filter images created before/since another image
docker images -f before=myapp:v2
docker images -f since=myapp:v1
```

**Sample Output:**
```
REPOSITORY          TAG         IMAGE ID       CREATED        SIZE
myapp               v1.0.0      3f4a2b1c9d8e   2 hours ago    245MB
myapp               latest      3f4a2b1c9d8e   2 hours ago    245MB
python              3.11-slim   a1b2c3d4e5f6   3 days ago     45.7MB
postgres            15-alpine   b2c3d4e5f6a7   5 days ago     238MB
redis               7-alpine    c3d4e5f6a7b8   5 days ago     29.9MB
<none>              <none>      d4e5f6a7b8c9   1 week ago     189MB  ← dangling
```

---

### `docker pull`

**What it does:** Downloads a Docker image from a registry (Docker Hub by default) to your local machine.

**Why you use it:** To get official base images (Python, Postgres, Redis, Nginx) before building or running them, or to pull a pre-built image from your team's registry.

```bash
# ── Basic Syntax ─────────────────────────────────────────────────────────
docker pull [OPTIONS] NAME[:TAG|@DIGEST]

# ── Most Common Usage ────────────────────────────────────────────────────

# Pull latest Python slim image (always prefer slim over full)
docker pull python:3.11-slim

# Pull a specific version (best practice — pinned versions)
docker pull python:3.11.9-slim-bookworm

# Pull PostgreSQL
docker pull postgres:15-alpine

# Pull Redis
docker pull redis:7-alpine

# Pull Nginx
docker pull nginx:1.25-alpine

# Pull all tags of an image
docker pull --all-tags python

# Pull from a private registry
docker pull myregistry.company.com/myapp:v1.0

# Pull using image digest (100% guaranteed specific image)
docker pull python@sha256:abc123def456...

# Verify the pull (check image is there)
docker pull python:3.11-slim && docker images python
```

**Django workflow tip:**
```bash
# Pull all images your stack needs upfront (faster when on good internet)
docker pull python:3.11-slim
docker pull postgres:15-alpine
docker pull redis:7-alpine
docker pull nginx:1.25-alpine
```

---

### `docker rmi`

**What it does:** Removes (deletes) one or more Docker images from your local machine.

**Why you use it:** Images accumulate quickly (each build may create a new image). Regular cleanup prevents running out of disk space.

```bash
# ── Basic Syntax ─────────────────────────────────────────────────────────
docker rmi [OPTIONS] IMAGE [IMAGE...]

# ── Most Common Usage ────────────────────────────────────────────────────

# Remove a specific image by name:tag
docker rmi myapp:v1.0.0

# Remove by image ID
docker rmi 3f4a2b1c9d8e

# Remove multiple images at once
docker rmi myapp:v1.0 myapp:v1.1 python:3.10-slim

# Force remove (even if containers exist from it)
docker rmi -f myapp:old

# Remove all dangling images (no tag, not used)
docker rmi $(docker images -f dangling=true -q)

# Remove ALL unused images
docker image prune -a

# Remove all images for a specific repo
docker rmi $(docker images myapp -q)
```

> ⚠️ **Warning:** You cannot remove an image if a container (even stopped) is using it. Stop and remove containers first.

---

### `docker image` (subcommands)

**What it does:** The modern, structured way to manage images (replaces older standalone commands).

```bash
# List images
docker image ls
docker image ls -a

# Build an image
docker image build -t myapp .

# Remove images
docker image rm myapp:v1
docker image rm $(docker image ls -q)   # Remove all images

# Remove dangling images
docker image prune

# Remove ALL unused images
docker image prune -a

# Show image history (all layers and their sizes)
docker image history myapp:latest

# Inspect image metadata (JSON output)
docker image inspect python:3.11-slim

# Tag an image
docker image tag myapp:latest myapp:v2.0

# Push an image
docker image push mycompany/myapp:v2.0

# Save image to a file
docker image save -o myapp.tar myapp:latest

# Load image from a file
docker image load -i myapp.tar
```

---

### `docker history`

**What it does:** Shows the **build history** of an image — every layer, its size, and the command that created it.

**Why you use it:** To understand why your image is large, to debug caching issues, and to see what commands were run in each layer.

```bash
# Show history of your Django image
docker history myapp:latest

# Show full commands (not truncated)
docker history --no-trunc myapp:latest

# Custom format
docker history --format "table {{.CreatedBy}}\t{{.Size}}" myapp:latest
```

**Sample Output:**
```
IMAGE          CREATED        CREATED BY                                      SIZE
3f4a2b1c9d8e   2 hours ago    CMD ["gunicorn" "myproject.wsgi:application"]   0B
<missing>      2 hours ago    USER django                                     0B
<missing>      2 hours ago    COPY . .                                        1.2MB    ← your code
<missing>      2 hours ago    RUN pip install -r requirements.txt             180MB    ← LARGEST layer
<missing>      2 hours ago    COPY requirements.txt .                         1.1kB
<missing>      2 hours ago    WORKDIR /app                                    0B
<missing>      3 days ago     /bin/sh -c #(nop) CMD ["python3"]               0B
```

🐍 **Django Use:** Run this to find which pip packages are bloating your image. If 180MB seems too large, check if you have unnecessary packages in `requirements.txt`.

---

### `docker save` and `docker load`

**What it does:** `save` exports an image to a `.tar` file. `load` imports it back. Used when you can't use a registry.

```bash
# Save a single image to a tar file
docker save -o myapp.tar myapp:v1.0

# Save multiple images to one tar
docker save -o all-images.tar myapp:v1 postgres:15 redis:7-alpine

# Save and compress (smaller file)
docker save myapp:v1 | gzip > myapp.tar.gz

# Load from tar file
docker load -i myapp.tar

# Load from compressed tar
docker load < myapp.tar.gz

# Load and verify
docker load -i myapp.tar && docker images myapp
```

🐍 **Use case:** Transferring Docker images to a server that has no internet access (air-gapped environment) or where Docker Hub is blocked.

---

## 2. Container Lifecycle Commands

Containers are **running instances of images** — like instantiated Python objects. This section covers their full lifecycle.

---

### `docker run`

**What it does:** **Creates a brand new container** from an image and immediately starts it. This is the most important and feature-rich Docker command.

**Why you use it:** Every time you want to start a container — for testing, development, running one-off commands (like Django migrations), or production deployment.

```bash
# ── Basic Syntax ─────────────────────────────────────────────────────────
docker run [OPTIONS] IMAGE [COMMAND] [ARG...]

# ── Simple Usage ─────────────────────────────────────────────────────────

# Run Python interactively
docker run -it python:3.11-slim python

# Run a command and remove container immediately after
docker run --rm python:3.11-slim python -c "import django; print(django.__version__)"

# ── Port Mapping ─────────────────────────────────────────────────────────

# Map container port 8000 to host port 8000
docker run -p 8000:8000 myapp

# Map to a different host port
docker run -p 80:8000 myapp      # host:80 → container:8000

# Bind to localhost only (not accessible from network)
docker run -p 127.0.0.1:8000:8000 myapp

# ── Detached Mode (Background) ────────────────────────────────────────────

# Run in background
docker run -d myapp

# Run in background with name
docker run -d --name django_web myapp

# ── Volumes & Bind Mounts ─────────────────────────────────────────────────

# Bind mount current directory (for development — live code reload)
docker run -v $(pwd):/app myapp

# Named volume (for persistent data like database)
docker run -v postgres_data:/var/lib/postgresql/data postgres:15

# Read-only bind mount
docker run -v $(pwd)/config:/app/config:ro myapp

# ── Environment Variables ─────────────────────────────────────────────────

# Set a single environment variable
docker run -e DEBUG=True myapp

# Set multiple environment variables
docker run -e DEBUG=True -e SECRET_KEY=mysecret myapp

# Load from an .env file
docker run --env-file .env myapp

# ── Naming & Cleanup ──────────────────────────────────────────────────────

# Name the container
docker run --name my_django_app myapp

# Auto-remove when stopped (great for one-off commands)
docker run --rm myapp python manage.py migrate

# ── Resource Limits ───────────────────────────────────────────────────────

# Limit memory
docker run --memory 512m myapp

# Limit CPU
docker run --cpus 1.0 myapp

# Both limits
docker run --memory 1g --cpus 2.0 myapp

# ── Networking ────────────────────────────────────────────────────────────

# Connect to a specific network
docker run --network mynetwork myapp

# Use host network (container shares host's network)
docker run --network host myapp

# ── Security ──────────────────────────────────────────────────────────────

# Run as specific user
docker run --user 1001:1001 myapp

# Read-only filesystem (extra security)
docker run --read-only --tmpfs /tmp myapp

# Drop all capabilities (minimum privileges)
docker run --cap-drop ALL myapp

# ── Restart Policies ──────────────────────────────────────────────────────

# Always restart (production services)
docker run -d --restart unless-stopped myapp

# Restart on failure only
docker run -d --restart on-failure:5 myapp

# ── Init process ──────────────────────────────────────────────────────────

# Use Docker's built-in init (handles signals + zombie processes properly)
docker run --init myapp

# ── FULL PRODUCTION-LIKE COMMAND ─────────────────────────────────────────

docker run \
  -d \                                          # Background
  --name django_web \                           # Name it
  --restart unless-stopped \                    # Auto-restart
  -p 8000:8000 \                                # Expose port
  -e DEBUG=False \                              # Env vars
  --env-file .env.production \                  # Env file
  -v static_files:/app/staticfiles \            # Named volume
  --memory 1g \                                 # Memory limit
  --cpus 1.5 \                                  # CPU limit
  --init \                                      # Init process
  --user 1001:1001 \                            # Non-root user
  --network django_network \                    # Custom network
  mycompany/django-app:v2.1.0                   # Image
```

---

### `docker start` / `docker stop` / `docker restart`

**What it does:**
- `start` — Start one or more **stopped** containers
- `stop` — Gracefully stop a running container (sends SIGTERM, waits, then SIGKILL)
- `restart` — Stop + start a container

**Why you use it:** Managing the lifecycle of existing containers without recreating them.

```bash
# ── docker start ─────────────────────────────────────────────────────────

# Start a stopped container
docker start django_web

# Start multiple containers
docker start django_web django_db django_redis

# Start and attach to output
docker start -a django_web

# Start interactively
docker start -ai django_web

# ── docker stop ──────────────────────────────────────────────────────────

# Gracefully stop a container (waits up to 10s for cleanup)
docker stop django_web

# Stop with a custom timeout (give app 30 seconds to shut down)
docker stop --time 30 django_web

# Stop multiple containers
docker stop django_web django_db django_redis

# Stop ALL running containers
docker stop $(docker ps -q)

# ── docker restart ────────────────────────────────────────────────────────

# Restart a container
docker restart django_web

# Restart with a custom stop timeout
docker restart --time 30 django_web

# Restart multiple containers
docker restart django_web django_celery
```

🐍 **Django Example:**
```bash
# After deploying a new version, restart the web container
docker stop django_web
docker rm django_web
docker run -d --name django_web myapp:v2.0  # Start fresh with new image

# OR: just restart if using the same image
docker restart django_web
```

---

### `docker kill`

**What it does:** Sends a signal to a container — by default `SIGKILL` (immediate termination), but you can send any Unix signal.

**Why you use it:** When `docker stop` is too slow or a container is unresponsive.

```bash
# Immediately kill a container (SIGKILL — no cleanup)
docker kill django_web

# Send a specific signal
docker kill --signal SIGTERM django_web     # Graceful shutdown request
docker kill --signal SIGHUP django_web      # Reload config (some apps support this)
docker kill --signal SIGUSR1 django_web     # Custom signal

# Kill all running containers
docker kill $(docker ps -q)
```

> ⚠️ **Prefer `docker stop` over `docker kill`** in production. `kill` is like pulling the power plug — open database connections, in-flight requests, and file writes may be corrupted.

---

### `docker rm`

**What it does:** Removes (deletes) stopped containers. Does NOT remove the image.

**Why you use it:** Stopped containers still take up disk space and namespace. Regular cleanup keeps things tidy.

```bash
# ── Basic Syntax ─────────────────────────────────────────────────────────

# Remove a stopped container
docker rm django_web

# Remove multiple containers
docker rm django_web django_db old_container

# Force remove a RUNNING container (same as stop + rm)
docker rm -f django_web

# Also remove associated anonymous volumes
docker rm -v django_web

# ── Bulk Removal ──────────────────────────────────────────────────────────

# Remove all stopped containers
docker container prune

# Remove all stopped containers (alternative)
docker rm $(docker ps -a -q --filter status=exited)

# Remove all containers — both running and stopped (⚠️ destructive)
docker rm -f $(docker ps -a -q)
```

---

### `docker create`

**What it does:** Creates a container from an image **without starting it**. The container exists but isn't running.

**Why you use it:** Pre-creating containers for later, or in scripts where you want to separate creation from starting.

```bash
# Create a container (don't start it)
docker create --name django_web -p 8000:8000 myapp

# Start it later
docker start django_web

# Create with all options
docker create \
  --name django_web \
  -p 8000:8000 \
  -e DEBUG=False \
  --restart unless-stopped \
  myapp:latest
```

---

### `docker pause` / `docker unpause`

**What it does:** Freezes (`pause`) or resumes (`unpause`) all processes in a container without stopping it.

**Why you use it:** Temporarily freeze a container for backups or resource debugging, without losing its state.

```bash
# Pause a running container (freezes all its processes)
docker pause django_web

# Unpause (resume)
docker unpause django_web

# Pause all running containers
docker pause $(docker ps -q)
```

🐍 **Use case:** Before taking a PostgreSQL backup — pause the container, snapshot the volume, unpause. Ensures a consistent backup.

---

### `docker rename`

**What it does:** Renames an existing container.

```bash
# Rename container
docker rename old_name new_name

# Example
docker rename my_app_1 django_web
```

---

### `docker wait`

**What it does:** Blocks until a container stops, then prints its exit code.

**Why you use it:** In scripts, to wait for a container to finish before proceeding.

```bash
# Wait for a migration to finish before starting the app
docker run --name migrations myapp python manage.py migrate
docker wait migrations        # Blocks until migrations container exits
echo "Exit code: $?"          # 0 = success, non-zero = failure
docker rm migrations
```

---

### `docker port`

**What it does:** Lists the port mappings for a container.

```bash
# Show all port mappings
docker port django_web

# Show mapping for a specific container port
docker port django_web 8000

# Output: 0.0.0.0:8000
```

---

## 3. Container Inspection & Debugging

---

### `docker logs`

**What it does:** Fetches the stdout/stderr logs from a container — everything your Django app prints goes here.

**Why you use it:** This is your primary debugging tool. Every `print()`, `logging.error()`, Django request log, and Gunicorn access log appears here.

```bash
# ── Basic Syntax ─────────────────────────────────────────────────────────
docker logs [OPTIONS] CONTAINER

# ── Most Common Usage ────────────────────────────────────────────────────

# View all logs
docker logs django_web

# Follow logs in real time (Ctrl+C to stop)
docker logs -f django_web

# Show last N lines only
docker logs --tail 50 django_web
docker logs --tail 100 django_web

# Show logs with timestamps
docker logs -t django_web

# Show logs since a time/duration
docker logs --since 30m django_web          # Last 30 minutes
docker logs --since 2h django_web           # Last 2 hours
docker logs --since "2024-01-15T10:00:00" django_web  # Since specific time

# Show logs until a time
docker logs --until "2024-01-15T11:00:00" django_web

# ── Power Combinations ────────────────────────────────────────────────────

# Follow last 100 lines with timestamps
docker logs -f -t --tail 100 django_web

# Follow and grep for errors only
docker logs -f django_web 2>&1 | grep -i error

# Follow and grep for a specific user's requests
docker logs -f django_web 2>&1 | grep "admin@example.com"

# Save logs to file
docker logs django_web > django_logs.txt 2>&1

# Save and follow simultaneously
docker logs -f django_web 2>&1 | tee django_logs.txt
```

🐍 **Django Tip:** Make sure `PYTHONUNBUFFERED=1` is set in your Dockerfile or environment, otherwise Django's output may not appear in logs immediately.

---

### `docker exec`

**What it does:** Runs a command **inside an already running container**. Unlike `docker run` (which creates a new container), `docker exec` enters an existing one.

**Why you use it:** This is how you run Django management commands, open a shell for debugging, inspect the filesystem, or run tests inside a running container.

```bash
# ── Basic Syntax ─────────────────────────────────────────────────────────
docker exec [OPTIONS] CONTAINER COMMAND [ARG...]

# ── Interactive Shell ─────────────────────────────────────────────────────

# Open a bash shell inside the container
docker exec -it django_web bash

# Open a sh shell (for alpine-based images that don't have bash)
docker exec -it django_web sh

# Open Python interpreter
docker exec -it django_web python

# ── Django Management Commands ────────────────────────────────────────────

# Run database migrations
docker exec django_web python manage.py migrate

# Run migrations and show output
docker exec -it django_web python manage.py migrate --verbosity=2

# Open Django shell
docker exec -it django_web python manage.py shell

# Open Django shell_plus (if django-extensions installed)
docker exec -it django_web python manage.py shell_plus

# Create superuser
docker exec -it django_web python manage.py createsuperuser

# Check Django deployment readiness
docker exec django_web python manage.py check --deploy

# Run Django tests
docker exec -it django_web python manage.py test

# Run specific test
docker exec -it django_web python manage.py test myapp.tests.test_views

# Collect static files
docker exec django_web python manage.py collectstatic --noinput

# Show applied migrations
docker exec django_web python manage.py showmigrations

# Show pending SQL for a migration
docker exec -it django_web python manage.py sqlmigrate myapp 0001

# Create a data dump
docker exec django_web python manage.py dumpdata --indent=2 > backup.json

# Load data from fixture
docker exec django_web python manage.py loaddata fixtures/initial_data.json

# ── System Inspection ─────────────────────────────────────────────────────

# Check installed Python packages
docker exec django_web pip list
docker exec django_web pip show django

# Check environment variables
docker exec django_web env

# Check disk space inside container
docker exec django_web df -h

# Check running processes inside container
docker exec django_web ps aux

# Check network connectivity from inside container
docker exec django_web ping db             # Can Django reach the DB?
docker exec django_web curl http://redis:6379  # Can it reach Redis?

# ── Running as Different User ──────────────────────────────────────────────

# Run as root (for troubleshooting permission issues)
docker exec -u root -it django_web bash

# Run as specific UID
docker exec -u 1001 django_web python manage.py migrate

# ── Set Environment Variables for exec ────────────────────────────────────

# Run with an additional env var
docker exec -e DJANGO_DEBUG=True django_web python manage.py shell
```

---

### `docker inspect`

**What it does:** Returns detailed **JSON metadata** about any Docker object — containers, images, volumes, networks.

**Why you use it:** For deep debugging — finding a container's IP, checking its config, seeing its mounts, understanding its network setup.

```bash
# ── Basic Syntax ─────────────────────────────────────────────────────────
docker inspect [OPTIONS] NAME|ID [NAME|ID...]

# ── Container Inspection ──────────────────────────────────────────────────

# Full JSON dump of container
docker inspect django_web

# Get container's IP address
docker inspect --format='{{.NetworkSettings.IPAddress}}' django_web

# Get container's IP on a specific network
docker inspect --format='{{.NetworkSettings.Networks.django_network.IPAddress}}' django_web

# Get container status
docker inspect --format='{{.State.Status}}' django_web

# Get container's environment variables
docker inspect --format='{{range .Config.Env}}{{println .}}{{end}}' django_web

# Get container's restart policy
docker inspect --format='{{.HostConfig.RestartPolicy.Name}}' django_web

# Get container's mount points
docker inspect --format='{{range .Mounts}}{{.Source}} → {{.Destination}}{{println}}{{end}}' django_web

# Get image used by container
docker inspect --format='{{.Config.Image}}' django_web

# Get container's start time
docker inspect --format='{{.State.StartedAt}}' django_web

# ── Image Inspection ──────────────────────────────────────────────────────

# Inspect an image
docker inspect python:3.11-slim

# Get image's environment variables
docker inspect --format='{{range .Config.Env}}{{println .}}{{end}}' python:3.11-slim

# Get exposed ports
docker inspect --format='{{.Config.ExposedPorts}}' myapp

# Get image layers
docker inspect --format='{{.RootFS.Layers}}' myapp

# ── Volume Inspection ─────────────────────────────────────────────────────

docker inspect postgres_data_volume

# Find where a volume is stored on host
docker inspect --format='{{.Mountpoint}}' postgres_data_volume
```

---

### `docker ps`

**What it does:** Lists containers. By default, shows only **running** containers.

**Why you use it:** The most frequently used command for checking what's running, how long it's been up, and what ports are mapped.

```bash
# ── Basic Syntax ─────────────────────────────────────────────────────────
docker ps [OPTIONS]

# ── Most Common Usage ────────────────────────────────────────────────────

# List running containers
docker ps

# List ALL containers (running + stopped)
docker ps -a
docker ps --all

# Show only container IDs (for piping into other commands)
docker ps -q
docker ps -a -q

# Show last N containers created (running or not)
docker ps -n 5

# Show the last created container
docker ps -l

# ── Filtering ─────────────────────────────────────────────────────────────

# Filter by status
docker ps -f status=running
docker ps -f status=exited
docker ps -f status=paused

# Filter by name
docker ps -f name=django

# Filter by image
docker ps -f ancestor=myapp:latest

# Filter by label
docker ps -f label=env=production

# Filter by network
docker ps -f network=django_network

# ── Custom Formatting ─────────────────────────────────────────────────────

# Clean tabular output
docker ps --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}\t{{.Image}}"

# Just names and status
docker ps --format "{{.Names}}: {{.Status}}"

# JSON output
docker ps --format "{{json .}}"
```

**Understanding the Output:**
```
CONTAINER ID   IMAGE           COMMAND                  CREATED        STATUS         PORTS                    NAMES
3f4a2b1c9d8e   myapp:v1        "gunicorn myproject…"   2 hours ago    Up 2 hours     0.0.0.0:8000->8000/tcp   django_web
b2c3d4e5f6a7   postgres:15     "docker-entrypoint.s…"  2 hours ago    Up 2 hours     5432/tcp                 django_db
```

---

### `docker stats`

**What it does:** Shows **real-time resource usage** statistics for running containers — like a live dashboard.

**Why you use it:** Monitor memory usage (critical for Django — OOM kills are common), CPU spikes from Celery tasks, and network activity.

```bash
# ── Basic Syntax ─────────────────────────────────────────────────────────
docker stats [OPTIONS] [CONTAINER...]

# Live stats for ALL running containers
docker stats

# Stats for specific containers
docker stats django_web django_db django_celery

# Single snapshot (no live update)
docker stats --no-stream

# No truncation of container names
docker stats --no-trunc

# Custom format
docker stats --format "table {{.Name}}\t{{.CPUPerc}}\t{{.MemUsage}}\t{{.MemPerc}}\t{{.NetIO}}"

# Stats snapshot for all containers, formatted
docker stats --no-stream --format "table {{.Name}}\t{{.CPUPerc}}\t{{.MemUsage}}"
```

**Sample Output:**
```
NAME            CPU %     MEM USAGE / LIMIT     MEM %     NET I/O
django_web      0.45%     142MiB / 1GiB         13.87%    5.2MB / 1.1MB
django_db       0.12%     89MiB / 512MiB        17.38%    1.2MB / 800kB
django_celery   1.23%     256MiB / 512MiB       50.00%    200kB / 50kB
django_redis    0.05%     12MiB / 256MiB        4.69%     400kB / 300kB
```

🐍 **Django Tip:** If `django_web` memory keeps climbing, you likely have a memory leak in your views. Use this to catch it early.

---

### `docker top`

**What it does:** Shows the running processes **inside** a container (like the `top` command but for a container).

```bash
# Show processes in a container
docker top django_web

# Show processes with specific ps options
docker top django_web aux

# Sample output:
# PID    USER   COMMAND
# 1234   django  /usr/local/bin/python gunicorn myproject.wsgi
# 1235   django  /usr/local/bin/python gunicorn myproject.wsgi [worker 1]
# 1236   django  /usr/local/bin/python gunicorn myproject.wsgi [worker 2]
```

---

### `docker diff`

**What it does:** Shows files that have been **Added (A)**, **Changed (C)**, or **Deleted (D)** in a container's filesystem compared to its base image.

**Why you use it:** Security auditing, debugging what a running container has changed, and understanding what `docker commit` would capture.

```bash
docker diff django_web

# Sample output:
# C /app
# A /app/staticfiles          ← collectstatic added this
# C /app/myproject/__pycache__  ← Python created bytecode cache
# A /tmp/django_cache          ← Django created temp files
```

---

### `docker cp`

**What it does:** Copies files/directories between a container and the local filesystem — in both directions.

**Why you use it:** Extract logs, configs, or database dumps from a container. Push configs or fixtures into a running container.

```bash
# ── Copy FROM container TO host ────────────────────────────────────────

# Copy a file
docker cp django_web:/app/logs/django.log ./django.log

# Copy a directory
docker cp django_web:/app/staticfiles ./local_staticfiles/

# Copy database dump
docker cp django_db:/var/lib/postgresql/backup.sql ./backup.sql

# ── Copy FROM host TO container ────────────────────────────────────────

# Copy a config file into container
docker cp ./local_settings.py django_web:/app/myproject/local_settings.py

# Copy a fixture into container
docker cp ./fixtures/test_data.json django_web:/app/fixtures/

# ── Common Django Workflows ────────────────────────────────────────────

# Backup Django data
docker exec django_web python manage.py dumpdata --indent=2 > /tmp/backup.json
docker cp django_web:/tmp/backup.json ./backup_$(date +%Y%m%d).json

# Copy media files for backup
docker cp django_web:/app/mediafiles ./media_backup/
```

---

### `docker attach`

**What it does:** Attaches your terminal to a running container's stdin/stdout/stderr — like stepping into the main process's console.

**Why you use it:** Primarily for debugging with `pdb` (Python debugger) inside Docker.

```bash
# Attach to a running container
docker attach django_web

# Detach without stopping: Ctrl+P then Ctrl+Q
# Stop the container: Ctrl+C
```

🐍 **Django pdb debugging:**
```yaml
# docker-compose.yml — Required for pdb to work
services:
  web:
    stdin_open: true   # Equivalent to -i
    tty: true          # Equivalent to -t
```
```python
# In your view
import pdb; pdb.set_trace()  # Breakpoint
```
```bash
docker attach django_web    # Now interact with pdb
```

---

### `docker events`

**What it does:** Shows a real-time stream of Docker events — container starts, stops, image pulls, network connections, etc.

**Why you use it:** Monitoring and auditing what's happening in your Docker environment in real-time.

```bash
# Watch all events
docker events

# Filter by object type
docker events --filter type=container
docker events --filter type=image
docker events --filter type=network
docker events --filter type=volume

# Filter by event action
docker events --filter event=start
docker events --filter event=stop
docker events --filter event=die

# Filter by container name
docker events --filter container=django_web

# Events from the last hour
docker events --since 1h

# Events in a specific time range
docker events --since "2024-01-15T00:00:00" --until "2024-01-15T23:59:59"

# JSON format output
docker events --format '{"time":"{{.Time}}","action":"{{.Action}}","actor":"{{.Actor.Attributes.name}}"}'
```

---

## 4. Volume Commands

Volumes provide **persistent storage** for containers. They're the solution to containers being ephemeral — your database data, media files, and logs live here.

---

### `docker volume create`

**What it does:** Creates a named volume that can be attached to containers.

```bash
# Create a named volume
docker volume create postgres_data

# Create with a specific driver
docker volume create --driver local postgres_data

# Create with labels
docker volume create --label env=production postgres_data

# Create with driver options (e.g., mount an NFS share)
docker volume create \
  --driver local \
  --opt type=nfs \
  --opt o=addr=192.168.1.100,rw \
  --opt device=:/exports/data \
  nfs_volume
```

---

### `docker volume ls`

**What it does:** Lists all Docker volumes.

```bash
# List all volumes
docker volume ls

# List only volume names
docker volume ls -q

# Filter by driver
docker volume ls -f driver=local

# Filter by label
docker volume ls -f label=env=production

# Filter dangling volumes (not attached to any container)
docker volume ls -f dangling=true
```

---

### `docker volume inspect`

**What it does:** Shows detailed information about a volume — where data is stored on the host, which containers use it, etc.

```bash
# Inspect a volume
docker volume inspect postgres_data

# Get the mount path on host (where data is actually stored)
docker volume inspect --format '{{.Mountpoint}}' postgres_data

# Output: /var/lib/docker/volumes/postgres_data/_data
```

---

### `docker volume rm` and `docker volume prune`

**What it does:** Removes volumes.

```bash
# Remove a specific volume (must not be in use)
docker volume rm postgres_data

# Remove multiple volumes
docker volume rm postgres_data redis_data static_files

# Remove ALL unused volumes (⚠️ DESTROYS DATA)
docker volume prune

# Remove unused volumes with a specific label
docker volume prune --filter label=env=test

# Force remove without confirmation
docker volume prune -f
```

> ⚠️ **CRITICAL WARNING:** Removing volumes destroys all data stored in them. Never run `docker volume prune` on a production machine without verifying you have backups.

---

### Backup and Restore Volumes

**Why you need this:** Named volumes store your PostgreSQL data, media uploads, etc. You need to back them up.

```bash
# ── Backup a Volume ─────────────────────────────────────────────────────

# Create a tar backup of a volume
docker run --rm \
  -v postgres_data:/source:ro \
  -v $(pwd):/backup \
  alpine \
  tar czf /backup/postgres_backup_$(date +%Y%m%d).tar.gz -C /source .

# ── Restore a Volume ─────────────────────────────────────────────────────

# Restore from backup
docker run --rm \
  -v postgres_data:/target \
  -v $(pwd):/backup \
  alpine \
  sh -c "cd /target && tar xzf /backup/postgres_backup_20240115.tar.gz"

# ── Copy data between volumes ─────────────────────────────────────────────

docker run --rm \
  -v old_volume:/source:ro \
  -v new_volume:/dest \
  alpine \
  cp -av /source/. /dest/
```

---

## 5. Network Commands

Docker networks allow containers to communicate securely and in isolation.

---

### `docker network create`

**What it does:** Creates a custom Docker network.

```bash
# Create a basic bridge network
docker network create django_network

# Create with a subnet
docker network create --subnet 172.20.0.0/16 django_network

# Create with a specific gateway
docker network create \
  --subnet 172.20.0.0/16 \
  --gateway 172.20.0.1 \
  django_network

# Create with labels
docker network create --label env=production django_network

# Create an overlay network (for Swarm)
docker network create --driver overlay swarm_network
```

---

### `docker network ls`

**What it does:** Lists all Docker networks.

```bash
# List all networks
docker network ls

# Filter by driver
docker network ls -f driver=bridge

# Show only IDs
docker network ls -q

# Custom format
docker network ls --format "table {{.Name}}\t{{.Driver}}\t{{.Scope}}"
```

**Default networks:**
```
NETWORK ID     NAME      DRIVER    SCOPE
abc123def456   bridge    bridge    local   ← default for standalone containers
def456ghi789   host      host      local   ← shares host network
ghi789jkl012   none      null      local   ← no networking
```

---

### `docker network inspect`

**What it does:** Shows detailed information about a network — which containers are connected, what IPs they have, subnet configuration.

```bash
# Inspect a network
docker network inspect django_network

# Get all container names and IPs on a network
docker network inspect \
  --format='{{range .Containers}}{{.Name}}: {{.IPv4Address}}{{println}}{{end}}' \
  django_network

# Output:
# django_web: 172.20.0.2/16
# django_db: 172.20.0.3/16
# django_redis: 172.20.0.4/16
```

---

### `docker network connect` / `docker network disconnect`

**What it does:** Connect or disconnect a running container to/from a network without restarting it.

```bash
# Connect a running container to a network
docker network connect django_network django_web

# Connect with a specific IP
docker network connect --ip 172.20.0.10 django_network django_web

# Connect with an alias (container accessible by alias name on that network)
docker network connect --alias web django_network django_web

# Disconnect
docker network disconnect django_network django_web

# Force disconnect
docker network disconnect -f django_network django_web
```

---

### `docker network rm` / `docker network prune`

```bash
# Remove a network (no containers can be connected)
docker network rm django_network

# Remove multiple networks
docker network rm django_network test_network

# Remove all unused networks
docker network prune

# Force prune
docker network prune -f
```

---

## 6. Docker Compose Commands

Docker Compose manages **multi-container applications**. For Django developers, this is your most-used tool day-to-day.

---

### `docker compose up`

**What it does:** Creates and starts all services defined in `docker-compose.yml`. If images don't exist, it builds them first.

```bash
# ── Basic Syntax ─────────────────────────────────────────────────────────
docker compose up [OPTIONS] [SERVICE...]

# ── Most Common Usage ────────────────────────────────────────────────────

# Start all services (foreground — see all logs)
docker compose up

# Start all services in background (detached)
docker compose up -d

# Start and rebuild images (use after Dockerfile or requirements.txt changes)
docker compose up --build

# Rebuild and start in background
docker compose up -d --build

# Start specific services only
docker compose up -d db redis        # Only start DB and Redis

# Force recreate containers even if config hasn't changed
docker compose up --force-recreate

# Don't start linked/dependent services
docker compose up --no-deps web

# Scale a service to multiple replicas
docker compose up --scale web=3 --scale celery=2

# Remove orphaned containers (containers for services no longer in compose file)
docker compose up -d --remove-orphans

# Pull latest images before starting
docker compose up --pull always

# Timeout for service startup
docker compose up -d --timeout 120
```

---

### `docker compose down`

**What it does:** Stops and **removes** containers and networks. Does NOT remove volumes by default.

```bash
# Stop and remove containers + networks
docker compose down

# Also remove named volumes (⚠️ deletes database data)
docker compose down -v
docker compose down --volumes

# Also remove images built by compose
docker compose down --rmi all
docker compose down --rmi local   # Only locally built images

# Remove everything (volumes + images)
docker compose down -v --rmi all

# Custom timeout before force kill
docker compose down -t 30
```

---

### `docker compose start` / `docker compose stop` / `docker compose restart`

```bash
# Stop containers without removing them
docker compose stop

# Stop specific services
docker compose stop web celery

# Start previously stopped containers
docker compose start

# Start specific services
docker compose start web celery

# Restart all services
docker compose restart

# Restart specific services
docker compose restart web

# Restart with a timeout
docker compose restart -t 30 web
```

---

### `docker compose run`

**What it does:** Runs a **one-off command** in a **new container** for a service. The container is separate from any running service containers.

**Why you use it:** The #1 way to run Django management commands in Docker.

```bash
# ── Basic Syntax ─────────────────────────────────────────────────────────
docker compose run [OPTIONS] SERVICE [COMMAND] [ARG...]

# ── Django Management Commands ────────────────────────────────────────────

# Run database migrations
docker compose run --rm web python manage.py migrate

# Create superuser
docker compose run --rm web python manage.py createsuperuser

# Open Django shell
docker compose run --rm web python manage.py shell

# Run tests
docker compose run --rm web python manage.py test

# Run pytest
docker compose run --rm web pytest -v

# Collect static files
docker compose run --rm web python manage.py collectstatic --noinput

# Create a new Django app
docker compose run --rm web python manage.py startapp myapp

# Run a custom management command
docker compose run --rm web python manage.py send_weekly_email

# Load fixtures
docker compose run --rm web python manage.py loaddata fixtures/initial_data.json

# Check Django configuration
docker compose run --rm web python manage.py check

# Generate migration files
docker compose run --rm web python manage.py makemigrations

# ── Options ───────────────────────────────────────────────────────────────

# --rm: Remove container after command finishes (always use this)
docker compose run --rm web python manage.py migrate

# -e: Set environment variable
docker compose run --rm -e DJANGO_SETTINGS_MODULE=myproject.settings.test web pytest

# --no-deps: Don't start linked services (use when DB is already running)
docker compose run --rm --no-deps web python -c "import django; print('OK')"

# -u: Run as specific user
docker compose run --rm -u root web bash

# -v: Override volume
docker compose run --rm -v $(pwd)/test_fixtures:/app/fixtures web python manage.py loaddata fixtures/test.json

# -p: Override port mapping
docker compose run --rm -p 8001:8000 web python manage.py runserver 0.0.0.0:8000

# --service-ports: Use the service's defined port mappings
docker compose run --service-ports web python manage.py runserver 0.0.0.0:8000
```

---

### `docker compose exec`

**What it does:** Runs a command in an **already running** service container. Unlike `run`, it doesn't create a new container.

```bash
# ── Basic Syntax ─────────────────────────────────────────────────────────
docker compose exec [OPTIONS] SERVICE COMMAND

# ── Usage ─────────────────────────────────────────────────────────────────

# Open shell in running web container
docker compose exec web bash

# Run Django shell in running container
docker compose exec web python manage.py shell

# Run migration in running container
docker compose exec web python manage.py migrate

# Check environment variables
docker compose exec web env | grep DJANGO

# Access PostgreSQL CLI
docker compose exec db psql -U django -d django_db

# Access Redis CLI
docker compose exec redis redis-cli

# Run as root
docker compose exec -u root web bash

# Run in specific container when scaled (index starts at 1)
docker compose exec --index=2 web bash
```

**`run` vs `exec` — when to use which:**
```bash
# Use 'run' when:
# - Running one-off commands (migrations, shell, tests)
# - Services might not be running
# - You want a fresh container
docker compose run --rm web python manage.py migrate   ✅

# Use 'exec' when:
# - Services are already running
# - You want to inspect/debug a running container
# - Faster (no container creation overhead)
docker compose exec web python manage.py shell         ✅
```

---

### `docker compose logs`

**What it does:** Shows logs from services defined in your Compose file.

```bash
# View logs for all services
docker compose logs

# Follow all logs in real-time
docker compose logs -f

# Logs for specific service
docker compose logs web
docker compose logs -f web

# Last 100 lines
docker compose logs --tail 100 web

# With timestamps
docker compose logs -t web

# Multiple services
docker compose logs -f web celery

# Since a duration
docker compose logs --since 1h web
```

---

### `docker compose ps`

**What it does:** Lists containers for the current Compose project.

```bash
# List all service containers
docker compose ps

# List only container IDs
docker compose ps -q

# Show status of specific service
docker compose ps web

# Custom format
docker compose ps --format "table {{.Name}}\t{{.Status}}\t{{.Ports}}"
```

---

### `docker compose build`

**What it does:** Builds or rebuilds service images without starting them.

```bash
# Build all services
docker compose build

# Build specific service
docker compose build web

# Build without using cache
docker compose build --no-cache web

# Build with build args
docker compose build --build-arg ENV=production web

# Pull latest base images before building
docker compose build --pull
```

---

### `docker compose pull` and `docker compose push`

```bash
# Pull latest images for all services
docker compose pull

# Pull for specific services
docker compose pull db redis

# Push built images to registry
docker compose push

# Push specific service
docker compose push web
```

---

### `docker compose config`

**What it does:** Validates and shows the parsed/merged Compose configuration.

**Why you use it:** Debug Compose file issues, see the result of variable interpolation, verify override files merged correctly.

```bash
# Validate and show the composed configuration
docker compose config

# With multiple files (shows merged result)
docker compose -f docker-compose.yml -f docker-compose.prod.yml config

# Show only service names
docker compose config --services

# Show only volume names
docker compose config --volumes
```

---

### `docker compose top`

**What it does:** Shows processes running inside Compose service containers.

```bash
docker compose top

# For specific service
docker compose top web
```

---

### `docker compose cp`

**What it does:** Copy files between service containers and the host (Compose version of `docker cp`).

```bash
# From container to host
docker compose cp web:/app/logs/django.log ./

# From host to container
docker compose cp ./local_settings.py web:/app/myproject/

# From service (when scaled, defaults to first container)
docker compose cp web:/app/staticfiles ./staticfiles/
```

---

### `docker compose scale` (deprecated) → Use `--scale` flag

```bash
# Old way (deprecated)
docker compose scale web=3

# Modern way
docker compose up --scale web=3 --scale celery=2
```

---

## 7. Registry & Distribution Commands

---

### `docker login` / `docker logout`

```bash
# Login to Docker Hub
docker login

# Login to Docker Hub with credentials
docker login -u yourusername -p yourpassword

# Login to private registry
docker login myregistry.company.com

# Login to AWS ECR
aws ecr get-login-password --region us-east-1 | \
  docker login --username AWS --password-stdin \
  123456789012.dkr.ecr.us-east-1.amazonaws.com

# Login to GitHub Container Registry
echo $GITHUB_TOKEN | docker login ghcr.io -u USERNAME --password-stdin

# Logout
docker logout
docker logout myregistry.company.com
```

---

### `docker tag`

**What it does:** Creates an additional tag for an existing image. Doesn't copy the image — just adds a new name pointing to the same image.

```bash
# ── Basic Syntax ─────────────────────────────────────────────────────────
docker tag SOURCE_IMAGE[:TAG] TARGET_IMAGE[:TAG]

# ── Common Usage ─────────────────────────────────────────────────────────

# Tag for Docker Hub
docker tag myapp:v1.0.0 yourusername/myapp:v1.0.0
docker tag myapp:v1.0.0 yourusername/myapp:latest

# Tag for private registry
docker tag myapp:v1.0.0 myregistry.company.com/team/myapp:v1.0.0

# Tag for AWS ECR
docker tag myapp:v1.0.0 123456789.dkr.ecr.us-east-1.amazonaws.com/myapp:v1.0.0

# Tag with Git commit hash (CI/CD best practice)
GIT_SHA=$(git rev-parse --short HEAD)
docker tag myapp:latest myapp:${GIT_SHA}
```

---

### `docker push`

**What it does:** Uploads a local image to a registry.

```bash
# Push to Docker Hub
docker push yourusername/myapp:v1.0.0
docker push yourusername/myapp:latest

# Push to private registry
docker push myregistry.company.com/team/myapp:v1.0.0

# Push all tags
docker push --all-tags yourusername/myapp
```

---

### `docker search`

**What it does:** Searches Docker Hub for images.

```bash
# Search for Django-related images
docker search django

# Search for Python
docker search python

# Filter: only official images
docker search --filter is-official=true python

# Filter by minimum stars
docker search --filter stars=100 python

# Limit results
docker search --limit 5 python

# Format output
docker search --format "table {{.Name}}\t{{.StarCount}}\t{{.IsOfficial}}" python
```

---

## 8. System & Cleanup Commands

---

### `docker system info` / `docker info`

**What it does:** Shows system-wide Docker information — Docker version, storage driver, number of containers/images, resource limits.

```bash
# Full system info
docker info

# Docker version info
docker version

# JSON format
docker info --format '{{json .}}'

# Specific fields
docker info --format '{{.ServerVersion}}'
docker info --format '{{.Containers}}'
docker info --format '{{.Driver}}'          # Storage driver (should be overlay2)
```

---

### `docker system df`

**What it does:** Shows **disk usage** by Docker objects — images, containers, volumes, build cache.

```bash
# Summary of disk usage
docker system df

# Verbose — show individual items
docker system df -v
```

**Sample Output:**
```
TYPE            TOTAL     ACTIVE    SIZE      RECLAIMABLE
Images          15        3         4.23GB    3.10GB (73%)
Containers      6         2         1.20MB    0B (0%)
Local Volumes   8         3         2.15GB    1.20GB (55%)
Build Cache     47        0         890MB     890MB
```

---

### `docker system prune`

**What it does:** Removes all **unused** Docker objects — stopped containers, dangling images, unused networks, build cache.

```bash
# Remove unused objects (asks for confirmation)
docker system prune

# Force (no confirmation prompt)
docker system prune -f

# Also remove unused volumes (⚠️ DESTROYS DATA)
docker system prune -v
docker system prune --volumes

# Also remove unused images (not just dangling)
docker system prune -a

# Nuclear option — remove EVERYTHING unused
docker system prune -af --volumes

# Prune objects older than 24 hours
docker system prune --filter until=24h
```

> ⚠️ **Never run `docker system prune -af --volumes` on a production server.** This will delete your database data stored in volumes.

---

### `docker container prune` / `docker image prune` / `docker volume prune` / `docker network prune`

Fine-grained cleanup commands:

```bash
# Remove all stopped containers
docker container prune

# Remove dangling images (untagged)
docker image prune

# Remove ALL unused images
docker image prune -a

# Remove unused volumes (⚠️)
docker volume prune

# Remove unused networks
docker network prune

# All with force (no confirmation)
docker container prune -f
docker image prune -af
docker volume prune -f
docker network prune -f
```

---

### `docker system events`

Same as `docker events` — covered in the inspection section.

---

## 9. 🔗 Combined & Chained Commands

These are **power-user patterns** — combining multiple commands for efficient workflows. This is where Docker really shines.

---

### 🔗 Stop and Remove All Containers

```bash
# Stop all running containers, then remove all containers
docker stop $(docker ps -q) && docker rm $(docker ps -a -q)

# Force remove all containers in one command
docker rm -f $(docker ps -a -q)

# One-liner: stop + remove + prune (clean everything)
docker stop $(docker ps -q) 2>/dev/null; docker container prune -f
```

---

### 🔗 Build, Tag, and Push in One Go

```bash
# Build → Tag → Push pipeline
APP_NAME="mycompany/django-app"
VERSION="v2.1.0"
GIT_SHA=$(git rev-parse --short HEAD)

docker build -t ${APP_NAME}:${VERSION} -t ${APP_NAME}:latest . \
  && docker tag ${APP_NAME}:${VERSION} ${APP_NAME}:${GIT_SHA} \
  && docker push ${APP_NAME}:${VERSION} \
  && docker push ${APP_NAME}:${GIT_SHA} \
  && docker push ${APP_NAME}:latest \
  && echo "✅ Successfully built and pushed ${APP_NAME}:${VERSION}"
```

---

### 🔗 Full Deploy Command (Pull + Migrate + Restart)

```bash
# Pull new image, run migrations, restart web service
IMAGE="mycompany/django-app:v2.1.0"

docker pull ${IMAGE} \
  && docker compose -f docker-compose.prod.yml run --rm web python manage.py migrate \
  && docker compose -f docker-compose.prod.yml up -d --no-deps web \
  && echo "✅ Deployment complete"
```

---

### 🔗 Run Migrations and Start Server

```bash
# Run migrations, then start the server
docker compose run --rm web python manage.py migrate \
  && docker compose up -d web
```

---

### 🔗 Full Django Setup from Clone

```bash
# One script to set up a fresh Django Docker environment
docker compose up -d db redis \
  && sleep 5 \
  && docker compose run --rm web python manage.py migrate \
  && docker compose run --rm web python manage.py collectstatic --noinput \
  && docker compose up -d \
  && echo "✅ Django stack is up at http://localhost:8000"
```

---

### 🔗 Remove All Images for a Repository

```bash
# Remove all versions of your app image
docker rmi $(docker images mycompany/django-app -q)

# Remove all images except those in use
docker images -q | xargs docker rmi 2>/dev/null || true
```

---

### 🔗 Get Into a Running Container Fast (by partial name)

```bash
# Connect to first container matching 'web'
docker exec -it $(docker ps -qf name=web | head -1) bash

# Or as a shell alias (add to ~/.bashrc or ~/.zshrc)
alias dshell='docker exec -it $(docker ps -qf name=web | head -1) bash'
dshell  # Just type this to get into your web container
```

---

### 🔗 Watch Container Logs While Running Migrations

```bash
# Terminal 1: Follow logs
docker compose logs -f web &

# Terminal 2: Run migrations
docker compose run --rm web python manage.py migrate

# Or: see DB logs while migrating
docker compose logs -f db &
docker compose run --rm web python manage.py migrate
```

---

### 🔗 Build With Cache From Registry (CI/CD Speed Boost)

```bash
# Pull cache image first, build using it, push new cache
docker pull myregistry.com/myapp:cache || true
docker build \
  --cache-from myregistry.com/myapp:cache \
  --build-arg BUILDKIT_INLINE_CACHE=1 \
  -t myapp:latest \
  -t myregistry.com/myapp:cache \
  .
docker push myregistry.com/myapp:cache
docker push myapp:latest
```

---

### 🔗 Clean Build (Remove Old Image First)

```bash
# Remove old image, rebuild fresh, run
docker rmi myapp:latest 2>/dev/null; \
docker build --no-cache -t myapp:latest . \
  && docker compose up -d --force-recreate web
```

---

### 🔗 Database Backup Workflow

```bash
# Dump PostgreSQL data from container, compress, and timestamp it
BACKUP_FILE="backup_$(date +%Y%m%d_%H%M%S).sql.gz"

docker compose exec -T db \
  pg_dump -U django django_db \
  | gzip > ${BACKUP_FILE} \
  && echo "✅ Backup saved: ${BACKUP_FILE}"
```

---

### 🔗 Database Restore Workflow

```bash
# Restore a compressed backup into the running PostgreSQL container
BACKUP_FILE="backup_20240115_120000.sql.gz"

gunzip -c ${BACKUP_FILE} | \
  docker compose exec -T db \
  psql -U django django_db \
  && echo "✅ Database restored from ${BACKUP_FILE}"
```

---

### 🔗 Rebuild Only Changed Service

```bash
# Rebuild only the web service and restart it (without touching db/redis)
docker compose build web \
  && docker compose up -d --no-deps web \
  && echo "✅ Web service restarted with new build"
```

---

### 🔗 Run Tests and Show Coverage

```bash
docker compose run --rm \
  -e DJANGO_SETTINGS_MODULE=myproject.settings.test \
  web \
  bash -c "coverage run manage.py test && coverage report && coverage html"
```

---

### 🔗 Full Cleanup Before a New Project Setup

```bash
# Remove everything related to a project before re-initializing
docker compose down -v --rmi local --remove-orphans \
  && docker network prune -f \
  && echo "✅ Project cleaned, ready for fresh start" \
  && docker compose up -d --build
```

---

### 🔗 Find Which Container Is Using a Port

```bash
# Find the container using port 8000
docker ps --format "{{.Names}}\t{{.Ports}}" | grep 8000

# Or more precisely
docker port $(docker ps -q) 2>/dev/null | grep 8000
```

---

### 🔗 Copy Files From All Running Containers

```bash
# Copy logs from all running containers to a logs directory
mkdir -p ./container_logs
for container in $(docker ps -q); do
  name=$(docker inspect --format='{{.Name}}' $container | sed 's/\///')
  docker cp $container:/app/logs/. ./container_logs/${name}/ 2>/dev/null && \
    echo "Copied logs from $name"
done
```

---

### 🔗 Rolling Deployment (Zero Downtime)

```bash
# Requires a load balancer (e.g., Nginx) in front
# Scale up new containers, then scale down old ones

# 1. Build new image
docker build -t myapp:new .

# 2. Start new containers alongside old ones
docker compose up -d --scale web=4 --no-recreate

# 3. Wait for health checks to pass
sleep 30

# 4. Remove old containers
docker compose up -d --scale web=2 --no-recreate
```

---

### 🔗 One-Shot Shell Alias Setup for Django

```bash
# Add these to your ~/.bashrc or ~/.zshrc for fast day-to-day work

# Quick access to Django shell
alias dshell='docker compose exec web python manage.py shell'

# Quick migrations
alias dmigrate='docker compose run --rm web python manage.py migrate'

# Quick makemigrations
alias dmkmigrations='docker compose run --rm web python manage.py makemigrations'

# Quick tests
alias dtest='docker compose run --rm web python manage.py test'

# Quick logs
alias dlogs='docker compose logs -f web'

# Quick restart web
alias drestart='docker compose restart web'

# Get into the container
alias dsh='docker compose exec web bash'

# Check status
alias dps='docker compose ps'
```

---

## 10. Django-Specific Command Workflows

Complete day-to-day command sequences for Django developers.

---

### 🔁 Daily Development Workflow

```bash
# Morning: Start your stack
docker compose up -d                              # Start everything
docker compose logs -f web                        # Watch logs

# After pulling new code from git
git pull origin main
docker compose build web                          # Rebuild if Dockerfile/requirements changed
docker compose run --rm web python manage.py migrate  # Apply any new migrations
docker compose restart web                        # Restart web service

# End of day: shut down
docker compose down                               # Stop and remove containers (volumes preserved)
```

---

### 🗄️ Database Management Workflow

```bash
# Create and apply new migration
docker compose exec web python manage.py makemigrations myapp
docker compose exec web python manage.py migrate

# Check migration status
docker compose exec web python manage.py showmigrations

# Access PostgreSQL directly
docker compose exec db psql -U django -d django_db

# Common psql commands:
# \dt          — list tables
# \d tablename — describe table
# \q           — quit

# Backup the database
docker compose exec -T db pg_dump -U django django_db > backup.sql

# Restore
cat backup.sql | docker compose exec -T db psql -U django django_db

# Reset database (⚠️ destroys all data — dev only!)
docker compose down -v && docker compose up -d db \
  && sleep 5 \
  && docker compose run --rm web python manage.py migrate \
  && docker compose run --rm web python manage.py createsuperuser
```

---

### 🧪 Testing Workflow

```bash
# Run all tests
docker compose run --rm web python manage.py test

# Run specific app tests
docker compose run --rm web python manage.py test myapp

# Run specific test class
docker compose run --rm web python manage.py test myapp.tests.test_views.UserViewTest

# Run with verbose output
docker compose run --rm web python manage.py test -v 2

# Run with pytest
docker compose run --rm web pytest

# Run with coverage
docker compose run --rm web bash -c "
  coverage run manage.py test &&
  coverage report --show-missing &&
  coverage html
"

# Run tests with a test database in RAM (faster)
docker compose run --rm \
  -e DATABASE_URL=postgres://test:test@db:5432/test_$(date +%s) \
  web python manage.py test
```

---

### 📦 Dependency Management Workflow

```bash
# Add a new package
echo "djangorestframework==3.15.2" >> requirements.txt

# Rebuild the image with new dependency
docker compose build --no-cache web

# Restart with new image
docker compose up -d --no-deps web

# Verify package is installed
docker compose exec web pip show djangorestframework

# List all installed packages
docker compose exec web pip list

# Check for security vulnerabilities
docker compose run --rm web pip audit    # if pip-audit is installed
docker compose run --rm web safety check # if safety is installed
```

---

### 🚀 Production Deployment Workflow

```bash
#!/bin/bash
# deploy.sh — Safe production deployment script

set -e  # Exit on error

IMAGE="mycompany/django-app"
VERSION=$1  # e.g., ./deploy.sh v2.1.0

echo "🚀 Deploying ${IMAGE}:${VERSION}..."

# 1. Pull new image
docker pull ${IMAGE}:${VERSION}

# 2. Run pre-deployment checks
docker run --rm --env-file .env.production \
  ${IMAGE}:${VERSION} \
  python manage.py check --deploy

# 3. Backup database before migration
docker compose exec -T db pg_dump -U django django_db | \
  gzip > backup_pre_${VERSION}_$(date +%Y%m%d_%H%M%S).sql.gz

# 4. Run migrations
docker compose -f docker-compose.prod.yml run --rm \
  -e DJANGO_IMAGE=${IMAGE}:${VERSION} \
  web python manage.py migrate --noinput

# 5. Collect static files
docker compose -f docker-compose.prod.yml run --rm web \
  python manage.py collectstatic --noinput

# 6. Update service to new image
docker compose -f docker-compose.prod.yml up -d --no-deps web

# 7. Wait and verify
sleep 10
docker compose -f docker-compose.prod.yml exec web \
  curl -f http://localhost:8000/health/ \
  && echo "✅ Deployment successful: ${IMAGE}:${VERSION}" \
  || echo "❌ Health check failed — rolling back"
```

---

## 📋 Ultimate Quick Reference Cheat Sheet

```bash
# ╔══════════════════════════════════════════════════════════════════════╗
# ║            DOCKER QUICK REFERENCE FOR DJANGO DEVELOPERS            ║
# ╚══════════════════════════════════════════════════════════════════════╝

# ── BUILD ──────────────────────────────────────────────────────────────
docker build -t myapp .                        # Build image
docker build -t myapp:v1 --no-cache .          # Build fresh (no cache)
docker build -f Dockerfile.prod -t myapp:prod . # Custom Dockerfile

# ── RUN ────────────────────────────────────────────────────────────────
docker run myapp                               # Run (foreground)
docker run -d myapp                            # Run (background)
docker run -d -p 8000:8000 --name web myapp   # Named, port mapped
docker run --rm -it myapp bash                 # Interactive, auto-remove
docker run --env-file .env myapp               # With .env file

# ── CONTAINER MANAGEMENT ───────────────────────────────────────────────
docker ps                                      # Running containers
docker ps -a                                   # All containers
docker stop web                                # Graceful stop
docker start web                               # Start stopped container
docker restart web                             # Restart
docker rm web                                  # Remove stopped container
docker rm -f web                               # Force remove running

# ── INSPECT & DEBUG ────────────────────────────────────────────────────
docker logs -f web                             # Follow logs
docker logs --tail 100 web                     # Last 100 lines
docker exec -it web bash                       # Shell inside container
docker exec web python manage.py migrate       # Run command
docker stats                                   # Resource usage
docker inspect web                             # Full metadata
docker top web                                 # Processes in container

# ── IMAGES ─────────────────────────────────────────────────────────────
docker images                                  # List images
docker pull python:3.11-slim                   # Pull image
docker rmi myapp:old                           # Remove image
docker history myapp                           # Show layers
docker tag myapp:v1 myregistry/myapp:v1        # Tag image

# ── VOLUMES ────────────────────────────────────────────────────────────
docker volume ls                               # List volumes
docker volume create mydata                    # Create volume
docker volume inspect mydata                   # Inspect volume
docker volume rm mydata                        # Remove volume
docker volume prune                            # Remove unused volumes

# ── NETWORKS ───────────────────────────────────────────────────────────
docker network ls                              # List networks
docker network create mynet                    # Create network
docker network inspect mynet                   # Inspect network
docker network connect mynet web               # Connect container
docker network prune                           # Remove unused networks

# ── DOCKER COMPOSE ─────────────────────────────────────────────────────
docker compose up -d                           # Start all (background)
docker compose up -d --build                   # Rebuild + start
docker compose down                            # Stop + remove containers
docker compose down -v                         # Also remove volumes ⚠️
docker compose logs -f web                     # Follow service logs
docker compose ps                              # Service status
docker compose restart web                     # Restart service
docker compose build web                       # Rebuild service image

# ── COMPOSE ONE-OFF COMMANDS ───────────────────────────────────────────
docker compose run --rm web python manage.py migrate
docker compose run --rm web python manage.py createsuperuser
docker compose run --rm web python manage.py shell
docker compose run --rm web python manage.py test
docker compose run --rm web python manage.py collectstatic --noinput
docker compose run --rm web python manage.py makemigrations

# ── REGISTRY ───────────────────────────────────────────────────────────
docker login                                   # Login to Docker Hub
docker push myregistry/myapp:v1               # Push image
docker pull myregistry/myapp:v1               # Pull image

# ── CLEANUP ────────────────────────────────────────────────────────────
docker system prune                            # Remove unused objects
docker system prune -af                        # Remove ALL unused
docker system prune -af --volumes              # ⚠️ Also volumes (DATA LOSS)
docker system df                               # Check disk usage
docker image prune -a                          # Remove unused images

# ── COMBINED COMMANDS ──────────────────────────────────────────────────
docker stop $(docker ps -q)                    # Stop all running
docker rm $(docker ps -a -q)                   # Remove all containers
docker rmi $(docker images -q)                 # Remove all images
docker build -t myapp . && docker compose up -d --no-deps web  # Build+deploy
docker compose run --rm web python manage.py migrate && docker compose up -d
```

---

> 📝 **Remember:** Practice these commands daily on a real Django project. The best way to learn is to spin up a Django + PostgreSQL + Redis stack with Docker Compose and work through the commands hands-on. Every management command you used to run with `python manage.py X` now becomes `docker compose run --rm web python manage.py X`.
