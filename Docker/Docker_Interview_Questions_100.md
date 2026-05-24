# 🐳 100 Docker Interview Questions & Answers
### From Zero to Production-Ready — Tailored for Python/Django Backend Developers

> **How to use this guide:** Read it top to bottom at least once. Every concept builds on the previous one. Pay special attention to the 🐍 **Django/Python Analogy** sections — they connect Docker ideas to things you already know deeply.

---

## 📋 Table of Contents

| Phase | Topic | Questions |
|-------|-------|-----------|
| **Phase 1** | Docker Basics & Core Concepts | Q1 – Q25 |
| **Phase 2** | Intermediate Container Operations & Networking | Q26 – Q55 |
| **Phase 3** | Advanced Architecture, Volumes & Security | Q56 – Q80 |
| **Phase 4** | Django-Specific Dockerization & Best Practices | Q81 – Q100 |

---

# 🟢 Phase 1: Docker Basics & Core Concepts
### *(For complete beginners — start here)*

---

### Q1. What is Docker, and why should a backend developer care about it?

**Answer:**
Docker is a platform that package software into standardized units called containers.Docker is a **containerization platform** that packages your application and all its dependencies (Python version, libraries, system packages, environment variables) into a single, portable unit called a **container**.

**The Analogy:** Think of Docker like a virtual environment `(venv)` taken to the next level. While `venv` only isolates Python packages, Docker isolates the entire operating system, system libraries, Python version, and database drivers so your app runs identically everywhere.

Before Docker, the classic problem was: *"It works on my machine but not on the server."* Docker solves this by ensuring your application runs in the **exact same environment** everywhere  your laptop, staging server, or production cloud.

**Why you should care as a Django developer:**
- Your Django app + PostgreSQL + Redis + Nginx can all be defined in one file and spun up with one command.
- No more manually installing Python versions, setting up virtualenvs on servers, or fixing OS-level dependency conflicts.
- Industry standard — almost every DevOps pipeline expects Docker knowledge.

```bash
# Without Docker: hours of manual setup
# With Docker: one command
docker compose up
```
#### Docker Image:
- A Docker Image is a read-only blueprint containing the source code, libraries, and dependencies needed to run an application.

#### Docker Container:
- A Docker Container is a live, runnable instance of that image.

**The Analogy:** In Object-Oriented Programming (OOP), a Docker Image is the Python Class, and a Docker Container is an Instantiated Object created from that class.

---

### Q2. What is the difference between a Virtual Machine (VM) and a Docker container?

**Answer:**

| Feature         | Virtual Machine | Docker Container        |
|-----------------|-----------------|-------------------------|
| **Includes**    | Full OS + App   | Only App + dependencies |
| **Size**        | GBs             | MBs                     |
| **Startup Time**| Minutes         | Seconds                 |
| **Isolation**   | Hardware-level  | Process-level           |
| **Performance** | Slower          | Near-native             |
| **Use Case**    | Full OS isolation | App isolation         |

**The key difference:** A VM virtualizes the **hardware** (so it runs a full OS). A Docker container virtualizes the **operating system** — it shares the host machine's OS kernel but isolates the application's processes and filesystem.

🐍 **Django Analogy:**
> A VM is like running a completely separate physical computer inside your computer. A Docker container is like running a separate Python **virtualenv** — same OS, but completely isolated dependencies and environment.

---

### Q3. What is a Docker Image?

**Answer:**

A **Docker Image** is a **read-only, immutable template** that contains everything needed to run your application: the base OS layer, runtime (Python), libraries, your code, and startup commands. Think of it as a **blueprint**.

Images are built in **layers**. Each instruction in a Dockerfile adds a new layer on top of the previous one.

```bash
# Pull an existing image from Docker Hub
docker pull python:3.11-slim

# List all images on your machine
docker images
```

🐍 **Django/Python Analogy:**
> A Docker Image is like a **Python class**. It defines the structure and behavior, but by itself, it doesn't *do* anything. You can't "run" a class — you instantiate it first. Similarly, you can't "use" an image — you run it as a container.

---

### Q4. What is a Docker Container?

**Answer:**

A **Docker Container** is a **running instance of a Docker Image**. When you run an image, Docker creates a container — an isolated process with its own filesystem, network, and process space. Containers are **ephemeral** by default (their data is lost when stopped, unless you use volumes).

```bash
# Run a container from the python:3.11-slim image
docker run python:3.11-slim python --version

# Run interactively (like opening a Python shell)
docker run -it python:3.11-slim bash

# List running containers
docker ps

# List all containers (including stopped)
docker ps -a
```

🐍 **Django/Python Analogy:**
> A Docker Container is like an **instantiated Python object** created from a class (Image). You can create multiple objects (containers) from the same class (image), each running independently. Just like `user1 = User()` and `user2 = User()` are separate objects, two containers from the same image are separate, isolated environments.

---

### Q5. What is Docker Hub?

**Answer:**

**Docker Hub** (`hub.docker.com`) is the **official public registry** for Docker Images — think of it as the **PyPI of Docker**. It hosts thousands of pre-built images (Python, PostgreSQL, Nginx, Redis, etc.) that you can pull and use immediately.

Key concepts:
- **Official Images**: Maintained by Docker/vendors (e.g., `python`, `postgres`, `nginx`)
- **User Images**: Community-contributed (e.g., `username/my-django-app`)
- **Tags**: Versions of an image (e.g., `python:3.11`, `python:3.11-slim`, `python:3.11-alpine`)

```bash
# Pull the official Python 3.11 slim image
docker pull python:3.11-slim

# Search for images
docker search django

# Push your own image (after logging in)
docker login
docker push yourusername/my-django-app:v1.0
```

🐍 **Django/Python Analogy:**
> Docker Hub is like **PyPI**. Just as you `pip install django` to get Django from PyPI, you `docker pull python:3.11` to get a Python environment from Docker Hub.

---

### Q6. What is a Dockerfile?

**Answer:**

A **Dockerfile** is a plain-text file (literally named `Dockerfile`, no extension) containing a series of **instructions** that tell Docker *how to build a Docker Image* step by step. It's the **recipe** for your image.

```dockerfile
# Example Dockerfile for a Django app
FROM python:3.11-slim          # Start from official Python image

WORKDIR /app                    # Set working directory inside container

COPY requirements.txt .         # Copy requirements first (for layer caching)
RUN pip install -r requirements.txt  # Install dependencies

COPY . .                        # Copy the rest of the project code

EXPOSE 8000                     # Document that the app listens on port 8000

CMD ["python", "manage.py", "runserver", "0.0.0.0:8000"]  # Default command
```

🐍 **Django/Python Analogy:**
> A Dockerfile is like your project's **`requirements.txt` + `setup.py` + startup script** combined into one file. It describes exactly what your app needs and how to start it — but for the entire environment (OS + Python + packages + code), not just Python packages.

---

### Q7. What are the most common Dockerfile instructions?

**Answer:**

| Instruction | Purpose | Example |
|---|---|---|
| `FROM` | Base image to start from | `FROM python:3.11-slim` |
| `WORKDIR` | Set working directory | `WORKDIR /app` |
| `COPY` | Copy files from host to image | `COPY . /app` |
| `ADD` | Like COPY but supports URLs & tar extraction | `ADD app.tar.gz /app` |
| `RUN` | Execute a command during build | `RUN pip install django` |
| `ENV` | Set environment variables | `ENV DEBUG=False` |
| `EXPOSE` | Document which port the app uses | `EXPOSE 8000` |
| `CMD` | Default command when container starts | `CMD ["gunicorn", "wsgi:app"]` |
| `ENTRYPOINT` | Fixed command that always runs | `ENTRYPOINT ["python"]` |
| `ARG` | Build-time variables | `ARG VERSION=1.0` |
| `VOLUME` | Declare a mount point | `VOLUME /data` |
| `USER` | Set the user to run as | `USER appuser` |

---

### Q8. What is the difference between `CMD` and `ENTRYPOINT`?

**Answer:**

Both define what command runs when a container starts, but they behave differently when you pass arguments:

- **`CMD`**: Sets the **default command** that can be **overridden** at runtime.
- **`ENTRYPOINT`**: Sets the **fixed command** that always runs. Arguments passed at runtime are **appended** to it, not replaced.

```dockerfile
# Using CMD — can be overridden
CMD ["python", "manage.py", "runserver"]

# Using ENTRYPOINT — always runs python
ENTRYPOINT ["python"]
CMD ["manage.py", "runserver"]  # default argument
```

```bash
# Overrides CMD entirely — runs shell instead
docker run myimage bash

# With ENTRYPOINT, this runs: python manage.py migrate
docker run myimage manage.py migrate
```

🐍 **Django/Python Analogy:**
> `ENTRYPOINT` is like the `python` interpreter you always need. `CMD` is like the script name — you can swap the script (`manage.py runserver` vs `manage.py migrate`) but you always use Python. Together: `ENTRYPOINT ["python"]` + `CMD ["manage.py", "runserver"]`.

---

### Q9. What is the difference between `COPY` and `ADD` in a Dockerfile?

**Answer:**

Both copy files into the image, but `ADD` has extra powers:

| Feature | `COPY` | `ADD` |
|---|---|---|
| Copy local files | ✅ Yes | ✅ Yes |
| Copy from URL | ❌ No | ✅ Yes |
| Auto-extract `.tar.gz` | ❌ No | ✅ Yes |
| **Recommended for?** | Most cases | Only when you need URL/tar features |

**Best Practice:** Always prefer `COPY` unless you specifically need `ADD`'s extra features. `COPY` is more transparent and predictable.

```dockerfile
# ✅ Preferred — clear intent
COPY requirements.txt /app/

# ❌ Avoid unless needed
ADD requirements.txt /app/
```

---

### Q10. How do you build a Docker Image from a Dockerfile?

**Answer:**

Use the `docker build` command. Run it from the directory containing your Dockerfile.

```bash
# Basic build — tags the image as "my-django-app"
docker build -t my-django-app .

# Build with a version tag
docker build -t my-django-app:v1.0 .

# Build from a specific Dockerfile path
docker build -f ./docker/Dockerfile.prod -t my-django-app:prod .

# Build with build arguments
docker build --build-arg DEBUG=False -t my-django-app .
```

**Flags explained:**
- `-t` or `--tag`: Name and optionally tag the image (`name:tag`)
- `-f`: Specify a custom Dockerfile path
- `.`: The **build context** — the directory Docker sends to the daemon (usually current directory)

---

### Q11. What is a Docker Registry?

**Answer:**

A **Docker Registry** is a **storage and distribution system** for Docker Images. It's like a server where images are stored and from where they can be pulled.

Types of registries:
- **Public Registry**: Docker Hub (`hub.docker.com`) — free, public images
- **Private Registry**: Self-hosted or cloud-managed registries for proprietary images
- **Cloud Registries**:
  - AWS ECR (Elastic Container Registry)
  - Google Artifact Registry (GCR)
  - GitHub Container Registry (GHCR)
  - Azure Container Registry (ACR)

```bash
# Pull from Docker Hub (default registry)
docker pull python:3.11

# Pull from a private registry (e.g., AWS ECR)
docker pull 123456789.dkr.ecr.us-east-1.amazonaws.com/my-django-app:latest

# Tag and push to a registry
docker tag my-django-app:latest myregistry.io/myteam/my-django-app:v1
docker push myregistry.io/myteam/my-django-app:v1
```

---

### Q12. What does `docker run` do and what are its key flags?

**Answer:**

`docker run` **creates and starts** a new container from a specified image. It's the most-used Docker command.

```bash
# Basic run
docker run python:3.11-slim

# Common flags
docker run \
  -d \                          # Detached mode (run in background)
  -it \                         # Interactive + TTY (for shell access)
  --name my-django-container \  # Give the container a name
  -p 8000:8000 \                # Map host port 8000 to container port 8000
  -v $(pwd):/app \              # Mount current directory to /app in container
  -e DEBUG=True \               # Set environment variable
  --rm \                        # Auto-remove container when it exits
  my-django-app:latest          # Image to use
```

**Key flags cheatsheet:**
| Flag | Meaning |
|---|---|
| `-d` | Run in background (detached) |
| `-it` | Interactive terminal |
| `-p host:container` | Port mapping |
| `-v host:container` | Volume mount |
| `-e KEY=VALUE` | Environment variable |
| `--name` | Container name |
| `--rm` | Remove container on exit |
| `--network` | Connect to a network |

---

### Q13. What is Docker Compose?

**Answer:**

**Docker Compose** is a tool for defining and running **multi-container Docker applications**. Instead of running multiple `docker run` commands manually, you define all your services in a single `docker-compose.yml` file and start them all with `docker compose up`.

```yaml
# docker-compose.yml — defines a full Django stack
version: "3.9"

services:
  web:                          # Django app service
    build: .
    ports:
      - "8000:8000"
    environment:
      - DEBUG=True
      - DATABASE_URL=postgres://user:pass@db:5432/mydb
    depends_on:
      - db
      - redis

  db:                           # PostgreSQL service
    image: postgres:15
    environment:
      POSTGRES_USER: user
      POSTGRES_PASSWORD: pass
      POSTGRES_DB: mydb
    volumes:
      - postgres_data:/var/lib/postgresql/data

  redis:                        # Redis service
    image: redis:7-alpine

volumes:
  postgres_data:
```

```bash
# Start all services
docker compose up

# Start in background
docker compose up -d

# Stop all services
docker compose down

# Rebuild and start
docker compose up --build
```

🐍 **Django Analogy:**
> Docker Compose is like your project's **`docker-compose.yml` = full infrastructure manifest**. Just as `manage.py` orchestrates Django commands, `docker compose` orchestrates your entire application stack (web + DB + cache + queue).

---

### Q14. What is the Docker Architecture?

**Answer:**

Docker uses a **client-server architecture** with three main components:

```
┌─────────────────────────────────────────────────────┐
│                   Docker Client                      │
│         (docker CLI — what you type into)            │
└────────────────────┬────────────────────────────────┘
                     │ REST API calls
┌────────────────────▼────────────────────────────────┐
│                  Docker Daemon (dockerd)             │
│    (server process that manages everything)          │
│                                                      │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐ │
│  │  Containers │  │   Images    │  │  Networks   │ │
│  └─────────────┘  └─────────────┘  └─────────────┘ │
└─────────────────────────────────────────────────────┘
                     │
         Pulls from / Pushes to
┌────────────────────▼────────────────────────────────┐
│               Docker Registry (Hub/ECR/etc.)         │
└─────────────────────────────────────────────────────┘
```

- **Docker Client**: The CLI you interact with (`docker build`, `docker run`)
- **Docker Daemon (`dockerd`)**: Background service that does the actual work — builds images, starts containers, manages networks and volumes
- **Docker Registry**: Remote store for images

---

### Q15. What is a Docker layer and why does it matter?

**Answer:**

Docker images are built in **layers** — each Dockerfile instruction (`FROM`, `RUN`, `COPY`, etc.) creates a **new read-only layer** stacked on top of previous layers. Layers are **cached** and **shared** between images.

```dockerfile
FROM python:3.11-slim      # Layer 1: Base Python image
WORKDIR /app               # Layer 2: Set directory
COPY requirements.txt .    # Layer 3: Copy requirements
RUN pip install -r requirements.txt  # Layer 4: Install packages (HEAVY)
COPY . .                   # Layer 5: Copy app code
CMD ["python", "manage.py", "runserver"]  # Layer 6
```

**Why this matters for performance:**
- If Layer 5 changes (you edit your code), Docker only rebuilds from Layer 5 onwards
- Layers 1–4 are **cached** and reused instantly
- This is why you always `COPY requirements.txt` **before** `COPY . .` — requirements rarely change, but code changes constantly

🐍 **Django Analogy:**
> Layers are like **Git commits**. Each commit (layer) is a snapshot of changes. Docker caches unchanged layers just like Git doesn't re-transmit unchanged commits — only the diff is sent/rebuilt.

---

### Q16. What is a `.dockerignore` file?

**Answer:**

A `.dockerignore` file tells Docker which files and directories to **exclude from the build context** when building an image. It works exactly like `.gitignore`.

Without it, Docker sends your entire project directory (including `node_modules`, `.git`, `__pycache__`, etc.) to the daemon — making builds slow and images bloated.

```dockerignore
# .dockerignore for a Django project
__pycache__
*.pyc
*.pyo
.git
.gitignore
.env
*.env
venv/
.venv/
node_modules/
*.log
.DS_Store
.pytest_cache/
htmlcov/
.coverage
README.md
docker-compose*.yml
```

**Rule of thumb:** If it's in `.gitignore`, it probably belongs in `.dockerignore` too.

---

### Q17. What is the difference between `docker stop` and `docker kill`?

**Answer:**

| Command | Signal Sent | Behavior |
|---|---|---|
| `docker stop` | `SIGTERM` then `SIGKILL` (after 10s) | **Graceful** shutdown — app can clean up |
| `docker kill` | `SIGKILL` immediately | **Forceful** shutdown — app is immediately terminated |

```bash
# Graceful stop (wait up to 10 seconds for cleanup)
docker stop my-container

# Forceful immediate kill
docker kill my-container

# Stop with a custom timeout (30 seconds)
docker stop --time 30 my-container
```

🐍 **Django/Python Analogy:**
> `docker stop` is like pressing `Ctrl+C` in your terminal — Django/Gunicorn gets a SIGTERM signal and finishes current requests before shutting down. `docker kill` is like `kill -9` — immediate death, no cleanup, open connections may be corrupted.

**Best Practice:** Always use `docker stop` in production to allow graceful shutdown.

---

### Q18. How do you view logs from a Docker container?

**Answer:**

Use `docker logs` to see the stdout/stderr output of a container.

```bash
# View logs of a container
docker logs my-container

# Follow logs in real-time (like tail -f)
docker logs -f my-container

# Show last 100 lines
docker logs --tail 100 my-container

# Show logs with timestamps
docker logs -t my-container

# Combine flags — last 50 lines, follow, with timestamps
docker logs -f -t --tail 50 my-container
```

🐍 **Django Analogy:**
> `docker logs -f my-container` is equivalent to `tail -f django.log`. Any `print()` statement or `logging.info()` call in your Django app appears here.

---

### Q19. How do you execute a command inside a running container?

**Answer:**

Use `docker exec` to run commands inside an **already running** container.

```bash
# Open an interactive bash shell inside the container
docker exec -it my-container bash

# Run a specific command
docker exec my-container python manage.py migrate

# Run as a specific user
docker exec -u root my-container bash

# Check installed packages inside container
docker exec my-container pip list

# Open Django shell
docker exec -it my-container python manage.py shell
```

**Key difference:**
- `docker run` = **creates a new container** and runs the command
- `docker exec` = **runs a command inside an existing running container**

---

### Q20. What is the difference between `docker run` and `docker exec`?

**Answer:**

| Feature | `docker run` | `docker exec` |
|---|---|---|
| **Action** | Creates a NEW container | Runs command in EXISTING container |
| **Container state** | Container doesn't need to exist | Container must be running |
| **Use case** | Starting fresh workloads | Debugging, running migrations |

```bash
# docker run — creates a brand new container
docker run -it python:3.11-slim bash

# docker exec — enters an already-running container
docker exec -it my-running-container bash
```

🐍 **Django Analogy:**
> `docker run` is like launching a new Python process. `docker exec` is like attaching to an already running Gunicorn process — you're entering an existing process, not starting a new one.

---

### Q21. How do you remove containers, images, and volumes?

**Answer:**

```bash
# ── CONTAINERS ──────────────────────────────────
docker rm container_name               # Remove stopped container
docker rm -f container_name            # Force remove running container
docker container prune                 # Remove ALL stopped containers

# ── IMAGES ──────────────────────────────────────
docker rmi image_name:tag              # Remove an image
docker rmi -f image_name:tag           # Force remove
docker image prune                     # Remove dangling (untagged) images
docker image prune -a                  # Remove ALL unused images

# ── VOLUMES ─────────────────────────────────────
docker volume rm volume_name           # Remove a volume
docker volume prune                    # Remove all unused volumes

# ── NUCLEAR OPTION — Clean everything ───────────
docker system prune                    # Remove stopped containers, unused networks, dangling images
docker system prune -a --volumes       # Remove EVERYTHING unused (containers, images, volumes, networks)
```

⚠️ **Warning:** `docker system prune -a --volumes` is irreversible. Double-check before running in production.

---

### Q22. What are Docker tags and how do you use them?

**Answer:**

A **tag** is a **label** applied to a Docker image to identify a specific version. It's the part after the colon in `image:tag`. Without a tag, Docker defaults to `:latest`.

```bash
# These are all the same image with different tags
python:3.11          # Specific minor version
python:3.11-slim     # Slim variant (smaller size)
python:3.11-alpine   # Alpine Linux variant (even smaller)
python:latest        # Always the newest version (avoid in production!)
python:3             # Locks to Python 3.x (minor version may change)
```

**Tagging your own images:**
```bash
# Build with a tag
docker build -t my-django-app:v1.2.3 .
docker build -t my-django-app:latest .

# Tag an existing image
docker tag my-django-app:v1.2.3 my-django-app:stable
```

**Best Practice:** Never use `:latest` in production Dockerfiles. Always pin to a specific version like `python:3.11.9-slim` for reproducible builds.

---

### Q23. What is the `docker inspect` command?

**Answer:**

`docker inspect` returns **detailed low-level JSON information** about Docker objects (containers, images, volumes, networks).

```bash
# Inspect a container
docker inspect my-container

# Inspect an image
docker inspect python:3.11-slim

# Extract specific field with --format
docker inspect --format='{{.NetworkSettings.IPAddress}}' my-container

# Get environment variables of a container
docker inspect --format='{{range .Config.Env}}{{println .}}{{end}}' my-container

# Get the container's status
docker inspect --format='{{.State.Status}}' my-container
```

🐍 **Python Analogy:**
> `docker inspect` is like calling `vars(obj)` or `obj.__dict__` in Python — it dumps all the internal state and configuration of a Docker object.

---

### Q24. What is the difference between an image's base and parent image?

**Answer:**

- **Base Image**: An image with no parent — it's built from scratch (`FROM scratch`). Usually official OS images like `ubuntu`, `alpine`, or `debian`.
- **Parent Image**: The image in the `FROM` instruction of your Dockerfile. It doesn't have to be a base image — it could itself be built on top of something else.

```dockerfile
# ubuntu is the BASE image (built from scratch)
FROM ubuntu:22.04

# python:3.11-slim is the PARENT image of this Dockerfile
# python:3.11-slim itself uses debian as its base
FROM python:3.11-slim
```

**Image inheritance chain:**
```
scratch (base)
  └── debian:bookworm-slim
        └── python:3.11-slim      ← parent of YOUR Dockerfile
              └── my-django-app   ← your image
```

---

### Q25. What does "ephemeral" mean in the context of containers?

**Answer:**

**Ephemeral** means **temporary / short-lived**. Containers are designed to be ephemeral by default — when a container is stopped and removed, **all data written inside the container's filesystem is lost**.

This is actually a feature, not a bug:
- Containers should be **stateless** — they can be created, destroyed, and replaced at any time
- Application state should be stored in **volumes** (databases, file uploads), not inside the container
- This enables easy scaling: spin up 10 containers, tear down 10 containers, no data concerns

```bash
# Data INSIDE a container is lost when it's removed
docker run --rm python:3.11-slim python -c "
import os
with open('/tmp/test.txt', 'w') as f:
    f.write('this will disappear')
"
# After this command, the container is gone and so is test.txt

# Data in a VOLUME persists
docker run -v myvolume:/data python:3.11-slim python -c "
with open('/data/test.txt', 'w') as f:
    f.write('this persists!')
"
```

🐍 **Django Analogy:**
> The container filesystem is like a **request's local variables** in Django — they live for the duration of the request (container run) and disappear when it's done. Django models saved to PostgreSQL are like Docker **volumes** — they persist beyond any individual request (container).

---

# 🔵 Phase 2: Intermediate Container Operations & Networking
### *(Hands-on operations, networking, and Docker Compose deep dive)*

---

### Q26. What is Docker Networking and why is it important?

**Answer:**

Docker Networking allows **containers to communicate with each other and with the outside world**. By default, Docker creates isolated network namespaces for containers — they can't talk to each other unless connected to the same network.

This is crucial because in a real Django stack:
- Your Django app needs to talk to PostgreSQL
- Django needs to reach Redis
- Nginx needs to route traffic to Gunicorn
- None of these should be publicly exposed except Nginx

```bash
# List all Docker networks
docker network ls

# Default networks created by Docker:
# bridge    — default for standalone containers
# host      — container shares host's network stack
# none      — no networking at all
```

---

### Q27. What are the types of Docker networks?

**Answer:**

| Network Type | Description | Use Case |
|---|---|---|
| **bridge** | Default. Containers on same bridge can communicate | Local dev, simple apps |
| **host** | Container shares host machine's network | High-performance, no isolation |
| **none** | No network access at all | Security-critical, offline batch jobs |
| **overlay** | Multi-host networking for Docker Swarm | Distributed/clustered environments |
| **macvlan** | Container gets its own MAC address | Legacy apps expecting direct network access |

```bash
# Create a custom bridge network
docker network create my-django-network

# Run containers on the same network (they can reach each other by name)
docker run -d --name db --network my-django-network postgres:15
docker run -d --name web --network my-django-network my-django-app
# Now 'web' can connect to 'db' using hostname 'db'
```

---

### Q28. How do containers communicate with each other?

**Answer:**

Containers on the **same Docker network** can communicate using their **container name as the hostname**. Docker has a built-in DNS that resolves container names to their IP addresses.

```yaml
# docker-compose.yml — Django connects to PostgreSQL
services:
  web:
    build: .
    environment:
      # Use 'db' as hostname — Docker DNS resolves it to the DB container's IP
      DATABASE_URL: postgres://user:pass@db:5432/mydb
                                         # ^^^ container name = hostname

  db:
    image: postgres:15
```

```python
# settings.py — Django database config
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.postgresql',
        'HOST': 'db',     # This resolves to the 'db' container's IP
        'PORT': '5432',
        'NAME': 'mydb',
        'USER': 'user',
        'PASSWORD': 'pass',
    }
}
```

🐍 **Django Analogy:**
> Docker DNS for container names is like Django's `INSTALLED_APPS` — you reference services by their name (`'db'`), and Docker figures out the actual address, just like Django resolves app names to actual module paths.

---

### Q29. What is port mapping/publishing in Docker?

**Answer:**

By default, container ports are only accessible **inside Docker's network**. **Port mapping** exposes a container port to the **host machine** (and potentially the internet).

Syntax: `-p host_port:container_port`

```bash
# Map host port 8000 to container port 8000
docker run -p 8000:8000 my-django-app

# Map host port 80 to container's port 8000
docker run -p 80:8000 my-django-app

# Bind to a specific host IP (only allow local access)
docker run -p 127.0.0.1:8000:8000 my-django-app

# Map multiple ports
docker run -p 8000:8000 -p 443:443 my-django-app

# Random host port (Docker chooses)
docker run -p 8000 my-django-app
```

```yaml
# In docker-compose.yml
services:
  web:
    ports:
      - "8000:8000"   # host:container
      - "127.0.0.1:8000:8000"  # bind to localhost only
```

---

### Q30. What is a Docker Volume and what problem does it solve?

**Answer:**

A **Docker Volume** is a **persistent storage mechanism** managed by Docker. It allows data to survive even after a container is stopped or removed.

Without volumes:
- Your PostgreSQL data is **lost** every time the container restarts
- Your Django media uploads are **lost** every time you redeploy
- Your logs are **lost** when the container dies

```bash
# Create a named volume
docker volume create postgres_data

# Use a volume when running a container
docker run -v postgres_data:/var/lib/postgresql/data postgres:15

# List volumes
docker volume ls

# Inspect a volume (find where data is stored on host)
docker volume inspect postgres_data
```

```yaml
# docker-compose.yml with persistent volumes
services:
  db:
    image: postgres:15
    volumes:
      - postgres_data:/var/lib/postgresql/data  # named volume

volumes:
  postgres_data:  # declare the named volume
```

---

### Q31. What are the three types of Docker storage mounts?

**Answer:**

| Mount Type | Managed By | Location | Use Case |
|---|---|---|---|
| **Volume** | Docker | Docker's storage area | Databases, persistent app data |
| **Bind Mount** | Host OS | Anywhere on host | Development (live code reload) |
| **tmpfs Mount** | Memory | RAM (not disk) | Sensitive temporary data |

```bash
# Named Volume (Docker manages it)
docker run -v postgres_data:/var/lib/postgresql/data postgres:15

# Bind Mount (you specify the host path)
docker run -v /home/user/myproject:/app my-django-app
# or with --mount (explicit and recommended)
docker run --mount type=bind,source=$(pwd),target=/app my-django-app

# tmpfs Mount (stored in RAM, not written to disk)
docker run --mount type=tmpfs,destination=/tmp my-django-app
```

🐍 **Django Dev Use Case:**
> **Bind mounts** are what you use during development! Mount your Django project directory into the container so that when you edit `views.py` on your host machine, the change is immediately reflected in the running container — no rebuild needed.

---

### Q32. What is the difference between a named volume and a bind mount?

**Answer:**

| Feature | Named Volume | Bind Mount |
|---|---|---|
| **Location** | Managed by Docker (hidden path) | Explicit host path you control |
| **Use in dev?** | Rarely | ✅ Yes — live code reload |
| **Use in prod?** | ✅ Yes — databases, persistent data | Rarely |
| **Portability** | ✅ Portable across machines | ❌ Path must exist on host |
| **Performance** | High | Slightly lower on macOS/Windows |

```yaml
# Development: bind mount for live reloading
services:
  web:
    volumes:
      - .:/app                 # Bind mount: local code → container

# Production: named volume for database persistence
services:
  db:
    volumes:
      - postgres_data:/var/lib/postgresql/data  # Named volume
```

---

### Q33. What is `docker-compose.yml` and what is its structure?

**Answer:**

`docker-compose.yml` is a **YAML file** that defines multi-container Docker applications. It's the single source of truth for your entire local infrastructure.

```yaml
version: "3.9"              # Compose file format version

services:                   # Each service = one container
  web:                      # Service name (used as hostname in Docker DNS)
    build:                  # Build from Dockerfile
      context: .
      dockerfile: Dockerfile
    image: my-django-app:dev
    container_name: django_web
    ports:
      - "8000:8000"
    environment:
      DEBUG: "True"
      DATABASE_URL: postgres://user:pass@db:5432/mydb
    env_file:               # Load from .env file
      - .env
    volumes:
      - .:/app              # Bind mount for dev
      - static_files:/app/staticfiles
    depends_on:
      db:
        condition: service_healthy  # Wait for DB to be healthy
    command: python manage.py runserver 0.0.0.0:8000
    networks:
      - backend

  db:
    image: postgres:15-alpine
    environment:
      POSTGRES_USER: user
      POSTGRES_PASSWORD: pass
      POSTGRES_DB: mydb
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U user -d mydb"]
      interval: 10s
      timeout: 5s
      retries: 5
    networks:
      - backend

volumes:
  postgres_data:
  static_files:

networks:
  backend:
    driver: bridge
```

---

### Q34. What does `depends_on` do in Docker Compose?

**Answer:**

`depends_on` tells Docker Compose the **startup order** of services. It ensures one service starts before another.

**Important caveat:** By default, `depends_on` only waits for the container to **start**, not for the service inside to be **ready** (e.g., PostgreSQL might be starting up but not yet accepting connections).

```yaml
services:
  web:
    depends_on:
      - db       # Waits for db container to START (not to be ready)

  db:
    image: postgres:15
```

**Solution — use health checks:**
```yaml
services:
  web:
    depends_on:
      db:
        condition: service_healthy  # Wait until DB passes health check

  db:
    image: postgres:15
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U user"]
      interval: 5s
      timeout: 3s
      retries: 5
```

🐍 **Django Analogy:**
> This is like Django signals — `depends_on: service_healthy` is saying "only start Django after the `post_save` signal from the database confirms it's ready."

---

### Q35. How do you pass environment variables to Docker containers?

**Answer:**

Three methods, from simplest to most secure:

**Method 1: `-e` flag (command line)**
```bash
docker run -e DEBUG=True -e SECRET_KEY=mysecret my-django-app
```

**Method 2: `--env-file` flag**
```bash
# .env file
DEBUG=True
SECRET_KEY=supersecretkey
DATABASE_URL=postgres://user:pass@db:5432/mydb

# Run with env file
docker run --env-file .env my-django-app
```

**Method 3: In `docker-compose.yml`**
```yaml
services:
  web:
    environment:            # Direct values (visible in compose file)
      DEBUG: "True"
    env_file:               # Load from file (keep secrets out of compose)
      - .env
```

🐍 **Django Best Practice:**
> Use `django-environ` or `python-decouple` to read these environment variables in `settings.py`. Your `settings.py` should never hardcode credentials — it should read from `os.environ` or a `.env` file.

```python
# settings.py with django-environ
import environ
env = environ.Env()
environ.Env.read_env()  # Reads .env file

SECRET_KEY = env('SECRET_KEY')
DATABASE_URL = env('DATABASE_URL')
```

---

### Q36. What are Docker Compose override files?

**Answer:**

Docker Compose supports **multiple compose files** that are **merged together**. The `docker-compose.override.yml` file is automatically loaded alongside `docker-compose.yml` and merges with it.

```yaml
# docker-compose.yml (base — works in all environments)
services:
  web:
    build: .
    environment:
      DATABASE_URL: postgres://user:pass@db:5432/mydb

  db:
    image: postgres:15
```

```yaml
# docker-compose.override.yml (auto-loaded in development)
services:
  web:
    volumes:
      - .:/app                    # Live code reloading
    command: python manage.py runserver 0.0.0.0:8000
    environment:
      DEBUG: "True"
```

```yaml
# docker-compose.prod.yml (production — load explicitly)
services:
  web:
    command: gunicorn myproject.wsgi:application --workers 4
    environment:
      DEBUG: "False"
```

```bash
# Development (base + override.yml are auto-merged)
docker compose up

# Production (explicitly specify files)
docker compose -f docker-compose.yml -f docker-compose.prod.yml up
```

---

### Q37. How do you scale services with Docker Compose?

**Answer:**

You can run **multiple instances** (replicas) of a service using `--scale`.

```bash
# Scale the web service to 3 instances
docker compose up --scale web=3

# This creates: web_1, web_2, web_3
```

**Important:** When scaling, you **cannot** use `container_name` (it must be unique) and you should use a **port range** or a **load balancer** (Nginx) instead of a direct port mapping.

```yaml
# docker-compose.yml configured for scaling
services:
  web:
    build: .
    # No 'container_name' — let Docker auto-name them
    # No fixed port like '8000:8000' — use nginx as load balancer
    expose:
      - "8000"        # Expose to other containers, not to host

  nginx:
    image: nginx:alpine
    ports:
      - "80:80"       # Only nginx is exposed to the host
```

---

### Q38. What is the difference between `EXPOSE` and port publishing (`-p`)?

**Answer:**

| Feature | `EXPOSE` | `-p` / `ports:` |
|---|---|---|
| **What it does** | Documents the port | Actually publishes the port to host |
| **Makes port accessible?** | No — it's just metadata/documentation | ✅ Yes — accessible from host |
| **Required for containers to communicate?** | No — containers on same network can communicate without it | Not needed for container-to-container |

```dockerfile
# Dockerfile
EXPOSE 8000   # This is documentation — "hey, this app uses port 8000"
```

```bash
# This actually makes it accessible from your host browser
docker run -p 8000:8000 my-django-app
```

🐍 **Analogy:**
> `EXPOSE` is like a **docstring** on a Python function — it tells humans (and tools) what port the app uses, but doesn't actually do anything functional. `-p` is the actual implementation that connects the port.

---

### Q39. How do you use Docker health checks?

**Answer:**

A **health check** is a command Docker runs periodically to verify that a container is working correctly — not just that it's running, but that it's *healthy*.

```dockerfile
# Dockerfile — define health check
HEALTHCHECK --interval=30s --timeout=10s --start-period=5s --retries=3 \
  CMD curl -f http://localhost:8000/health/ || exit 1
```

```yaml
# docker-compose.yml — define health check
services:
  web:
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8000/health/"]
      interval: 30s      # How often to check
      timeout: 10s       # How long to wait for response
      start_period: 40s  # Grace period at startup (don't fail during startup)
      retries: 3         # Failures before marking as unhealthy

  db:
    image: postgres:15
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER} -d ${POSTGRES_DB}"]
      interval: 10s
      retries: 5
```

🐍 **Django Best Practice:**
> Create a simple `/health/` endpoint in Django that returns HTTP 200 and checks DB connectivity:

```python
# views.py
from django.db import connection
from django.http import JsonResponse

def health_check(request):
    connection.ensure_connection()
    return JsonResponse({"status": "healthy"})
```

---

### Q40. What is a multi-stage build and why is it useful?

**Answer:**

A **multi-stage build** uses **multiple `FROM` statements** in a single Dockerfile. Each stage can use a different base image. The key benefit: you can **copy only the necessary artifacts** from a build stage into the final image, leaving behind build tools — resulting in a **much smaller final image**.

```dockerfile
# ── Stage 1: Builder ─────────────────────────────
FROM python:3.11-slim AS builder

WORKDIR /app
COPY requirements.txt .

# Install dependencies into a virtual environment
RUN python -m venv /opt/venv
ENV PATH="/opt/venv/bin:$PATH"
RUN pip install --no-cache-dir -r requirements.txt

# ── Stage 2: Production Image ─────────────────────
FROM python:3.11-slim AS production

# Copy ONLY the virtual environment from the builder stage
COPY --from=builder /opt/venv /opt/venv

ENV PATH="/opt/venv/bin:$PATH"

WORKDIR /app
COPY . .

# Create non-root user
RUN adduser --disabled-password --gecos '' appuser
USER appuser

CMD ["gunicorn", "myproject.wsgi:application", "--workers", "4", "--bind", "0.0.0.0:8000"]
```

**Result:** The final image contains only Python + your venv + your code — no pip, no build tools, no cache. Images can be **50-80% smaller**.

---

### Q41. How do you handle secrets in Docker without hardcoding them?

**Answer:**

**Never** put secrets (passwords, API keys, tokens) directly in your Dockerfile or docker-compose.yml (they end up in version control and image layers).

**Methods from least to most secure:**

**Method 1: `.env` file (development)**
```bash
# .env (add to .gitignore!)
DATABASE_PASSWORD=supersecret
SECRET_KEY=django-secret-key

# docker-compose.yml
env_file: .env
```

**Method 2: Docker Secrets (Swarm mode)**
```bash
echo "supersecret" | docker secret create db_password -
```

```yaml
services:
  db:
    secrets:
      - db_password
secrets:
  db_password:
    external: true
```

**Method 3: External Secret Managers (Production Best Practice)**
- AWS Secrets Manager
- HashiCorp Vault
- GCP Secret Manager
- Azure Key Vault

These inject secrets at runtime as environment variables, not stored in images at all.

---

### Q42. What is the `docker cp` command?

**Answer:**

`docker cp` copies files between a container and the host machine — in both directions.

```bash
# Copy from container to host
docker cp my-container:/app/logs/error.log ./local-error.log

# Copy from host to container
docker cp ./local-config.py my-container:/app/config.py

# Copy a directory
docker cp my-container:/app/staticfiles ./staticfiles/
```

🐍 **Django Use Case:**
```bash
# Grab the Django database dump from inside the container
docker exec my-container python manage.py dumpdata > backup.json
docker cp my-container:/app/backup.json ./backup.json
```

---

### Q43. What does `docker ps` show and what are its useful flags?

**Answer:**

`docker ps` lists **running containers**. `docker ps -a` lists **all containers** (including stopped ones).

```bash
# List running containers
docker ps

# List all containers
docker ps -a

# Show only container IDs (useful for scripting)
docker ps -q

# Show last N created containers
docker ps -n 5

# Filter by status
docker ps -f status=exited

# Filter by name
docker ps -f name=django

# Custom format
docker ps --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"
```

**Output columns explained:**
| Column | Meaning |
|---|---|
| `CONTAINER ID` | Unique short ID |
| `IMAGE` | Image it was created from |
| `COMMAND` | Command being run |
| `CREATED` | When it was created |
| `STATUS` | Current state + uptime |
| `PORTS` | Port mappings |
| `NAMES` | Container name |

---

### Q44. What is `docker pull` vs `docker push`?

**Answer:**

```bash
# PULL: Download an image from a registry to your local machine
docker pull python:3.11-slim
docker pull 123456789.dkr.ecr.us-east-1.amazonaws.com/my-app:latest

# PUSH: Upload your local image to a registry
docker login
docker tag my-django-app:v1 yourusername/my-django-app:v1
docker push yourusername/my-django-app:v1
```

🐍 **Git Analogy:**
> `docker pull` = `git pull` (download). `docker push` = `git push` (upload). The registry is like GitHub — a remote location to store and share your images.

**Full workflow:**
```bash
# 1. Build your image
docker build -t my-django-app:v1.0 .

# 2. Tag it for your registry
docker tag my-django-app:v1.0 yourusername/my-django-app:v1.0

# 3. Push to Docker Hub
docker push yourusername/my-django-app:v1.0

# 4. On your server, pull and run it
docker pull yourusername/my-django-app:v1.0
docker run -d yourusername/my-django-app:v1.0
```

---

### Q45. What is `docker save` and `docker load`?

**Answer:**

These commands transfer images as **tar archive files** — useful when you can't use a registry (air-gapped environments, secure networks).

```bash
# Save an image to a tar file
docker save -o my-django-app.tar my-django-app:v1.0

# Load an image from a tar file
docker load -i my-django-app.tar

# Save multiple images
docker save -o all-images.tar my-django-app:v1 postgres:15 redis:7
```

🐍 **Analogy:**
> `docker save` is like creating a Python wheel file (`.whl`) — packaging everything into a portable file. `docker load` is like `pip install mypackage.whl` — installing from that file directly without going to PyPI.

---

### Q46. What is `docker stats`?

**Answer:**

`docker stats` shows **real-time resource usage** (CPU, memory, network I/O, disk I/O) for running containers — like `htop` or `top` but for containers.

```bash
# Live stats for all containers
docker stats

# Stats for specific containers
docker stats web db redis

# Single snapshot (no live update)
docker stats --no-stream

# Custom format
docker stats --format "table {{.Name}}\t{{.CPUPerc}}\t{{.MemUsage}}"
```

**Output explained:**
| Column | Meaning |
|---|---|
| `CPU %` | CPU usage percentage |
| `MEM USAGE / LIMIT` | Memory used / Memory limit |
| `MEM %` | Memory usage percentage |
| `NET I/O` | Network bytes in/out |
| `BLOCK I/O` | Disk bytes read/written |
| `PIDS` | Number of processes in container |

---

### Q47. How do you limit resources (CPU and memory) for a container?

**Answer:**

Resource limits prevent a misbehaving container from consuming all host resources.

```bash
# Limit memory to 512MB
docker run --memory 512m my-django-app

# Limit CPU to 0.5 cores
docker run --cpus 0.5 my-django-app

# Combine limits
docker run --memory 1g --cpus 1.0 my-django-app
```

```yaml
# docker-compose.yml
services:
  web:
    deploy:
      resources:
        limits:
          memory: 512M
          cpus: "0.5"
        reservations:
          memory: 256M
          cpus: "0.25"
```

🐍 **Django Use Case:**
> Limit your Celery worker containers so a runaway task doesn't kill the entire host. For example: `--memory 256m --cpus 0.5` per worker.

---

### Q48. What is `docker network inspect` and how do you debug network issues?

**Answer:**

```bash
# List all networks
docker network ls

# Inspect a network (see all containers connected to it)
docker network inspect bridge
docker network inspect my-django-network

# Connect a running container to a network
docker network connect my-network my-container

# Disconnect a container from a network
docker network disconnect my-network my-container

# Debug: ping from inside container to another container
docker exec my-web ping db

# Debug: test if port is reachable
docker exec my-web nc -zv db 5432

# Debug: check DNS resolution
docker exec my-web nslookup db
```

---

### Q49. What is an init process and why is it needed in Docker containers?

**Answer:**

The **init process** (PID 1) in a Unix system is responsible for:
- Reaping zombie processes (dead child processes)
- Forwarding signals to child processes

The problem: When your app is PID 1 in a container and gets a `SIGTERM` (e.g., from `docker stop`), many apps (including Django's `manage.py`) don't handle signals properly.

**Solutions:**
```dockerfile
# Option 1: Use tini (lightweight init system)
FROM python:3.11-slim
RUN apt-get install -y tini
ENTRYPOINT ["/usr/bin/tini", "--"]
CMD ["gunicorn", "myproject.wsgi:application"]

# Option 2: Docker's built-in init
# docker run --init my-django-app
```

```yaml
# docker-compose.yml
services:
  web:
    init: true  # Use Docker's built-in tini
```

---

### Q50. What is Docker's default bridge network vs a custom bridge network?

**Answer:**

| Feature | Default Bridge | Custom Bridge |
|---|---|---|
| **Container DNS** | ❌ No (use IP addresses) | ✅ Yes (use container names) |
| **Auto-created** | ✅ Yes | ❌ You must create it |
| **Isolation** | Shares with all containers | Only connected containers |
| **Recommended?** | Development/testing only | ✅ Always use in production |

```bash
# Default bridge — containers can't reach each other by name
docker run -d --name db postgres:15
docker run -d --name web my-django-app
# web CANNOT connect to db using hostname 'db'

# Custom bridge — containers can reach each other by name
docker network create mynet
docker run -d --name db --network mynet postgres:15
docker run -d --name web --network mynet my-django-app
# web CAN connect to db using hostname 'db' ✅
```

Docker Compose **automatically creates a custom bridge network** for each project — that's why container DNS works in Compose without any extra configuration.

---

### Q51. What is `docker commit` and when should you avoid it?

**Answer:**

`docker commit` creates a new image from a **running container's current state**. It captures all changes made to the container's filesystem.

```bash
# Commit a container to a new image
docker commit my-container my-custom-image:v1

# With metadata
docker commit --message "Added numpy" --author "Dev" my-container my-image:v2
```

**⚠️ Why you should almost NEVER use it:**
- The resulting image is a **black box** — no Dockerfile, no reproducibility
- Changes can't be reviewed or version-controlled
- Violates the "infrastructure as code" principle
- Security scanning tools can't analyze what's inside

**Rule:** Always use a `Dockerfile` instead. `docker commit` is only useful for debugging/experimentation — never for production images.

🐍 **Django Analogy:**
> `docker commit` is like manually editing production database data directly in psql instead of writing a proper Django migration. It works, but it's untrackable, unrepeatable, and dangerous.

---

### Q52. What is `docker diff`?

**Answer:**

`docker diff` shows which files have been **added (A), changed (C), or deleted (D)** in a container's filesystem compared to its original image.

```bash
docker diff my-container

# Output example:
# C /app
# A /app/newfile.py      ← Added
# C /app/views.py        ← Changed
# D /app/old_config.py   ← Deleted
```

Useful for:
- Debugging what a running container has changed
- Understanding what `docker commit` would capture
- Security auditing

---

### Q53. What is Docker's layer caching mechanism and how do you optimize it?

**Answer:**

Docker caches each layer and **only rebuilds layers that have changed** (and all layers after them). The cache is **invalidated** by:
1. Changing a Dockerfile instruction
2. Changing a file in a `COPY` instruction

**Optimization strategy: Order instructions from LEAST to MOST frequently changing**

```dockerfile
# ❌ BAD — Code changes invalidate pip install (rebuilds every time!)
FROM python:3.11-slim
WORKDIR /app
COPY . .                            # Code changes break cache here
RUN pip install -r requirements.txt # ← Runs every time you change any code

# ✅ GOOD — pip install is cached unless requirements.txt changes
FROM python:3.11-slim
WORKDIR /app
COPY requirements.txt .             # Only changes when dependencies change
RUN pip install -r requirements.txt # ← Cached! Only runs when requirements.txt changes
COPY . .                            # Code changes only invalidate this layer
```

**Result:** With the good pattern, changing `views.py` only rebuilds the last `COPY . .` layer — no pip reinstall.

---

### Q54. How do you run one-off commands with Docker Compose?

**Answer:**

`docker compose run` starts a **new container** for a service and runs a command. It's different from `docker compose exec` (which runs in an existing container).

```bash
# Run Django migrations (common use case!)
docker compose run --rm web python manage.py migrate

# Create a Django superuser
docker compose run --rm web python manage.py createsuperuser

# Open Django shell
docker compose run --rm web python manage.py shell

# Run tests
docker compose run --rm web python manage.py test

# Collect static files
docker compose run --rm web python manage.py collectstatic --noinput

# --rm: automatically remove the container after it exits
```

🐍 **This is your day-to-day workflow** as a Django developer using Docker:
```bash
docker compose up -d db              # Start just the database
docker compose run --rm web python manage.py migrate    # Run migrations
docker compose up -d web             # Start the web server
docker compose run --rm web python manage.py createsuperuser
```

---

### Q55. What is the difference between `docker compose up`, `docker compose start`, and `docker compose run`?

**Answer:**

| Command | What it does |
|---|---|
| `docker compose up` | Create + start containers (builds if needed) |
| `docker compose start` | Start **already created** but stopped containers |
| `docker compose run` | Run a **one-off command** in a new container |
| `docker compose exec` | Run a command in an **already running** container |

```bash
# First time setup: creates containers and starts them
docker compose up -d

# Stop containers (keeps them)
docker compose stop

# Restart the stopped containers
docker compose start

# Run a one-off migration
docker compose run --rm web python manage.py migrate

# Execute a command in running web container
docker compose exec web python manage.py shell
```

---

# 🔴 Phase 3: Advanced Architecture, Volumes & Security
### *(Deep internals, production patterns, security hardening)*

---

### Q56. What are Docker namespaces?

**Answer:**

**Namespaces** are the core Linux kernel feature that makes container isolation possible. Docker uses namespaces to create isolated environments where each container thinks it's the only process on the machine.

| Namespace | What it isolates |
|---|---|
| **pid** | Process IDs (container sees only its own processes) |
| **net** | Network interfaces, routing tables, ports |
| **mnt** | Filesystem mount points |
| **uts** | Hostname and domain name |
| **ipc** | Inter-process communication (shared memory, semaphores) |
| **user** | User and group IDs |

🐍 **Python Analogy:**
> Namespaces are like Python's module namespaces. Inside a module, `os` refers to Python's `os` module. Inside a container's namespace, `PID 1` refers to the container's init process — even though the host machine has a completely different PID 1. Same name, different isolated context.

---

### Q57. What are Linux cgroups (control groups) and how do they relate to Docker?

**Answer:**

**cgroups (Control Groups)** are a Linux kernel feature that **limits, accounts for, and isolates** resource usage (CPU, memory, disk I/O, network) for a group of processes.

While **namespaces** provide *isolation* (hiding), **cgroups** provide *resource control* (limiting).

Together:
- **Namespaces** = "What can a container see?" (process isolation)
- **cgroups** = "How much can a container use?" (resource limits)

```bash
# When you run this Docker command:
docker run --memory 512m --cpus 0.5 my-django-app

# Docker creates:
# 1. A namespace (isolated view of the system)
# 2. A cgroup (limits: max 512MB RAM, max 0.5 CPU cores)
```

---

### Q58. What is the Union File System (UnionFS) in Docker?

**Answer:**

**UnionFS** is the technology that enables Docker's layered image system. It allows files and directories from separate layers to be **transparently overlaid**, forming a single coherent filesystem.

```
Container Layer (writable)   ← Your container's changes go here
─────────────────────────────
Layer 4: COPY . .            (your application code)
Layer 3: RUN pip install     (installed packages)
Layer 2: WORKDIR /app        
Layer 1: FROM python:3.11    (base Python image — shared with ALL Python containers)
```

**Key property:** Layers are **read-only and shared**. Multiple containers can use the same base image layers without duplicating them on disk. Only the writable container layer is unique per container.

**Storage drivers** implement UnionFS:
- `overlay2` (most common, recommended)
- `devicemapper`
- `btrfs`
- `aufs` (legacy)

---

### Q59. What is Docker Swarm?

**Answer:**

**Docker Swarm** is Docker's **native container orchestration** system for managing a cluster of Docker nodes (servers). It allows you to deploy and scale containers across **multiple machines**.

```
                 ┌──────────────────────┐
                 │   Swarm Manager Node │
                 │   (orchestrates)     │
                 └────────┬─────────────┘
                          │
           ┌──────────────┼──────────────┐
           ▼              ▼              ▼
     Worker Node 1   Worker Node 2   Worker Node 3
     (runs tasks)    (runs tasks)    (runs tasks)
```

```bash
# Initialize a Swarm
docker swarm init

# Deploy a stack
docker stack deploy -c docker-compose.yml mystack

# Scale a service
docker service scale mystack_web=5

# List services
docker service ls
```

**Docker Swarm vs Kubernetes:**
- Swarm is simpler to set up, built into Docker
- Kubernetes is more powerful, more complex, industry standard for large scale
- For most Django apps: Swarm is sufficient; enterprise-scale: use Kubernetes

---

### Q60. What is Docker Content Trust (DCT)?

**Answer:**

**Docker Content Trust (DCT)** is a security feature that uses **cryptographic signing** to verify the authenticity and integrity of Docker images. It ensures you're running the exact image that was published, not a tampered version.

```bash
# Enable Docker Content Trust
export DOCKER_CONTENT_TRUST=1

# Now all pulls/pushes are signed and verified
docker pull python:3.11-slim   # Verifies signature
docker push myimage:v1         # Signs the image

# Disable (NOT recommended in production)
export DOCKER_CONTENT_TRUST=0
```

🐍 **Python Analogy:**
> DCT is like verifying a Python package's checksum after downloading it from PyPI — ensuring the file wasn't tampered with in transit. `pip` does this automatically; Docker Content Trust does it for images.

---

### Q61. What is the principle of least privilege in Docker security?

**Answer:**

**Principle of Least Privilege** means giving a process only the minimum permissions it needs to function. In Docker:

1. **Run as non-root user**
```dockerfile
# Create a non-root user and switch to it
RUN addgroup --system appgroup && adduser --system --group appuser
USER appuser
```

2. **Drop Linux capabilities**
```bash
# Drop ALL capabilities, add back only what's needed
docker run --cap-drop ALL --cap-add NET_BIND_SERVICE my-app
```

3. **Read-only filesystem**
```bash
docker run --read-only --tmpfs /tmp my-app
```

4. **No privileged mode**
```bash
# NEVER do this in production:
docker run --privileged my-app  # Gives full host access — dangerous!
```

5. **Use `seccomp` profiles**
```bash
# Restrict system calls the container can make
docker run --security-opt seccomp=my-seccomp-profile.json my-app
```

---

### Q62. Why should you never run containers as root?

**Answer:**

Running as root inside a container is dangerous because:

1. **Container escape**: If an attacker breaks out of the container, they have **root access on the host**
2. **File permission issues**: Files created in volumes are owned by root, causing permission headaches
3. **Violates least privilege**: Your Django app doesn't need root to serve HTTP

```dockerfile
# ✅ Secure Dockerfile
FROM python:3.11-slim

WORKDIR /app

# Install dependencies as root (need to write to system directories)
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

# Create non-root user
RUN addgroup --system --gid 1001 django && \
    adduser --system --uid 1001 --gid 1001 --no-create-home django

# Change ownership of app directory
RUN chown -R django:django /app

# Switch to non-root user
USER django

CMD ["gunicorn", "myproject.wsgi:application", "--bind", "0.0.0.0:8000"]
```

---

### Q63. What is a distroless image?

**Answer:**

**Distroless images** (by Google) contain only your application and its runtime dependencies — **no shell, no package manager, no OS utilities**. This dramatically reduces the attack surface.

```dockerfile
# Multi-stage: build with full Python image, run with distroless
FROM python:3.11-slim AS builder
WORKDIR /app
COPY requirements.txt .
RUN pip install --target=/app/packages -r requirements.txt
COPY . .

FROM gcr.io/distroless/python3-debian12 AS production
WORKDIR /app
COPY --from=builder /app /app
ENV PYTHONPATH=/app/packages
CMD ["myproject/wsgi.py"]
```

**Benefits:**
- Much smaller images
- Massively reduced attack surface
- No shell = can't `exec` into container (security feature AND inconvenience)
- Fewer CVEs (no bash, no curl, no apt)

**Tradeoff:** Very hard to debug since you can't `docker exec` into a shell.

---

### Q64. What are Docker secrets and how do they differ from environment variables?

**Answer:**

**Docker Secrets** (available in Swarm mode) store sensitive data as **encrypted files** mounted in containers at `/run/secrets/`. They are more secure than environment variables because:

- Secrets are **encrypted at rest and in transit**
- Secrets are **not visible** in `docker inspect` or `docker ps`
- Secrets are **only available to services** that are explicitly granted access
- Secrets are stored in memory (tmpfs), not on disk

```bash
# Create a secret
echo "my-super-secret-db-password" | docker secret create db_password -

# Use in a service
docker service create \
  --name my-django-app \
  --secret db_password \
  my-django-app:latest
```

```yaml
# docker-compose.yml (Swarm mode)
services:
  web:
    secrets:
      - db_password

secrets:
  db_password:
    external: true  # Pre-created secret
```

Inside the container, the secret is at `/run/secrets/db_password`:
```python
# settings.py — read secret from file
with open('/run/secrets/db_password', 'r') as f:
    DB_PASSWORD = f.read().strip()
```

---

### Q65. What is image vulnerability scanning?

**Answer:**

**Image scanning** analyzes Docker images for known **CVEs (Common Vulnerabilities and Exposures)** in OS packages and application dependencies. It's a critical DevSecOps practice.

```bash
# Docker Scout (built into Docker Desktop)
docker scout cves python:3.11-slim

# Trivy (open source, popular)
trivy image python:3.11-slim
trivy image my-django-app:v1.0

# Snyk
snyk container test my-django-app:v1.0
```

**Integration in CI/CD:**
```yaml
# GitHub Actions — scan on every push
- name: Scan image for vulnerabilities
  uses: aquasecurity/trivy-action@master
  with:
    image-ref: 'my-django-app:latest'
    severity: 'CRITICAL,HIGH'
    exit-code: '1'  # Fail the build if found
```

🐍 **Django Connection:**
> Just like you run `pip audit` or `safety check` to scan your Python dependencies for vulnerabilities, `trivy image` scans your entire Docker image (including OS packages and Python packages).

---

### Q66. What is a Docker BuildKit?

**Answer:**

**BuildKit** is Docker's modern, next-generation image build engine (default since Docker 23.0). It offers significant improvements over the classic builder:

| Feature | Classic Builder | BuildKit |
|---|---|---|
| **Parallel build steps** | ❌ Sequential | ✅ Parallel |
| **Build cache** | Basic | ✅ Advanced (remote cache) |
| **Build secrets** | Not supported | ✅ `--secret` flag |
| **SSH forwarding** | Not supported | ✅ For private repos |
| **`--no-cache-filter`** | Not available | ✅ Available |

```bash
# Enable BuildKit (older Docker versions)
DOCKER_BUILDKIT=1 docker build -t my-app .

# Modern Docker (23.0+) — BuildKit is default
docker build -t my-app .

# Build with a secret (never baked into image layers)
docker build \
  --secret id=github_token,env=GITHUB_TOKEN \
  -t my-app .
```

```dockerfile
# Use a secret in Dockerfile (not stored in image!)
RUN --mount=type=secret,id=github_token \
    pip install git+https://$(cat /run/secrets/github_token)@github.com/org/private-pkg.git
```

---

### Q67. What is Docker's `--no-cache` flag?

**Answer:**

`--no-cache` forces Docker to **rebuild every layer** from scratch, ignoring all cached layers. Useful when you suspect stale cache is causing issues.

```bash
# Build completely from scratch
docker build --no-cache -t my-django-app .

# Force-pull fresh base images too
docker build --no-cache --pull -t my-django-app .
```

**When to use:**
- After adding new system packages (cache may have old `apt-get update`)
- Debugging caching issues
- When a `RUN curl` or external dependency may have changed
- In CI/CD when you want guaranteed fresh builds

**When NOT to use:** Local development — you'll waste minutes waiting for pip to reinstall everything.

---

### Q68. What is a Docker entrypoint script?

**Answer:**

An **entrypoint script** is a shell script used as the container's `ENTRYPOINT`. It runs setup tasks before the main application starts — like an `__init__` method for your container.

```bash
#!/bin/bash
# entrypoint.sh

set -e  # Exit on any error

echo "Running database migrations..."
python manage.py migrate --noinput

echo "Collecting static files..."
python manage.py collectstatic --noinput --clear

echo "Starting application..."
exec "$@"   # Execute the CMD (passed as arguments to this script)
```

```dockerfile
# Dockerfile
COPY entrypoint.sh /entrypoint.sh
RUN chmod +x /entrypoint.sh

ENTRYPOINT ["/entrypoint.sh"]
CMD ["gunicorn", "myproject.wsgi:application", "--bind", "0.0.0.0:8000"]
```

🐍 **Django Analogy:**
> The entrypoint script is like Django's `AppConfig.ready()` method — it runs initialization logic before the application is fully started.

---

### Q69. What is Docker image tagging strategy for production?

**Answer:**

A good tagging strategy ensures **traceability, rollback capability, and clarity**.

**Common strategies:**

```bash
# ❌ Anti-pattern: Only using :latest
docker build -t myapp:latest .  # Can't roll back!

# ✅ Semantic Versioning
docker build -t myapp:1.2.3 .
docker build -t myapp:1.2 .
docker build -t myapp:latest .

# ✅ Git SHA (best for CI/CD traceability)
docker build -t myapp:$(git rev-parse --short HEAD) .
# Result: myapp:a3f8c21

# ✅ Combined (version + git SHA)
docker build -t myapp:v1.2.3-a3f8c21 .

# ✅ Environment-specific tags
docker build -t myapp:staging .
docker build -t myapp:production .
```

**GitOps strategy in CI/CD:**
```bash
# Tag with branch name + commit SHA
IMAGE_TAG="${BRANCH_NAME}-$(git rev-parse --short HEAD)"
docker build -t myapp:${IMAGE_TAG} .
docker push myapp:${IMAGE_TAG}
```

---

### Q70. What is a Docker registry mirror?

**Answer:**

A **registry mirror** is a **cached copy** of Docker Hub (or other registries) running locally or on your network. It speeds up pulls by caching images locally and reduces bandwidth costs.

**Why use it?**
- Docker Hub has **pull rate limits** (100 pulls/6 hours for unauthenticated, 200/6h for free accounts)
- Faster builds in CI/CD (pull from local cache, not the internet)
- Works in air-gapped environments

```json
// /etc/docker/daemon.json
{
  "registry-mirrors": [
    "https://mirror.gcr.io",
    "https://your-internal-mirror.company.com"
  ]
}
```

---

### Q71. How do you handle database migrations in a Dockerized Django application?

**Answer:**

This is one of the most important architectural decisions for Dockerized Django. You should **never run migrations as part of `CMD`** because:
- If you have multiple replicas, all would run migrations simultaneously (race conditions)
- The app container starts before migrations finish

**Recommended patterns:**

**Pattern 1: Entrypoint script (for single-instance apps)**
```bash
# entrypoint.sh
python manage.py migrate --noinput
exec gunicorn myproject.wsgi:application
```

**Pattern 2: Separate migration job (for production/multi-instance)**
```yaml
# docker-compose.yml
services:
  migrate:
    build: .
    command: python manage.py migrate --noinput
    depends_on:
      db:
        condition: service_healthy

  web:
    build: .
    command: gunicorn myproject.wsgi:application
    depends_on:
      migrate:
        condition: service_completed_successfully
```

**Pattern 3: Kubernetes Init Container**
```yaml
initContainers:
  - name: migrate
    image: my-django-app:v1.2
    command: ["python", "manage.py", "migrate", "--noinput"]
```

---

### Q72. What is a dangling image?

**Answer:**

A **dangling image** is an image that:
- Has **no tag** (shows as `<none>:<none>` in `docker images`)
- Is no longer referenced by any container

They accumulate over time when you rebuild images with the same tag — the old image loses its tag (which moves to the new build) and becomes dangling.

```bash
# See dangling images
docker images -f dangling=true

# Remove dangling images
docker image prune

# Remove ALL unused images (dangling + unreferenced)
docker image prune -a

# Automatic cleanup in CI/CD
docker system prune -f  # Remove dangling images, stopped containers, unused networks
```

🐍 **Analogy:**
> Dangling images are like orphaned Python `.pyc` files — they accumulate invisibly, waste disk space, and serve no purpose. Regular cleanup (`find . -name "*.pyc" -delete` → `docker image prune`) is a good habit.

---

### Q73. What is `docker system df`?

**Answer:**

`docker system df` shows how much **disk space** Docker objects are using — images, containers, volumes, and build cache.

```bash
docker system df

# Output:
# TYPE            TOTAL     ACTIVE    SIZE      RECLAIMABLE
# Images          15        3         4.23GB    3.1GB (73%)
# Containers      6         2         1.2MB     0B (0%)
# Local Volumes   8         3         2.15GB    1.2GB (55%)
# Build Cache     47        0         890MB     890MB

# Verbose breakdown
docker system df -v
```

Use this regularly to understand disk usage and decide what to prune.

---

### Q74. What is the difference between `docker compose down` and `docker compose stop`?

**Answer:**

| Command | Stops Containers | Removes Containers | Removes Networks | Removes Volumes |
|---|---|---|---|---|
| `docker compose stop` | ✅ | ❌ | ❌ | ❌ |
| `docker compose down` | ✅ | ✅ | ✅ | ❌ |
| `docker compose down -v` | ✅ | ✅ | ✅ | ✅ |

```bash
# Just stop containers (can restart with 'docker compose start')
docker compose stop

# Stop + remove containers and networks (volumes preserved)
docker compose down

# ⚠️ Stop + remove EVERYTHING including your DB data
docker compose down -v  # Use with caution!
```

---

### Q75. What is OCI (Open Container Initiative)?

**Answer:**

The **Open Container Initiative (OCI)** is a Linux Foundation project that defines **open standards** for container formats and runtimes. It ensures containers from different tools are interoperable.

**OCI Standards:**
- **Image Spec**: Defines the format of container images
- **Runtime Spec**: Defines how to run containers (`runc` is the reference implementation)
- **Distribution Spec**: Defines how registries distribute images

**Why it matters:**
- Docker images are OCI-compliant — they work with Kubernetes, Podman, and any OCI-compatible tool
- `containerd` (Docker's runtime) is OCI-compliant
- Kubernetes uses OCI images natively

🐍 **Python Analogy:**
> OCI is like Python's **WSGI standard** — a common interface that ensures Django, Flask, and FastAPI apps all work with any WSGI-compatible server (Gunicorn, uWSGI, Waitress) without modification.

---

### Q76. What is `containerd` and how does it relate to Docker?

**Answer:**

`containerd` is a **high-level container runtime** — it's the core engine that actually manages the full container lifecycle (image storage, container execution, networking). Docker is built on top of `containerd`.

```
Docker CLI
    │
Docker Daemon (dockerd)
    │
containerd      ← industry-standard runtime
    │
runc            ← low-level OCI runtime (actually creates containers)
    │
Linux Kernel (namespaces + cgroups)
```

**Why this matters:**
- Kubernetes dropped Docker as its container runtime in 2020 (because it didn't need all of Docker's features) and uses `containerd` or `CRI-O` directly
- Docker images still work perfectly — they're OCI-compliant
- Understanding this helps in Kubernetes interviews

---

### Q77. What is Dockerfile best practices for production images?

**Answer:**

```dockerfile
# ✅ Production-Ready Dockerfile

# 1. Pin specific versions (never :latest)
FROM python:3.11.9-slim-bookworm AS base

# 2. Set environment variables to prevent Python buffering
ENV PYTHONUNBUFFERED=1 \
    PYTHONDONTWRITEBYTECODE=1 \
    PIP_NO_CACHE_DIR=1 \
    PIP_DISABLE_PIP_VERSION_CHECK=1

WORKDIR /app

# 3. Install system dependencies separately (changes rarely)
RUN apt-get update && apt-get install -y --no-install-recommends \
    libpq-dev \
    && rm -rf /var/lib/apt/lists/*  # Clean apt cache

# 4. Copy and install requirements BEFORE app code (cache optimization)
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# 5. Copy application code
COPY . .

# 6. Create and use non-root user
RUN addgroup --system --gid 1001 django && \
    adduser --system --uid 1001 --gid 1001 --no-create-home django && \
    chown -R django:django /app

USER django

# 7. Expose port (documentation)
EXPOSE 8000

# 8. Health check
HEALTHCHECK --interval=30s --timeout=10s --start-period=60s --retries=3 \
    CMD curl -f http://localhost:8000/health/ || exit 1

# 9. Use exec form (not shell form) for CMD
CMD ["gunicorn", "myproject.wsgi:application", \
     "--bind", "0.0.0.0:8000", \
     "--workers", "4", \
     "--timeout", "120"]
```

---

### Q78. What is `PYTHONUNBUFFERED` and `PYTHONDONTWRITEBYTECODE` in Docker?

**Answer:**

These are Python-specific environment variables commonly set in Django Dockerfiles:

```dockerfile
ENV PYTHONUNBUFFERED=1
ENV PYTHONDONTWRITEBYTECODE=1
```

**`PYTHONUNBUFFERED=1`:**
- By default, Python **buffers stdout and stderr** output (batches it before sending)
- In a container, this means `print()` statements may not appear in `docker logs` immediately
- Setting `PYTHONUNBUFFERED=1` forces **immediate output** — you see logs in real-time

**`PYTHONDONTWRITEBYTECODE=1`:**
- Prevents Python from writing `.pyc` (bytecode cache) files
- In containers, `.pyc` files waste space and can cause issues with read-only filesystems
- Since containers are ephemeral, caching bytecode provides no benefit

```bash
# Without PYTHONUNBUFFERED:
docker logs my-app  # Logs appear delayed or missing

# With PYTHONUNBUFFERED=1:
docker logs my-app  # Logs appear immediately ✅
```

---

### Q79. What is the Dockerfile `ONBUILD` instruction?

**Answer:**

`ONBUILD` triggers instructions that execute **when the image is used as a base image** by another Dockerfile. It's a deferred instruction — it does nothing when the image is built, but runs when someone does `FROM your-image`.

```dockerfile
# django-base/Dockerfile
FROM python:3.11-slim
WORKDIR /app
ONBUILD COPY requirements.txt .              # Runs in child builds
ONBUILD RUN pip install -r requirements.txt  # Runs in child builds
ONBUILD COPY . .                             # Runs in child builds
```

```dockerfile
# myproject/Dockerfile — child of django-base
FROM myorg/django-base:latest
# The three ONBUILD instructions automatically run here!
# No explicit COPY or RUN needed
CMD ["gunicorn", "myproject.wsgi:application"]
```

**Use Case:** Creating standardized base images for multiple Django projects in your organization.

---

### Q80. What is Docker's `--restart` policy?

**Answer:**

The `--restart` flag defines **what Docker should do when a container exits** — useful for ensuring services automatically recover from crashes.

| Policy | Behavior |
|---|---|
| `no` (default) | Never restart |
| `always` | Always restart (even after `docker stop`) |
| `unless-stopped` | Restart unless explicitly stopped by user |
| `on-failure` | Restart only if container exits with non-zero code |
| `on-failure:5` | Restart on failure, maximum 5 times |

```bash
# For production web services
docker run -d --restart unless-stopped my-django-app

# For Celery workers (restart on crash, not on normal stop)
docker run -d --restart on-failure:5 celery-worker
```

```yaml
# docker-compose.yml
services:
  web:
    restart: unless-stopped

  celery:
    restart: on-failure
```

---

# 🟣 Phase 4: Django-Specific Dockerization & Best Practices
### *(Real-world Django + Docker patterns for interviews and production)*

---

### Q81. What is the complete Django + PostgreSQL + Redis Docker Compose setup?

**Answer:**

```yaml
# docker-compose.yml — Full Django Stack
version: "3.9"

services:
  # ── PostgreSQL Database ──────────────────────
  db:
    image: postgres:15-alpine
    container_name: django_db
    restart: unless-stopped
    environment:
      POSTGRES_USER: ${POSTGRES_USER:-django}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD:-django}
      POSTGRES_DB: ${POSTGRES_DB:-django_db}
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER:-django}"]
      interval: 10s
      timeout: 5s
      retries: 5

  # ── Redis Cache/Broker ───────────────────────
  redis:
    image: redis:7-alpine
    container_name: django_redis
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5

  # ── Django Web Application ───────────────────
  web:
    build:
      context: .
      dockerfile: Dockerfile
    container_name: django_web
    restart: unless-stopped
    volumes:
      - .:/app                         # Dev: live code reload
      - static_files:/app/staticfiles
      - media_files:/app/mediafiles
    ports:
      - "8000:8000"
    environment:
      DEBUG: ${DEBUG:-True}
      SECRET_KEY: ${SECRET_KEY}
      DATABASE_URL: postgres://${POSTGRES_USER:-django}:${POSTGRES_PASSWORD:-django}@db:5432/${POSTGRES_DB:-django_db}
      REDIS_URL: redis://redis:6379/0
      ALLOWED_HOSTS: ${ALLOWED_HOSTS:-localhost,127.0.0.1}
    depends_on:
      db:
        condition: service_healthy
      redis:
        condition: service_healthy
    command: python manage.py runserver 0.0.0.0:8000

  # ── Celery Worker ────────────────────────────
  celery:
    build: .
    container_name: django_celery
    restart: unless-stopped
    command: celery -A myproject worker -l info
    volumes:
      - .:/app
    environment:
      DATABASE_URL: postgres://${POSTGRES_USER:-django}:${POSTGRES_PASSWORD:-django}@db:5432/${POSTGRES_DB:-django_db}
      REDIS_URL: redis://redis:6379/0
    depends_on:
      - web
      - redis

  # ── Celery Beat Scheduler ────────────────────
  celery-beat:
    build: .
    container_name: django_celery_beat
    restart: unless-stopped
    command: celery -A myproject beat -l info
    volumes:
      - .:/app
    depends_on:
      - celery

volumes:
  postgres_data:
  static_files:
  media_files:
```

---

### Q82. How do you configure Django `settings.py` for Docker environments?

**Answer:**

Use **environment variables** for all environment-specific configuration. Never hardcode anything.

```python
# settings.py — Docker-compatible configuration
import os
import environ

env = environ.Env(
    DEBUG=(bool, False),
    ALLOWED_HOSTS=(list, ['localhost', '127.0.0.1']),
)
environ.Env.read_env(os.path.join(BASE_DIR, '.env'))

# ── Core Settings ────────────────────────────
SECRET_KEY = env('SECRET_KEY')
DEBUG = env('DEBUG')
ALLOWED_HOSTS = env('ALLOWED_HOSTS')

# ── Database ─────────────────────────────────
# DATABASE_URL=postgres://user:pass@db:5432/mydb
# 'db' is the Docker service name — resolves via Docker DNS
DATABASES = {
    'default': env.db('DATABASE_URL')
}

# ── Redis Cache ───────────────────────────────
# REDIS_URL=redis://redis:6379/0
CACHES = {
    'default': {
        'BACKEND': 'django_redis.cache.RedisCache',
        'LOCATION': env('REDIS_URL'),
    }
}

# ── Celery ────────────────────────────────────
CELERY_BROKER_URL = env('REDIS_URL')
CELERY_RESULT_BACKEND = env('REDIS_URL')

# ── Static and Media Files ────────────────────
STATIC_URL = '/static/'
STATIC_ROOT = os.path.join(BASE_DIR, 'staticfiles')  # Mount as volume in prod

MEDIA_URL = '/media/'
MEDIA_ROOT = os.path.join(BASE_DIR, 'mediafiles')    # Mount as volume
```

---

### Q83. How do you handle Django static files in Docker?

**Answer:**

Static files are tricky in Docker because Django's development server serves them, but production should use Nginx or a CDN.

**Development (docker-compose):**
```yaml
services:
  web:
    volumes:
      - .:/app  # Bind mount — Django dev server serves static files
```

**Production Pattern:**
```yaml
services:
  web:
    command: gunicorn myproject.wsgi:application --bind 0.0.0.0:8000
    volumes:
      - static_files:/app/staticfiles    # Named volume shared with nginx

  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
    volumes:
      - static_files:/app/staticfiles:ro # Nginx reads static files directly
      - ./nginx.conf:/etc/nginx/conf.d/default.conf
    depends_on:
      - web

volumes:
  static_files:
```

```nginx
# nginx.conf
server {
    listen 80;

    location /static/ {
        alias /app/staticfiles/;  # Serve directly from volume
    }

    location / {
        proxy_pass http://web:8000;  # Proxy dynamic requests to Gunicorn
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

**Collect static files during deployment:**
```bash
docker compose run --rm web python manage.py collectstatic --noinput
```

---

### Q84. How do you run Gunicorn in Docker for production?

**Answer:**

**Gunicorn** is the production WSGI server for Django. Replace `python manage.py runserver` with Gunicorn in production.

```dockerfile
# Production Dockerfile
CMD ["gunicorn", \
     "myproject.wsgi:application", \
     "--bind", "0.0.0.0:8000", \
     "--workers", "4", \
     "--threads", "2", \
     "--worker-class", "gthread", \
     "--timeout", "120", \
     "--keep-alive", "5", \
     "--log-level", "info", \
     "--access-logfile", "-", \
     "--error-logfile", "-"]
```

**Worker count formula:** `(2 × CPU cores) + 1`
- 1 CPU → 3 workers
- 2 CPUs → 5 workers
- 4 CPUs → 9 workers

```yaml
# docker-compose.prod.yml
services:
  web:
    environment:
      GUNICORN_WORKERS: 4
      GUNICORN_TIMEOUT: 120
    command: >
      gunicorn myproject.wsgi:application
      --bind 0.0.0.0:8000
      --workers ${GUNICORN_WORKERS:-4}
      --timeout ${GUNICORN_TIMEOUT:-120}
```

---

### Q85. How do you set up Nginx as a reverse proxy for Django in Docker?

**Answer:**

```nginx
# nginx.conf
upstream django {
    server web:8000;        # 'web' = Django service name in Docker DNS
}

server {
    listen 80;
    server_name example.com;

    client_max_body_size 20M;  # For file uploads

    # Static files — served directly by Nginx
    location /static/ {
        alias /app/staticfiles/;
        expires 30d;
        add_header Cache-Control "public, immutable";
    }

    # Media files
    location /media/ {
        alias /app/mediafiles/;
    }

    # Django application — proxied to Gunicorn
    location / {
        proxy_pass http://django;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_connect_timeout 60s;
        proxy_read_timeout 120s;
    }
}
```

```yaml
# docker-compose.yml
services:
  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx.conf:/etc/nginx/conf.d/default.conf:ro
      - static_files:/app/staticfiles:ro
      - media_files:/app/mediafiles:ro
    depends_on:
      - web
```

---

### Q86. How do you handle Django's `ALLOWED_HOSTS` in Docker?

**Answer:**

In Docker, the container's hostname, the service name, and the host machine's IP all need to be considered.

```python
# settings.py
import os

# Read from environment variable (comma-separated)
ALLOWED_HOSTS = os.environ.get('ALLOWED_HOSTS', 'localhost').split(',')

# Or with django-environ
ALLOWED_HOSTS = env.list('ALLOWED_HOSTS', default=['localhost', '127.0.0.1'])
```

```yaml
# docker-compose.yml
services:
  web:
    environment:
      # Development
      ALLOWED_HOSTS: "localhost,127.0.0.1,0.0.0.0"

      # Production
      ALLOWED_HOSTS: "yourdomain.com,www.yourdomain.com"
```

**Also configure:**
```python
# For Django behind a reverse proxy (Nginx)
USE_X_FORWARDED_HOST = True
SECURE_PROXY_SSL_HEADER = ('HTTP_X_FORWARDED_PROTO', 'https')
```

---

### Q87. How do you use Docker for Django testing in CI/CD?

**Answer:**

```yaml
# docker-compose.test.yml
version: "3.9"

services:
  db:
    image: postgres:15-alpine
    environment:
      POSTGRES_USER: test
      POSTGRES_PASSWORD: test
      POSTGRES_DB: test_db
    tmpfs:
      - /var/lib/postgresql/data  # Use RAM for speed in tests

  test:
    build: .
    command: python manage.py test --verbosity=2
    environment:
      DATABASE_URL: postgres://test:test@db:5432/test_db
      DJANGO_SETTINGS_MODULE: myproject.settings.test
    depends_on:
      db:
        condition: service_healthy
```

```bash
# Run tests
docker compose -f docker-compose.test.yml run --rm test

# Run specific test
docker compose -f docker-compose.test.yml run --rm test \
  python manage.py test myapp.tests.test_views

# With pytest
docker compose -f docker-compose.test.yml run --rm test \
  pytest -v --cov=myapp
```

**GitHub Actions CI:**
```yaml
# .github/workflows/test.yml
- name: Run Tests
  run: |
    docker compose -f docker-compose.test.yml run --rm test
```

---

### Q88. How do you use different Dockerfiles for development and production?

**Answer:**

```
project/
├── Dockerfile              # Production
├── Dockerfile.dev          # Development
├── docker-compose.yml      # Base/Production
└── docker-compose.dev.yml  # Development overrides
```

```dockerfile
# Dockerfile.dev — includes dev tools
FROM python:3.11-slim

WORKDIR /app

COPY requirements.txt requirements-dev.txt ./
RUN pip install -r requirements.txt -r requirements-dev.txt
# requirements-dev.txt includes: pytest, black, flake8, django-debug-toolbar

EXPOSE 8000
CMD ["python", "manage.py", "runserver", "0.0.0.0:8000"]
```

```dockerfile
# Dockerfile (production)
FROM python:3.11-slim

WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .
RUN adduser --system --group django
USER django

EXPOSE 8000
CMD ["gunicorn", "myproject.wsgi:application", "--bind", "0.0.0.0:8000"]
```

```yaml
# docker-compose.dev.yml
services:
  web:
    build:
      context: .
      dockerfile: Dockerfile.dev
    volumes:
      - .:/app    # Live code reload
    environment:
      DEBUG: "True"
```

```bash
# Development
docker compose -f docker-compose.dev.yml up

# Production
docker compose up
```

---

### Q89. How do you handle Celery with Docker?

**Answer:**

```yaml
# docker-compose.yml — Full Celery setup
services:
  web:
    build: .
    command: gunicorn myproject.wsgi:application --bind 0.0.0.0:8000

  celery-worker:
    build: .
    command: celery -A myproject worker --loglevel=info --concurrency=4
    environment:
      DATABASE_URL: postgres://user:pass@db:5432/mydb
      CELERY_BROKER_URL: redis://redis:6379/0
    depends_on:
      - redis
      - db
    restart: unless-stopped

  celery-beat:
    build: .
    command: celery -A myproject beat --loglevel=info --scheduler django_celery_beat.schedulers:DatabaseScheduler
    depends_on:
      - celery-worker
    restart: unless-stopped

  flower:
    build: .
    command: celery -A myproject flower --port=5555
    ports:
      - "5555:5555"
    depends_on:
      - redis
```

**Scaling workers:**
```bash
# Run 4 Celery worker containers
docker compose up --scale celery-worker=4
```

---

### Q90. How do you manage database connections in a Dockerized Django app?

**Answer:**

With multiple Django containers/workers, you need **connection pooling** to avoid overwhelming PostgreSQL.

```python
# settings.py — PgBouncer or django-db-geventpool
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.postgresql',
        'HOST': 'db',
        'CONN_MAX_AGE': 60,  # Reuse connections for 60 seconds (persistent connections)
    }
}
```

**Better: Add PgBouncer for connection pooling**
```yaml
services:
  pgbouncer:
    image: edoburu/pgbouncer
    environment:
      DB_USER: django
      DB_PASSWORD: django
      DB_HOST: db
      DB_NAME: django_db
      POOL_MODE: transaction
      MAX_CLIENT_CONN: 100
      DEFAULT_POOL_SIZE: 20
    ports:
      - "6432:5432"

  web:
    environment:
      DATABASE_URL: postgres://django:django@pgbouncer:5432/django_db
```

---

### Q91. What is the wait-for-it pattern in Docker?

**Answer:**

`depends_on: service_healthy` requires a health check. An alternative for simple cases is `wait-for-it.sh` — a script that polls until a TCP port is available.

```bash
#!/bin/bash
# wait-for-it.sh usage
./wait-for-it.sh db:5432 --timeout=30 -- python manage.py migrate
```

```dockerfile
# Download wait-for-it in Dockerfile
ADD https://raw.githubusercontent.com/vishnubob/wait-for-it/master/wait-for-it.sh /wait-for-it.sh
RUN chmod +x /wait-for-it.sh
```

```yaml
# docker-compose.yml
services:
  web:
    command: /wait-for-it.sh db:5432 --timeout=30 -- gunicorn myproject.wsgi:application
```

**Modern preferred approach:** Use `depends_on: condition: service_healthy` with proper healthchecks instead — it's cleaner and built into Compose.

---

### Q92. How do you set up SSL/HTTPS with Django in Docker?

**Answer:**

```yaml
# docker-compose.prod.yml — with Let's Encrypt SSL
services:
  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx.conf:/etc/nginx/conf.d/default.conf:ro
      - certbot_certs:/etc/letsencrypt:ro
      - certbot_www:/var/www/certbot:ro

  certbot:
    image: certbot/certbot
    volumes:
      - certbot_certs:/etc/letsencrypt
      - certbot_www:/var/www/certbot
    command: certonly --webroot -w /var/www/certbot -d yourdomain.com --email you@email.com --agree-tos --non-interactive

volumes:
  certbot_certs:
  certbot_www:
```

```nginx
# nginx.conf with SSL
server {
    listen 80;
    location /.well-known/acme-challenge/ { root /var/www/certbot; }
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl;
    ssl_certificate /etc/letsencrypt/live/yourdomain.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/yourdomain.com/privkey.pem;
    location / { proxy_pass http://web:8000; }
}
```

```python
# settings.py — Django HTTPS settings
SECURE_SSL_REDIRECT = True
SESSION_COOKIE_SECURE = True
CSRF_COOKIE_SECURE = True
SECURE_HSTS_SECONDS = 31536000
```

---

### Q93. What is a Docker-based Django development workflow?

**Answer:**

**Day-to-day developer workflow:**

```bash
# 1. Clone the project and start everything
git clone https://github.com/org/project.git
cd project
cp .env.example .env            # Set up environment
docker compose up -d            # Start all services in background

# 2. Run migrations
docker compose exec web python manage.py migrate

# 3. Create superuser
docker compose exec web python manage.py createsuperuser

# 4. Access your app
# http://localhost:8000 — Django app
# http://localhost:5555 — Flower (Celery monitor)

# 5. Make code changes — live reload because of bind mount (.:/app)

# 6. Run tests
docker compose exec web python manage.py test

# 7. Install a new package
echo "djangorestframework==3.15.0" >> requirements.txt
docker compose build web        # Rebuild to pick up new dependency
docker compose up -d web        # Restart the service

# 8. View logs
docker compose logs -f web

# 9. Clean up
docker compose down             # Stop and remove containers
docker compose down -v          # Also remove volumes (wipes DB!)
```

---

### Q94. How do you debug a Dockerized Django application?

**Answer:**

**Method 1: `pdb` with Docker**
```yaml
# docker-compose.yml — enable stdin for pdb
services:
  web:
    stdin_open: true   # -i flag
    tty: true          # -t flag
```

```python
# views.py
import pdb; pdb.set_trace()  # Breakpoint
```
```bash
docker attach django_web    # Attach to the container to interact with pdb
```

**Method 2: `debugpy` for VS Code remote debugging**
```python
# Add to manage.py or a debug view
import debugpy
debugpy.listen(("0.0.0.0", 5678))
debugpy.wait_for_client()  # Pause until VS Code connects
```

```yaml
services:
  web:
    ports:
      - "5678:5678"  # Debug port
```

**Method 3: Django Debug Toolbar**
```python
# settings.py
if DEBUG:
    INSTALLED_APPS += ['debug_toolbar']
    INTERNAL_IPS = ['127.0.0.1', '172.17.0.0/16']  # Docker network IPs
```

**Method 4: Logs**
```bash
docker compose logs -f web    # Follow all logs
docker compose logs -f web db # Multiple services
```

---

### Q95. How do you optimize Docker images for a Django application?

**Answer:**

**Complete optimization checklist:**

```dockerfile
# 1. ✅ Use slim/alpine base images
FROM python:3.11-slim    # ~45MB vs python:3.11 (~920MB)

# 2. ✅ Use multi-stage builds (covered in Q40)

# 3. ✅ Order layers for maximum caching (requirements before code)
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt  # Cached layer
COPY . .                                             # Only this changes

# 4. ✅ Clean up in the SAME layer
RUN apt-get update && apt-get install -y \
    libpq-dev \
    && rm -rf /var/lib/apt/lists/*   # Remove apt cache in same layer

# 5. ✅ Use .dockerignore aggressively
# (exclude venv, .git, __pycache__, tests, docs)

# 6. ✅ Set PYTHONDONTWRITEBYTECODE=1 (no .pyc files)
# 7. ✅ Use --no-cache-dir in pip install (no pip cache)
# 8. ✅ Run as non-root user
# 9. ✅ Don't install dev dependencies in prod image
```

**Typical size reduction:**
| Stage | Image Size |
|---|---|
| `python:3.11` (full) | ~920MB |
| `python:3.11-slim` | ~45MB |
| Slim + optimized Dockerfile | ~180MB (with dependencies) |
| Multi-stage build | ~120MB |

---

### Q96. What are the key differences between development and production Docker setups for Django?

**Answer:**

| Aspect | Development | Production |
|---|---|---|
| **Server** | `manage.py runserver` | Gunicorn |
| **Debug** | `DEBUG=True` | `DEBUG=False` |
| **Code mount** | Bind mount (`.:/app`) | Code baked into image |
| **Requirements** | `requirements.txt` + `requirements-dev.txt` | `requirements.txt` only |
| **Restart** | Optional | `unless-stopped` |
| **Secrets** | `.env` file | Docker Secrets / Vault |
| **Static files** | Django serves them | Nginx serves them |
| **HTTPS** | HTTP only | HTTPS with SSL cert |
| **Logging** | stdout (verbose) | Structured logging |
| **Health checks** | Optional | Mandatory |
| **User** | Often root (acceptable) | Non-root (mandatory) |
| **Build cache** | Use aggressively | Use selectively |

---

### Q97. How do you implement a CI/CD pipeline with Docker for a Django app?

**Answer:**

```yaml
# .github/workflows/deploy.yml — GitHub Actions
name: Build, Test, Deploy

on:
  push:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Build test image
        run: docker build -t myapp:test --target builder .

      - name: Run tests
        run: |
          docker compose -f docker-compose.test.yml up -d db
          docker compose -f docker-compose.test.yml run --rm \
            -e DATABASE_URL=postgres://test:test@db:5432/test_db \
            web python manage.py test

  build-and-push:
    needs: test
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Login to registry
        run: echo "${{ secrets.DOCKER_PASSWORD }}" | docker login -u "${{ secrets.DOCKER_USERNAME }}" --password-stdin

      - name: Build and push
        run: |
          IMAGE_TAG="${{ github.sha }}"
          docker build -t myorg/myapp:${IMAGE_TAG} -t myorg/myapp:latest .
          docker push myorg/myapp:${IMAGE_TAG}
          docker push myorg/myapp:latest

  deploy:
    needs: build-and-push
    runs-on: ubuntu-latest
    steps:
      - name: Deploy to server
        run: |
          ssh user@myserver "
            docker pull myorg/myapp:latest
            docker compose -f /app/docker-compose.prod.yml up -d --no-deps web
            docker compose -f /app/docker-compose.prod.yml run --rm web python manage.py migrate
          "
```

---

### Q98. How do you handle Django media files (user uploads) in Docker?

**Answer:**

User-uploaded files (via `ImageField`, `FileField`) need **persistent, shared storage** — a named volume or cloud storage.

**Option 1: Named Volume (single server)**
```yaml
services:
  web:
    volumes:
      - media_files:/app/mediafiles

  nginx:
    volumes:
      - media_files:/app/mediafiles:ro  # Read-only for nginx

volumes:
  media_files:
```

```python
# settings.py
MEDIA_URL = '/media/'
MEDIA_ROOT = '/app/mediafiles'
```

**Option 2: S3/Cloud Storage (recommended for production)**
```python
# settings.py — using django-storages with S3
DEFAULT_FILE_STORAGE = 'storages.backends.s3boto3.S3Boto3Storage'
AWS_STORAGE_BUCKET_NAME = os.environ.get('AWS_STORAGE_BUCKET_NAME')
AWS_S3_REGION_NAME = os.environ.get('AWS_S3_REGION_NAME')
```

```yaml
# docker-compose.yml
environment:
  AWS_ACCESS_KEY_ID: ${AWS_ACCESS_KEY_ID}
  AWS_SECRET_ACCESS_KEY: ${AWS_SECRET_ACCESS_KEY}
  AWS_STORAGE_BUCKET_NAME: ${AWS_STORAGE_BUCKET_NAME}
```

**Why S3?** If you have multiple web containers (scaled), they all need access to the same media files. Named volumes only work on a single host. S3 works across all containers and servers.

---

### Q99. What is a common mistake beginners make when Dockerizing Django apps?

**Answer:**

**Top 10 common mistakes:**

1. **Running as root** — Always create a non-root user
2. **Using `:latest` tag** — Pin specific versions for reproducibility
3. **Copying `.env` into the image** — Use `--env-file` at runtime, never bake secrets into images
4. **Not using `.dockerignore`** — Huge build contexts slow everything down
5. **Wrong COPY order** — Putting `COPY . .` before `pip install` defeats layer caching
6. **Not setting `PYTHONUNBUFFERED=1`** — Logs won't appear in `docker logs`
7. **Using `manage.py runserver` in production** — Use Gunicorn
8. **Running migrations in CMD** — Causes race conditions with multiple replicas
9. **Hardcoding `localhost` in settings** — Use Docker service names (e.g., `db`, `redis`)
10. **Not having health checks** — `depends_on` without `condition: service_healthy` causes startup race conditions

---

### Q100. What are the top Docker interview topics you must master?

**Answer:**

This is your **interview preparation checklist** — make sure you can confidently explain and demonstrate each:

**🟢 Must-Know Basics:**
- [ ] Image vs Container (Python class vs object analogy)
- [ ] Writing a Dockerfile from scratch for a Django app
- [ ] `docker build`, `docker run`, `docker ps`, `docker logs`, `docker exec`
- [ ] Layer caching and why `COPY requirements.txt` comes before `COPY . .`
- [ ] `.dockerignore` and why it matters

**🔵 Intermediate:**
- [ ] Docker Compose for multi-service apps (Django + Postgres + Redis)
- [ ] Named volumes vs bind mounts (when to use each)
- [ ] Container networking — how Django finds `db` by name
- [ ] Port mapping (`-p 8000:8000`) and when to use `EXPOSE`
- [ ] `depends_on` with health checks
- [ ] Environment variables and secrets management

**🔴 Advanced:**
- [ ] Multi-stage builds (builder vs production stage)
- [ ] Namespaces and cgroups (the "why" behind containers)
- [ ] Running as non-root user (security hardening)
- [ ] Docker image scanning (Trivy, Snyk)
- [ ] `docker system prune` and managing disk usage

**🟣 Django-Specific:**
- [ ] Full Compose stack: Django + Postgres + Redis + Celery + Nginx
- [ ] Static files with Nginx (shared volume pattern)
- [ ] Database migrations in Docker (entrypoint vs separate job)
- [ ] Development vs production Dockerfiles
- [ ] CI/CD pipeline with Docker build and push

**Sample interview answer framework:**
> "When asked 'How would you Dockerize a Django app?', walk through: 1) Dockerfile with multi-stage build, 2) docker-compose.yml with all services, 3) entrypoint script for migrations, 4) nginx for static files, 5) named volumes for data persistence, 6) .env for configuration."

---

## 🎯 Quick Reference Cheat Sheet

```bash
# ── Build ──────────────────────────────────────────────
docker build -t myapp:v1 .                    # Build image
docker build --no-cache -t myapp:v1 .         # Build without cache

# ── Run ────────────────────────────────────────────────
docker run -d -p 8000:8000 --name web myapp   # Run detached
docker run --rm -it myapp bash                # Interactive, auto-remove

# ── Compose ────────────────────────────────────────────
docker compose up -d                          # Start all services
docker compose up -d --build                  # Rebuild and start
docker compose down                           # Stop and remove
docker compose down -v                        # Also remove volumes
docker compose logs -f web                    # Follow logs
docker compose exec web bash                  # Shell in running container
docker compose run --rm web python manage.py migrate  # One-off command

# ── Inspect ────────────────────────────────────────────
docker ps                                     # Running containers
docker ps -a                                  # All containers
docker images                                 # All images
docker stats                                  # Live resource usage
docker inspect mycontainer                    # Full container details
docker system df                              # Disk usage

# ── Cleanup ────────────────────────────────────────────
docker system prune -a --volumes              # Remove everything unused
docker image prune                            # Remove dangling images
docker volume prune                           # Remove unused volumes
```

---

*📝 Good luck with your Docker journey! Remember: the best way to learn Docker is to Dockerize a real Django project. Start with `docker compose up` getting your Django + PostgreSQL running, and build from there.*
