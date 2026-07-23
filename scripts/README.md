# 🛠️ DebForge Setup Scripts

Automation for standing up the **container-first Debian dev environment** described in
[`../docs/dev-environment-debian.md`](../docs/dev-environment-debian.md).

There are **two scripts**, run in order. The split is deliberate: one changes the
**machine** (needs root), the other configures **your user + the containers** (must *not* be root).

| # | Script | Runs as | Changes | What it does |
|---|--------|---------|---------|--------------|
| 1 | [`setup-host.sh`](./setup-host.sh) | **root** (auto-elevates) | system-wide | Installs Docker CE from Docker's official repo, buildx, Compose v2; adds you to the `docker` group; enables the daemon + BuildKit. |
| 2 | [`setup-dev.sh`](./setup-dev.sh) | **your user** (refuses root) | your repo | Creates `.env`, sets your `UID`/`GID`, then builds, starts, and health-checks the stack. |

> 💡 Run `setup-host.sh` **once per machine**. Run `setup-dev.sh` **once per clone** (and again any time you want to (re)start the stack).

---

## ⚡ TL;DR

```bash
# 1. Prepare the host (installs Docker, adds you to the docker group)
sudo ./scripts/setup-host.sh

# 2. Activate your new docker-group membership in the current shell
newgrp docker            # or just log out and back in

# 3. From the project root: configure + build + start + verify the stack
./scripts/setup-dev.sh
```

✅ When it finishes you'll have the stack running at <http://localhost:8080>
(health: <http://localhost:8080/api/health>).

---

## ✅ Prerequisites

- **Debian 13 "trixie"** (reference) or Debian 12 "bookworm". A Debian-family
  `ID`/`ID_LIKE` in `/etc/os-release` is required — the host script refuses to run elsewhere.
- Internet access to `download.docker.com` (for the APT repo) and Docker Hub (for images).
- `sudo` available for step 1 (or run it directly as root).
- The project repo cloned somewhere under your home directory.

Everything else — Node, PostgreSQL, etc. — lives **in containers**. You do not install
language runtimes on the host.

---

## 1️⃣ `setup-host.sh` — host system preparation (root)

Automates the [Host Setup on Debian trixie](../docs/dev-environment-debian.md#-host-setup-on-debian-trixie)
section. It **requires root** because it edits APT sources, installs packages, manages
the `docker` systemd service, and modifies group membership. If you forget `sudo`, it
**re-executes itself under sudo** automatically.

### What it does (idempotent)

1. Removes conflicting distro packages (`docker.io`, `docker-compose`, `containerd`, `runc`, …).
2. Adds Docker's **official** GPG key + APT repository (not the distro `docker.io`).
3. Installs `docker-ce`, `docker-ce-cli`, `containerd.io`, `docker-buildx-plugin`, `docker-compose-plugin`.
4. Creates the `docker` group and adds the target user to it.
5. Enables/starts the daemon, turns on **BuildKit**, and creates a `devbuilder` buildx builder.
6. Verifies the host (`hello-world`, `docker compose version`, buildx, group membership).

### Usage

```bash
sudo ./scripts/setup-host.sh                 # full host preparation
sudo ./scripts/setup-host.sh --user alice    # add a specific user to the docker group
sudo ./scripts/setup-host.sh --verify        # only run verification checks (no changes)
./scripts/setup-host.sh --debug              # (auto-elevates) trace every command
./scripts/setup-host.sh --help
```

### Options

| Flag | Meaning |
|------|---------|
| `--user NAME` | User to add to the `docker` group. Default: the `sudo` caller (`$SUDO_USER`), else `logname`, else the first regular account (UID ≥ 1000). |
| `--verify` | Run only the verification checks; make no changes. |
| `--debug` | Shell tracing (`set -x`) + `DEBUG` log lines. |
| `-h`, `--help` | Show help. |

### Fallbacks & safety

- **Repo suite fallback** — if Docker hasn't published the `trixie` suite yet, it validates
  and falls back to `bookworm` (compatible), with a warning to re-run later.
- **Init system fallback** — uses `systemctl` when systemd is active, otherwise `service`.
- **Non-destructive daemon config** — an existing `/etc/docker/daemon.json` is **never**
  overwritten (BuildKit is on by default in modern Docker anyway).
- **Log file:** `/var/log/debforge-setup-host.log` (falls back to `$TMPDIR` if not writable).

---

## 2️⃣ `setup-dev.sh` — user config + containerized environment (non-root)

Automates the per-developer steps: [Onboarding](../docs/dev-environment-debian.md#️-onboarding-a-debian-workstation),
[Environment & Secrets](../docs/dev-environment-debian.md#-environment--secrets), and the
[Quickstart](../docs/dev-environment-debian.md#-quickstart-tldr) bring-up.

It **refuses to run as root** — it configures *your* user and writes `.env`, and using
`sudo` here would create root-owned files (exactly the UID/GID bind-mount problem the
docs warn about).

### What it does (idempotent)

1. **Preflight** — confirms Docker is reachable **without sudo**, Compose v2 is present,
   and (best-effort) buildx works. Distinguishes "not installed" from "docker group not
   active in this shell yet."
2. **Configure** — creates `.env` from `.env.example` (minimal fallback if the template is
   missing), upserts your real `UID`/`GID`, and checks `.env` is git-ignored.
3. **Bring-up** — builds + starts the stack. Prefers `make up`; falls back to
   `docker compose up -d --build`.
4. **Verify** — waits for services to become healthy (per-container, with a timeout), then
   probes the frontend and `/api/health` (auto-detecting the proxy port).

### Usage

```bash
./scripts/setup-dev.sh                       # configure + build + start + verify
./scripts/setup-dev.sh --no-up               # configure .env / UID-GID only
./scripts/setup-dev.sh --verify              # only run health + endpoint checks
./scripts/setup-dev.sh --down                # stop & remove the stack (keeps data)
./scripts/setup-dev.sh --url http://localhost:8081   # override the probed URL
./scripts/setup-dev.sh --project-dir /path/to/repo   # point at a specific repo root
./scripts/setup-dev.sh --debug
./scripts/setup-dev.sh --help
```

### Options

| Flag | Meaning |
|------|---------|
| `--no-up` | Only configure `.env`/`UID`/`GID`; don't start containers. |
| `--verify` | Only run health + endpoint checks against an already-running stack. |
| `--down` | Stop and remove containers (named volumes/data are preserved). |
| `--url URL` | App base URL to probe. Default: auto-detected from the `proxy` port mapping, else `http://localhost:8080`. |
| `--project-dir DIR` | Repo root. Default: auto-detected by walking up from the current dir to a `docker-compose.yml`. |
| `--debug` | Shell tracing (`set -x`) + `DEBUG` log lines. |
| `-h`, `--help` | Show help. |

### Fallbacks & safety

- **Task-runner fallback** — uses `make` targets when a `Makefile` is present, otherwise
  raw `docker compose`.
- **Health wait with timeout** — polls each container (`healthy` if it has a healthcheck,
  else `running`) for up to 180s instead of hanging forever; on timeout it prints status
  and log hints.
- **Port auto-detect** — reads the published `proxy:80` port so the endpoint probe works
  even if `8080` was remapped.
- **Graceful when scaffolding is missing** — if there's no compose file yet, it still
  configures `.env` and clearly skips the bring-up.
- **Log file:** `<repo>/setup-dev.log` (falls back to `$TMPDIR`).

---

## 🧭 End-to-end flow

```text
┌─────────────────────────────┐        ┌──────────────────────────────┐
│  sudo ./setup-host.sh        │        │  ./setup-dev.sh               │
│  (root — once per machine)   │        │  (you — once per clone)       │
│                              │        │                               │
│  • Docker CE + buildx        │        │  • preflight (no sudo!)       │
│  • compose v2                │  ───▶  │  • .env + UID/GID             │
│  • docker group + you        │        │  • make up / compose up       │
│  • daemon + BuildKit         │        │  • wait for healthy           │
│  • verify host               │        │  • probe /api/health          │
└─────────────────────────────┘        └──────────────────────────────┘
        then: `newgrp docker`  ────────────────▲
        (activate group in your shell)
```

---

## 🧯 Troubleshooting

| Symptom | Cause | Fix |
|---------|-------|-----|
| `setup-dev.sh` says *"Docker works only with sudo"* | Your shell hasn't picked up the `docker` group yet | Run `newgrp docker` (or log out/in), then re-run. |
| `setup-dev.sh` exits: *"Do NOT run as root"* | You ran it with `sudo` | Run it as your normal user; only `setup-host.sh` needs root. |
| `apt-get update` 404s on the Docker repo | Docker hasn't published the `trixie` suite yet | The host script auto-falls back to `bookworm`; re-run later to switch back. |
| Health wait times out | A service is crash-looping or slow to start | `docker compose logs -f` (or `docker compose logs -f api`) to see why. |
| `bind: address already in use` | Port 8080 (or another) is taken | Remap in `docker-compose.override.yml`; find the process with `ss -ltnp`. |
| Files owned by `root` on bind mounts | `UID`/`GID` mismatch | Re-run `setup-dev.sh` (it sets `UID`/`GID` in `.env`), then `make rebuild`. |

For anything else, check the log files (`/var/log/debforge-setup-host.log`,
`<repo>/setup-dev.log`) — every command is recorded there — and see the full
[Troubleshooting section](../docs/dev-environment-debian.md#-troubleshooting) in the main doc.

---

## 📎 Notes

- Both scripts are **idempotent** — safe to re-run.
- Both use strict mode (`set -Eeuo pipefail`) with an error trap that reports the failing
  line, plus timestamped, leveled logging (`INFO`/`WARN`/`OK`/`ERROR`, `DEBUG` with `--debug`).
- They target **Debian only**. For a mixed Windows/macOS/Linux team, see the cross-platform
  edition of the environment doc.
