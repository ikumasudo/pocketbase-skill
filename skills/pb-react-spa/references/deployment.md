# Production Deployment

Docker, binary distribution, and reverse proxy configurations for PocketBase + React SPA.

---

## Overview

PocketBase operates in two modes, each with multiple deployment options:

| | Executable (binary) | Single Dockerfile | Docker Compose |
|---|---|---|---|
| **Binary mode** (standalone) | 1A | 2A | 3A / 3B |
| **Go package mode** | 1B | 2B | 3A / 3B |

**Common prerequisites:**

- Frontend SPA is built (`npm run build`) and placed in `pb_public/` or copied during Docker build
- `pb_data/` is **never** baked into a Docker image — always use a volume mount
- PocketBase listens on port **8080** inside containers (default), mapped to **8090** externally
- The PocketBase JS SDK uses `new PocketBase()` (no URL argument) — works in both dev and production because API requests are always same-origin

---

## 1. Executable (Binary) Distribution

### 1A. Binary Mode (Standalone)

Download the PocketBase binary for the target platform:

```bash
VERSION=$(curl -s https://api.github.com/repos/pocketbase/pocketbase/releases/latest \
  | python3 -c "import sys,json; print(json.load(sys.stdin)['tag_name'].lstrip('v'))")
ARCH=$(uname -m | sed 's/x86_64/amd64/;s/aarch64/arm64/')
OS=$(uname -s | tr A-Z a-z)
curl -sL "https://github.com/pocketbase/pocketbase/releases/download/v${VERSION}/pocketbase_${VERSION}_${OS}_${ARCH}.zip" \
  -o pb.zip && unzip -o pb.zip pocketbase && rm pb.zip
```

**Build the SPA and deploy:**

```bash
cd frontend && npm run build
mkdir -p ../backend/pb_public
cp -r dist/* ../backend/pb_public/
```

**Deployment directory structure:**

```
deploy/
├── pocketbase              # Binary for target platform
├── pb_public/              # Frontend build artifacts
├── pb_migrations/          # JS migration files
├── pb_hooks/               # JSVM hook files (if any)
└── pb_data/                # Created at runtime (add to .gitignore)
```

**Start the server:**

```bash
./pocketbase serve --http=0.0.0.0:8090
```

This single process serves the API, SPA, and Admin UI.

**First-time superuser creation:**

```bash
./pocketbase superuser create admin@example.com yourpassword
```

### 1B. Go Package Mode

**CGO build options:**

| Setting | Pros | Cons |
|---|---|---|
| `CGO_ENABLED=0` (recommended) | Static binary, no system deps, easy cross-compile | No FTS5 / ICU tokenizer |
| `CGO_ENABLED=1` | Full SQLite features (FTS5, ICU) | Requires gcc/musl-dev, harder to cross-compile |

**Build:**

```bash
# Static build (recommended)
CGO_ENABLED=0 go build -o myapp .

# Cross-compile example (Linux AMD64 from macOS/Windows)
CGO_ENABLED=0 GOOS=linux GOARCH=amd64 go build -o myapp .
```

**Deployment directory structure:**

```
deploy/
├── myapp                   # Compiled Go binary (migrations are embedded)
├── pb_public/              # Frontend build artifacts
└── pb_data/                # Created at runtime (add to .gitignore)
```

> Go migrations are compiled into the binary — no need to copy `migrations/` or `pb_migrations/` to the deploy directory.

**Integrated build script (`build.sh`):**

```bash
#!/bin/bash
set -euo pipefail

# Build frontend
cd frontend && npm ci && npm run build && cd ..

# Build backend
CGO_ENABLED=0 go build -o myapp .

# Assemble deploy directory
mkdir -p deploy/pb_public
cp myapp deploy/
cp -r frontend/dist/* deploy/pb_public/

echo "Build complete → deploy/"
```

**Start:**

```bash
cd deploy && ./myapp serve --http=0.0.0.0:8090
```

---

## 2. Single Dockerfile

### 2A. Binary Mode

```dockerfile
# ---- Stage 1: Build SPA ----
FROM node:22-alpine AS frontend
WORKDIR /app
COPY frontend/package.json frontend/package-lock.json ./
RUN npm ci
COPY frontend/ .
RUN npm run build

# ---- Stage 2: Runtime ----
FROM alpine:3.21
ARG PB_VERSION=0.28.2
ARG TARGETARCH

RUN apk add --no-cache ca-certificates curl unzip \
    && curl -sL "https://github.com/pocketbase/pocketbase/releases/download/v${PB_VERSION}/pocketbase_${PB_VERSION}_linux_${TARGETARCH}.zip" \
       -o /tmp/pb.zip \
    && unzip /tmp/pb.zip pocketbase -d /usr/local/bin/ \
    && rm /tmp/pb.zip \
    && apk del curl unzip

WORKDIR /app

# Copy SPA build
COPY --from=frontend /app/dist ./pb_public/

# Copy migrations and hooks
COPY backend/pb_migrations/ ./pb_migrations/
COPY backend/pb_hooks/ ./pb_hooks/

EXPOSE 8080

HEALTHCHECK --interval=30s --timeout=5s --start-period=5s --retries=3 \
    CMD wget -qO- http://localhost:8080/api/health || exit 1

CMD ["/usr/local/bin/pocketbase", "serve", "--http=0.0.0.0:8080"]
```

> Update `PB_VERSION` to match the PocketBase version you use.
> `TARGETARCH` is set automatically by `docker buildx` (amd64, arm64).

**Build and run:**

```bash
docker build -t myapp .
docker run -d -p 127.0.0.1:8090:8080 -v pb_data:/app/pb_data --name myapp myapp
```

### 2B. Go Package Mode

```dockerfile
# ---- Stage 1: Build SPA ----
FROM node:22-alpine AS frontend
WORKDIR /app
COPY frontend/package.json frontend/package-lock.json ./
RUN npm ci
COPY frontend/ .
RUN npm run build

# ---- Stage 2: Build Go binary ----
FROM golang:1.23-alpine AS backend
WORKDIR /src
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 go build -o /myapp .

# ---- Stage 3: Runtime ----
FROM alpine:3.21
RUN apk add --no-cache ca-certificates

WORKDIR /app

# Copy Go binary
COPY --from=backend /myapp .

# Copy SPA build
COPY --from=frontend /app/dist ./pb_public/

EXPOSE 8080

HEALTHCHECK --interval=30s --timeout=5s --start-period=5s --retries=3 \
    CMD wget -qO- http://localhost:8080/api/health || exit 1

CMD ["./myapp", "serve", "--http=0.0.0.0:8080"]
```

> Go migrations are compiled into the binary — no COPY for `migrations/` needed in the runtime stage.

**CGO_ENABLED=1 variant** (if you need FTS5/ICU):

Replace the backend stage with:

```dockerfile
FROM golang:1.23-alpine AS backend
RUN apk add --no-cache gcc musl-dev
WORKDIR /src
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=1 go build -o /myapp .
```

**Build and run:**

```bash
docker build -t myapp .
docker run -d -p 127.0.0.1:8090:8080 -v pb_data:/app/pb_data --name myapp myapp
```

### .dockerignore

Create a `.dockerignore` in the project root:

```
node_modules/
frontend/node_modules/
pb_data/
.env
.git/
.gitignore
*.md
```

---

## 3. Docker Compose

### 3A. Basic Configuration

Works for both binary mode and Go package mode — just change the Dockerfile.

```yaml
services:
  pocketbase:
    build: .
    ports:
      - "127.0.0.1:8090:8080"
    volumes:
      - pb_data:/app/pb_data
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "wget", "-qO-", "http://localhost:8080/api/health"]
      interval: 30s
      timeout: 5s
      start_period: 5s
      retries: 3

volumes:
  pb_data:
```

```bash
docker compose up -d
docker compose logs -f pocketbase
```

### 3B. With Reverse Proxy (Caddy)

Caddy provides automatic HTTPS via Let's Encrypt.

```yaml
services:
  pocketbase:
    build: .
    expose:
      - "8080"
    volumes:
      - pb_data:/app/pb_data
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "wget", "-qO-", "http://localhost:8080/api/health"]
      interval: 30s
      timeout: 5s
      start_period: 5s
      retries: 3

  caddy:
    image: caddy:2-alpine
    ports:
      - "80:80"
      - "443:443"
      - "443:443/udp"
    volumes:
      - ./Caddyfile:/etc/caddy/Caddyfile:ro
      - caddy_data:/data
      - caddy_config:/config
    restart: unless-stopped
    depends_on:
      pocketbase:
        condition: service_healthy

volumes:
  pb_data:
  caddy_data:
  caddy_config:
```

**Caddyfile:**

```
example.com {
    reverse_proxy pocketbase:8080
}
```

> Replace `example.com` with your actual domain. Caddy automatically provisions and renews TLS certificates.

**Nginx alternative:**

If you prefer Nginx, replace the Caddy service:

```yaml
  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
    volumes:
      - ./nginx.conf:/etc/nginx/conf.d/default.conf:ro
    restart: unless-stopped
    depends_on:
      pocketbase:
        condition: service_healthy
```

```nginx
server {
    listen 80;
    server_name example.com;

    location / {
        proxy_pass http://pocketbase:8080;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        # SSE (realtime) support
        proxy_buffering off;
        proxy_cache off;
    }
}
```

> `proxy_buffering off` is required for PocketBase's realtime SSE subscriptions.

---

## 4. First-Time Superuser Creation

After the container starts for the first time, create a superuser:

```bash
docker exec -it myapp ./pocketbase superuser create admin@example.com yourpassword

# Or for Go package mode:
docker exec -it myapp ./myapp superuser create admin@example.com yourpassword
```

For Docker Compose:

```bash
docker compose exec pocketbase /app/pocketbase superuser create admin@example.com yourpassword

# Go package mode:
docker compose exec pocketbase /app/myapp superuser create admin@example.com yourpassword
```

---

## 5. Production Checklist

- [ ] **SPA build** — `npm run build` succeeds and `pb_public/` contains `index.html`
- [ ] **pb_data volume** — `pb_data/` is mounted as a named volume (never baked into the image)
- [ ] **HTTPS** — Reverse proxy (Caddy/Nginx) or cloud provider handles TLS termination
- [ ] **Superuser** — Created via CLI after first start
- [ ] **Backups** — Scheduled backup strategy for `pb_data/` (contains SQLite DB and uploaded files)
- [ ] **Firewall** — PocketBase port (8080/8090) is not directly exposed to the internet (behind reverse proxy)
- [ ] **Health check** — `/api/health` endpoint is monitored
- [ ] **Restart policy** — `restart: unless-stopped` or equivalent systemd configuration
