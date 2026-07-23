# 🐳 DebForge — Container-First Fullstack Dev Environment (Debian Edition)

> A single source of truth for spinning up an identical fullstack development environment on **Debian**. Reproducible, production-parity, and "clone + one command" fast. The reference host/server OS is **Debian 13 "trixie" (stable)** — the *same* OS in dev and prod.

**Legend:** 💡 tip · ⚠️ warning · ✅ verify · 🐧 Debian

> 💡 **Note:** Placeholders like `{{FRONTEND}}` mark **swappable** stack choices. The concrete stack shown (Vite + React + TS, Node + TS, PostgreSQL, Caddy) is illustrative — every *principle* here is stack-agnostic; only the *commands* are concrete so they stay runnable. Swap the parts, keep the structure.

> 💡 **Note:** This edition targets **Debian only**. If your team also runs Windows or macOS laptops, use the cross-platform edition instead — everything below assumes a native Debian host or a Debian server.

---

## 📑 Table of Contents

1. [📖 Overview & Philosophy](#-overview--philosophy)
2. [⚡ Quickstart (TL;DR)](#-quickstart-tldr)
3. [✅ Prerequisites](#-prerequisites)
4. [🐧 Host Setup on Debian trixie](#-host-setup-on-debian-trixie)
5. [🖥️ Onboarding a Debian Workstation](#️-onboarding-a-debian-workstation)
6. [🗂️ Project Structure](#️-project-structure)
7. [📦 Dockerfiles](#-dockerfiles)
8. [🧩 Compose Files](#-compose-files)
9. [🔐 Environment & Secrets](#-environment--secrets)
10. [🔁 Daily Dev Workflow](#-daily-dev-workflow)
11. [♻️ Reproducibility Guardrails](#️-reproducibility-guardrails)
12. [🚀 Dev/Prod Parity & Deployment](#-devprod-parity--deployment)
13. [🛠️ Task Runner Reference](#️-task-runner-reference)
14. [🧯 Troubleshooting](#-troubleshooting)
15. [📋 New-Hire Checklist](#-new-hire-checklist)

---

## 📖 Overview & Philosophy

**Why container-first?** Because the machine a developer sits at should not decide whether the app runs. The classic failure mode — *"works on my machine"* — happens when the app depends on things scattered across a host: a specific Node version, a locally-installed Postgres, an OpenSSL quirk, a `PATH` entry someone set two years ago. Containers move **all** of that into an image that is built from a recipe (`Dockerfile`) and pinned to exact versions. The host becomes almost boring: it needs only **Docker + Git + an editor**.

That single decision delivers the four goals:

| Goal | How container-first delivers it |
|---|---|
| 🐧 **Team standard on Debian** | Every developer and every server runs the same Debian + Docker Engine, so the same images and `docker compose` commands behave identically from laptop to production. |
| ♻️ **Reproducible** | Images are pinned **by digest**, dependencies are locked in committed lockfiles, and the whole recipe is version-controlled. Same inputs → same environment, on any machine, next year. |
| 💻 **Works on my machine** | A new hire runs `git clone` + `make up`. No language runtimes to install, no version juggling. |
| 🚀 **Works in prod** | Dev and prod build from the **same base images** and the **same Dockerfile stages** on the **same OS**. Config comes from the environment (12-factor), so promoting an image to prod changes *configuration*, not *code paths*. |

> 💡 **Tip:** The mental shift is "**the container is the unit of truth**, the host is just a Docker runtime." On Debian, dev and prod share not only the container but the host OS too — parity is as tight as it gets.

### Mental model

A minimal fullstack topology: the browser hits an **edge proxy**, which routes to the **frontend** and the **API**; the API talks to the **database**. In dev, everything is one Compose project on one network.

```mermaid
flowchart LR
    Browser["🌐 Browser"] --> Proxy["🔀 {{PROXY}}\nCaddy (edge)"]
    Proxy -->|/| Web["💻 {{FRONTEND}}\nVite + React + TS"]
    Proxy -->|/api| Api["⚙️ {{BACKEND}}\nNode + TS"]
    Api --> DB[("🗄️ {{DB}}\nPostgreSQL")]
    subgraph net["docker network: appnet"]
        Proxy
        Web
        Api
        DB
    end
```

ASCII fallback (same topology):

```text
  🌐 Browser
     │
     ▼
  🔀 proxy (Caddy)  ──/──►  💻 web  (Vite/React)
     │
     └──────────/api──────► ⚙️ api  (Node/TS) ──► 🗄️ db (Postgres)

  └───────────── docker network: appnet ─────────────┘
```

Each box is a container. Each arrow is a network call over the internal Compose network, addressed by **service name** (`api`, `db`, `web`) — not `localhost`.

---

## ⚡ Quickstart (TL;DR)

**The impatient path.** Assumes Docker + Git already work on your Debian host (see [Prerequisites](#-prerequisites) if not).

```bash
# 1. Clone the repo anywhere in your home directory
git clone https://example.com/your-org/your-app.git
cd your-app

# 2. Create your local env from the committed template
cp .env.example .env

# 3. Build + start everything (frontend, api, db, proxy) in the background
make up

# 4. Follow logs until healthy
make logs
```

✅ **Verify:** open <http://localhost:8080> — you should see the frontend, and <http://localhost:8080/api/health> should return `{"status":"ok"}`.

```bash
# Tear down when done (keeps volumes/data)
make down
```

> 💡 **Tip:** No `make`? Every target maps to a plain `docker compose` command — see the [Task Runner Reference](#️-task-runner-reference). You never *need* `make`; it's just ergonomics.

---

## ✅ Prerequisites

You need three things on the Debian host, and **nothing else** (no Node, no Python, no Postgres — those live in containers):

| Requirement | 🐧 Debian |
|---|---|
| **Container runtime** | Docker Engine (CE) from Docker's official APT repo — native |
| **Git** | `sudo apt-get install -y git` |
| **Editor** | VS Code / Neovim / any |
| **Shell** | bash |
| **`make`** (optional) | `sudo apt-get install -y make` |

Baseline versions (anything newer is fine):

- Docker Engine **≥ 26** with **Compose v2** (the `docker compose` subcommand — *not* the old hyphenated `docker-compose`).
- Git **≥ 2.40**.
- Debian 13 "trixie" is the reference; Debian 12 "bookworm" also works.

✅ **Verify** the runtime is healthy:

```bash
docker version --format '{{.Server.Version}}'   # prints a version, e.g. 27.x
docker compose version                          # "Docker Compose version v2.x"
docker run --rm hello-world                      # prints "Hello from Docker!"
```

> ⚠️ **Warning:** If `docker compose version` errors but `docker-compose version` works, you're on the deprecated v1 plugin. This guide uses **v2 only**. Install the `docker-compose-plugin` package (covered below) and always type `docker compose` (space, not hyphen).

> ⚠️ **Warning — do not assume Docker Desktop on Linux.** The native Docker Engine (installed below) is the reference and is faster. Docker Desktop for Linux runs an extra VM you don't need.

---

## 🐧 Host Setup on Debian trixie

**Why the official Docker repo and not `apt install docker.io`?** Debian's `docker.io` package lags upstream, doesn't ship the modern **buildx**/**Compose v2** plugins in the versions we want, and mixes poorly with BuildKit features we rely on for reproducible builds. We install **Docker CE from Docker's official APT repository** so dev and prod hosts run the same, current engine.

> 💡 **Tip:** These are the exact steps the bundled `scripts/setup-host.sh` automates. Run them by hand to understand them, or run the script for an idempotent one-shot.

### 1. Remove any conflicting distro packages (idempotent)

```bash
for pkg in docker.io docker-doc docker-compose podman-docker containerd runc; do
  sudo apt-get remove -y "$pkg" 2>/dev/null || true
done
```

### 2. Add Docker's official GPG key and APT repository

```bash
sudo apt-get update
sudo apt-get install -y ca-certificates curl gnupg

# Keyring dir (idempotent)
sudo install -m 0755 -d /etc/apt/keyrings

# Docker's GPG key
sudo curl -fsSL https://download.docker.com/linux/debian/gpg \
  -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

# Add the repo, auto-detecting the codename (trixie)
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] \
  https://download.docker.com/linux/debian \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" \
  | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

> 💡 **Note:** On a brand-new trixie release, Docker's repo occasionally publishes the `trixie` suite a little after the OS ships. If `apt-get update` 404s on the Docker repo, temporarily replace `$(. /etc/os-release && echo "$VERSION_CODENAME")` with the previous stable codename `bookworm` — the packages are compatible — and switch back once `trixie` is published.

### 3. Install Docker Engine, CLI, containerd, Buildx & Compose v2

```bash
sudo apt-get update
sudo apt-get install -y \
  docker-ce docker-ce-cli containerd.io \
  docker-buildx-plugin docker-compose-plugin
```

### 4. Add your user to the `docker` group (run Docker without `sudo`)

```bash
sudo groupadd docker 2>/dev/null || true
sudo usermod -aG docker "$USER"
# Apply the new group WITHOUT logging out (or just open a new shell):
newgrp docker
```

> ⚠️ **Warning:** Membership in `docker` is effectively **root-equivalent** (you can bind-mount `/` into a container). That's fine on a personal dev box; on shared servers, prefer rootless Docker or restricted access.

### 5. Enable & start the service; confirm BuildKit/buildx

```bash
sudo systemctl enable --now docker
docker buildx version           # buildx present
docker buildx create --use --name devbuilder 2>/dev/null || docker buildx use devbuilder
```

BuildKit is the default builder in modern Docker; we make it explicit for reproducible, cached builds. Belt-and-suspenders, you can also set it in the daemon config:

```bash
echo '{ "features": { "buildkit": true } }' | sudo tee /etc/docker/daemon.json
sudo systemctl restart docker
```

### ✅ Verify the whole host

```bash
docker run --rm hello-world
docker compose version
docker buildx ls
id -nG | tr ' ' '\n' | grep -q docker && echo "✅ in docker group"
```

### 🧱 Debian trixie realities you should know

| Topic | What's true on trixie | Why you care |
|---|---|---|
| **Repo choice** | Use Docker's **official** repo, not `docker.io`. | Current engine + buildx + Compose v2, matching prod. |
| **cgroup v2** | trixie uses **cgroup v2** (unified hierarchy) by default. | Docker CE supports it natively — good. Resource limits (`cpus`, `mem_limit`) behave correctly. If you run *ancient* tooling that needs cgroup v1, you'd add `systemd.unified_cgroup_hierarchy=0` to the kernel cmdline — **but don't**; stay on v2. |
| **iptables-nft** | trixie uses the **nftables** backend (`iptables` is `iptables-nft`). | Docker CE works with nft. If container networking misbehaves after upgrades, ensure `iptables` points at the nft backend: `sudo update-alternatives --config iptables` → pick `iptables-nft`. Avoid mixing legacy `iptables-legacy` rules with Docker's nft chains. |
| **firewalld/ufw** | If you enable a host firewall, Docker punches its own nft rules. | Docker bypasses `ufw` by default for published ports. Don't assume `ufw deny` protects a published container port — bind sensitive ports to `127.0.0.1` in Compose instead. |

---

## 🖥️ Onboarding a Debian Workstation

The goal: **an identical container experience on every Debian machine**. Because containers already run on the same Linux kernel your host uses, there's no VM, no filesystem-boundary tax, and no cross-OS surprises — this is the fast path.

### 🐧 Native Debian — the reference path

1. **Install the host tooling** — follow [Host Setup on Debian trixie](#-host-setup-on-debian-trixie) once per machine (or run `scripts/setup-host.sh`).
2. **Clone the repo** anywhere under your home directory — bind mounts are native, so file I/O and inotify-based hot reload are full speed:

   ```bash
   cd ~/work && git clone https://example.com/your-org/your-app.git
   ```

3. **Edit with any editor.** VS Code, Neovim, JetBrains — they all operate directly on the local files. No remote-attach dance needed.

✅ **Verify:** `docker run --rm hello-world` works **without** `sudo`, and `id -nG | grep -q docker && echo ok` prints `ok`.

> 💡 **Tip:** On native Linux, bind mounts are as fast as the disk and file-watching "just works" via inotify — so you do **not** need macOS-style `:cached` flags or `CHOKIDAR_USEPOLLING` polling. Leave polling off for lower CPU usage; only enable it if you're editing over a network filesystem (NFS/SMB).

> 💡 **Note — other distros:** if a teammate is on Ubuntu/Fedora rather than Debian, the *only* difference is the APT/DNF repo line — Docker publishes an official repo per distro. The images, Compose files, and workflow are identical. Standardizing on Debian keeps dev/prod parity tightest.

### At-a-glance

| Concern | 🐧 Debian (native) |
|---|---|
| Where code lives | Anywhere in your home dir (native filesystem) |
| Bind-mount speed | Native (fastest) — no VM boundary |
| File-watching / hot reload | Native inotify — no polling needed |
| Runtime | Docker Engine CE from the official repo |
| Main gotcha | Remember to join the `docker` group (`newgrp docker`) |

---

## 🗂️ Project Structure

A clean, annotated tree. Everything a new hire needs is committed; only real secrets and local state are ignored.

```text
your-app/
├── .dockerignore              # keeps build context small & builds reproducible
├── .gitattributes             # normalizes line endings (LF) repo-wide
├── .gitignore                 # ignores .env, node_modules, build output
├── .env.example               # ✅ committed template of every required var (no secrets)
├── .env                        # 🔐 git-ignored, real local values (NEVER committed)
├── Makefile                   # task runner: make up/down/logs/exec/... (swappable: justfile)
├── docker-compose.yml         # base topology: services, network, volumes (shared dev+prod)
├── docker-compose.override.yml# 🧑‍💻 DEV overrides: bind mounts, hot reload, exposed ports
├── docker-compose.prod.yml    # 🚀 PROD overrides: prod targets, no source mounts, limits
├── Caddyfile                  # {{PROXY}} config (swappable: traefik.yml / nginx.conf)
├── scripts/
│   └── setup-host.sh          # 🐧 idempotent Debian host bootstrap (Docker CE + buildx)
├── db/
│   └── init/                  # optional SQL run on first DB init
├── apps/
│   ├── web/                   # {{FRONTEND}} — Vite + React + TS
│   │   ├── Dockerfile         # multi-stage: deps → dev → build → prod
│   │   ├── package.json
│   │   ├── package-lock.json  # ♻️ committed lockfile (pinned deps)
│   │   └── src/
│   └── api/                   # {{BACKEND}} — Node + TS
│       ├── Dockerfile         # multi-stage: deps → dev → build → prod
│       ├── package.json
│       ├── package-lock.json  # ♻️ committed lockfile
│       └── src/
└── docs/
    └── dev-environment-debian.md  # 📖 this document
```

> 💡 **Tip:** A monorepo layout (`apps/web`, `apps/api`) keeps one clone = whole system, which is exactly what goal #3 ("clone + one command") wants. Swap in `packages/`, `services/`, or a polyrepo if your org differs — the Compose wiring stays the same.

---

## 📦 Dockerfiles

**Principles baked into every image:**

- **Multi-stage** — one Dockerfile, several `FROM` stages. Dev target has toolchains + source mounts; prod target is a tiny runtime with only built artifacts.
- **Pinned by tag *and* digest** — never `latest`. The digest (`@sha256:...`) makes the base image byte-for-byte reproducible.
- **Non-root user** — the container's default user is a normal user, matching a build-arg UID/GID to avoid root-owned files on bind mounts (the classic 🔑 UID/GID gotcha).
- **`.dockerignore`** keeps the build context (and thus the cache) small and deterministic.

> ⚠️ **Warning — pin by digest:** get the current digest with
> `docker buildx imagetools inspect node:22-bookworm-slim` (look for the `Digest:` line), then paste it. Digests below are **placeholders** — replace with real ones and re-pin deliberately when you upgrade.

### `apps/api/Dockerfile` — Node + TypeScript (swappable)

```dockerfile
# syntax=docker/dockerfile:1.7
# ---- Base: pinned by tag AND digest (replace the sha with a real one) ----
FROM node:22-bookworm-slim@sha256:REPLACE_WITH_REAL_DIGEST AS base
ENV NODE_ENV=production
WORKDIR /app

# Create a non-root user whose UID/GID can match the host developer (🔑 fixes bind-mount ownership)
ARG UID=1000
ARG GID=1000
RUN groupadd -g "${GID}" app 2>/dev/null || true \
 && useradd -m -u "${UID}" -g "${GID}" -s /bin/bash app 2>/dev/null || true

# ---- Deps: install with the committed lockfile, cached separately from source ----
FROM base AS deps
COPY apps/api/package.json apps/api/package-lock.json ./
# BuildKit cache mount speeds up repeated installs without baking cache into the layer
RUN --mount=type=cache,target=/root/.npm \
    npm ci --include=dev

# ---- Dev: full toolchain; source is bind-mounted at runtime, so we don't COPY it ----
FROM deps AS dev
ENV NODE_ENV=development
USER app
EXPOSE 3000
# Hot reload dev server (tsx/nodemon watches the bind-mounted source)
CMD ["npm", "run", "dev"]

# ---- Build: compile TS -> JS, then prune to production deps ----
FROM deps AS build
COPY apps/api/ ./
RUN npm run build \
 && npm prune --omit=dev

# ---- Prod: minimal runtime, only built output + prod deps, non-root ----
FROM base AS prod
ENV NODE_ENV=production
COPY --from=build /app/node_modules ./node_modules
COPY --from=build /app/dist ./dist
COPY --from=build /app/package.json ./package.json
USER app
EXPOSE 3000
# Container-native healthcheck (used by Compose/orchestrator)
HEALTHCHECK --interval=10s --timeout=3s --start-period=20s --retries=5 \
  CMD node -e "fetch('http://localhost:3000/health').then(r=>process.exit(r.ok?0:1)).catch(()=>process.exit(1))"
CMD ["node", "dist/server.js"]
```

### `apps/web/Dockerfile` — Vite + React + TS (swappable)

```dockerfile
# syntax=docker/dockerfile:1.7
FROM node:22-bookworm-slim@sha256:REPLACE_WITH_REAL_DIGEST AS base
WORKDIR /app
ARG UID=1000
ARG GID=1000
RUN groupadd -g "${GID}" app 2>/dev/null || true \
 && useradd -m -u "${UID}" -g "${GID}" -s /bin/bash app 2>/dev/null || true

FROM base AS deps
COPY apps/web/package.json apps/web/package-lock.json ./
RUN --mount=type=cache,target=/root/.npm npm ci --include=dev

# ---- Dev: Vite dev server with HMR over the bind-mounted source ----
FROM deps AS dev
ENV NODE_ENV=development
USER app
EXPOSE 5173
# --host makes Vite listen on 0.0.0.0 so the proxy/host can reach it
CMD ["npm", "run", "dev", "--", "--host", "0.0.0.0"]

# ---- Build: static assets ----
FROM deps AS build
COPY apps/web/ ./
RUN npm run build      # outputs /app/dist

# ---- Prod: static files served by a tiny, pinned web server ----
FROM caddy:2.8-alpine@sha256:REPLACE_WITH_REAL_DIGEST AS prod
COPY --from=build /app/dist /usr/share/caddy
# caddy image already runs as non-root and has a sane default; serves :80
```

> 💡 **Tip — why not `COPY` source in the dev stage?** In dev we **bind-mount** the source at runtime so edits are instant (hot reload). Copying it would bake a stale snapshot into the image. Prod does the opposite: it `COPY`s and builds so the image is self-contained and immutable.

---

## 🧩 Compose Files

**Why three files?** Compose **merges** files in order, later values overriding earlier ones. This lets one base file describe the *topology* while thin override files describe *dev* vs *prod* differences — the essence of dev/prod parity.

- `docker compose up` auto-loads **`docker-compose.yml` + `docker-compose.override.yml`** → your dev environment.
- For prod you explicitly pick the base + prod file: `docker compose -f docker-compose.yml -f docker-compose.prod.yml ...`.

### `docker-compose.yml` — base (shared)

```yaml
# Compose v2 — no `version:` key needed (obsolete in v2)
name: your-app

services:
  proxy:
    image: caddy:2.8-alpine@sha256:REPLACE_WITH_REAL_DIGEST   # {{PROXY}}
    depends_on:
      web:
        condition: service_started
      api:
        condition: service_healthy
    networks: [appnet]

  web:
    build:
      context: .
      dockerfile: apps/web/Dockerfile
      args:
        UID: ${UID:-1000}
        GID: ${GID:-1000}
    networks: [appnet]

  api:
    build:
      context: .
      dockerfile: apps/api/Dockerfile
      args:
        UID: ${UID:-1000}
        GID: ${GID:-1000}
    environment:
      DATABASE_URL: postgres://${POSTGRES_USER}:${POSTGRES_PASSWORD}@db:5432/${POSTGRES_DB}
      NODE_ENV: ${NODE_ENV:-production}
    depends_on:
      db:
        condition: service_healthy    # ⏳ wait until DB healthcheck passes
    networks: [appnet]

  db:
    image: postgres:16-bookworm@sha256:REPLACE_WITH_REAL_DIGEST   # {{DB}}
    environment:
      POSTGRES_USER: ${POSTGRES_USER}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
      POSTGRES_DB: ${POSTGRES_DB}
    volumes:
      - db-data:/var/lib/postgresql/data     # named volume: data survives `down`
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER} -d ${POSTGRES_DB}"]
      interval: 5s
      timeout: 3s
      retries: 10
      start_period: 10s
    networks: [appnet]

networks:
  appnet:

volumes:
  db-data:
```

### `docker-compose.override.yml` — dev (auto-loaded)

```yaml
services:
  proxy:
    ports:
      - "8080:80"          # 🔌 remap here if 8080 is taken (e.g. "8081:80")
    volumes:
      - ./Caddyfile:/etc/caddy/Caddyfile:ro

  web:
    build:
      target: dev          # use the dev stage (Vite HMR)
    volumes:
      - ./apps/web:/app              # native Debian bind mount — full-speed, no :cached needed
      - /app/node_modules            # anonymous volume: keep deps inside the container
    ports:
      - "5173:5173"        # optional direct access to Vite

  api:
    build:
      target: dev          # dev stage (tsx/nodemon watch)
    volumes:
      - ./apps/api:/app
      - /app/node_modules
    environment:
      NODE_ENV: development
    ports:
      - "3000:3000"        # optional direct API access; remap on conflict
```

> 💡 **Tip — the `node_modules` trick:** the anonymous volume `- /app/node_modules` **shadows** the bind mount for that one directory, so the container uses the deps it installed at build time (correct arch/platform) instead of whatever the host has. On native Debian this also avoids clobbering the container's Linux-built native modules with the host tree.

> 💡 **Tip:** On native Debian, inotify propagates file events across the bind mount, so Vite HMR and the API watcher fire instantly. You don't need `CHOKIDAR_USEPOLLING` — only add polling if you're editing over NFS/SMB.

### `docker-compose.prod.yml` — prod overrides

```yaml
services:
  proxy:
    ports:
      - "80:80"
      - "443:443"          # real TLS at the edge
    volumes:
      - ./Caddyfile:/etc/caddy/Caddyfile:ro
    restart: unless-stopped

  web:
    build:
      target: prod         # static assets served by Caddy — NO source mount
    restart: unless-stopped

  api:
    build:
      target: prod         # compiled JS, prod deps only, non-root — NO source mount
    environment:
      NODE_ENV: production
    restart: unless-stopped
    deploy:
      resources:
        limits:
          cpus: "1.0"
          memory: 512M

  db:
    restart: unless-stopped
    deploy:
      resources:
        limits:
          memory: 1G
```

✅ **Verify** which config is actually in effect (renders the merged result):

```bash
docker compose config                                            # dev (base + override)
docker compose -f docker-compose.yml -f docker-compose.prod.yml config   # prod
```

---

## 🔐 Environment & Secrets

**Why 12-factor config?** Because the *same image* must run in dev, CI, and prod with only its **environment** changing. Hard-coding config into the image breaks parity and leaks secrets into version control.

### The `.env` strategy

- **`.env.example`** — committed. Lists **every** variable the app needs, with safe placeholder/dummy values and comments. It's the contract: if it's not here, it's not a supported config knob.
- **`.env`** — git-ignored. Each developer's real local values. Compose auto-loads `.env` from the project root for `${VAR}` interpolation.

`.env.example`:

```dotenv
# ---- Postgres (dev defaults; override in real .env) ----
POSTGRES_USER=app
POSTGRES_PASSWORD=devpassword_change_me
POSTGRES_DB=app

# ---- API ----
NODE_ENV=development

# ---- Host user mapping (fixes 🔑 UID/GID bind-mount ownership) ----
# Run:  id -u   and   id -g   and paste the results (usually 1000/1000 on Debian)
UID=1000
GID=1000
```

`.gitignore` (the important lines):

```gitignore
.env
*.env.local
node_modules/
apps/*/dist/
```

✅ **Verify** `.env` is ignored (should print nothing):

```bash
git check-ignore -v .env    # prints ".gitignore:1:.env  .env"  → good, it IS ignored
git status --porcelain | grep -E '(^|/)\.env$' && echo "⚠️ .env is tracked!" || echo "✅ .env not tracked"
```

### Config precedence (highest wins)

1. Values set directly in the shell / CI runner environment.
2. `environment:` keys in the active Compose file(s).
3. `${VAR}` interpolation from the project `.env` file.
4. Image defaults (`ENV` in the Dockerfile).

### 🔑 Set your UID/GID once

```bash
# Append your real ids so bind-mounted files aren't owned by root
echo "UID=$(id -u)" >> .env
echo "GID=$(id -g)" >> .env
```

### Secrets — the rules

> ⚠️ **Warning:** **Never commit real secrets.** `.env` is git-ignored precisely so credentials don't land in history. If a secret is ever committed, rotate it — deleting the commit is not enough.

- **Dev:** dummy values in `.env` are fine (they're not real credentials).
- **Prod:** inject secrets from your platform's mechanism — Docker/Swarm **secrets**, Kubernetes Secrets, or a manager like Vault / AWS Secrets Manager / SOPS-encrypted files — as environment variables or mounted files at runtime. Compose supports first-class `secrets:` that mount to `/run/secrets/<name>`:

```yaml
# docker-compose.prod.yml (excerpt) — file-based secret, not an env var in history
services:
  api:
    secrets: [db_password]
    environment:
      DATABASE_URL: postgres://app@db:5432/app   # password read from the secret file
secrets:
  db_password:
    file: ./secrets/db_password.txt   # this file is git-ignored & provisioned out-of-band
```

---

## 🔁 Daily Dev Workflow

Everything runs **inside containers** — you never install a runtime on the host. Below, each command shows the `make` shortcut and the raw `docker compose` it wraps.

### Start / stop

```bash
make up        # docker compose up -d --build         → build + start in background
make down      # docker compose down                  → stop & remove containers (keeps volumes)
make restart   # docker compose restart api           → bounce a single service
```

### Logs

```bash
make logs              # docker compose logs -f --tail=100          → all services
docker compose logs -f api    # follow just the API
```

### Exec into a running container (the container-first replacement for local CLIs)

```bash
make sh SVC=api        # docker compose exec api bash    → shell inside the API container
docker compose exec db psql -U app -d app     # a psql prompt, no host Postgres needed
```

### Hot reload

With the dev bind mounts + watchers configured above, **edit a file on the host and the change is picked up automatically** — Vite HMR refreshes the browser; the API restarts via its watcher. On native Debian this is instant via inotify. No rebuild needed for source changes.

> 💡 **Tip:** You only need `--build` (`make up` / `make rebuild`) when **dependencies or the Dockerfile change**. Pure source edits are live.

### Rebuild after dependency/Dockerfile changes

```bash
make rebuild           # docker compose build --no-cache && docker compose up -d
# Or targeted:
docker compose build api && docker compose up -d api
```

### Migrations & tests — run them *in* the container

```bash
# Run DB migrations using the api image's tooling (example; swap for your migrator)
make migrate           # docker compose exec api npm run migrate

# Run the test suite inside a throwaway container (same image, isolated)
make test              # docker compose run --rm api npm test
```

✅ **Verify a healthy stack:**

```bash
docker compose ps      # STATUS should show "healthy" for db and api
curl -fsS http://localhost:8080/api/health && echo   # → {"status":"ok"}
```

---

## ♻️ Reproducibility Guardrails

These are the habits that make "same result on any machine, any time" true rather than aspirational.

| Guardrail | What to do | Why |
|---|---|---|
| 🏷️ **Pin by tag + digest** | `FROM node:22-bookworm-slim@sha256:...`; images in Compose too. | A tag can be re-pointed upstream; a digest is immutable. Kills "the base image changed under us." |
| 🔒 **Commit lockfiles** | `package-lock.json` (or `pnpm-lock.yaml`, `poetry.lock`, `go.sum`) is tracked; build with `npm ci` (not `npm install`). | `ci` installs the **exact** locked tree and fails on drift. |
| 🧹 **`.dockerignore`** | Exclude `node_modules`, `.git`, `dist`, `.env`, logs. | Smaller, deterministic build context; faster builds; no secret leakage into images. |
| 🔤 **`.gitattributes`** | `* text=auto eol=lf`. | Normalizes line endings so shell scripts stay LF even if someone commits from a non-Unix editor. |
| ⚡ **BuildKit cache** | `# syntax=...` + `--mount=type=cache` for package managers. | Fast rebuilds without baking cache into layers. |
| 👤 **Non-root user** | `USER app` in images; `UID/GID` build args. | Least privilege + fixes bind-mount ownership. |
| 🧱 **Multi-stage** | dev/build/prod stages in one Dockerfile. | Small prod images; dev/prod share a base. |

`.dockerignore`:

```gitignore
.git
node_modules
**/node_modules
**/dist
.env
*.env.local
*.log
.DS_Store
docs
```

`.gitattributes`:

```gitattributes
# Normalize all text files to LF in the repo; check out LF everywhere.
* text=auto eol=lf

# Keep genuinely binary files untouched
*.png binary
*.jpg binary
*.woff2 binary
```

> 💡 **Tip — line endings:** even on an all-Debian team, keep `.gitattributes`. It guarantees shell scripts and Dockerfiles stay LF regardless of who commits (a stray CRLF in a `#!/bin/sh` script makes Linux fail with `bad interpreter: No such file or directory`). If CRLF ever sneaks in, renormalize once:
> ```bash
> git add --renormalize .
> git commit -m "Normalize line endings to LF"
> ```

✅ **Verify pins & locks:**

```bash
grep -RnE 'FROM .*:latest|image: .*:latest' . && echo "⚠️ found a latest tag" || echo "✅ no latest tags"
grep -Rn '@sha256:' apps/*/Dockerfile docker-compose.yml | head    # digests present
test -f apps/api/package-lock.json && echo "✅ api lockfile committed"
```

---

## 🚀 Dev/Prod Parity & Deployment

**The parity contract:** dev and prod build from the **same Dockerfile** and the **same base image**, on the **same Debian host OS**, differing only by *build target* and *config*. That's what makes "works in prod" a near-certainty rather than a hope.

### Build once, promote the artifact

Don't rebuild for prod on a whim — **build the image once, tag it, push it, and promote the exact digest** through environments (CI → staging → prod). What changes between environments is the **environment variables/secrets**, not the bits.

```bash
# CI: build the prod target and tag by immutable content
docker buildx build \
  --target prod \
  --file apps/api/Dockerfile \
  --tag registry.example.com/your-app/api:1.4.2 \
  --push .

# Promote by digest (never by :latest)
docker buildx imagetools inspect registry.example.com/your-app/api:1.4.2   # note the digest
# deploy referencing registry.example.com/your-app/api@sha256:<that-digest>
```

### What differs dev → prod

| Aspect | 🧑‍💻 Dev (`override`) | 🚀 Prod (`prod`) |
|---|---|---|
| Build target | `dev` (toolchain, watchers) | `prod` (compiled, minimal) |
| Source | **bind-mounted** (hot reload) | **baked into image** (immutable) |
| Ports | high dev ports exposed | 80/443 at the edge only |
| Restart policy | none (you control it) | `unless-stopped` |
| Resource limits | none | `cpus` / `memory` limits set |
| Config | dummy `.env` | real secrets from a manager |
| TLS | plain HTTP on 8080 | real certs at proxy |

### Healthchecks & startup ordering

Both `depends_on: condition: service_healthy` and container `HEALTHCHECK`s (defined earlier) mean the API waits for a **truly ready** DB, and orchestrators can gate traffic on readiness — identical behavior in dev and prod.

### Deploy the prod stack

The reference prod host is the same Debian trixie + Docker Engine you set up for dev:

```bash
docker compose -f docker-compose.yml -f docker-compose.prod.yml pull
docker compose -f docker-compose.yml -f docker-compose.prod.yml up -d
```

✅ **Verify prod bring-up:**

```bash
docker compose -f docker-compose.yml -f docker-compose.prod.yml ps    # all "healthy"/"running"
```

> 💡 **Note:** Plain Compose is a fine target for a single Debian VPS/host. For multi-node, the *same images* deploy to Swarm or Kubernetes — the reproducibility work you did (digests, 12-factor config, healthchecks) carries straight over.

---

## 🛠️ Task Runner Reference

**Why a task runner?** So "the commands" live in the repo, not in someone's memory. New hires (and CI) call the same named targets. Below is a `Makefile`; `just` is a great swappable alternative (`justfile` with near-identical recipes).

```makefile
# Makefile — thin wrappers over `docker compose`. Requires: docker, compose v2.
# Usage: make <target>            e.g. make up

COMPOSE      := docker compose
COMPOSE_PROD := docker compose -f docker-compose.yml -f docker-compose.prod.yml
SVC          ?= api          # default service for exec/logs; override: make sh SVC=web

.DEFAULT_GOAL := help

.PHONY: help up down restart logs sh exec ps build rebuild migrate test seed prod-up prod-down config doctor

help:            ## List available targets
	@grep -E '^[a-zA-Z_-]+:.*?## .*$$' $(MAKEFILE_LIST) | \
	 awk 'BEGIN{FS=":.*?## "}{printf "  \033[36m%-12s\033[0m %s\n", $$1, $$2}'

up:              ## Build + start the dev stack in the background
	$(COMPOSE) up -d --build

down:            ## Stop & remove dev containers (keeps volumes/data)
	$(COMPOSE) down

restart:         ## Restart one service (SVC=api by default)
	$(COMPOSE) restart $(SVC)

logs:            ## Follow logs for all services (last 100 lines)
	$(COMPOSE) logs -f --tail=100

sh:              ## Open a shell inside a service container (SVC=api)
	$(COMPOSE) exec $(SVC) bash

exec:            ## Run an arbitrary command: make exec SVC=api CMD="npm run lint"
	$(COMPOSE) exec $(SVC) $(CMD)

ps:              ## Show service status/health
	$(COMPOSE) ps

build:           ## Build images (uses cache)
	$(COMPOSE) build

rebuild:         ## Rebuild images from scratch, then restart
	$(COMPOSE) build --no-cache && $(COMPOSE) up -d

migrate:         ## Run DB migrations inside the api container
	$(COMPOSE) exec $(SVC) npm run migrate

test:            ## Run the test suite in a throwaway container
	$(COMPOSE) run --rm $(SVC) npm test

seed:            ## Seed the database
	$(COMPOSE) exec $(SVC) npm run seed

config:          ## Render the merged dev config (sanity check)
	$(COMPOSE) config

prod-up:         ## Bring up the PROD stack (base + prod overrides)
	$(COMPOSE_PROD) up -d

prod-down:       ## Tear down the PROD stack
	$(COMPOSE_PROD) down

doctor:          ## Environment sanity checks
	@docker version --format 'Server: {{.Server.Version}}'
	@$(COMPOSE) version
	@test -f .env && echo "✅ .env present" || echo "⚠️ missing .env — run: cp .env.example .env"
	@git check-ignore -q .env && echo "✅ .env git-ignored" || echo "⚠️ .env NOT ignored!"
```

✅ **Verify:** `make help` lists every target; `make doctor` reports a green environment.

> 💡 **Tip:** Install `make` on Debian with `sudo apt-get install -y make`. Everything it wraps is a plain `docker compose` command, so you can always run those directly.

---

## 🧯 Troubleshooting

| 🩺 Symptom | 🔍 Cause | 🛠️ Fix |
|---|---|---|
| `docker` needs `sudo` / permission denied | User not in `docker` group | `sudo usermod -aG docker $USER` then `newgrp docker` (or re-login). |
| Bind-mounted files owned by `root`; app can't write | 🔑 Container user UID ≠ host UID | Set `UID`/`GID` in `.env` (`id -u`, `id -g`), rebuild with those build args; images run `USER app`. |
| `bad interpreter: No such file or directory` on a `.sh` | ↩️ CRLF line endings crept in | Add `.gitattributes` (`* text=auto eol=lf`), `git add --renormalize .`; set editor to LF. |
| Hot reload doesn't fire; edits ignored | Editing over a network FS (NFS/SMB), or inotify watch limit hit | Set `CHOKIDAR_USEPOLLING=true` for network mounts; raise the limit: `sudo sysctl fs.inotify.max_user_watches=524288`. |
| `bind: address already in use` on `up` | 🔌 Host port already taken | Remap the left side in `docker-compose.override.yml` (`"8081:80"`); find the hog: `sudo lsof -i :8080` or `ss -ltnp`. |
| Image behaves differently than last week | 🏷️ Unpinned/`latest` tag drifted | Pin `@sha256:` digests; rebuild; commit the new digest deliberately. |
| API starts before DB ready, connection refused | ⏳ Missing readiness gate | Use `depends_on: condition: service_healthy` + DB healthcheck (shown in Compose). |
| `docker compose` unknown, `docker-compose` works | Deprecated v1 installed | Install `docker-compose-plugin`; always use `docker compose` (v2). |
| `apt-get update` 404s on the Docker repo | Docker hasn't published the `trixie` suite yet | Temporarily point the repo at `bookworm` (compatible); switch back once `trixie` is published. |
| Container networking broke after a trixie upgrade | 🧱 iptables legacy vs nft mismatch | `sudo update-alternatives --config iptables` → pick `iptables-nft`; `sudo systemctl restart docker`. |
| Published port unreachable despite `ufw deny` / still reachable | Docker writes its own nft rules, bypassing ufw | Bind to `127.0.0.1:PORT:...` in Compose for host-only ports; don't rely on ufw for containers. |
| `docker.service` won't start / "cgroup" errors | cgroup v1 was forced on the kernel cmdline | Leave trixie on the default **cgroup v2**; remove any `systemd.unified_cgroup_hierarchy=0` you added. |
| Build pulls stale deps despite lockfile | Used `npm install` in image | Use `npm ci` against the committed lockfile. |

---

## 📋 New-Hire Checklist

Copy this into your onboarding ticket and tick as you go. Goal: **productive from a single clone + one command.**

```markdown
### Host setup (Debian)
- [ ] 🐧 Docker Engine (CE) installed from Docker's official APT repo (ran scripts/setup-host.sh or the manual steps)
- [ ] Added to the `docker` group; `docker run --rm hello-world` works WITHOUT sudo
- [ ] `docker compose version` shows v2.x
- [ ] `docker buildx version` works (BuildKit enabled)
- [ ] Git + an editor installed

### Repo
- [ ] Repo cloned anywhere under your home directory
- [ ] `cp .env.example .env`
- [ ] Added UID/GID: `echo "UID=$(id -u)" >> .env && echo "GID=$(id -g)" >> .env`
- [ ] `git check-ignore .env` confirms .env is ignored (no secrets committed)

### First run
- [ ] `make up` completes without errors
- [ ] `make doctor` all green
- [ ] `docker compose ps` shows db + api "healthy"
- [ ] http://localhost:8080 loads the frontend
- [ ] http://localhost:8080/api/health returns {"status":"ok"}

### Prove the loop works
- [ ] Edit a frontend file → browser hot-reloads automatically
- [ ] Edit an API file → server restarts automatically
- [ ] `make sh SVC=api` opens a shell in the container
- [ ] `make test` runs the suite inside a container
- [ ] `make migrate` applies DB migrations

### Understand the model
- [ ] I know code runs in containers; the host only has Docker+Git+editor
- [ ] I know dev and prod share the same Debian OS, base images, and Dockerfile
- [ ] I know `.env` is local & git-ignored; real secrets never get committed
- [ ] I know dev & prod differ by build target + config, not code
- [ ] I read the Troubleshooting table

✅ When every box is checked, you're productive. Welcome aboard! 🚀
```
