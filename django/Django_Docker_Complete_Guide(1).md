# 🐳 Complete Django Project — Docker Guide
### Step-by-Step from Zero to Production-Ready

> **Reading Guide:** Follow every step in order. Each step builds on the previous one.
> Every file shown is complete — you can copy it directly into your project.
> Comments inside code blocks explain *why* each line exists, not just what it does.

---

## 📋 Table of Contents

| Step | Topic | What You'll Build |
|------|-------|-------------------|
| [Step 0](#step-0-understand-the-target-architecture) | Architecture Overview | Full picture before you start |
| [Step 1](#step-1-project-structure) | Project Structure | Correct folder layout |
| [Step 2](#step-2-django-project-setup) | Django Project Setup | The actual Django app |
| [Step 3](#step-3-requirements-files) | Requirements Files | Separate dev/prod dependencies |
| [Step 4](#step-4-environment-variables-env-files) | Environment Variables | `.env` files for all configs |
| [Step 5](#step-5-the-dockerfile-development) | Dockerfile (Dev) | Build your dev image |
| [Step 6](#step-6-the-dockerfile-production) | Dockerfile (Prod) | Optimized production image |
| [Step 7](#step-7-dockerignore) | `.dockerignore` | Keep images lean |
| [Step 8](#step-8-django-settings-configuration) | Django Settings | Docker-compatible settings |
| [Step 9](#step-9-entrypoint-script) | Entrypoint Script | Auto-migrations on startup |
| [Step 10](#step-10-nginx-configuration) | Nginx Config | Reverse proxy + static files |
| [Step 11](#step-11-docker-compose-development) | Compose (Dev) | Full dev stack |
| [Step 12](#step-12-docker-compose-production) | Compose (Prod) | Production stack |
| [Step 13](#step-13-first-run-complete-workflow) | First Run | Spin up everything |
| [Step 14](#step-14-daily-development-workflow) | Dev Workflow | Day-to-day commands |
| [Step 15](#step-15-adding-celery--celery-beat) | Celery Setup | Async tasks |
| [Step 16](#step-16-adding-flower-celery-monitoring) | Flower | Monitor Celery tasks |
| [Step 17](#step-17-health-checks--readiness) | Health Checks | Production readiness |
| [Step 18](#step-18-cicd-pipeline-github-actions) | CI/CD Pipeline | GitHub Actions |
| [Step 19](#step-19-ssl--https-with-lets-encrypt) | SSL / HTTPS | Secure production |
| [Step 20](#step-20-production-deployment-checklist) | Deploy Checklist | Final verification |

---

## Step 0: Understand the Target Architecture

Before writing a single file, understand what you are building.

### 🏗️ Development Architecture
```
Your Code Editor (localhost)
        │
        │  (bind mount — live code changes)
        ▼
┌───────────────────────────────────────────────────────┐
│              Docker Network: django_dev                │
│                                                       │
│  ┌─────────────────┐    ┌──────────────────────┐     │
│  │   web (Django)  │───▶│  db (PostgreSQL 15)  │     │
│  │ manage.py       │    │  Port: 5432 (internal)│     │
│  │ runserver:8000  │    └──────────────────────┘     │
│  └────────┬────────┘                                  │
│           │              ┌──────────────────────┐     │
│           └─────────────▶│  redis (Redis 7)     │     │
│                          │  Port: 6379 (internal)│     │
│                          └──────────────────────┘     │
└───────────────────────────────────────────────────────┘
        │
        │ localhost:8000
        ▼
   Your Browser
```

### 🏗️ Production Architecture
```
   Internet
      │
      ▼ Port 80 / 443
┌─────────────────────────────────────────────────────────────┐
│                    Docker Network: django_prod               │
│                                                             │
│  ┌──────────────────┐                                       │
│  │  nginx           │──── /static/ ──▶ shared volume        │
│  │  Port: 80/443    │──── /media/  ──▶ shared volume        │
│  │  (reverse proxy) │──── /        ──▶ web:8000             │
│  └──────────────────┘                                       │
│          │                                                   │
│          ▼ Port 8000 (internal only)                        │
│  ┌──────────────────┐    ┌──────────────────────────┐      │
│  │  web (Gunicorn)  │───▶│  db (PostgreSQL 15)      │      │
│  │  Django App      │    │  named volume: pg_data    │      │
│  └────────┬─────────┘    └──────────────────────────┘      │
│           │                                                  │
│           │              ┌──────────────────────────┐      │
│           └─────────────▶│  redis (Redis 7)          │      │
│                          └──────────────────────────┘      │
│                                                             │
│  ┌──────────────────┐    ┌──────────────────────────┐      │
│  │  celery (worker) │    │  celery-beat (scheduler)  │      │
│  └──────────────────┘    └──────────────────────────┘      │
└─────────────────────────────────────────────────────────────┘
```

### 📦 Services Summary
| Service | Image | Purpose | Port |
|---------|-------|---------|------|
| `web` | Custom (your Django app) | Django/Gunicorn app server | 8000 (internal) |
| `db` | `postgres:15-alpine` | Primary database | 5432 (internal) |
| `redis` | `redis:7-alpine` | Cache + Celery broker | 6379 (internal) |
| `nginx` | `nginx:1.25-alpine` | Reverse proxy + static files | 80, 443 (public) |
| `celery` | Custom (same as web) | Async task worker | — |
| `celery-beat` | Custom (same as web) | Periodic task scheduler | — |
| `flower` | Custom (same as web) | Celery monitoring UI | 5555 (dev only) |

---

## Step 1: Project Structure

Create this **exact folder structure** before writing any Docker file. Getting the structure right first prevents confusion later.

```bash
# Run these commands to create the structure
mkdir -p myproject/{config/{nginx,postgres},docker,scripts}
cd myproject
```

### 📁 Complete Project Layout
```
myproject/                          ← Your git repository root
│
├── 📄 Dockerfile                   ← Production image definition
├── 📄 Dockerfile.dev               ← Development image definition
├── 📄 docker-compose.yml           ← Production compose file
├── 📄 docker-compose.dev.yml       ← Development compose file
├── 📄 .dockerignore                ← Files excluded from Docker build context
│
├── 📄 .env                         ← 🔴 NEVER commit — local development secrets
├── 📄 .env.example                 ← ✅ Commit this — template with no real values
├── 📄 .env.production              ← 🔴 NEVER commit — production secrets
│
├── 📄 manage.py                    ← Django management script (auto-generated)
├── 📄 requirements.txt             ← Production Python dependencies
├── 📄 requirements.dev.txt         ← Development-only dependencies
│
├── 📄 entrypoint.sh                ← Container startup script
│
├── 📁 config/                      ← Infrastructure configuration files
│   ├── 📁 nginx/
│   │   ├── 📄 nginx.conf           ← Nginx main config
│   │   └── 📄 django.conf          ← Nginx server block for Django
│   └── 📁 postgres/
│       └── 📄 init.sql             ← DB initialization SQL (optional)
│
├── 📁 scripts/                     ← Utility shell scripts
│   ├── 📄 backup_db.sh             ← Database backup script
│   └── 📄 restore_db.sh           ← Database restore script
│
└── 📁 src/                         ← All Django application code lives here
    ├── 📄 manage.py                ← Django CLI entry point
    │
    ├── 📁 myproject/               ← Django project config package
    │   ├── 📄 __init__.py
    │   ├── 📄 settings/            ← Split settings by environment
    │   │   ├── 📄 __init__.py
    │   │   ├── 📄 base.py          ← Shared settings (all environments)
    │   │   ├── 📄 development.py   ← Dev overrides
    │   │   └── 📄 production.py    ← Prod overrides
    │   ├── 📄 urls.py
    │   ├── 📄 wsgi.py
    │   └── 📄 asgi.py
    │
    └── 📁 apps/                    ← Your Django applications
        └── 📁 core/                ← Example app
            ├── 📄 __init__.py
            ├── 📄 models.py
            ├── 📄 views.py
            ├── 📄 urls.py
            └── 📄 tests.py
```

---

## Step 2: Django Project Setup

Initialize the Django project inside the `src/` directory.

```bash
# Install Django temporarily to scaffold the project
pip install django

# Create Django project (note the dot at the end — creates in current dir)
django-admin startproject myproject src/

# Create your first app
cd src
python manage.py startapp core apps/core

# Go back to project root
cd ..
```

After running these commands, your `src/` will have the standard Django structure.

---

## Step 3: Requirements Files

Split your dependencies into **two files** — production needs minimal, clean packages. Development needs debugging tools that should never go into production.

### 📄 `requirements.txt` — Production Dependencies

```txt
# ─────────────────────────────────────────────────────────────────────────────
# requirements.txt — PRODUCTION dependencies only
# Install with: pip install -r requirements.txt
#
# IMPORTANT: Always pin exact versions (==) for reproducible builds.
# Use 'pip freeze > requirements.txt' after testing to get exact versions.
# ─────────────────────────────────────────────────────────────────────────────

# ── Django Core ──────────────────────────────────────────────────────────────
Django==4.2.11                    # The framework itself — pin to minor version
gunicorn==21.2.0                  # Production WSGI server (replaces runserver)
whitenoise==6.6.0                 # Serve static files efficiently without Nginx (optional)

# ── Database ─────────────────────────────────────────────────────────────────
psycopg2-binary==2.9.9            # PostgreSQL adapter for Django
                                  # Use psycopg2 (not binary) in production if possible
                                  # Binary is fine for containers, avoids compilation

# ── Environment & Configuration ───────────────────────────────────────────────
django-environ==0.11.2            # Read settings from environment variables
                                  # Provides env() helper and .env file support

# ── Redis & Caching ───────────────────────────────────────────────────────────
redis==5.0.3                      # Redis Python client
django-redis==5.4.0               # Django cache backend using Redis

# ── Celery (Async Tasks) ──────────────────────────────────────────────────────
celery==5.3.6                     # Distributed task queue
django-celery-beat==2.6.0         # Periodic task scheduling (stores in DB)
django-celery-results==2.5.1      # Store task results in Django DB

# ── API (optional — remove if not building REST API) ──────────────────────────
djangorestframework==3.15.1       # Django REST Framework
django-cors-headers==4.3.1        # Handle CORS for frontend apps

# ── Security ─────────────────────────────────────────────────────────────────
django-csp==3.8                   # Content Security Policy headers
```

---

### 📄 `requirements.dev.txt` — Development-Only Dependencies

```txt
# ─────────────────────────────────────────────────────────────────────────────
# requirements.dev.txt — DEVELOPMENT dependencies
# Install with: pip install -r requirements.txt -r requirements.dev.txt
#
# These packages are ONLY for development. They are NEVER installed
# in the production image — keeping it lean and secure.
# ─────────────────────────────────────────────────────────────────────────────

# Include all production dependencies first
-r requirements.txt

# ── Development Server ────────────────────────────────────────────────────────
Werkzeug==3.0.1                   # Better debugger for runserver_plus
django-extensions==3.2.3          # Extra management commands (shell_plus, etc.)

# ── Debugging ─────────────────────────────────────────────────────────────────
django-debug-toolbar==4.3.0       # In-browser debugging panel (SQL, cache, etc.)
ipython==8.22.2                   # Better Python shell (used by shell_plus)
ipdb==0.13.13                     # IPython-based debugger (better pdb)

# ── Testing ───────────────────────────────────────────────────────────────────
pytest==8.1.1                     # Test runner
pytest-django==4.8.0              # Django plugin for pytest
pytest-cov==5.0.0                 # Coverage reporting
factory-boy==3.3.0                # Model factories for tests (like Faker)
faker==24.3.0                     # Generate fake data for tests

# ── Code Quality ──────────────────────────────────────────────────────────────
black==24.3.0                     # Code formatter (auto-formats your code)
flake8==7.0.0                     # Linter (catches style and syntax issues)
isort==5.13.2                     # Sorts imports automatically
mypy==1.9.0                       # Static type checker
django-stubs==4.2.7               # Django type stubs for mypy

# ── Security Scanning ─────────────────────────────────────────────────────────
pip-audit==2.7.2                  # Scan dependencies for known vulnerabilities
safety==3.0.1                     # Another security scanner for dependencies
```

---

## Step 4: Environment Variables (.env Files)

**This step is critical for security.** Never hardcode credentials. Environment variables let you use the same Docker image across dev, staging, and production with different configs.

### 📄 `.env.example` — Template (✅ Commit to Git)

```bash
# ─────────────────────────────────────────────────────────────────────────────
# .env.example — TEMPLATE FILE
#
# HOW TO USE:
#   cp .env.example .env
#   Then fill in the real values in .env
#
# This file is committed to Git.
# The actual .env file is NEVER committed (it's in .gitignore).
# ─────────────────────────────────────────────────────────────────────────────

# ── Django Core Settings ──────────────────────────────────────────────────────

# Generate with: python -c "from django.core.management.utils import get_random_secret_key; print(get_random_secret_key())"
SECRET_KEY=your-secret-key-here-generate-a-new-one

# Set to False in production ALWAYS
DEBUG=True

# Comma-separated list of allowed hosts
# Dev: localhost,127.0.0.1
# Prod: yourdomain.com,www.yourdomain.com
ALLOWED_HOSTS=localhost,127.0.0.1,0.0.0.0

# Which settings module to use
# Dev: myproject.settings.development
# Prod: myproject.settings.production
DJANGO_SETTINGS_MODULE=myproject.settings.development

# ── Database ──────────────────────────────────────────────────────────────────

# PostgreSQL connection URL format:
# postgres://USER:PASSWORD@HOST:PORT/DBNAME
# HOST is the Docker service name (e.g., 'db') — Docker DNS resolves it
DATABASE_URL=postgres://django_user:django_password@db:5432/django_db

# Individual DB settings (used by docker-compose to configure postgres container)
POSTGRES_DB=django_db
POSTGRES_USER=django_user
POSTGRES_PASSWORD=django_password

# ── Redis ─────────────────────────────────────────────────────────────────────

# Redis connection URL
# 'redis' is the Docker service name
REDIS_URL=redis://redis:6379/0

# ── Email ─────────────────────────────────────────────────────────────────────

EMAIL_BACKEND=django.core.mail.backends.console.EmailBackend
EMAIL_HOST=smtp.gmail.com
EMAIL_PORT=587
EMAIL_USE_TLS=True
EMAIL_HOST_USER=your-email@gmail.com
EMAIL_HOST_PASSWORD=your-app-password

# ── AWS S3 (for media files in production) ────────────────────────────────────

AWS_ACCESS_KEY_ID=your-access-key
AWS_SECRET_ACCESS_KEY=your-secret-key
AWS_STORAGE_BUCKET_NAME=your-bucket-name
AWS_S3_REGION_NAME=us-east-1

# ── Celery ────────────────────────────────────────────────────────────────────

CELERY_BROKER_URL=redis://redis:6379/1
CELERY_RESULT_BACKEND=redis://redis:6379/2
```

---

### 📄 `.env` — Your Local Development Values (🔴 Never Commit)

```bash
# ─────────────────────────────────────────────────────────────────────────────
# .env — LOCAL DEVELOPMENT ENVIRONMENT
# 
# ⚠️  DO NOT COMMIT THIS FILE TO GIT ⚠️
# It's already in .gitignore — keep it that way.
# Use simple passwords in dev — they never leave your machine.
# ─────────────────────────────────────────────────────────────────────────────

SECRET_KEY=django-insecure-dev-key-only-for-local-development-not-for-production

DEBUG=True

ALLOWED_HOSTS=localhost,127.0.0.1,0.0.0.0

DJANGO_SETTINGS_MODULE=myproject.settings.development

# ── Database ──────────────────────────────────────────────────────────────────
# Simple passwords are fine in dev — this only runs on your local machine
DATABASE_URL=postgres://django_user:django_password@db:5432/django_db

POSTGRES_DB=django_db
POSTGRES_USER=django_user
POSTGRES_PASSWORD=django_password    # Simple password is fine for local dev

# ── Redis ─────────────────────────────────────────────────────────────────────
REDIS_URL=redis://redis:6379/0

# ── Email — Console backend in dev (prints emails to terminal) ────────────────
EMAIL_BACKEND=django.core.mail.backends.console.EmailBackend

# ── Celery ────────────────────────────────────────────────────────────────────
CELERY_BROKER_URL=redis://redis:6379/1
CELERY_RESULT_BACKEND=redis://redis:6379/2
```

---

### Update `.gitignore`

```gitignore
# ─────────────────────────────────────────────────────────────────────────────
# .gitignore — Files that must NEVER be committed to Git
# ─────────────────────────────────────────────────────────────────────────────

# ── Environment Files (CRITICAL — never commit real secrets) ──────────────────
.env
.env.production
.env.staging
*.env

# ── Python ────────────────────────────────────────────────────────────────────
__pycache__/
*.py[cod]
*$py.class
*.so
.Python
venv/
.venv/
env/
*.egg-info/
dist/
build/

# ── Django ────────────────────────────────────────────────────────────────────
*.log
local_settings.py
db.sqlite3
db.sqlite3-journal
media/                          # Never commit uploaded media files

# ── Testing ───────────────────────────────────────────────────────────────────
.coverage
htmlcov/
.pytest_cache/
.tox/

# ── IDE ───────────────────────────────────────────────────────────────────────
.vscode/
.idea/
*.swp
*.swo

# ── OS ────────────────────────────────────────────────────────────────────────
.DS_Store
Thumbs.db
```

---

## Step 5: The Dockerfile (Development)

The development Dockerfile prioritizes **developer experience** over image size. It includes all dev tools.

### 📄 `Dockerfile.dev`

```dockerfile
# ─────────────────────────────────────────────────────────────────────────────
# Dockerfile.dev — Development Image
#
# PURPOSE: Fast iteration during development.
# CHARACTERISTICS:
#   - Installs dev dependencies (pytest, black, ipython, etc.)
#   - Uses 'manage.py runserver' with auto-reload
#   - Code is NOT baked in (mounted via bind volume for live reload)
#   - Slightly larger image — we don't care in dev
#   - NOT for production use
# ─────────────────────────────────────────────────────────────────────────────

# ── Base Image ────────────────────────────────────────────────────────────────
# python:3.11-slim is the official Debian-based Python image with minimal extras.
# 'slim' removes documentation, man pages, and some tools — cuts size significantly.
# We pin to 3.11 (not 3.11.x) for dev — minor patches are fine.
# In production we pin the full version (e.g., 3.11.9).
FROM python:3.11-slim

# ── Labels ────────────────────────────────────────────────────────────────────
# Metadata about the image — useful for auditing and tooling.
LABEL maintainer="your-email@example.com"
LABEL description="Django development image"
LABEL environment="development"

# ── Python Environment Variables ──────────────────────────────────────────────

# PYTHONUNBUFFERED=1: Forces Python to flush stdout/stderr immediately.
# Without this, your print() and logging calls won't show in 'docker logs'
# in real time — they'll be buffered and appear in batches or not at all.
ENV PYTHONUNBUFFERED=1

# PYTHONDONTWRITEBYTECODE=1: Prevents Python from creating .pyc bytecode files.
# In containers (especially with bind mounts), these cause confusion because:
# 1. They persist in your local filesystem after container is gone
# 2. They can cause "module was already imported" issues
# 3. They waste space — containers are ephemeral, so caching bytecode is pointless
ENV PYTHONDONTWRITEBYTECODE=1

# PIP_NO_CACHE_DIR=1: Tells pip not to cache downloaded packages.
# In containers we never rebuild from cache anyway, so this saves disk space.
ENV PIP_NO_CACHE_DIR=1

# PIP_DISABLE_PIP_VERSION_CHECK=1: Suppresses "new version of pip available" warnings.
# These warnings are noise in container build logs.
ENV PIP_DISABLE_PIP_VERSION_CHECK=1

# ── System Dependencies ───────────────────────────────────────────────────────
# Install OS-level packages that Python packages depend on at compile time.
# We do this in ONE RUN command to keep it as a single layer.
# The '&& rm -rf /var/lib/apt/lists/*' at the end removes the apt package
# cache from this layer — if we don't do this, the cache stays in the image
# and bloats it by ~40MB.
RUN apt-get update && apt-get install -y --no-install-recommends \
    # libpq-dev: Required to compile psycopg2 (the PostgreSQL adapter)
    libpq-dev \
    # gcc: C compiler needed for packages that have C extensions
    gcc \
    # curl: Used in health checks and debugging
    curl \
    # git: Needed if you pip install packages directly from Git repos
    git \
    # netcat-openbsd: 'nc' command — used to check if a port is open
    # (useful in entrypoint scripts to wait for PostgreSQL to start)
    netcat-openbsd \
    # Clean up apt cache in the SAME layer to save space
    && rm -rf /var/lib/apt/lists/*

# ── Working Directory ─────────────────────────────────────────────────────────
# All subsequent commands run from /app inside the container.
# This directory is created automatically by WORKDIR.
# We use /app as convention — it's clean and avoids conflicts with system dirs.
WORKDIR /app

# ── Install Python Dependencies ───────────────────────────────────────────────
# CRITICAL ORDERING: We copy requirements files BEFORE copying application code.
#
# WHY? Docker layer caching.
# If you do 'COPY . .' first, then 'pip install', then every single code change
# (even changing one line in views.py) invalidates the pip install layer and
# Docker reinstalls ALL packages from scratch. That takes 3-5 minutes every time.
#
# With this order:
# - requirements files change rarely → pip install layer stays cached
# - You change views.py → only the 'COPY . .' layer is rebuilt (fast!)
# Result: code change rebuilds in seconds, not minutes.
COPY requirements.dev.txt .

# Install ALL dependencies (prod + dev) for development
RUN pip install -r requirements.dev.txt

# ── Port Documentation ────────────────────────────────────────────────────────
# EXPOSE documents which port the app uses. It does NOT actually publish the port.
# The actual publishing happens in docker-compose.yml with the 'ports:' key.
# This is just metadata — like a README in the Dockerfile.
EXPOSE 8000

# ── Default Command ───────────────────────────────────────────────────────────
# Note: We do NOT copy code here (no 'COPY . .')
# In development, code is mounted via a bind volume in docker-compose.dev.yml.
# This means code changes are instantly visible — NO rebuild required.
#
# '0.0.0.0:8000' means listen on ALL network interfaces, not just localhost.
# This is REQUIRED in containers — without 0.0.0.0, Django only listens on
# 127.0.0.1 inside the container, which is not accessible from outside.
CMD ["python", "manage.py", "runserver", "0.0.0.0:8000"]
```

---

## Step 6: The Dockerfile (Production)

The production Dockerfile prioritizes **security, size, and performance**. It uses a multi-stage build.

### 📄 `Dockerfile` (Production)

```dockerfile
# ─────────────────────────────────────────────────────────────────────────────
# Dockerfile — Production Image
#
# PURPOSE: Lean, secure, production-ready image.
# TECHNIQUE: Multi-stage build
#   Stage 1 (builder): Install all dependencies in a full environment
#   Stage 2 (production): Copy only what's needed into a clean, minimal image
#
# RESULT: The final image contains ONLY:
#   - Python runtime (slim)
#   - Your virtual environment with installed packages
#   - Your application code
#   - A non-root user to run the app
#
# WHY MULTI-STAGE?
# If you do 'pip install' in a single stage, the image contains:
#   - gcc, libpq-dev, and other build tools (used to compile psycopg2, etc.)
#   - pip's compilation artifacts
#   - Build caches
# These are needed to BUILD packages but not to RUN them.
# Multi-stage lets us compile in one environment and run in another.
# Typical size reduction: 500MB → 150MB
# ─────────────────────────────────────────────────────────────────────────────

# ═════════════════════════════════════════════════════════════════════════════
# STAGE 1: builder
# Full environment with build tools — we throw this away after building
# ═════════════════════════════════════════════════════════════════════════════

# Pin the FULL version in production for 100% reproducibility.
# 'slim-bookworm' = Debian Bookworm (stable) + minimal packages.
FROM python:3.11.9-slim-bookworm AS builder

# Same environment variables as dev
ENV PYTHONUNBUFFERED=1 \
    PYTHONDONTWRITEBYTECODE=1 \
    PIP_NO_CACHE_DIR=1 \
    PIP_DISABLE_PIP_VERSION_CHECK=1

# ── Build Dependencies ────────────────────────────────────────────────────────
# We need build tools HERE (in the builder stage) to compile packages.
# They will NOT be in the final image.
RUN apt-get update && apt-get install -y --no-install-recommends \
    libpq-dev \        # Needed to build psycopg2
    gcc \              # C compiler for native extensions
    && rm -rf /var/lib/apt/lists/*

WORKDIR /app

# ── Create Virtual Environment ────────────────────────────────────────────────
# We create a venv specifically so we can COPY it to the next stage cleanly.
# All dependencies go into /opt/venv — one directory to copy.
RUN python -m venv /opt/venv

# Activate the venv for subsequent RUN commands
ENV PATH="/opt/venv/bin:$PATH"

# ── Install Python Dependencies ───────────────────────────────────────────────
# Only production requirements (no dev tools, no debuggers, no test frameworks).
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt


# ═════════════════════════════════════════════════════════════════════════════
# STAGE 2: production
# The FINAL image — clean, minimal, secure
# ═════════════════════════════════════════════════════════════════════════════

FROM python:3.11.9-slim-bookworm AS production

# Labels for production image metadata
LABEL maintainer="your-email@example.com"
LABEL description="Django production image"
LABEL environment="production"

ENV PYTHONUNBUFFERED=1 \
    PYTHONDONTWRITEBYTECODE=1 \
    # Add the venv to PATH so python/pip commands use it automatically
    PATH="/opt/venv/bin:$PATH"

# ── Runtime Dependencies Only ─────────────────────────────────────────────────
# In the FINAL image, we only need RUNTIME libraries (not build tools).
# libpq5 is the PostgreSQL runtime client (smaller than libpq-dev which includes headers).
# curl is for health checks.
RUN apt-get update && apt-get install -y --no-install-recommends \
    libpq5 \           # PostgreSQL runtime library (not the -dev build version)
    curl \             # For health checks in HEALTHCHECK instruction
    && rm -rf /var/lib/apt/lists/*

WORKDIR /app

# ── Copy Virtual Environment from Builder ─────────────────────────────────────
# This is the key multi-stage magic: we ONLY copy the installed packages.
# gcc, libpq-dev, build artifacts — all gone. Never in the final image.
COPY --from=builder /opt/venv /opt/venv

# ── Copy Application Code ─────────────────────────────────────────────────────
# In production, code IS baked into the image (unlike dev where it's bind-mounted).
# This makes the image fully self-contained and deployable anywhere.
COPY . .

# ── Create Non-Root User ──────────────────────────────────────────────────────
# SECURITY: Never run as root in production.
# If an attacker exploits your app and breaks out of the container,
# they'd have root on the HOST machine if running as root.
# With a non-root user, their blast radius is contained.
#
# --system: Creates a system account (no login shell, no home dir)
# --gid 1001: Specific GID for predictable permissions
# --uid 1001: Specific UID for predictable permissions
RUN groupadd --system --gid 1001 django && \
    useradd --system --uid 1001 --gid django --no-create-home django

# ── Set Ownership ─────────────────────────────────────────────────────────────
# Give the django user ownership of the app directory.
# Without this, the app can't write to any files (logs, uploads, etc.)
RUN chown -R django:django /app

# ── Static Files Directory ────────────────────────────────────────────────────
# Create the staticfiles directory and set ownership.
# 'collectstatic' will write here, and Nginx will read from here.
RUN mkdir -p /app/staticfiles /app/mediafiles && \
    chown -R django:django /app/staticfiles /app/mediafiles

# ── Switch to Non-Root User ───────────────────────────────────────────────────
# All subsequent commands (including CMD) run as 'django', not root.
USER django

# ── Port Documentation ────────────────────────────────────────────────────────
EXPOSE 8000

# ── Health Check ──────────────────────────────────────────────────────────────
# Docker will periodically run this command to check if the container is healthy.
# --interval=30s: Check every 30 seconds
# --timeout=10s: If the check takes longer than 10s, it's failed
# --start-period=60s: Give the app 60 seconds to start before checking
#                     (Django startup + migration time)
# --retries=3: Mark as unhealthy after 3 consecutive failures
HEALTHCHECK --interval=30s --timeout=10s --start-period=60s --retries=3 \
    CMD curl -f http://localhost:8000/health/ || exit 1

# ── Production Command ────────────────────────────────────────────────────────
# Gunicorn is a production-grade WSGI server.
# '--workers 4': 4 worker processes (formula: 2 × CPU cores + 1)
# '--worker-class gthread': Threaded workers (better for I/O-bound Django views)
# '--threads 2': 2 threads per worker
# '--bind 0.0.0.0:8000': Listen on all interfaces at port 8000
# '--timeout 120': Workers that don't respond in 120s are killed and restarted
# '--keep-alive 5': Keep connections alive for 5 seconds (behind Nginx)
# '--access-logfile -': Log access to stdout (captured by 'docker logs')
# '--error-logfile -': Log errors to stderr (captured by 'docker logs')
# '--log-level info': Log level
CMD ["gunicorn", \
     "myproject.wsgi:application", \
     "--workers", "4", \
     "--worker-class", "gthread", \
     "--threads", "2", \
     "--bind", "0.0.0.0:8000", \
     "--timeout", "120", \
     "--keep-alive", "5", \
     "--access-logfile", "-", \
     "--error-logfile", "-", \
     "--log-level", "info"]
```

---

## Step 7: `.dockerignore`

This file prevents unnecessary files from being sent to the Docker daemon during builds. Without it, your entire project directory (including `node_modules`, `.git`, etc.) is uploaded every time.

### 📄 `.dockerignore`

```dockerignore
# ─────────────────────────────────────────────────────────────────────────────
# .dockerignore — Files excluded from the Docker build context
#
# WHY THIS MATTERS:
# When you run 'docker build .', Docker sends all files in '.' to the daemon.
# Without .dockerignore, this includes your .git folder (hundreds of MB),
# node_modules, virtual environments, etc. — massively slowing builds.
#
# Rule of thumb: If it's in .gitignore, it probably belongs here too.
# ─────────────────────────────────────────────────────────────────────────────

# ── Version Control ───────────────────────────────────────────────────────────
# .git contains your entire commit history — never needed in the image.
# Including it also invalidates ALL build cache on every commit.
.git
.gitignore
.gitattributes

# ── Python Cache Files ────────────────────────────────────────────────────────
# Bytecode cache files — rebuilt inside the container as needed.
__pycache__
*.py[cod]
*$py.class
*.pyo
*.pyd

# ── Virtual Environments ──────────────────────────────────────────────────────
# Your local venv has the wrong platform binaries (your OS vs container OS).
# Docker installs its own packages via requirements.txt — don't include yours.
venv/
.venv/
env/
.env_local/

# ── Environment Files ─────────────────────────────────────────────────────────
# CRITICAL: Never bake secrets into the image.
# Settings are passed as environment variables at runtime, not build time.
.env
.env.*
*.env

# ── Testing & Coverage ────────────────────────────────────────────────────────
# Test files only used during CI/CD, not needed in the production image.
.coverage
htmlcov/
.pytest_cache/
.tox/
nosetests.xml
coverage.xml

# ── Development Tools ─────────────────────────────────────────────────────────
# Editor configs and IDE files — no place in a container.
.vscode/
.idea/
*.swp
*.swo
.DS_Store
Thumbs.db

# ── Documentation ─────────────────────────────────────────────────────────────
# Docs are not needed to run the application.
docs/
README.md
CHANGELOG.md
*.md
!requirements*.txt    # Exception: keep requirements files (they end in .txt not .md, but being explicit)

# ── Docker Files ──────────────────────────────────────────────────────────────
# The Compose files and other Dockerfiles are not needed INSIDE the image.
# They're used to BUILD the image, not to run the app.
docker-compose*.yml
Dockerfile*

# ── Media & Static Build Artifacts ───────────────────────────────────────────
# Generated files — recreated inside the container.
staticfiles/
media/
node_modules/

# ── Logs ──────────────────────────────────────────────────────────────────────
*.log
logs/

# ── Backup Files ──────────────────────────────────────────────────────────────
*.sql
*.sql.gz
backups/

# ── CI/CD ─────────────────────────────────────────────────────────────────────
.github/
.circleci/
.travis.yml
Jenkinsfile
```

---

## Step 8: Django Settings Configuration

Split your settings into **base + environment-specific** files. This is the industry standard for Docker deployments.

### 📄 `src/myproject/settings/base.py` — Shared Settings

```python
# ─────────────────────────────────────────────────────────────────────────────
# settings/base.py — Settings shared across ALL environments
#
# This file is imported by development.py and production.py.
# It should contain settings that DON'T change between environments.
# Environment-specific values come from environment variables via django-environ.
# ─────────────────────────────────────────────────────────────────────────────

import os
from pathlib import Path

import environ

# ── Path Setup ────────────────────────────────────────────────────────────────
# BASE_DIR points to the 'src/' directory (where manage.py lives).
# We use Path for cross-platform compatibility.
BASE_DIR = Path(__file__).resolve().parent.parent.parent  # → src/

# ── django-environ Setup ──────────────────────────────────────────────────────
# environ.Env() creates an env reader.
# The arguments define TYPE CASTING and DEFAULT VALUES for each variable.
# Format: env_var_name=(type, default_value)
env = environ.Env(
    DEBUG=(bool, False),                          # Default: False (safe default)
    ALLOWED_HOSTS=(list, ['localhost']),           # Default: localhost only
    INTERNAL_IPS=(list, ['127.0.0.1']),           # For debug toolbar
    CORS_ALLOWED_ORIGINS=(list, []),              # For DRF CORS
)

# Read the .env file if it exists.
# In Docker, env vars are passed directly — .env file may or may not exist.
# This line handles BOTH cases gracefully.
# In production containers, env vars come from docker-compose 'environment:' key.
environ.Env.read_env(os.path.join(BASE_DIR, '..', '.env'))

# ── Security Keys ─────────────────────────────────────────────────────────────
# env() raises ImproperlyConfigured if SECRET_KEY is not set.
# This is intentional — you cannot accidentally run without a secret key.
SECRET_KEY = env('SECRET_KEY')

# ── Application Definition ────────────────────────────────────────────────────
DJANGO_APPS = [
    'django.contrib.admin',
    'django.contrib.auth',
    'django.contrib.contenttypes',
    'django.contrib.sessions',
    'django.contrib.messages',
    'django.contrib.staticfiles',
]

THIRD_PARTY_APPS = [
    'rest_framework',            # DRF — only if you're building an API
    'corsheaders',               # CORS headers
    'django_celery_beat',        # Periodic tasks stored in DB
    'django_celery_results',     # Task results stored in DB
]

LOCAL_APPS = [
    'apps.core',                 # Your custom apps
]

INSTALLED_APPS = DJANGO_APPS + THIRD_PARTY_APPS + LOCAL_APPS

# ── Middleware ────────────────────────────────────────────────────────────────
MIDDLEWARE = [
    'django.middleware.security.SecurityMiddleware',
    # WhiteNoise must be right after SecurityMiddleware
    # It serves static files efficiently without a separate web server
    'whitenoise.middleware.WhiteNoiseMiddleware',
    'django.contrib.sessions.middleware.SessionMiddleware',
    'corsheaders.middleware.CorsMiddleware',    # Must be before CommonMiddleware
    'django.middleware.common.CommonMiddleware',
    'django.middleware.csrf.CsrfViewMiddleware',
    'django.contrib.auth.middleware.AuthenticationMiddleware',
    'django.contrib.messages.middleware.MessageMiddleware',
    'django.middleware.clickjacking.XFrameOptionsMiddleware',
]

ROOT_URLCONF = 'myproject.urls'
WSGI_APPLICATION = 'myproject.wsgi.application'
ASGI_APPLICATION = 'myproject.asgi.application'

# ── Templates ─────────────────────────────────────────────────────────────────
TEMPLATES = [
    {
        'BACKEND': 'django.template.backends.django.DjangoTemplates',
        'DIRS': [BASE_DIR / 'templates'],
        'APP_DIRS': True,
        'OPTIONS': {
            'context_processors': [
                'django.template.context_processors.debug',
                'django.template.context_processors.request',
                'django.contrib.auth.context_processors.auth',
                'django.contrib.messages.context_processors.messages',
            ],
        },
    },
]

# ── Database ──────────────────────────────────────────────────────────────────
# env.db() reads a DATABASE_URL string and returns a Django DATABASES dict.
# Example: postgres://user:password@db:5432/mydb
# → {'ENGINE': 'django.db.backends.postgresql', 'HOST': 'db', ...}
#
# 'db' in the URL is the DOCKER SERVICE NAME — Docker DNS resolves it
# to the PostgreSQL container's IP address automatically.
# This is why container-to-container communication works by service name!
DATABASES = {
    'default': env.db('DATABASE_URL')
}

# Connection pooling — reuse connections for up to 60 seconds
# Prevents opening a new DB connection for every HTTP request
DATABASES['default']['CONN_MAX_AGE'] = 60

# ── Cache (Redis) ─────────────────────────────────────────────────────────────
# Using Redis as the cache backend.
# 'redis' in the URL is the Docker service name — same DNS magic as DB above.
CACHES = {
    'default': {
        'BACKEND': 'django_redis.cache.RedisCache',
        'LOCATION': env('REDIS_URL', default='redis://redis:6379/0'),
        'OPTIONS': {
            'CLIENT_CLASS': 'django_redis.client.DefaultClient',
            # Redis connection timeout — fail fast if Redis is down
            'SOCKET_CONNECT_TIMEOUT': 5,
            'SOCKET_TIMEOUT': 5,
        }
    }
}

# ── Session ───────────────────────────────────────────────────────────────────
# Store sessions in Redis (faster than DB, cleared automatically)
SESSION_ENGINE = 'django.contrib.sessions.backends.cache'
SESSION_CACHE_ALIAS = 'default'

# ── Password Validation ────────────────────────────────────────────────────────
AUTH_PASSWORD_VALIDATORS = [
    {'NAME': 'django.contrib.auth.password_validation.UserAttributeSimilarityValidator'},
    {'NAME': 'django.contrib.auth.password_validation.MinimumLengthValidator'},
    {'NAME': 'django.contrib.auth.password_validation.CommonPasswordValidator'},
    {'NAME': 'django.contrib.auth.password_validation.NumericPasswordValidator'},
]

# ── Internationalization ───────────────────────────────────────────────────────
LANGUAGE_CODE = 'en-us'
TIME_ZONE = 'UTC'                  # Always use UTC in servers — convert in frontend
USE_I18N = True
USE_TZ = True

# ── Static Files ──────────────────────────────────────────────────────────────
STATIC_URL = '/static/'
# STATIC_ROOT: Where 'collectstatic' gathers all static files.
# In docker-compose, this directory is shared with Nginx via a named volume.
STATIC_ROOT = BASE_DIR / 'staticfiles'

# WhiteNoise compression: Gzip + Brotli compress static files automatically
STATICFILES_STORAGE = 'whitenoise.storage.CompressedManifestStaticFilesStorage'

# ── Media Files (User Uploads) ─────────────────────────────────────────────────
MEDIA_URL = '/media/'
MEDIA_ROOT = BASE_DIR / 'mediafiles'

# ── Default Primary Key ────────────────────────────────────────────────────────
DEFAULT_AUTO_FIELD = 'django.db.models.BigAutoField'

# ── Celery Configuration ───────────────────────────────────────────────────────
# 'redis' is the Docker service name for the Redis container.
CELERY_BROKER_URL = env('CELERY_BROKER_URL', default='redis://redis:6379/1')
CELERY_RESULT_BACKEND = env('CELERY_RESULT_BACKEND', default='redis://redis:6379/2')
CELERY_ACCEPT_CONTENT = ['application/json']
CELERY_TASK_SERIALIZER = 'json'
CELERY_RESULT_SERIALIZER = 'json'
CELERY_TIMEZONE = TIME_ZONE
# Use django-celery-beat to store schedules in database (so you can edit them in admin)
CELERY_BEAT_SCHEDULER = 'django_celery_beat.schedulers:DatabaseScheduler'

# ── REST Framework ─────────────────────────────────────────────────────────────
REST_FRAMEWORK = {
    'DEFAULT_AUTHENTICATION_CLASSES': [
        'rest_framework.authentication.SessionAuthentication',
    ],
    'DEFAULT_PERMISSION_CLASSES': [
        'rest_framework.permissions.IsAuthenticated',
    ],
    'DEFAULT_RENDERER_CLASSES': [
        'rest_framework.renderers.JSONRenderer',
    ],
    'DEFAULT_PAGINATION_CLASS': 'rest_framework.pagination.PageNumberPagination',
    'PAGE_SIZE': 20,
}

# ── Logging ────────────────────────────────────────────────────────────────────
# Structured logging — all output goes to stdout/stderr.
# Docker captures these and makes them available via 'docker logs'.
# NEVER log to files in containers — files are ephemeral and you can't 'docker logs' them.
LOGGING = {
    'version': 1,
    'disable_existing_loggers': False,
    'formatters': {
        'verbose': {
            # Format: [2024-01-15 10:30:00] INFO django.request GET /api/users/ 200
            'format': '[{asctime}] {levelname} {name} {message}',
            'style': '{',
        },
    },
    'handlers': {
        # 'console' sends logs to stdout — Docker captures this with 'docker logs'
        'console': {
            'class': 'logging.StreamHandler',
            'formatter': 'verbose',
        },
    },
    'root': {
        'handlers': ['console'],
        'level': 'INFO',
    },
    'loggers': {
        'django': {
            'handlers': ['console'],
            'level': 'INFO',
            'propagate': False,
        },
        'django.db.backends': {
            # Set to 'DEBUG' to log all SQL queries (very verbose — dev only)
            'handlers': ['console'],
            'level': 'WARNING',
            'propagate': False,
        },
    },
}
```

---

### 📄 `src/myproject/settings/development.py`

```python
# ─────────────────────────────────────────────────────────────────────────────
# settings/development.py — Development-only settings
#
# Overrides base settings for local development.
# Selected with: DJANGO_SETTINGS_MODULE=myproject.settings.development
# ─────────────────────────────────────────────────────────────────────────────

from .base import *   # noqa: Import all base settings

# ── Debug Mode ────────────────────────────────────────────────────────────────
# In development, debug=True gives you the detailed error page.
# NEVER True in production — it leaks sensitive config info.
DEBUG = True

# ── Allowed Hosts ─────────────────────────────────────────────────────────────
# In development, allow all these typical local addresses.
ALLOWED_HOSTS = ['localhost', '127.0.0.1', '0.0.0.0', '*']

# ── Debug Toolbar ─────────────────────────────────────────────────────────────
# Shows SQL queries, cache hits, template rendering time in your browser.
# Only installed in dev (it's in requirements.dev.txt, not requirements.txt).
INSTALLED_APPS += ['debug_toolbar']

MIDDLEWARE += ['debug_toolbar.middleware.DebugToolbarMiddleware']

# Docker containers use non-loopback IPs (172.x.x.x).
# Debug toolbar only shows for IPs in INTERNAL_IPS.
# This allows it to work inside Docker containers.
INTERNAL_IPS = ['127.0.0.1', '172.17.0.1', '172.18.0.1']

# Alternative: show for ALL IPs in development (simpler)
DEBUG_TOOLBAR_CONFIG = {
    'SHOW_TOOLBAR_CALLBACK': lambda request: DEBUG,
}

# ── Database ──────────────────────────────────────────────────────────────────
# In dev, we don't need connection pooling — keep it simple.
DATABASES['default']['CONN_MAX_AGE'] = 0

# ── Email ─────────────────────────────────────────────────────────────────────
# Print emails to the terminal instead of actually sending them.
# You can see the email content in 'docker compose logs web'.
EMAIL_BACKEND = 'django.core.mail.backends.console.EmailBackend'

# ── Static Files ──────────────────────────────────────────────────────────────
# In development, Django serves static files directly via 'runserver'.
# No need for Nginx or collectstatic — simplifies setup.
STATICFILES_STORAGE = 'django.contrib.staticfiles.storage.StaticFilesStorage'

# ── Django Extensions (shell_plus etc.) ──────────────────────────────────────
INSTALLED_APPS += ['django_extensions']

# ── Logging ───────────────────────────────────────────────────────────────────
# In development, show all SQL queries (useful for debugging N+1 queries).
LOGGING['loggers']['django.db.backends']['level'] = 'DEBUG'
```

---

### 📄 `src/myproject/settings/production.py`

```python
# ─────────────────────────────────────────────────────────────────────────────
# settings/production.py — Production-only settings
#
# Security-hardened settings for production deployment.
# Selected with: DJANGO_SETTINGS_MODULE=myproject.settings.production
# ─────────────────────────────────────────────────────────────────────────────

from .base import *   # noqa

# ── Debug ─────────────────────────────────────────────────────────────────────
DEBUG = False    # MUST be False in production

# ── Allowed Hosts ─────────────────────────────────────────────────────────────
# Read from environment — set to your actual domain(s) in .env.production.
ALLOWED_HOSTS = env.list('ALLOWED_HOSTS')

# ── Security Headers ──────────────────────────────────────────────────────────
# These tell browsers to use HTTPS only, protect against XSS and clickjacking.

# Redirect all HTTP requests to HTTPS.
SECURE_SSL_REDIRECT = True

# Tell browsers to only ever connect via HTTPS for 1 year.
# max_age=31536000 = 1 year in seconds.
SECURE_HSTS_SECONDS = 31536000
SECURE_HSTS_INCLUDE_SUBDOMAINS = True   # Apply to subdomains too
SECURE_HSTS_PRELOAD = True              # Allow browser HSTS preload list

# Only send session cookie over HTTPS.
SESSION_COOKIE_SECURE = True

# Only send CSRF cookie over HTTPS.
CSRF_COOKIE_SECURE = True

# X-Content-Type-Options: nosniff — prevent MIME type sniffing.
SECURE_CONTENT_TYPE_NOSNIFF = True

# X-XSS-Protection header.
SECURE_BROWSER_XSS_FILTER = True

# Clickjacking protection — only allow your site to be iframed by itself.
X_FRAME_OPTIONS = 'DENY'

# ── CSRF Trusted Origins ──────────────────────────────────────────────────────
# Required when running behind a reverse proxy (Nginx).
# Django 4.0+ requires the full origin (scheme + domain) in this list.
CSRF_TRUSTED_ORIGINS = env.list(
    'CSRF_TRUSTED_ORIGINS',
    default=['https://yourdomain.com', 'https://www.yourdomain.com']
)

# ── Database ──────────────────────────────────────────────────────────────────
# In production, enable connection persistence to avoid opening a new
# connection for every request (big performance improvement).
DATABASES['default']['CONN_MAX_AGE'] = 60
DATABASES['default']['OPTIONS'] = {
    'connect_timeout': 10,    # Fail fast if DB is unreachable
}

# ── Email ─────────────────────────────────────────────────────────────────────
EMAIL_BACKEND = 'django.core.mail.backends.smtp.EmailBackend'
EMAIL_HOST = env('EMAIL_HOST', default='smtp.gmail.com')
EMAIL_PORT = env.int('EMAIL_PORT', default=587)
EMAIL_USE_TLS = True
EMAIL_HOST_USER = env('EMAIL_HOST_USER')
EMAIL_HOST_PASSWORD = env('EMAIL_HOST_PASSWORD')
DEFAULT_FROM_EMAIL = env('DEFAULT_FROM_EMAIL', default='noreply@yourdomain.com')

# ── Logging ───────────────────────────────────────────────────────────────────
# In production: WARNING level (not DEBUG) to reduce log noise.
# Don't log all SQL queries in production — too verbose and a security risk.
LOGGING['root']['level'] = 'WARNING'
LOGGING['loggers']['django']['level'] = 'WARNING'
```

---

### 📄 `src/myproject/urls.py` — Add Health Check URL

```python
# ─────────────────────────────────────────────────────────────────────────────
# urls.py — Add a /health/ endpoint for Docker health checks
#
# Docker's HEALTHCHECK and load balancers ping this URL to verify the app
# is running and can connect to the database.
# ─────────────────────────────────────────────────────────────────────────────

from django.contrib import admin
from django.urls import path, include
from django.http import JsonResponse
from django.db import connection


def health_check(request):
    """
    Health check endpoint.
    
    Returns 200 if:
    - Django is running
    - Database connection is working
    
    Returns 503 if the database is unreachable.
    
    Docker uses this via:
    HEALTHCHECK CMD curl -f http://localhost:8000/health/ || exit 1
    """
    try:
        # Actually test the database connection — a real connectivity check.
        # If PostgreSQL is down, this raises an exception → returns 503.
        connection.ensure_connection()
        return JsonResponse({
            "status": "healthy",
            "database": "connected"
        }, status=200)
    except Exception as e:
        return JsonResponse({
            "status": "unhealthy",
            "database": "disconnected",
            "error": str(e)
        }, status=503)


urlpatterns = [
    path('admin/', admin.site.urls),
    
    # Health check — used by Docker HEALTHCHECK and load balancers.
    # Must be lightweight and fast — it's called every 30 seconds.
    path('health/', health_check, name='health_check'),
    
    # Your app URLs
    path('api/', include('apps.core.urls')),
    
    # Debug toolbar — only active in development (guard with settings check)
    path('__debug__/', include('debug_toolbar.urls')),
]
```

---

## Step 9: Entrypoint Script

The entrypoint script runs every time the container starts. It handles startup tasks that must happen **before** the main process.

### 📄 `entrypoint.sh`

```bash
#!/bin/bash
# ─────────────────────────────────────────────────────────────────────────────
# entrypoint.sh — Container startup script
#
# This script runs EVERY TIME the container starts, before the main CMD.
# It handles:
#   1. Waiting for the database to be ready (critical — Django crashes if DB not ready)
#   2. Running database migrations (keeps schema up to date on deploy)
#   3. Collecting static files (if in production mode)
#   4. Starting the main application (Django or Celery)
#
# HOW IT WORKS with Dockerfile:
#   ENTRYPOINT ["/entrypoint.sh"]   ← This script always runs first
#   CMD ["gunicorn", "..."]         ← CMD is passed as arguments to this script
#   The script ends with 'exec "$@"' which executes whatever CMD was passed.
# ─────────────────────────────────────────────────────────────────────────────

# set -e: Exit immediately if ANY command fails.
# Without this, the script continues even if 'migrate' fails —
# then your app starts with an outdated schema (bugs!).
set -e

# set -o pipefail: If a command in a pipe fails, the whole pipe fails.
# Example: 'bad_command | grep x' — without this, the pipe "succeeds"
# because grep succeeded, hiding the bad_command failure.
set -o pipefail

# ── Color Output ──────────────────────────────────────────────────────────────
RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
NC='\033[0m'  # No Color

log_info()    { echo -e "${GREEN}[ENTRYPOINT]${NC} $1"; }
log_warning() { echo -e "${YELLOW}[ENTRYPOINT]${NC} $1"; }
log_error()   { echo -e "${RED}[ENTRYPOINT]${NC} $1"; }

# ── Wait for PostgreSQL ───────────────────────────────────────────────────────
# CRITICAL: Docker starts containers simultaneously. Even with 'depends_on',
# Django might start before PostgreSQL is ready to accept connections.
# Without this wait, Django crashes on startup with "Connection refused".
#
# We extract the database host from DATABASE_URL environment variable.
# DATABASE_URL format: postgres://user:password@host:port/dbname
# We need the 'host' part.

log_info "Waiting for PostgreSQL to be ready..."

# Extract DB host from DATABASE_URL
# Example: postgres://django_user:password@db:5432/django_db → db
DB_HOST=$(echo $DATABASE_URL | python3 -c "
import sys
from urllib.parse import urlparse
url = urlparse(sys.stdin.read().strip())
print(url.hostname)
")

# Extract DB port
DB_PORT=$(echo $DATABASE_URL | python3 -c "
import sys
from urllib.parse import urlparse
url = urlparse(sys.stdin.read().strip())
print(url.port or 5432)
")

# Loop until PostgreSQL accepts connections on port 5432.
# 'nc' (netcat) checks if a TCP port is open.
# We try every second for up to 60 seconds.
MAX_RETRIES=60
RETRY_COUNT=0

while ! nc -z "$DB_HOST" "$DB_PORT" 2>/dev/null; do
    RETRY_COUNT=$((RETRY_COUNT + 1))
    
    if [ $RETRY_COUNT -ge $MAX_RETRIES ]; then
        log_error "PostgreSQL at $DB_HOST:$DB_PORT did not become ready in ${MAX_RETRIES}s. Exiting."
        exit 1
    fi
    
    log_warning "PostgreSQL not ready (attempt $RETRY_COUNT/$MAX_RETRIES) — waiting 1s..."
    sleep 1
done

log_info "✅ PostgreSQL is ready at $DB_HOST:$DB_PORT"

# ── Wait for Redis (optional — uncomment if your app uses Redis on startup) ────
# REDIS_HOST=$(echo $REDIS_URL | python3 -c "
# import sys
# from urllib.parse import urlparse
# url = urlparse(sys.stdin.read().strip())
# print(url.hostname)
# ")
# 
# log_info "Waiting for Redis..."
# while ! nc -z "$REDIS_HOST" 6379 2>/dev/null; do
#     sleep 1
# done
# log_info "✅ Redis is ready"

# ── Database Migrations ───────────────────────────────────────────────────────
# Run migrations automatically on every container start.
#
# WHY: When you deploy a new version with new migrations, running migrations
# here ensures the schema is updated before the app starts serving traffic.
#
# GOTCHA for multi-replica deployments:
# If you run 3 web containers, all 3 will try to run migrations simultaneously.
# Django's migration system uses database locks to handle this safely — only
# one migration runs at a time, others wait. But for large deployments,
# prefer a separate 'migration job' container that runs once before web starts.
#
# --noinput: Don't ask for confirmation (required for non-interactive container)
log_info "Running database migrations..."
python manage.py migrate --noinput

log_info "✅ Migrations complete"

# ── Collect Static Files ──────────────────────────────────────────────────────
# Gather all static files into STATIC_ROOT for Nginx to serve.
# Only needed in PRODUCTION — in dev, runserver handles static files.
#
# We check DJANGO_SETTINGS_MODULE to skip this in development.
if [[ "$DJANGO_SETTINGS_MODULE" == *"production"* ]]; then
    log_info "Collecting static files..."
    python manage.py collectstatic --noinput --clear
    log_info "✅ Static files collected"
fi

# ── Create Superuser (optional — development only) ────────────────────────────
# Automatically create a superuser in development for convenience.
# In production, superusers should be created manually.
if [[ "$DJANGO_SETTINGS_MODULE" == *"development"* ]] && [[ "$CREATE_SUPERUSER" == "true" ]]; then
    log_info "Creating development superuser..."
    python manage.py shell -c "
from django.contrib.auth import get_user_model
User = get_user_model()
if not User.objects.filter(username='admin').exists():
    User.objects.create_superuser('admin', 'admin@example.com', 'admin')
    print('Superuser created: admin/admin')
else:
    print('Superuser already exists')
"
fi

# ── Start Main Process ────────────────────────────────────────────────────────
# 'exec "$@"' replaces the shell process with the CMD passed to this script.
#
# WHY exec?
# Without exec: shell → gunicorn (two processes, signals to shell don't reach gunicorn)
# With exec: gunicorn replaces shell (one process, Docker signals reach gunicorn)
#
# This is critical for graceful shutdown — 'docker stop' sends SIGTERM.
# With exec, gunicorn receives it and shuts down gracefully (finishes requests).
# Without exec, the shell receives it, ignores it, and Docker eventually force-kills.
log_info "🚀 Starting: $@"
exec "$@"
```

```bash
# Make the script executable — critical, Docker won't run it otherwise
chmod +x entrypoint.sh
```

### Update Dockerfiles to Use Entrypoint

Add these lines to **both** `Dockerfile.dev` and `Dockerfile`:

```dockerfile
# Add to Dockerfile.dev (before CMD)
COPY entrypoint.sh /entrypoint.sh
RUN chmod +x /entrypoint.sh
ENTRYPOINT ["/entrypoint.sh"]
CMD ["python", "manage.py", "runserver", "0.0.0.0:8000"]

# Add to Dockerfile production (before CMD, after USER django)
COPY --chown=django:django entrypoint.sh /entrypoint.sh
RUN chmod +x /entrypoint.sh
ENTRYPOINT ["/entrypoint.sh"]
CMD ["gunicorn", "myproject.wsgi:application", ...]
```

---

## Step 10: Nginx Configuration

Nginx sits in front of Gunicorn as a **reverse proxy**. It handles SSL, static files, and load balancing.

### 📄 `config/nginx/nginx.conf` — Main Nginx Config

```nginx
# ─────────────────────────────────────────────────────────────────────────────
# nginx.conf — Main Nginx configuration
#
# This file configures Nginx at the global level.
# The server block (where we define our Django site) goes in django.conf.
# ─────────────────────────────────────────────────────────────────────────────

# user nginx: Run as the nginx user (not root) — security best practice.
user nginx;

# worker_processes auto: Automatically match number of CPU cores.
# On a 4-core machine → 4 worker processes.
worker_processes auto;

# Error log location and level.
# In Docker, we redirect to /dev/stderr so 'docker logs nginx' shows errors.
error_log /dev/stderr warn;

# pid: Where nginx writes its PID (process ID) file.
pid /var/run/nginx.pid;

events {
    # worker_connections: Max simultaneous connections per worker process.
    # Total max connections = worker_processes × worker_connections
    # Default 1024 is fine for most Django apps.
    worker_connections 1024;

    # epoll: Linux-specific event method — most efficient on Linux containers.
    use epoll;
    
    # multi_accept: Accept multiple connections at once (improves throughput).
    multi_accept on;
}

http {
    # ── MIME Types ────────────────────────────────────────────────────────────
    # Include the default MIME type mappings.
    # Without this, files served without correct Content-Type cause browser issues.
    include       /etc/nginx/mime.types;
    default_type  application/octet-stream;

    # ── Logging ────────────────────────────────────────────────────────────────
    # Log format — captures the real client IP even behind a proxy.
    # $http_x_forwarded_for: The actual client IP (passed by upstream proxy).
    log_format main '$remote_addr - $remote_user [$time_local] "$request" '
                    '$status $body_bytes_sent "$http_referer" '
                    '"$http_user_agent" "$http_x_forwarded_for"';

    # Send access logs to stdout — 'docker logs nginx' captures this.
    access_log /dev/stdout main;

    # ── Performance ────────────────────────────────────────────────────────────
    # sendfile: Serve files directly from OS kernel (bypasses user-space copy).
    # Dramatically faster for static file serving.
    sendfile on;
    
    # tcp_nopush: Batch send HTTP response headers with the start of the file.
    tcp_nopush on;
    
    # tcp_nodelay: Send small packets immediately (reduces latency for API responses).
    tcp_nodelay on;

    # ── Timeouts ───────────────────────────────────────────────────────────────
    # keepalive_timeout: How long to keep idle connections open.
    # Longer = fewer TCP handshakes = better performance.
    keepalive_timeout 65;

    # ── Compression ────────────────────────────────────────────────────────────
    # gzip: Compress responses before sending (reduces bandwidth 60-80%).
    gzip on;
    gzip_vary on;           # Add Vary: Accept-Encoding header
    gzip_min_length 1024;   # Only compress responses > 1KB
    gzip_types
        text/plain
        text/css
        text/xml
        text/javascript
        application/json
        application/javascript
        application/xml+rss
        image/svg+xml;

    # ── Security Headers ───────────────────────────────────────────────────────
    # Hide nginx version from HTTP headers (reduces attack surface).
    server_tokens off;

    # ── Upstream ───────────────────────────────────────────────────────────────
    # 'web' is the Docker service name for our Gunicorn container.
    # Docker DNS resolves 'web' to the container's IP automatically.
    # Port 8000 is where Gunicorn listens (internal, not exposed to internet).
    upstream django_app {
        server web:8000;
        
        # If you scale web to multiple replicas, add more servers here.
        # Nginx will round-robin between them.
        # server web:8000 weight=1;  # Load balancing with weights
        
        # keepalive: Keep persistent connections to Gunicorn workers.
        # Reduces connection overhead — especially important for high traffic.
        keepalive 32;
    }

    # ── Include Server Blocks ──────────────────────────────────────────────────
    # Load our Django server configuration.
    include /etc/nginx/conf.d/*.conf;
}
```

---

### 📄 `config/nginx/django.conf` — Server Block

```nginx
# ─────────────────────────────────────────────────────────────────────────────
# django.conf — Nginx server block for the Django application
#
# This handles:
#   1. HTTP → HTTPS redirect (port 80 → 443)
#   2. Static files (served directly by Nginx, NOT through Django)
#   3. Media files (user uploads, served directly)
#   4. Everything else → proxied to Gunicorn
# ─────────────────────────────────────────────────────────────────────────────

# ── HTTP Server (Port 80) — Redirect to HTTPS ─────────────────────────────────
server {
    listen 80;
    server_name yourdomain.com www.yourdomain.com;

    # Let's Encrypt certificate renewal needs to access /.well-known/ via HTTP.
    # This location handles that before the HTTPS redirect.
    location /.well-known/acme-challenge/ {
        root /var/www/certbot;
    }

    # Redirect all other HTTP traffic to HTTPS.
    # 301 = Permanent redirect (browsers remember this).
    location / {
        return 301 https://$host$request_uri;
    }
}

# ── HTTPS Server (Port 443) — Main Server ─────────────────────────────────────
server {
    listen 443 ssl http2;
    server_name yourdomain.com www.yourdomain.com;

    # ── SSL Certificate ─────────────────────────────────────────────────────────
    # Certificates from Let's Encrypt (generated by certbot container).
    # These files are in a shared Docker volume between nginx and certbot.
    ssl_certificate     /etc/letsencrypt/live/yourdomain.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/yourdomain.com/privkey.pem;

    # ── SSL Settings ────────────────────────────────────────────────────────────
    # Only allow modern, secure TLS versions.
    ssl_protocols TLSv1.2 TLSv1.3;
    
    # Strong cipher suites only.
    ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384;
    ssl_prefer_server_ciphers off;
    
    # Cache SSL sessions for 1 day — reduces handshake overhead.
    ssl_session_cache shared:SSL:10m;
    ssl_session_timeout 1d;

    # ── Upload Size ─────────────────────────────────────────────────────────────
    # Allow file uploads up to 20MB.
    # If your Django app uses FileField/ImageField, set this to your max upload size.
    client_max_body_size 20M;

    # ── Static Files ─────────────────────────────────────────────────────────────
    # Serve static files DIRECTLY from Nginx — bypasses Django/Gunicorn entirely.
    # This is MUCH faster because:
    # 1. No Python overhead — Nginx reads files from disk directly
    # 2. Nginx handles caching headers efficiently
    # 3. Gunicorn workers aren't tied up serving static files
    #
    # How the volume works:
    # - Django's 'collectstatic' writes to /app/staticfiles/ (in the web container)
    # - A named Docker volume 'static_files' is mounted at /app/staticfiles/ in web
    # - The SAME volume is mounted at /app/staticfiles/ in nginx (read-only)
    # - So nginx sees the same files that Django collected
    location /static/ {
        alias /app/staticfiles/;    # Files are in this directory in the container
        
        # Cache static files for 30 days in browser.
        # Safe because WhiteNoise adds content hash to filenames (cache-busting).
        expires 30d;
        add_header Cache-Control "public, immutable";
        
        # If file not found, return 404 immediately (don't proxy to Django).
        try_files $uri $uri/ =404;
    }

    # ── Media Files (User Uploads) ────────────────────────────────────────────
    # Same volume-sharing pattern as static files above.
    location /media/ {
        alias /app/mediafiles/;
        
        # Cache media files for 7 days.
        # Note: if users update profile pictures etc., the URL changes (UUIDs).
        expires 7d;
        add_header Cache-Control "public";
        
        try_files $uri $uri/ =404;
    }

    # ── Django Application ────────────────────────────────────────────────────
    # All other requests are PROXIED to Gunicorn.
    # 'django_app' is the upstream block defined in nginx.conf.
    location / {
        proxy_pass http://django_app;

        # ── Proxy Headers ──────────────────────────────────────────────────────
        # These headers tell Django the real client information.
        # Without them, Django thinks every request comes from the Nginx container.
        
        # Pass the original 'Host' header (your domain name).
        # Django uses this in ALLOWED_HOSTS check.
        proxy_set_header Host $http_host;
        
        # Pass the real client IP address.
        # Django uses this for rate limiting and logging.
        proxy_set_header X-Real-IP $remote_addr;
        
        # Pass the full chain of proxy IPs (important for logging).
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        
        # Tell Django whether the original request was HTTP or HTTPS.
        # Django uses this to generate correct absolute URLs.
        # Required for: SECURE_SSL_REDIRECT, SESSION_COOKIE_SECURE, etc.
        proxy_set_header X-Forwarded-Proto $scheme;
        
        # For WebSocket support (Django Channels).
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";

        # ── Timeouts ───────────────────────────────────────────────────────────
        # How long to wait for Gunicorn to respond.
        # For long-running views (reports, file processing), increase this.
        proxy_connect_timeout 60s;
        proxy_send_timeout    120s;
        proxy_read_timeout    120s;

        # ── Buffer Settings ────────────────────────────────────────────────────
        # Buffer responses from Gunicorn before sending to client.
        # Allows Gunicorn workers to finish quickly and accept new connections.
        proxy_buffering on;
        proxy_buffer_size 8k;
        proxy_buffers 8 8k;
    }

    # ── Health Check (no proxy — handled internally) ──────────────────────────
    # Optional: You can serve the health check directly from nginx.
    # location /nginx-health {
    #     return 200 'nginx ok\n';
    #     add_header Content-Type text/plain;
    # }
}
```

---

## Step 11: Docker Compose (Development)

This is what you run every day during development.

### 📄 `docker-compose.dev.yml`

```yaml
# ─────────────────────────────────────────────────────────────────────────────
# docker-compose.dev.yml — Development Environment
#
# Run with: docker compose -f docker-compose.dev.yml up
# Or create an alias: alias dcdev='docker compose -f docker-compose.dev.yml'
#
# CHARACTERISTICS:
#   - Live code reload (bind mount)
#   - Development Django server (runserver)
#   - All ports exposed for direct access
#   - Debug tools installed
#   - No Nginx (Django serves everything)
#   - Verbose logging
# ─────────────────────────────────────────────────────────────────────────────

# Compose file format version 3.9 supports all modern features.
version: "3.9"

# ── Named Volumes ─────────────────────────────────────────────────────────────
# Volumes declared here persist between 'docker compose down' and 'up' calls.
# They are ONLY deleted with 'docker compose down -v'.
volumes:
  # Persistent PostgreSQL data — your database survives container restarts.
  postgres_dev_data:
    name: myproject_postgres_dev_data   # Explicit name (easier to find with 'docker volume ls')

  # Persistent Redis data — optional, Redis works fine without persistence in dev.
  redis_dev_data:
    name: myproject_redis_dev_data

# ── Networks ──────────────────────────────────────────────────────────────────
# Custom bridge network so containers can reach each other by service name.
# Django connects to 'db:5432' and 'redis:6379' — Docker DNS resolves these names.
networks:
  django_dev:
    name: myproject_django_dev          # Explicit network name
    driver: bridge                      # Default driver, works well for single-host

# ─────────────────────────────────────────────────────────────────────────────
# SERVICES
# ─────────────────────────────────────────────────────────────────────────────
services:

  # ── PostgreSQL Database ──────────────────────────────────────────────────────
  db:
    # Use official PostgreSQL 15 Alpine image.
    # Alpine = very small Linux (5MB) — great for development.
    image: postgres:15-alpine

    # Give it a memorable name — used in 'docker exec' and 'docker logs'.
    container_name: myproject_db_dev

    # Restart policy for dev: 'unless-stopped' means it starts with Docker Desktop.
    restart: unless-stopped

    # ── PostgreSQL Configuration ───────────────────────────────────────────────
    # These env vars are read by the postgres container's entrypoint script.
    # They create the database and user on FIRST startup (not on subsequent starts).
    environment:
      # Read from your .env file (copied from .env.example).
      # The ${VAR:-default} syntax means: use VAR if set, else use 'default'.
      POSTGRES_DB: ${POSTGRES_DB:-django_db}
      POSTGRES_USER: ${POSTGRES_USER:-django_user}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD:-django_password}
      
      # Boost performance for development — faster but less durable.
      # NEVER use in production (data loss on crash).
      POSTGRES_INITDB_ARGS: "--encoding=UTF8 --lc-collate=C --lc-ctype=C"

    # ── Volumes ────────────────────────────────────────────────────────────────
    volumes:
      # Named volume for data persistence.
      # Data survives 'docker compose restart' and 'docker compose down'.
      # Data is DELETED only with 'docker compose down -v'.
      - postgres_dev_data:/var/lib/postgresql/data

      # Optional: Mount SQL initialization scripts.
      # Files here run on FIRST database creation only (not on restart).
      # - ./config/postgres/init.sql:/docker-entrypoint-initdb.d/init.sql:ro

    # ── Ports ─────────────────────────────────────────────────────────────────
    # Expose DB port to host in DEVELOPMENT ONLY.
    # This lets you connect with pgAdmin, TablePlus, DBeaver, or psql from your machine.
    # In PRODUCTION, NEVER expose the DB port publicly.
    ports:
      - "5432:5432"   # host_port:container_port

    # ── Health Check ──────────────────────────────────────────────────────────
    # 'depends_on: condition: service_healthy' in the web service uses this.
    # Without a health check, 'depends_on' only waits for the container to START,
    # not for PostgreSQL to actually accept connections (takes 2-5 more seconds).
    healthcheck:
      # pg_isready: Official PostgreSQL utility that checks if the server is ready.
      # -U: username, -d: database name
      test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER:-django_user} -d ${POSTGRES_DB:-django_db}"]
      interval: 5s        # Check every 5 seconds
      timeout: 3s         # Fail if check takes > 3 seconds
      retries: 10         # Mark unhealthy after 10 consecutive failures
      start_period: 10s   # Don't fail for first 10 seconds (startup time)

    networks:
      - django_dev

  # ── Redis ────────────────────────────────────────────────────────────────────
  redis:
    # Redis 7 Alpine — latest stable version, tiny image.
    image: redis:7-alpine

    container_name: myproject_redis_dev

    restart: unless-stopped

    # ── Redis Configuration ────────────────────────────────────────────────────
    # 'redis-server --appendonly yes' enables AOF persistence.
    # This means Redis data (cached sessions, Celery task states) survives restarts.
    # In production, you'd have more detailed redis.conf.
    command: redis-server --appendonly yes --maxmemory 256mb --maxmemory-policy allkeys-lru

    volumes:
      - redis_dev_data:/data   # Persist Redis data to named volume

    # Expose Redis port for development tools (Redis Commander, RedisInsight).
    ports:
      - "6379:6379"

    # Health Check: 'redis-cli ping' returns PONG if Redis is ready.
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s
      timeout: 3s
      retries: 5
      start_period: 5s

    networks:
      - django_dev

  # ── Django Web Application ────────────────────────────────────────────────────
  web:
    # BUILD from Dockerfile.dev (not from a pre-built image).
    # Every time you change Dockerfile.dev or requirements.dev.txt,
    # run 'docker compose build web' to rebuild.
    build:
      context: .              # Build context: send current directory to Docker daemon
      dockerfile: Dockerfile.dev  # Use the development Dockerfile

    container_name: myproject_web_dev

    # For Django manage.py pdb debugging — enables interactive terminal.
    # Without these, 'import pdb; pdb.set_trace()' won't work in Docker.
    stdin_open: true    # Equivalent to docker run -i
    tty: true           # Equivalent to docker run -t

    restart: unless-stopped

    # ── Volumes ────────────────────────────────────────────────────────────────
    volumes:
      # 🔑 THE KEY TO LIVE RELOADING IN DEVELOPMENT:
      # Bind mount: maps your local project root → /app in the container.
      # When you save a file in your editor, Django's runserver detects the change
      # and reloads automatically — NO rebuild needed.
      # This is why we DON'T copy code in Dockerfile.dev.
      - ./src:/app

      # Prevent the local virtual environment from overwriting the container's.
      # Without this, if you have a local 'venv/' folder, it overwrites /app/venv/
      # which is the container's Python environment → broken container.
      # This empty volume 'masks' any local venv from appearing in the container.
      # - /app/venv   # Uncomment if you have a venv/ inside src/

    # ── Ports ─────────────────────────────────────────────────────────────────
    # Map host port 8000 to container port 8000.
    # Access your app at: http://localhost:8000
    ports:
      - "8000:8000"

    # ── Environment Variables ──────────────────────────────────────────────────
    # Load from .env file first (lower priority).
    # Then environment: block overrides specific values (higher priority).
    env_file:
      - .env   # Loads .env file from project root

    environment:
      # Override specific settings for dev — these take priority over .env.
      DJANGO_SETTINGS_MODULE: myproject.settings.development
      DEBUG: "True"
      
      # DATABASE_URL uses the Docker service name 'db' as the hostname.
      # Docker DNS automatically resolves 'db' to the db container's IP.
      DATABASE_URL: postgres://${POSTGRES_USER:-django_user}:${POSTGRES_PASSWORD:-django_password}@db:5432/${POSTGRES_DB:-django_db}
      
      # REDIS_URL uses the 'redis' service name similarly.
      REDIS_URL: redis://redis:6379/0
      CELERY_BROKER_URL: redis://redis:6379/1
      CELERY_RESULT_BACKEND: redis://redis:6379/2

    # ── Service Dependencies ──────────────────────────────────────────────────
    # 'depends_on' with 'condition: service_healthy' means:
    # "Don't start 'web' until 'db' passes its health check."
    # This prevents Django from crashing on startup because PostgreSQL isn't ready.
    # Requires the 'healthcheck' block in the db service above.
    depends_on:
      db:
        condition: service_healthy    # Wait for PostgreSQL to pass health check
      redis:
        condition: service_healthy    # Wait for Redis to pass health check

    # ── Override Command ───────────────────────────────────────────────────────
    # Use Django's development server with auto-reload.
    # The entrypoint.sh runs first (migrations, etc.), then this CMD.
    # '0.0.0.0:8000' = listen on ALL interfaces (required in containers).
    command: python manage.py runserver 0.0.0.0:8000

    networks:
      - django_dev

  # ── Celery Worker (optional in dev — comment out if not using Celery) ─────────
  celery:
    # Reuse the same image as web — it has the same code and dependencies.
    build:
      context: .
      dockerfile: Dockerfile.dev

    container_name: myproject_celery_dev

    restart: unless-stopped

    volumes:
      - ./src:/app    # Same bind mount as web — live code reload for tasks too

    env_file:
      - .env

    environment:
      DJANGO_SETTINGS_MODULE: myproject.settings.development
      DATABASE_URL: postgres://${POSTGRES_USER:-django_user}:${POSTGRES_PASSWORD:-django_password}@db:5432/${POSTGRES_DB:-django_db}
      REDIS_URL: redis://redis:6379/0
      CELERY_BROKER_URL: redis://redis:6379/1
      CELERY_RESULT_BACKEND: redis://redis:6379/2

    # Override the CMD from Dockerfile to run Celery instead of Django.
    # -A myproject: Celery app module (your Django project's celery.py)
    # worker: Start a worker process
    # --loglevel=info: Verbose enough for development
    # --concurrency=2: 2 concurrent task workers (enough for dev)
    command: celery -A myproject worker --loglevel=info --concurrency=2

    depends_on:
      db:
        condition: service_healthy
      redis:
        condition: service_healthy

    networks:
      - django_dev

  # ── Celery Beat — Periodic Task Scheduler ─────────────────────────────────────
  celery-beat:
    build:
      context: .
      dockerfile: Dockerfile.dev

    container_name: myproject_celery_beat_dev

    restart: unless-stopped

    volumes:
      - ./src:/app

    env_file:
      - .env

    environment:
      DJANGO_SETTINGS_MODULE: myproject.settings.development
      DATABASE_URL: postgres://${POSTGRES_USER:-django_user}:${POSTGRES_PASSWORD:-django_password}@db:5432/${POSTGRES_DB:-django_db}
      CELERY_BROKER_URL: redis://redis:6379/1

    # beat: Start the scheduler that sends periodic tasks to the broker.
    # --scheduler: Use database scheduler (schedules stored in Django admin).
    command: celery -A myproject beat --loglevel=info --scheduler django_celery_beat.schedulers:DatabaseScheduler

    depends_on:
      db:
        condition: service_healthy
      redis:
        condition: service_healthy
      # Important: Celery Beat should start after Celery worker is ready.
      celery:
        condition: service_started

    networks:
      - django_dev

  # ── Flower — Celery Monitoring UI ─────────────────────────────────────────────
  flower:
    build:
      context: .
      dockerfile: Dockerfile.dev

    container_name: myproject_flower_dev

    restart: unless-stopped

    volumes:
      - ./src:/app

    env_file:
      - .env

    environment:
      CELERY_BROKER_URL: redis://redis:6379/1

    # Start Flower web UI on port 5555.
    # Access at: http://localhost:5555
    command: celery -A myproject flower --port=5555

    ports:
      - "5555:5555"   # Access Flower at http://localhost:5555

    depends_on:
      - redis
      - celery

    networks:
      - django_dev
```

---

## Step 12: Docker Compose (Production)

The production Compose file — security hardened, no dev tools, uses Nginx.

### 📄 `docker-compose.yml`

```yaml
# ─────────────────────────────────────────────────────────────────────────────
# docker-compose.yml — PRODUCTION Environment
#
# Run with: docker compose up -d
# (docker compose uses this file by default when no -f flag is given)
#
# DIFFERENCES FROM DEV:
#   - No bind mounts (code is baked into image)
#   - Nginx reverse proxy (production-grade HTTP server)
#   - Gunicorn (production WSGI server, not runserver)
#   - Non-root user
#   - No dev tools installed
#   - Only necessary ports exposed (only 80/443 — not 5432, 6379)
#   - Resource limits
#   - Stronger restart policies
# ─────────────────────────────────────────────────────────────────────────────

version: "3.9"

# ── Named Volumes ──────────────────────────────────────────────────────────────
volumes:
  # PostgreSQL data — the most critical volume. NEVER delete this in production.
  postgres_prod_data:
    name: myproject_postgres_prod_data

  # Redis data — for session persistence and Celery results.
  redis_prod_data:
    name: myproject_redis_prod_data

  # Static files — shared between web (writes via collectstatic) and nginx (reads).
  # This is how Nginx serves Django's static files without going through Python.
  static_files:
    name: myproject_static_files

  # Media files — user uploads shared between web and nginx.
  media_files:
    name: myproject_media_files

  # Let's Encrypt SSL certificates — shared between nginx and certbot.
  certbot_certs:
    name: myproject_certbot_certs

  certbot_www:
    name: myproject_certbot_www

# ── Networks ───────────────────────────────────────────────────────────────────
networks:
  django_prod:
    name: myproject_django_prod
    driver: bridge

# ─────────────────────────────────────────────────────────────────────────────
# SERVICES
# ─────────────────────────────────────────────────────────────────────────────
services:

  # ── PostgreSQL Database ──────────────────────────────────────────────────────
  db:
    image: postgres:15-alpine

    container_name: myproject_db_prod

    # always: restart even if explicitly stopped (production critical service).
    restart: always

    environment:
      POSTGRES_DB: ${POSTGRES_DB}         # Must be set — no defaults in production
      POSTGRES_USER: ${POSTGRES_USER}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}

    volumes:
      - postgres_prod_data:/var/lib/postgresql/data

    # ⚠️ NO 'ports:' block in production.
    # The database is only accessible INSIDE the Docker network.
    # External access = security vulnerability.
    # If you need to access it, use 'docker exec db psql' or an SSH tunnel.

    # ── Resource Limits ────────────────────────────────────────────────────────
    deploy:
      resources:
        limits:
          memory: 1G          # Max 1GB RAM for PostgreSQL
          cpus: '1.0'         # Max 1 CPU core
        reservations:
          memory: 512M        # Reserve 512MB guaranteed

    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER} -d ${POSTGRES_DB}"]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 30s

    networks:
      - django_prod

  # ── Redis ─────────────────────────────────────────────────────────────────────
  redis:
    image: redis:7-alpine

    container_name: myproject_redis_prod

    restart: always

    # Production Redis config:
    # --appendonly yes: AOF persistence (survive restart)
    # --requirepass: Password authentication
    # --maxmemory 512mb: Limit memory usage
    # --maxmemory-policy allkeys-lru: Evict least-recently-used keys when full
    command: >
      redis-server
      --appendonly yes
      --requirepass ${REDIS_PASSWORD}
      --maxmemory 512mb
      --maxmemory-policy allkeys-lru

    volumes:
      - redis_prod_data:/data

    # ⚠️ NO 'ports:' block — Redis not exposed externally.

    deploy:
      resources:
        limits:
          memory: 512M
          cpus: '0.5'

    healthcheck:
      # Must include auth password for health check.
      test: ["CMD", "redis-cli", "-a", "${REDIS_PASSWORD}", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5

    networks:
      - django_prod

  # ── Django Web Application ─────────────────────────────────────────────────────
  web:
    # In production, use a pre-built image (from your CI/CD registry).
    # ${IMAGE_TAG:-latest} means: use IMAGE_TAG if set, else 'latest'.
    image: ${DOCKER_REGISTRY:-mycompany}/django-app:${IMAGE_TAG:-latest}

    # Uncomment to build locally instead:
    # build:
    #   context: .
    #   dockerfile: Dockerfile
    #   target: production

    container_name: myproject_web_prod

    restart: always

    # ── Volumes ────────────────────────────────────────────────────────────────
    volumes:
      # Static files: collectstatic writes here, nginx reads from here.
      # 'web' has write access, 'nginx' has read-only access.
      - static_files:/app/staticfiles

      # Media files: Django writes uploads here, nginx serves them.
      - media_files:/app/mediafiles

    # ── NO 'ports:' block ─────────────────────────────────────────────────────
    # Web container is NOT directly exposed to the internet.
    # All traffic goes through Nginx → Nginx proxies to web:8000 internally.
    # 'expose:' (without 'ports:') tells Docker this container listens on 8000
    # but doesn't publish it to the host machine.
    expose:
      - "8000"

    environment:
      DJANGO_SETTINGS_MODULE: myproject.settings.production
      DEBUG: "False"
      SECRET_KEY: ${SECRET_KEY}
      ALLOWED_HOSTS: ${ALLOWED_HOSTS}
      CSRF_TRUSTED_ORIGINS: ${CSRF_TRUSTED_ORIGINS}
      DATABASE_URL: postgres://${POSTGRES_USER}:${POSTGRES_PASSWORD}@db:5432/${POSTGRES_DB}
      REDIS_URL: redis://:${REDIS_PASSWORD}@redis:6379/0
      CELERY_BROKER_URL: redis://:${REDIS_PASSWORD}@redis:6379/1
      CELERY_RESULT_BACKEND: redis://:${REDIS_PASSWORD}@redis:6379/2

    depends_on:
      db:
        condition: service_healthy
      redis:
        condition: service_healthy

    # ── Resource Limits ────────────────────────────────────────────────────────
    deploy:
      resources:
        limits:
          memory: 1G
          cpus: '1.5'
        reservations:
          memory: 512M

    healthcheck:
      # This calls the /health/ endpoint we defined in urls.py.
      # If Django or the DB is down, this returns non-200 → container marked unhealthy.
      test: ["CMD", "curl", "-f", "http://localhost:8000/health/"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 60s   # Give Django 60s to start (migrations take time)

    networks:
      - django_prod

  # ── Nginx Reverse Proxy ────────────────────────────────────────────────────────
  nginx:
    image: nginx:1.25-alpine

    container_name: myproject_nginx_prod

    restart: always

    volumes:
      # Nginx configuration files (read-only — nginx reads, doesn't modify them).
      - ./config/nginx/nginx.conf:/etc/nginx/nginx.conf:ro
      - ./config/nginx/django.conf:/etc/nginx/conf.d/default.conf:ro

      # Static files — read-only (nginx serves, web container writes via collectstatic).
      - static_files:/app/staticfiles:ro

      # Media files — read-only (nginx serves, Django writes uploads).
      - media_files:/app/mediafiles:ro

      # SSL certificates from Let's Encrypt (certbot writes, nginx reads).
      - certbot_certs:/etc/letsencrypt:ro
      - certbot_www:/var/www/certbot:ro

    # ── Ports ─────────────────────────────────────────────────────────────────
    # ONLY Nginx exposes ports to the internet.
    # 80: HTTP (redirects to HTTPS)
    # 443: HTTPS (main traffic)
    ports:
      - "80:80"
      - "443:443"

    depends_on:
      web:
        condition: service_healthy  # Wait for Django to be healthy before proxying

    deploy:
      resources:
        limits:
          memory: 128M   # Nginx is very memory efficient
          cpus: '0.5'

    healthcheck:
      test: ["CMD", "wget", "--no-verbose", "--tries=1", "--spider", "http://localhost/health/"]
      interval: 30s
      timeout: 10s
      retries: 3

    networks:
      - django_prod

  # ── Certbot — SSL Certificate Manager ─────────────────────────────────────────
  certbot:
    image: certbot/certbot:latest

    container_name: myproject_certbot

    # Certbot doesn't need to run continuously — it runs once to get certs,
    # then again every 60 days to renew. 'restart: no' is appropriate.
    restart: no

    volumes:
      - certbot_certs:/etc/letsencrypt
      - certbot_www:/var/www/certbot

    # Renew certificates. Run this manually or via cron every 60 days.
    # First run: use 'certonly' command (see Step 19).
    command: renew --quiet

    networks:
      - django_prod

  # ── Celery Worker ──────────────────────────────────────────────────────────────
  celery:
    image: ${DOCKER_REGISTRY:-mycompany}/django-app:${IMAGE_TAG:-latest}

    container_name: myproject_celery_prod

    restart: always

    environment:
      DJANGO_SETTINGS_MODULE: myproject.settings.production
      SECRET_KEY: ${SECRET_KEY}
      DATABASE_URL: postgres://${POSTGRES_USER}:${POSTGRES_PASSWORD}@db:5432/${POSTGRES_DB}
      REDIS_URL: redis://:${REDIS_PASSWORD}@redis:6379/0
      CELERY_BROKER_URL: redis://:${REDIS_PASSWORD}@redis:6379/1
      CELERY_RESULT_BACKEND: redis://:${REDIS_PASSWORD}@redis:6379/2

    # Production Celery: more concurrency, structured logging.
    command: >
      celery -A myproject worker
      --loglevel=info
      --concurrency=4
      --max-tasks-per-child=100

    depends_on:
      db:
        condition: service_healthy
      redis:
        condition: service_healthy

    deploy:
      resources:
        limits:
          memory: 512M
          cpus: '1.0'

    networks:
      - django_prod

  # ── Celery Beat ────────────────────────────────────────────────────────────────
  celery-beat:
    image: ${DOCKER_REGISTRY:-mycompany}/django-app:${IMAGE_TAG:-latest}

    container_name: myproject_celery_beat_prod

    restart: always

    environment:
      DJANGO_SETTINGS_MODULE: myproject.settings.production
      SECRET_KEY: ${SECRET_KEY}
      DATABASE_URL: postgres://${POSTGRES_USER}:${POSTGRES_PASSWORD}@db:5432/${POSTGRES_DB}
      CELERY_BROKER_URL: redis://:${REDIS_PASSWORD}@redis:6379/1

    command: >
      celery -A myproject beat
      --loglevel=info
      --scheduler django_celery_beat.schedulers:DatabaseScheduler

    depends_on:
      db:
        condition: service_healthy
      redis:
        condition: service_healthy

    deploy:
      resources:
        limits:
          memory: 256M
          cpus: '0.25'

    networks:
      - django_prod
```

---

## Step 13: First Run — Complete Workflow

Follow these steps **exactly in order** for the first time setup.

```bash
# ── STEP 1: Verify project structure ──────────────────────────────────────────
ls -la
# Should see: Dockerfile, Dockerfile.dev, docker-compose.yml,
#             docker-compose.dev.yml, .env, entrypoint.sh, src/

# ── STEP 2: Copy and configure .env ───────────────────────────────────────────
cp .env.example .env

# Open .env and set your values:
# - Generate SECRET_KEY: python -c "from django.core.management.utils import get_random_secret_key; print(get_random_secret_key())"
# - Set POSTGRES_PASSWORD to something you'll remember for dev
nano .env

# ── STEP 3: Build the development images ──────────────────────────────────────
# This downloads base images (Python, Postgres, Redis) and installs your
# Python packages. First build takes 3-5 minutes. Subsequent builds are faster (cache).
docker compose -f docker-compose.dev.yml build

# You should see output like:
# [+] Building 45.2s (12/12) FINISHED
# ✅ If you see this, your Dockerfile is correct.

# ── STEP 4: Start the database and Redis first ─────────────────────────────────
# Start only db and redis first — give them time to initialize before Django.
docker compose -f docker-compose.dev.yml up -d db redis

# Wait for health checks to pass (about 5-10 seconds)
docker compose -f docker-compose.dev.yml ps
# db should show: healthy
# redis should show: healthy

# ── STEP 5: Run database migrations ───────────────────────────────────────────
# '--rm': Remove the temporary container after migrations finish.
# This creates all Django tables in the PostgreSQL database.
docker compose -f docker-compose.dev.yml run --rm web python manage.py migrate

# Expected output:
# [ENTRYPOINT] Waiting for PostgreSQL to be ready...
# [ENTRYPOINT] ✅ PostgreSQL is ready
# [ENTRYPOINT] Running database migrations...
# Operations to perform: Apply all migrations...
# Running migrations: Applying contenttypes.0001... OK
# [ENTRYPOINT] ✅ Migrations complete

# ── STEP 6: Create a superuser ────────────────────────────────────────────────
docker compose -f docker-compose.dev.yml run --rm web python manage.py createsuperuser

# Enter: username, email, password when prompted.

# ── STEP 7: Start all services ────────────────────────────────────────────────
docker compose -f docker-compose.dev.yml up -d

# ── STEP 8: Verify all containers are running ──────────────────────────────────
docker compose -f docker-compose.dev.yml ps

# Expected output:
# NAME                          STATUS          PORTS
# myproject_db_dev              Up (healthy)    0.0.0.0:5432->5432/tcp
# myproject_redis_dev           Up (healthy)    0.0.0.0:6379->6379/tcp
# myproject_web_dev             Up              0.0.0.0:8000->8000/tcp
# myproject_celery_dev          Up
# myproject_celery_beat_dev     Up
# myproject_flower_dev          Up              0.0.0.0:5555->5555/tcp

# ── STEP 9: Access your application ────────────────────────────────────────────
echo "Django app:    http://localhost:8000"
echo "Django admin:  http://localhost:8000/admin"
echo "Health check:  http://localhost:8000/health/"
echo "Flower (tasks):http://localhost:5555"

# ── STEP 10: Watch logs ────────────────────────────────────────────────────────
docker compose -f docker-compose.dev.yml logs -f web
```

---

## Step 14: Daily Development Workflow

```bash
# ─────────────────────────────────────────────────────────────────────────────
# Create a Makefile for common commands (put in project root)
# ─────────────────────────────────────────────────────────────────────────────
```

### 📄 `Makefile`

```makefile
# ─────────────────────────────────────────────────────────────────────────────
# Makefile — Shortcuts for common Docker commands
#
# Usage: make <target>
# Example: make migrate
# ─────────────────────────────────────────────────────────────────────────────

# Development compose file shorthand
DC_DEV = docker compose -f docker-compose.dev.yml
DC_PROD = docker compose

# ── Development Commands ───────────────────────────────────────────────────────

# Start all development services
up:
	$(DC_DEV) up -d

# Start and rebuild (after Dockerfile or requirements changes)
up-build:
	$(DC_DEV) up -d --build

# Stop all services (keep volumes/data)
down:
	$(DC_DEV) down

# Stop and delete volumes (DESTROYS DATABASE — use with caution!)
down-clean:
	$(DC_DEV) down -v

# View logs for all services
logs:
	$(DC_DEV) logs -f

# View logs for specific service: make logs-web
logs-web:
	$(DC_DEV) logs -f web

logs-celery:
	$(DC_DEV) logs -f celery

logs-db:
	$(DC_DEV) logs -f db

# ── Django Commands ────────────────────────────────────────────────────────────

# Run migrations
migrate:
	$(DC_DEV) run --rm web python manage.py migrate

# Create migration files
makemigrations:
	$(DC_DEV) run --rm web python manage.py makemigrations

# Open Django shell
shell:
	$(DC_DEV) run --rm web python manage.py shell_plus

# Create superuser
superuser:
	$(DC_DEV) run --rm web python manage.py createsuperuser

# Collect static files
collectstatic:
	$(DC_DEV) run --rm web python manage.py collectstatic --noinput

# Run all tests
test:
	$(DC_DEV) run --rm web python manage.py test

# Run tests with coverage
test-coverage:
	$(DC_DEV) run --rm web bash -c "coverage run manage.py test && coverage report"

# Run a specific test: make test-app APP=core
test-app:
	$(DC_DEV) run --rm web python manage.py test apps.$(APP)

# ── Database Commands ─────────────────────────────────────────────────────────

# Access PostgreSQL CLI
psql:
	$(DC_DEV) exec db psql -U ${POSTGRES_USER} -d ${POSTGRES_DB}

# Backup database
db-backup:
	$(DC_DEV) exec -T db pg_dump -U ${POSTGRES_USER} ${POSTGRES_DB} | gzip > backup_$(shell date +%Y%m%d_%H%M%S).sql.gz
	@echo "✅ Database backed up"

# ── Build Commands ─────────────────────────────────────────────────────────────

# Build development image
build-dev:
	$(DC_DEV) build

# Build production image
build-prod:
	docker build -t myapp:latest .

# ── Production Commands ────────────────────────────────────────────────────────

# Deploy to production
prod-up:
	$(DC_PROD) up -d

prod-down:
	$(DC_PROD) down

prod-migrate:
	$(DC_PROD) run --rm web python manage.py migrate --noinput

# ── Cleanup ────────────────────────────────────────────────────────────────────

# Remove all unused Docker objects
prune:
	docker system prune -f

.PHONY: up up-build down down-clean logs logs-web logs-celery logs-db \
        migrate makemigrations shell superuser collectstatic \
        test test-coverage psql db-backup build-dev build-prod \
        prod-up prod-down prod-migrate prune
```

---

## Step 15: Adding Celery & Celery Beat

### 📄 `src/myproject/celery.py`

```python
# ─────────────────────────────────────────────────────────────────────────────
# myproject/celery.py — Celery application configuration
#
# This file configures the Celery application for Django.
# It's imported by the Celery worker container (via the 'command' in compose).
# ─────────────────────────────────────────────────────────────────────────────

import os
from celery import Celery

# Set the Django settings module for the 'celery' program.
# This is important — Celery needs to know your Django settings
# so it can access DATABASES, CACHES, CELERY_BROKER_URL, etc.
os.environ.setdefault('DJANGO_SETTINGS_MODULE', 'myproject.settings.development')

# Create the Celery application instance.
# 'myproject' is the name of this Celery app — used in task names.
app = Celery('myproject')

# Load Celery configuration from Django settings.
# All settings starting with 'CELERY_' in settings.py are used.
# namespace='CELERY' means only CELERY_* settings are loaded.
app.config_from_object('django.conf:settings', namespace='CELERY')

# Auto-discover tasks in all installed apps.
# Celery looks for a 'tasks.py' file in each installed app.
# This means you don't need to manually import every task.
app.autodiscover_tasks()


@app.task(bind=True, ignore_result=True)
def debug_task(self):
    """
    Simple debug task to verify Celery is working.
    Run with: docker compose exec web python manage.py shell
    >>> from myproject.celery import debug_task
    >>> debug_task.delay()
    """
    print(f'Request: {self.request!r}')
```

---

### 📄 `src/myproject/__init__.py`

```python
# ─────────────────────────────────────────────────────────────────────────────
# myproject/__init__.py
#
# This ensures that the Celery app is always imported when Django starts,
# so that @shared_task decorators use this Celery app.
# Without this, @shared_task might not find the Celery app.
# ─────────────────────────────────────────────────────────────────────────────

# Import the celery app so it's registered with Django.
from .celery import app as celery_app

# __all__ makes the celery_app available via 'from myproject import celery_app'
__all__ = ('celery_app',)
```

---

### 📄 `src/apps/core/tasks.py` — Example Tasks

```python
# ─────────────────────────────────────────────────────────────────────────────
# apps/core/tasks.py — Example Celery tasks
# ─────────────────────────────────────────────────────────────────────────────

from celery import shared_task
from celery.utils.log import get_task_logger

# Use Celery's logger — output appears in 'docker compose logs celery'
logger = get_task_logger(__name__)


@shared_task(
    bind=True,              # Access 'self' for retries
    max_retries=3,          # Retry up to 3 times on failure
    default_retry_delay=60, # Wait 60 seconds between retries
)
def send_welcome_email(self, user_id: int) -> dict:
    """
    Send a welcome email to a new user.
    
    This task runs asynchronously in the Celery worker container,
    NOT in the Django web container — keeping web responses fast.
    
    Usage in views.py:
        from apps.core.tasks import send_welcome_email
        send_welcome_email.delay(user.id)  # Fire and forget
        
        # Or with a delay:
        send_welcome_email.apply_async(args=[user.id], countdown=60)
    """
    from django.contrib.auth import get_user_model
    from django.core.mail import send_mail
    
    User = get_user_model()
    
    try:
        user = User.objects.get(id=user_id)
        logger.info(f"Sending welcome email to {user.email}")
        
        send_mail(
            subject='Welcome to Our App!',
            message=f'Hi {user.username}, welcome aboard!',
            from_email='noreply@example.com',
            recipient_list=[user.email],
        )
        
        logger.info(f"Welcome email sent to {user.email}")
        return {"status": "sent", "email": user.email}
        
    except User.DoesNotExist:
        logger.error(f"User {user_id} not found")
        return {"status": "error", "reason": "user_not_found"}
        
    except Exception as exc:
        logger.error(f"Failed to send email: {exc}")
        # Retry the task with exponential backoff
        raise self.retry(exc=exc, countdown=60 * (self.request.retries + 1))
```

---

## Step 16: Adding Flower (Celery Monitoring)

Flower is already configured in `docker-compose.dev.yml`. Access it at `http://localhost:5555`.

Add basic auth for production:

```yaml
# In docker-compose.yml (production), add to flower service:
  flower:
    image: ${DOCKER_REGISTRY:-mycompany}/django-app:${IMAGE_TAG:-latest}
    container_name: myproject_flower_prod
    restart: always
    command: >
      celery -A myproject flower
      --port=5555
      --basic-auth=${FLOWER_USER}:${FLOWER_PASSWORD}
    ports:
      - "5555:5555"    # Or put behind Nginx with /flower/ path
    environment:
      CELERY_BROKER_URL: redis://:${REDIS_PASSWORD}@redis:6379/1
    networks:
      - django_prod
```

---

## Step 17: Health Checks & Readiness

### 📄 `src/apps/core/views.py` — Enhanced Health Check

```python
# ─────────────────────────────────────────────────────────────────────────────
# Comprehensive health check for production readiness monitoring.
# ─────────────────────────────────────────────────────────────────────────────

import time
from django.db import connection
from django.core.cache import cache
from django.http import JsonResponse


def health_check(request):
    """
    Multi-component health check endpoint.
    
    Checks:
    - Database connectivity
    - Redis/Cache connectivity
    - Response time
    
    Returns 200 if all healthy, 503 if any component is down.
    Used by Docker HEALTHCHECK, load balancers, and monitoring tools.
    """
    health_status = {
        "status": "healthy",
        "timestamp": time.time(),
        "components": {}
    }
    
    http_status = 200
    
    # ── Check Database ─────────────────────────────────────────────────────────
    try:
        start = time.time()
        connection.ensure_connection()
        # Run a simple query to verify the connection works end-to-end
        with connection.cursor() as cursor:
            cursor.execute("SELECT 1")
        db_time = time.time() - start
        
        health_status["components"]["database"] = {
            "status": "healthy",
            "response_time_ms": round(db_time * 1000, 2)
        }
    except Exception as e:
        health_status["components"]["database"] = {
            "status": "unhealthy",
            "error": str(e)
        }
        health_status["status"] = "unhealthy"
        http_status = 503
    
    # ── Check Cache (Redis) ────────────────────────────────────────────────────
    try:
        start = time.time()
        cache.set("health_check", "ok", timeout=10)
        result = cache.get("health_check")
        cache_time = time.time() - start
        
        health_status["components"]["cache"] = {
            "status": "healthy" if result == "ok" else "unhealthy",
            "response_time_ms": round(cache_time * 1000, 2)
        }
        if result != "ok":
            health_status["status"] = "unhealthy"
            http_status = 503
    except Exception as e:
        health_status["components"]["cache"] = {
            "status": "unhealthy",
            "error": str(e)
        }
        health_status["status"] = "degraded"  # Cache down but DB is up
    
    return JsonResponse(health_status, status=http_status)
```

---

## Step 18: CI/CD Pipeline (GitHub Actions)

### 📄 `.github/workflows/deploy.yml`

```yaml
# ─────────────────────────────────────────────────────────────────────────────
# deploy.yml — GitHub Actions CI/CD Pipeline
#
# Triggers on push to main branch.
# Steps:
#   1. Run tests
#   2. Build Docker image
#   3. Push to registry
#   4. Deploy to production server
# ─────────────────────────────────────────────────────────────────────────────

name: Test, Build, and Deploy

on:
  push:
    branches: [main]          # Deploy on push to main
  pull_request:
    branches: [main]          # Test on PRs (but don't deploy)

env:
  # Docker image name — change to your registry/org/name
  IMAGE_NAME: ${{ secrets.DOCKER_REGISTRY }}/django-app

jobs:

  # ── Job 1: Run Tests ─────────────────────────────────────────────────────────
  test:
    name: Run Django Tests
    runs-on: ubuntu-latest

    services:
      # GitHub Actions can spin up service containers for testing.
      postgres:
        image: postgres:15-alpine
        env:
          POSTGRES_USER: test_user
          POSTGRES_PASSWORD: test_password
          POSTGRES_DB: test_db
        # Wait for postgres to be healthy before running tests.
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
        ports:
          - 5432:5432

      redis:
        image: redis:7-alpine
        options: >-
          --health-cmd "redis-cli ping"
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
        ports:
          - 6379:6379

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Set up Python 3.11
        uses: actions/setup-python@v5
        with:
          python-version: '3.11'
          # Cache pip packages between workflow runs — speeds up CI
          cache: 'pip'

      - name: Install dependencies
        run: |
          pip install -r requirements.dev.txt

      - name: Run Django system checks
        env:
          DJANGO_SETTINGS_MODULE: myproject.settings.development
          SECRET_KEY: ci-test-secret-key-not-for-production
          DATABASE_URL: postgres://test_user:test_password@localhost:5432/test_db
          REDIS_URL: redis://localhost:6379/0
        run: |
          cd src
          python manage.py check

      - name: Run tests with coverage
        env:
          DJANGO_SETTINGS_MODULE: myproject.settings.development
          SECRET_KEY: ci-test-secret-key-not-for-production
          DATABASE_URL: postgres://test_user:test_password@localhost:5432/test_db
          REDIS_URL: redis://localhost:6379/0
          CELERY_BROKER_URL: redis://localhost:6379/1
          CELERY_RESULT_BACKEND: redis://localhost:6379/2
        run: |
          cd src
          coverage run manage.py test
          coverage report --fail-under=80   # Fail if coverage drops below 80%

      - name: Upload coverage report
        uses: actions/upload-artifact@v4
        with:
          name: coverage-report
          path: src/htmlcov/

  # ── Job 2: Build and Push Docker Image ───────────────────────────────────────
  build:
    name: Build Docker Image
    runs-on: ubuntu-latest
    needs: test    # Only run if tests pass
    if: github.ref == 'refs/heads/main'   # Only on main branch (not PRs)

    outputs:
      # Pass the image tag to the deploy job
      image_tag: ${{ steps.meta.outputs.tags }}
      git_sha: ${{ steps.vars.outputs.sha_short }}

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Set up Docker Buildx
        # Buildx enables multi-platform builds and advanced caching.
        uses: docker/setup-buildx-action@v3

      - name: Get short commit SHA
        id: vars
        run: echo "sha_short=$(git rev-parse --short HEAD)" >> $GITHUB_OUTPUT

      - name: Login to Docker Registry
        uses: docker/login-action@v3
        with:
          registry: ${{ secrets.DOCKER_REGISTRY }}
          username: ${{ secrets.DOCKER_USERNAME }}
          password: ${{ secrets.DOCKER_PASSWORD }}

      - name: Build and push Docker image
        uses: docker/build-push-action@v5
        with:
          context: .
          file: ./Dockerfile
          target: production     # Build only the 'production' stage
          push: true
          tags: |
            ${{ env.IMAGE_NAME }}:latest
            ${{ env.IMAGE_NAME }}:${{ steps.vars.outputs.sha_short }}
          # Use GitHub Actions cache to speed up builds.
          # Layers from previous builds are reused where possible.
          cache-from: type=gha
          cache-to: type=gha,mode=max

  # ── Job 3: Deploy to Production ───────────────────────────────────────────────
  deploy:
    name: Deploy to Production
    runs-on: ubuntu-latest
    needs: build   # Only run if build succeeds
    if: github.ref == 'refs/heads/main'
    
    # Use GitHub's deployment environments for approval workflows.
    environment:
      name: production
      url: https://yourdomain.com

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Deploy to server via SSH
        uses: appleboy/ssh-action@v1
        with:
          host: ${{ secrets.PRODUCTION_HOST }}
          username: ${{ secrets.PRODUCTION_USER }}
          key: ${{ secrets.PRODUCTION_SSH_KEY }}
          script: |
            set -e
            
            echo "🚀 Starting deployment..."
            
            # Navigate to project directory on server
            cd /opt/myproject
            
            # Pull latest compose files
            git pull origin main
            
            # Export the image tag for docker-compose.yml substitution
            export IMAGE_TAG=${{ needs.build.outputs.git_sha }}
            
            # Pull the new image
            docker pull ${{ env.IMAGE_NAME }}:${IMAGE_TAG}
            
            # Run database migrations (before starting new version)
            docker compose run --rm web python manage.py migrate --noinput
            
            # Collect static files
            docker compose run --rm web python manage.py collectstatic --noinput
            
            # Restart web service with the new image
            docker compose up -d --no-deps --force-recreate web
            
            # Wait for health check to pass
            sleep 15
            docker compose exec web curl -f http://localhost:8000/health/ || exit 1
            
            # Also restart Celery workers (so they pick up new task code)
            docker compose up -d --no-deps --force-recreate celery celery-beat
            
            # Clean up old images
            docker image prune -f
            
            echo "✅ Deployment complete!"
```

---

## Step 19: SSL / HTTPS with Let's Encrypt

```bash
# ── STEP 1: Start Nginx with HTTP only first ────────────────────────────────
# Remove SSL directives from django.conf temporarily (use HTTP only server block).
# Then start nginx so certbot can verify domain ownership.

docker compose up -d nginx

# ── STEP 2: Get SSL Certificate ─────────────────────────────────────────────
# Replace yourdomain.com and email with your actual values.
docker compose run --rm certbot certonly \
  --webroot \
  --webroot-path /var/www/certbot \
  -d yourdomain.com \
  -d www.yourdomain.com \
  --email your-email@example.com \
  --agree-tos \
  --no-eff-email

# ── STEP 3: Restore SSL nginx config ────────────────────────────────────────
# Re-enable the SSL server block in django.conf (port 443 with certificate paths).
# Then reload nginx:
docker compose exec nginx nginx -s reload

# ── STEP 4: Test auto-renewal ────────────────────────────────────────────────
docker compose run --rm certbot renew --dry-run

# ── STEP 5: Set up cron job for auto-renewal ─────────────────────────────────
# Add to your server's crontab: renew certs every day at 3am
# crontab -e
# 0 3 * * * cd /opt/myproject && docker compose run --rm certbot renew --quiet && docker compose exec nginx nginx -s reload
```

---

## Step 20: Production Deployment Checklist

Before going live, verify **every item** in this checklist:

```bash
#!/bin/bash
# run_checklist.sh — Run this before every production deployment

echo "╔══════════════════════════════════════════════════════════╗"
echo "║         PRODUCTION DEPLOYMENT CHECKLIST                  ║"
echo "╚══════════════════════════════════════════════════════════╝"

# ── Security Checks ───────────────────────────────────────────────────────────
echo ""
echo "🔒 SECURITY CHECKS"

# 1. DEBUG must be False
echo -n "  [ ] DEBUG=False in production settings... "
docker compose exec web python -c "from django.conf import settings; assert not settings.DEBUG, 'DEBUG is True!'; print('✅ OK')"

# 2. SECRET_KEY must not be the default
echo -n "  [ ] SECRET_KEY is set and not default... "
docker compose exec web python -c "
from django.conf import settings
assert len(settings.SECRET_KEY) >= 50, 'SECRET_KEY too short'
assert 'insecure' not in settings.SECRET_KEY.lower(), 'Using insecure dev key!'
print('✅ OK')
"

# 3. Run Django deployment check
echo "  [ ] Django deployment checks..."
docker compose exec web python manage.py check --deploy

# 4. Verify non-root user
echo -n "  [ ] Running as non-root user... "
docker compose exec web id
# Should show 'uid=1001(django) gid=1001(django)'

# ── Database Checks ────────────────────────────────────────────────────────────
echo ""
echo "🗄️  DATABASE CHECKS"

# 5. All migrations applied
echo "  [ ] All migrations applied..."
docker compose exec web python manage.py showmigrations | grep '\[ \]' && echo "❌ Pending migrations!" || echo "✅ All migrations applied"

# 6. Database backup exists
echo -n "  [ ] Recent database backup... "
ls -lt *.sql.gz 2>/dev/null | head -1 || echo "⚠️  No backup found"

# ── Performance Checks ─────────────────────────────────────────────────────────
echo ""
echo "⚡ PERFORMANCE CHECKS"

# 7. Static files collected
echo -n "  [ ] Static files collected... "
docker compose exec web python -c "
import os
from django.conf import settings
count = sum(len(files) for _, _, files in os.walk(settings.STATIC_ROOT))
print(f'✅ {count} files in staticfiles/')
"

# 8. Health check passing
echo -n "  [ ] Health check passing... "
docker compose exec web curl -sf http://localhost:8000/health/ && echo "✅ OK" || echo "❌ FAILED"

# ── SSL Checks ────────────────────────────────────────────────────────────────
echo ""
echo "🔐 SSL CHECKS"

# 9. SSL certificate valid
echo -n "  [ ] SSL certificate valid... "
docker compose exec nginx nginx -t && echo "✅ Nginx config OK" || echo "❌ Nginx config invalid"

echo ""
echo "╔══════════════════════════════════════════════════════════╗"
echo "║  If all checks pass, you're ready to deploy! 🚀          ║"
echo "╚══════════════════════════════════════════════════════════╝"
```

---

## 🗂️ Final Project File Summary

```
myproject/
│
├── 📄 Dockerfile                   ← Production multi-stage build
├── 📄 Dockerfile.dev               ← Development image with tools
├── 📄 docker-compose.yml           ← Production: Nginx + Gunicorn + Postgres + Redis + Celery
├── 📄 docker-compose.dev.yml       ← Development: runserver + live reload + Flower
├── 📄 .dockerignore                ← Excludes git, venv, .env from build context
├── 📄 entrypoint.sh                ← Wait-for-db + migrate + exec CMD
├── 📄 Makefile                     ← Shorthand commands (make migrate, make test)
│
├── 📄 .env                         ← 🔴 Never commit: local dev secrets
├── 📄 .env.example                 ← ✅ Commit: template with no real values
├── 📄 .gitignore                   ← Excludes .env, venv, __pycache__, etc.
│
├── 📄 requirements.txt             ← Production Python packages (pinned versions)
├── 📄 requirements.dev.txt         ← Dev-only: pytest, black, ipython
│
├── 📁 config/
│   └── 📁 nginx/
│       ├── 📄 nginx.conf           ← Global Nginx config (worker_processes, gzip)
│       └── 📄 django.conf          ← Server block: HTTPS, static, proxy_pass
│
├── 📁 .github/
│   └── 📁 workflows/
│       └── 📄 deploy.yml           ← CI/CD: test → build → push → deploy
│
└── 📁 src/                         ← All Django code
    ├── 📄 manage.py
    ├── 📁 myproject/
    │   ├── 📄 __init__.py          ← Imports celery_app for @shared_task
    │   ├── 📄 celery.py            ← Celery app config
    │   ├── 📄 wsgi.py
    │   ├── 📄 asgi.py
    │   ├── 📄 urls.py              ← Includes /health/ endpoint
    │   └── 📁 settings/
    │       ├── 📄 base.py          ← Shared: DB, cache, celery, logging
    │       ├── 📄 development.py   ← Debug toolbar, console email
    │       └── 📄 production.py    ← HTTPS headers, SMTP email
    └── 📁 apps/
        └── 📁 core/
            ├── 📄 models.py
            ├── 📄 views.py         ← Health check view
            ├── 📄 tasks.py         ← Celery tasks
            └── 📄 tests.py
```

---

## 🚀 One-Command Quick Start

```bash
# Clone the project on any machine (dev or server) and run:

git clone https://github.com/yourorg/myproject.git
cd myproject
cp .env.example .env
# (edit .env to set SECRET_KEY and passwords)

# Development: Full stack in one command
docker compose -f docker-compose.dev.yml up -d --build \
  && docker compose -f docker-compose.dev.yml run --rm web python manage.py migrate \
  && docker compose -f docker-compose.dev.yml run --rm web python manage.py createsuperuser \
  && echo "✅ Ready at http://localhost:8000"

# Production: Same project, different compose file
docker compose up -d --build \
  && docker compose run --rm web python manage.py migrate \
  && docker compose run --rm web python manage.py collectstatic --noinput \
  && echo "✅ Production ready at https://yourdomain.com"
```

---

> 📌 **Key Principles to Remember:**
> 1. **Never commit `.env`** — use `.env.example` as template
> 2. **Layer caching** — copy `requirements.txt` before `COPY . .`
> 3. **Non-root in production** — always `USER django` in Dockerfile
> 4. **Healthchecks everywhere** — `depends_on: condition: service_healthy`
> 5. **Logs to stdout** — never log to files, always to console
> 6. **One concern per container** — web, db, redis, celery = separate services
> 7. **Named volumes for data** — never lose your database on `docker compose down`
