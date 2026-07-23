# Exported: `scripts/setup-dev.sh`

> Script #2 of 2 — **USER CONFIG + CONTAINERIZED ENV PREPARATION (no root)**. Source file:
> `scripts/setup-dev.sh`. Not natively Markdown — wrapped in a fenced block below
> for this export. See [`../scripts/README.md`](../scripts/README.md) for usage.

```bash
#!/usr/bin/env bash
#
# ============================================================================
# setup-dev.sh — Script #2 of 2: USER CONFIG + CONTAINERIZED ENV PREPARATION
# ============================================================================
#
# Run this AFTER setup-host.sh (the root host-prep script) has installed
# Docker CE and added you to the `docker` group. This script performs the
# per-developer, NON-root steps from docs/dev-environment-debian.md:
#
#   * Preflight: Docker reachable WITHOUT sudo, Compose v2, buildx, project files
#   * Config:    create .env from .env.example, inject your UID/GID,
#                confirm .env is git-ignored
#   * Bring-up:  build + start the dev stack (prefers `make up`, falls back to
#                `docker compose up -d --build`)
#   * Verify:    wait for healthchecks, probe the app endpoint
#
# It intentionally does NOT use sudo: it configures YOUR user and files, and
# talking to Docker via sudo here would create root-owned artifacts (exactly
# the UID/GID bind-mount gotcha the doc warns about).
#
# DESIGN GOALS (as requested):
#   * Verbose, timestamped, leveled logging to console AND a log file
#   * Strict error handling (set -Eeuo pipefail + ERR trap with line numbers)
#   * Fallbacks (make->docker compose, port autodetect, health polling with
#     timeout, .env.example missing, docker-group-not-active hints)
#   * Idempotent: safe to re-run
#
# Usage:
#   ./scripts/setup-dev.sh                  # configure + bring the stack up
#   ./scripts/setup-dev.sh --no-up          # only configure .env / UID-GID
#   ./scripts/setup-dev.sh --verify         # only run health/endpoint checks
#   ./scripts/setup-dev.sh --down           # tear the stack down
#   ./scripts/setup-dev.sh --url http://localhost:8081   # override app URL
#   ./scripts/setup-dev.sh --project-dir /path/to/repo
#   ./scripts/setup-dev.sh --debug
#   ./scripts/setup-dev.sh --help
# ============================================================================

set -Eeuo pipefail

# ---------------------------------------------------------------------------
# Constants & globals
# ---------------------------------------------------------------------------
readonly SCRIPT_NAME="$(basename "$0")"
readonly HEALTH_TIMEOUT=180     # seconds to wait for services to become healthy
readonly HEALTH_INTERVAL=5      # seconds between health polls

PROJECT_DIR=""                  # resolved project root (contains docker-compose.yml)
APP_URL=""                      # health endpoint base (auto-detected if empty)
ACTION="up"                     # up | config-only | verify | down
DEBUG=0
COMPOSE=(docker compose)        # base compose invocation (array = safe quoting)
USE_MAKE=0                      # whether a usable Makefile + make binary exist

# Log file lives in the project dir once known; bootstrap to a temp file first.
LOG_FILE="${TMPDIR:-/tmp}/debforge-setup-dev.log"

# ---------------------------------------------------------------------------
# Logging helpers (self-contained; mirror setup-host.sh for consistency).
# ---------------------------------------------------------------------------
if [[ -t 1 ]]; then
  C_DIM=$'\033[2m'; C_RED=$'\033[31m'; C_GRN=$'\033[32m'
  C_YEL=$'\033[33m'; C_CYN=$'\033[36m'; C_BLD=$'\033[1m'; C_RST=$'\033[0m'
else
  C_DIM=""; C_RED=""; C_GRN=""; C_YEL=""; C_CYN=""; C_BLD=""; C_RST=""
fi

_ts() { date +'%Y-%m-%dT%H:%M:%S%z' 2>/dev/null || echo "----"; }

_log() {
  local level="$1" color="$2"; shift 2
  local line="[$(_ts)] [${level}] $*"
  printf '%s%s%s\n' "${color}" "${line}" "${C_RST}"
  printf '%s\n' "${line}" >>"${LOG_FILE}" 2>/dev/null || true
}

log()   { _log "INFO " "" "$@"; }
debug() { [[ "${DEBUG}" -eq 1 ]] && _log "DEBUG" "${C_DIM}" "$@" || true; }
warn()  { _log "WARN " "${C_YEL}" "⚠️  $*"; }
ok()    { _log "OK   " "${C_GRN}" "✅ $*"; }
step()  { printf '\n'; _log "STEP " "${C_BLD}${C_CYN}" "▶ $*"; }
err()   { _log "ERROR" "${C_RED}${C_BLD}" "❌ $*"; }
die()   { err "$*"; exit 1; }

# run CMD... — verbose wrapper that streams output to the log file.
run() {
  debug "\$ $*"
  if "$@" >>"${LOG_FILE}" 2>&1; then
    return 0
  else
    local rc=$?
    warn "command failed (rc=${rc}): $*  (see ${LOG_FILE})"
    return "${rc}"
  fi
}

on_err() {
  local rc=$? line=$1
  err "Aborted at line ${line} (exit ${rc}). Full log: ${LOG_FILE}"
  exit "${rc}"
}
trap 'on_err $LINENO' ERR

# ---------------------------------------------------------------------------
# Usage
# ---------------------------------------------------------------------------
usage() {
  cat <<EOF
${SCRIPT_NAME} — per-developer config + containerized dev environment bring-up (NO root)

Options:
  --no-up             Configure .env / UID-GID only; don't start containers
  --verify            Only run health + endpoint checks against a running stack
  --down              Stop and remove the dev stack (keeps volumes/data)
  --url URL           App base URL to probe (default: auto-detect, else http://localhost:8080)
  --project-dir DIR   Path to the repo root (default: auto-detected)
  --debug             Verbose shell tracing (set -x) + DEBUG log lines
  -h, --help          Show this help

Typical flow:
  1) sudo ./scripts/setup-host.sh     # once per machine (root)
  2) newgrp docker                    # activate docker group in your shell
  3) ./scripts/setup-dev.sh           # this script
EOF
}

# ---------------------------------------------------------------------------
# Argument parsing
# ---------------------------------------------------------------------------
parse_args() {
  while [[ $# -gt 0 ]]; do
    case "$1" in
      --no-up)         ACTION="config-only"; shift ;;
      --verify)        ACTION="verify"; shift ;;
      --down)          ACTION="down"; shift ;;
      --url)           [[ $# -ge 2 ]] || die "--url needs a value"; APP_URL="$2"; shift 2 ;;
      --url=*)         APP_URL="${1#*=}"; shift ;;
      --project-dir)   [[ $# -ge 2 ]] || die "--project-dir needs a value"; PROJECT_DIR="$2"; shift 2 ;;
      --project-dir=*) PROJECT_DIR="${1#*=}"; shift ;;
      --debug)         DEBUG=1; shift ;;
      -h|--help)       usage; exit 0 ;;
      *) die "Unknown option '$1' (try --help)" ;;
    esac
  done
}

# ---------------------------------------------------------------------------
# Refuse to run as root: this script configures the *developer's* user/files.
# ---------------------------------------------------------------------------
ensure_not_root() {
  if [[ "${EUID}" -eq 0 ]]; then
    die "Do NOT run setup-dev.sh as root/sudo. Run it as your normal user so .env and bind mounts are owned by you. (Host prep is the only root step: setup-host.sh.)"
  fi
}

# ---------------------------------------------------------------------------
# Locate the project root. Priority: --project-dir > walk up from CWD > the
# script's parent directory. We anchor on docker-compose.yml.
# ---------------------------------------------------------------------------
resolve_project_dir() {
  if [[ -n "${PROJECT_DIR}" ]]; then
    [[ -d "${PROJECT_DIR}" ]] || die "--project-dir '${PROJECT_DIR}' is not a directory."
  else
    # Walk up from the current directory looking for a compose file.
    local d="${PWD}"
    while [[ "${d}" != "/" ]]; do
      if [[ -f "${d}/docker-compose.yml" || -f "${d}/compose.yaml" ]]; then
        PROJECT_DIR="${d}"; break
      fi
      d="$(dirname "${d}")"
    done
    # Fallback: assume the repo root is the script's parent's parent (scripts/..).
    if [[ -z "${PROJECT_DIR}" ]]; then
      local self_dir; self_dir="$(cd "$(dirname "$0")" && pwd)"
      PROJECT_DIR="$(dirname "${self_dir}")"
    fi
  fi

  cd "${PROJECT_DIR}" || die "Cannot cd into project dir '${PROJECT_DIR}'."
  PROJECT_DIR="${PWD}"

  # Re-home the log file inside the project (fallback to temp if not writable).
  local candidate="${PROJECT_DIR}/setup-dev.log"
  if ( : >>"${candidate}" ) 2>/dev/null; then
    LOG_FILE="${candidate}"
  fi

  log "Project dir : ${PROJECT_DIR}"
  log "Log file    : ${LOG_FILE}"

  if [[ ! -f "${PROJECT_DIR}/docker-compose.yml" && ! -f "${PROJECT_DIR}/compose.yaml" ]]; then
    warn "No docker-compose.yml/compose.yaml found in ${PROJECT_DIR}."
    warn "Configuration steps will still run; container bring-up will be skipped."
  fi
}

# ---------------------------------------------------------------------------
# Preflight: verify Docker works WITHOUT sudo, Compose v2, buildx.
# The most common failure here is "installed but docker group not active yet".
# ---------------------------------------------------------------------------
preflight() {
  step "Preflight checks"
  [[ "${DEBUG}" -eq 1 ]] && set -x || true

  command -v docker >/dev/null 2>&1 || die "docker not found. Run the host prep first: sudo ./scripts/setup-host.sh"

  # Can we talk to the daemon as this (non-root) user?
  if ! docker info >/dev/null 2>&1; then
    if command -v sudo >/dev/null 2>&1 && sudo -n docker info >/dev/null 2>&1; then
      die "Docker works only with sudo — your shell hasn't picked up the 'docker' group yet. Run 'newgrp docker' (or log out/in) and re-run this script."
    fi
    die "Cannot reach the Docker daemon. Is it running? Try: sudo systemctl start docker (or run setup-host.sh)."
  fi
  ok "Docker daemon reachable without sudo (Engine $(docker version --format '{{.Server.Version}}' 2>/dev/null || echo '?'))."

  # Compose v2 is mandatory (v1 hyphenated is deprecated per the doc).
  if docker compose version >/dev/null 2>&1; then
    ok "Compose v2: $(docker compose version --short 2>/dev/null || echo present)"
  else
    die "'docker compose' (v2) missing. Install docker-compose-plugin via setup-host.sh."
  fi

  # buildx is expected but not strictly required to bring the stack up.
  if docker buildx version >/dev/null 2>&1; then
    ok "buildx: $(docker buildx version | head -n1)"
  else
    warn "buildx not available — builds may be slower / lack cache mounts."
  fi

  # Decide whether we can use the Makefile task runner as a nicety.
  if command -v make >/dev/null 2>&1 && [[ -f "${PROJECT_DIR}/Makefile" ]]; then
    USE_MAKE=1
    debug "Makefile + make found — will use make targets where available."
  else
    debug "No make/Makefile — using raw docker compose."
  fi
}

# ---------------------------------------------------------------------------
# Configure .env: create from .env.example, then ensure UID/GID match the host
# user (fixes the root-owned-bind-mount gotcha).
# ---------------------------------------------------------------------------
configure_env() {
  step "Configuring environment (.env, UID/GID)"

  local env_file="${PROJECT_DIR}/.env"
  local example="${PROJECT_DIR}/.env.example"

  if [[ -f "${env_file}" ]]; then
    ok ".env already exists — leaving your values in place (idempotent)."
  elif [[ -f "${example}" ]]; then
    cp "${example}" "${env_file}"
    ok "Created .env from .env.example."
  else
    # Fallback: no template committed. Create a minimal, safe dev .env so the
    # developer isn't blocked, and clearly flag that it's a stopgap.
    warn "No .env.example found — writing a minimal fallback .env (review it!)."
    cat >"${env_file}" <<'EOF'
# Fallback .env generated by setup-dev.sh (no .env.example was present).
# Review these values — dummy dev credentials only, never use in prod.
POSTGRES_USER=app
POSTGRES_PASSWORD=devpassword_change_me
POSTGRES_DB=app
NODE_ENV=development
EOF
  fi

  # --- Inject / update UID and GID so bind-mounted files are owned by us. ----
  local uid gid
  uid="$(id -u)"; gid="$(id -g)"
  upsert_env_var "${env_file}" "UID" "${uid}"
  upsert_env_var "${env_file}" "GID" "${gid}"
  ok "Set UID=${uid} GID=${gid} in .env."

  verify_env_ignored
}

# upsert_env_var FILE KEY VALUE — set KEY=VALUE, replacing any existing line.
# Idempotent and safe for re-runs.
upsert_env_var() {
  local file="$1" key="$2" value="$3"
  if grep -qE "^${key}=" "${file}" 2>/dev/null; then
    # Replace in place; use a temp file to avoid sed -i portability issues.
    local tmp; tmp="$(mktemp)"
    grep -vE "^${key}=" "${file}" >"${tmp}"
    printf '%s=%s\n' "${key}" "${value}" >>"${tmp}"
    mv "${tmp}" "${file}"
    debug "Updated ${key} in ${file}."
  else
    printf '%s=%s\n' "${key}" "${value}" >>"${file}"
    debug "Appended ${key} to ${file}."
  fi
}

# verify_env_ignored — confirm .env won't be committed. Best-effort (only if
# this is a git repo).
verify_env_ignored() {
  if command -v git >/dev/null 2>&1 && git -C "${PROJECT_DIR}" rev-parse --is-inside-work-tree >/dev/null 2>&1; then
    if git -C "${PROJECT_DIR}" check-ignore -q .env; then
      ok ".env is git-ignored (secrets won't be committed)."
    else
      warn ".env is NOT git-ignored! Add '.env' to .gitignore before committing anything."
    fi
  else
    debug "Not a git work tree — skipping .env ignore check."
  fi
}

# ---------------------------------------------------------------------------
# Bring the stack up. Prefer `make up`; fall back to raw docker compose.
# ---------------------------------------------------------------------------
compose_up() {
  step "Building and starting the dev stack"

  if [[ ! -f "${PROJECT_DIR}/docker-compose.yml" && ! -f "${PROJECT_DIR}/compose.yaml" ]]; then
    warn "No compose file present — skipping bring-up. (Add docker-compose.yml, then re-run.)"
    return 0
  fi

  if [[ "${USE_MAKE}" -eq 1 ]] && make -n up >/dev/null 2>&1; then
    log "Using 'make up'."
    run make up || die "'make up' failed (see ${LOG_FILE})."
  else
    log "Using 'docker compose up -d --build'."
    run "${COMPOSE[@]}" up -d --build || die "'docker compose up' failed (see ${LOG_FILE})."
  fi
  ok "Containers started."
}

# ---------------------------------------------------------------------------
# Wait for services to become healthy. Containers WITH a healthcheck must
# report 'healthy'; containers WITHOUT one only need to be 'running'.
# Times out after HEALTH_TIMEOUT seconds (fallback: warn, don't hang forever).
# ---------------------------------------------------------------------------
wait_for_health() {
  step "Waiting for services to become healthy (timeout ${HEALTH_TIMEOUT}s)"

  local deadline=$(( SECONDS + HEALTH_TIMEOUT ))
  local ids id status state unhealthy

  while :; do
    # Collect container IDs for this compose project.
    ids="$("${COMPOSE[@]}" ps -q 2>/dev/null || true)"
    if [[ -z "${ids}" ]]; then
      warn "No containers found for this project yet."
    fi

    unhealthy=0
    while IFS= read -r id; do
      [[ -n "${id}" ]] || continue
      # Health status is empty for containers without a healthcheck.
      status="$(docker inspect -f '{{if .State.Health}}{{.State.Health.Status}}{{end}}' "${id}" 2>/dev/null || true)"
      state="$(docker inspect -f '{{.State.Status}}' "${id}" 2>/dev/null || echo unknown)"
      local name; name="$(docker inspect -f '{{.Name}}' "${id}" 2>/dev/null | sed 's#^/##' || echo "${id:0:12}")"

      if [[ -n "${status}" ]]; then
        # Has a healthcheck — must be 'healthy'.
        if [[ "${status}" != "healthy" ]]; then
          unhealthy=$((unhealthy+1))
          debug "  ${name}: health=${status} state=${state}"
        fi
      else
        # No healthcheck — 'running' is good enough.
        if [[ "${state}" != "running" ]]; then
          unhealthy=$((unhealthy+1))
          debug "  ${name}: state=${state} (no healthcheck)"
        fi
      fi
    done <<< "${ids}"

    if [[ -n "${ids}" && "${unhealthy}" -eq 0 ]]; then
      ok "All services are healthy/running."
      return 0
    fi

    if [[ "${SECONDS}" -ge "${deadline}" ]]; then
      warn "Timed out waiting for health. Current status:"
      "${COMPOSE[@]}" ps || true
      warn "Inspect logs with: docker compose logs -f"
      return 1
    fi

    log "…${unhealthy} service(s) not ready; re-checking in ${HEALTH_INTERVAL}s"
    sleep "${HEALTH_INTERVAL}"
  done
}

# ---------------------------------------------------------------------------
# Auto-detect the app URL from the dev override's proxy port mapping, so the
# endpoint probe still works if someone remapped 8080 -> 8081.
# ---------------------------------------------------------------------------
detect_app_url() {
  [[ -n "${APP_URL}" ]] && return 0   # explicit --url wins

  local host_port=""
  # Ask Compose for the published port of the 'proxy' service, container port 80.
  host_port="$("${COMPOSE[@]}" port proxy 80 2>/dev/null | sed -E 's/.*:([0-9]+)$/\1/' | head -n1 || true)"

  if [[ -n "${host_port}" ]]; then
    APP_URL="http://localhost:${host_port}"
    debug "Detected app URL from compose: ${APP_URL}"
  else
    APP_URL="http://localhost:8080"   # documented default
    debug "Falling back to default app URL: ${APP_URL}"
  fi
}

# ---------------------------------------------------------------------------
# Probe the app + health endpoint. Best-effort: a failure is a warning, not a
# hard error (the app may take a moment, or curl may be absent).
# ---------------------------------------------------------------------------
verify_endpoints() {
  step "Verifying application endpoints"
  detect_app_url
  log "App URL: ${APP_URL}"

  if ! command -v curl >/dev/null 2>&1; then
    warn "curl not installed — skipping endpoint probe. Open ${APP_URL} in a browser."
    return 0
  fi

  local health="${APP_URL%/}/api/health"
  if curl -fsS --max-time 10 "${health}" >/dev/null 2>&1; then
    ok "Health endpoint OK: ${health}"
  else
    warn "Health endpoint not responding yet: ${health}"
    warn "It may still be starting. Check: docker compose logs -f api"
  fi

  if curl -fsS --max-time 10 -o /dev/null "${APP_URL}"; then
    ok "Frontend reachable: ${APP_URL}"
  else
    warn "Frontend not reachable yet at ${APP_URL} (check the proxy/web logs)."
  fi
}

# ---------------------------------------------------------------------------
# Tear-down (for --down).
# ---------------------------------------------------------------------------
compose_down() {
  step "Stopping the dev stack"
  if [[ "${USE_MAKE}" -eq 1 ]] && make -n down >/dev/null 2>&1; then
    run make down || warn "'make down' failed."
  else
    run "${COMPOSE[@]}" down || warn "'docker compose down' failed."
  fi
  ok "Stack stopped (named volumes/data preserved)."
}

# ---------------------------------------------------------------------------
# Final banner with the daily-workflow cheatsheet from the doc.
# ---------------------------------------------------------------------------
finish() {
  step "Developer environment ready"
  cat <<EOF
${C_GRN}${C_BLD}
  🎉 You're set up. Handy commands:
${C_RST}    make up          # build + start (or: docker compose up -d --build)
    make logs        # follow logs        (docker compose logs -f)
    make sh SVC=api  # shell in a service (docker compose exec api bash)
    make test        # run tests          (docker compose run --rm api npm test)
    make down        # stop the stack     (docker compose down)

    App:    ${APP_URL:-http://localhost:8080}
    Health: ${APP_URL:-http://localhost:8080}/api/health
    Log:    ${LOG_FILE}
EOF
}

# ---------------------------------------------------------------------------
# Main
# ---------------------------------------------------------------------------
main() {
  parse_args "$@"
  ensure_not_root
  resolve_project_dir
  log "${SCRIPT_NAME} starting (action=${ACTION})"

  case "${ACTION}" in
    down)
      preflight
      compose_down
      ;;
    verify)
      preflight
      wait_for_health || true
      verify_endpoints
      ;;
    config-only)
      preflight
      configure_env
      ok "Configuration complete (--no-up: skipped container bring-up)."
      ;;
    up)
      preflight
      configure_env
      compose_up
      wait_for_health || warn "Proceeding despite health timeout — inspect logs if the app misbehaves."
      verify_endpoints
      finish
      ;;
    *)
      die "Unknown action '${ACTION}'."
      ;;
  esac
}

main "$@"
```
