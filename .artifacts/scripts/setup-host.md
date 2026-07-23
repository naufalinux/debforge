# Exported: `scripts/setup-host.sh`

> Script #1 of 2 — **HOST SYSTEM PREPARATION (requires root)**. Source file:
> `scripts/setup-host.sh`. Not natively Markdown — wrapped in a fenced block below
> for this export. See [`../scripts/README.md`](../scripts/README.md) for usage.

```bash
#!/usr/bin/env bash
#
# ============================================================================
# setup-host.sh — Script #1 of 2: HOST SYSTEM PREPARATION (requires root)
# ============================================================================
#
# Automates the "Host Setup on Debian trixie" section of
# docs/dev-environment-debian.md:
#
#   1. Remove conflicting distro packages (docker.io, docker-compose, ...)
#   2. Add Docker's OFFICIAL APT repo + GPG key (NOT the distro docker.io)
#   3. Install docker-ce, docker-ce-cli, containerd.io, buildx & compose v2
#   4. Create the `docker` group and add the target (non-root) user to it
#   5. Enable/start the daemon, enable BuildKit, set up a buildx builder
#   6. Verify the whole host
#
# This script changes system state (packages, apt sources, systemd services,
# group membership) and therefore REQUIRES root. It will auto-elevate with
# sudo if you forget. The per-user, no-root steps live in setup-dev.sh.
#
# DESIGN GOALS (as requested):
#   * Verbose, timestamped, leveled logging to console AND a log file
#   * Strict error handling (set -Eeuo pipefail + ERR trap with line numbers)
#   * Fallbacks (repo suite trixie->bookworm, systemctl->service, existing
#     daemon.json is preserved, missing sudo handled)
#   * Idempotent: safe to re-run
#
# Usage:
#   sudo ./scripts/setup-host.sh                 # full host preparation
#   sudo ./scripts/setup-host.sh --user alice    # add a specific user to docker group
#   sudo ./scripts/setup-host.sh --verify        # only run verification checks
#   ./scripts/setup-host.sh --debug              # (auto-elevates) trace every command
#   ./scripts/setup-host.sh --help
# ============================================================================

# --- Strict mode -----------------------------------------------------------
# -E  : ERR trap is inherited by functions/subshells
# -e  : exit on any unhandled non-zero status
# -u  : error on use of unset variables
# -o pipefail : a pipeline fails if ANY stage fails, not just the last
set -Eeuo pipefail

# ---------------------------------------------------------------------------
# Constants & globals
# ---------------------------------------------------------------------------
readonly SCRIPT_NAME="$(basename "$0")"
readonly DOCKER_GPG_URL="https://download.docker.com/linux/debian/gpg"
readonly KEYRING="/etc/apt/keyrings/docker.asc"
readonly APT_SOURCE="/etc/apt/sources.list.d/docker.list"
readonly DAEMON_JSON="/etc/docker/daemon.json"
readonly FALLBACK_SUITE="bookworm"   # used if Docker hasn't published the host codename yet

# Populated later
CODENAME=""          # Debian codename detected from /etc/os-release
DOCKER_SUITE=""      # the suite actually used for the Docker repo
TARGET_USER=""       # the non-root user to add to the docker group
MODE="install"       # install | verify
DEBUG=0

# Log file: prefer /var/log (we're root); fall back to a temp file if not writable.
LOG_FILE="/var/log/debforge-setup-host.log"

# ---------------------------------------------------------------------------
# Logging helpers — everything is timestamped, leveled, and tee'd to LOG_FILE.
# ---------------------------------------------------------------------------
if [[ -t 1 ]]; then
  C_DIM=$'\033[2m'; C_RED=$'\033[31m'; C_GRN=$'\033[32m'
  C_YEL=$'\033[33m'; C_CYN=$'\033[36m'; C_BLD=$'\033[1m'; C_RST=$'\033[0m'
else
  C_DIM=""; C_RED=""; C_GRN=""; C_YEL=""; C_CYN=""; C_BLD=""; C_RST=""
fi

_ts() { date +'%Y-%m-%dT%H:%M:%S%z' 2>/dev/null || echo "----"; }

# _log LEVEL COLOR MESSAGE... — write one line to console + log file.
_log() {
  local level="$1" color="$2"; shift 2
  local line="[$(_ts)] [${level}] $*"
  printf '%s%s%s\n' "${color}" "${line}" "${C_RST}"
  # Append plain (uncolored) to the log file; ignore failures (e.g. read-only FS).
  printf '%s\n' "${line}" >>"${LOG_FILE}" 2>/dev/null || true
}

log()   { _log "INFO " "" "$@"; }
debug() { [[ "${DEBUG}" -eq 1 ]] && _log "DEBUG" "${C_DIM}" "$@" || true; }
warn()  { _log "WARN " "${C_YEL}" "⚠️  $*"; }
ok()    { _log "OK   " "${C_GRN}" "✅ $*"; }
step()  { printf '\n'; _log "STEP " "${C_BLD}${C_CYN}" "▶ $*"; }
err()   { _log "ERROR" "${C_RED}${C_BLD}" "❌ $*"; }

# die MESSAGE — log an error and exit non-zero.
die() { err "$*"; exit 1; }

# run CMD... — verbose command wrapper: logs the command, runs it, and logs
# failures. Keeps a consistent audit trail in the log file for debugging.
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

# ---------------------------------------------------------------------------
# ERR / EXIT traps — turn an unexpected failure into a clear, debuggable message
# instead of a silent abort.
# ---------------------------------------------------------------------------
on_err() {
  local rc=$? line=$1
  err "Aborted at line ${line} (exit ${rc})."
  err "Full log: ${LOG_FILE}"
  err "The script is idempotent — fix the issue above and re-run."
  exit "${rc}"
}
trap 'on_err $LINENO' ERR

# ---------------------------------------------------------------------------
# Usage
# ---------------------------------------------------------------------------
usage() {
  cat <<EOF
${SCRIPT_NAME} — Debian host preparation for a container-first dev environment (requires root)

Options:
  --user NAME    User to add to the 'docker' group (default: the sudo caller)
  --verify       Only run verification checks (no changes)
  --debug        Verbose shell tracing (set -x) + DEBUG log lines
  -h, --help     Show this help

Steps performed (idempotent):
  1. Remove conflicting distro packages
  2. Add Docker's official GPG key + APT repo (trixie, falling back to ${FALLBACK_SUITE})
  3. Install Docker CE + buildx + compose v2 plugins
  4. Add the target user to the 'docker' group
  5. Enable the daemon, BuildKit, and a buildx builder
  6. Verify the host

Log file: ${LOG_FILE}
EOF
}

# ---------------------------------------------------------------------------
# Argument parsing
# ---------------------------------------------------------------------------
parse_args() {
  while [[ $# -gt 0 ]]; do
    case "$1" in
      --user)   [[ $# -ge 2 ]] || die "--user needs a value"; TARGET_USER="$2"; shift 2 ;;
      --user=*) TARGET_USER="${1#*=}"; shift ;;
      --verify) MODE="verify"; shift ;;
      --debug)  DEBUG=1; shift ;;
      -h|--help) usage; exit 0 ;;
      *) die "Unknown option '$1' (try --help)" ;;
    esac
  done
}

# ---------------------------------------------------------------------------
# Ensure we're root. If not, transparently re-exec under sudo (fallback: fail
# with a clear instruction if sudo is unavailable).
# ---------------------------------------------------------------------------
ensure_root() {
  if [[ "${EUID}" -eq 0 ]]; then
    return 0
  fi
  if command -v sudo >/dev/null 2>&1; then
    log "Not running as root — re-executing under sudo..."
    # -E preserves our environment (so --debug etc. survive); pass all args through.
    exec sudo -E bash "$0" "$@"
  fi
  die "This script must run as root and 'sudo' is not installed. Re-run as root."
}

# ---------------------------------------------------------------------------
# Set up the log file (now that we're root). Fall back to a temp file if
# /var/log is not writable (e.g. exotic mounts / containers).
# ---------------------------------------------------------------------------
init_logging() {
  if ! ( : >>"${LOG_FILE}" ) 2>/dev/null; then
    local tmp="${TMPDIR:-/tmp}/debforge-setup-host.log"
    LOG_FILE="${tmp}"
    : >>"${LOG_FILE}" 2>/dev/null || LOG_FILE="/dev/null"
  fi
  log "${SCRIPT_NAME} starting (mode=${MODE}); logging to ${LOG_FILE}"
  [[ "${DEBUG}" -eq 1 ]] && { set -x; } || true
}

# ---------------------------------------------------------------------------
# Determine which non-root user should join the docker group.
# Priority: explicit --user > sudo caller > logname > first UID>=1000 human.
# ---------------------------------------------------------------------------
resolve_target_user() {
  if [[ -n "${TARGET_USER}" ]]; then
    :
  elif [[ -n "${SUDO_USER:-}" && "${SUDO_USER}" != "root" ]]; then
    TARGET_USER="${SUDO_USER}"
  elif TARGET_USER="$(logname 2>/dev/null)" && [[ -n "${TARGET_USER}" && "${TARGET_USER}" != "root" ]]; then
    :
  else
    # Fallback: first regular user account (UID 1000-59999).
    TARGET_USER="$(awk -F: '$3>=1000 && $3<60000 {print $1; exit}' /etc/passwd || true)"
  fi

  if [[ -z "${TARGET_USER}" ]]; then
    warn "Could not determine a non-root user. The 'docker' group step will be skipped."
    warn "Re-run with '--user <name>' to add a specific user."
  elif ! id "${TARGET_USER}" >/dev/null 2>&1; then
    die "Target user '${TARGET_USER}' does not exist."
  else
    log "Target user for docker group: ${TARGET_USER}"
  fi
}

# ---------------------------------------------------------------------------
# Preflight: confirm Debian-family Linux, capture codename + arch.
# ---------------------------------------------------------------------------
preflight() {
  step "Preflight checks"

  [[ "$(uname -s)" == "Linux" ]] || die "This host-prep script is for Linux (Debian). Detected: $(uname -s)."

  [[ -r /etc/os-release ]] || die "/etc/os-release not found — cannot identify the distribution."
  # shellcheck disable=SC1091
  . /etc/os-release

  if [[ "${ID:-}" != "debian" && "${ID_LIKE:-}" != *debian* ]]; then
    die "Detected '${PRETTY_NAME:-unknown}'. This automation targets Debian. For other distros, use Docker's official repo for that distro."
  fi

  CODENAME="${VERSION_CODENAME:-}"
  [[ -n "${CODENAME}" ]] || die "Could not detect VERSION_CODENAME from /etc/os-release."

  log "Distribution : ${PRETTY_NAME:-Debian}"
  log "Codename     : ${CODENAME}"
  log "Architecture : $(dpkg --print-architecture 2>/dev/null || uname -m)"

  if [[ "${CODENAME}" != "trixie" ]]; then
    warn "Reference OS is Debian trixie; you're on '${CODENAME}'. Continuing (Docker's repo supports it)."
  fi

  command -v apt-get >/dev/null 2>&1 || die "apt-get not found — this is not a Debian/APT system."
  ok "Preflight passed."
}

# ---------------------------------------------------------------------------
# Step 1 — Remove conflicting distro packages (idempotent).
# ---------------------------------------------------------------------------
remove_conflicts() {
  step "1/6 Removing conflicting distro packages"
  local pkg
  for pkg in docker.io docker-doc docker-compose docker-compose-v2 podman-docker containerd runc; do
    if dpkg -s "${pkg}" >/dev/null 2>&1; then
      log "Removing ${pkg} ..."
      run apt-get remove -y "${pkg}" || warn "Could not remove ${pkg} (continuing)."
    else
      debug "${pkg} not installed — skipping."
    fi
  done
  ok "Conflicting packages handled."
}

# ---------------------------------------------------------------------------
# Step 2 — Docker's official GPG key + APT repository.
# Fallback: if the host codename suite isn't published yet, use bookworm.
# ---------------------------------------------------------------------------
add_repo() {
  step "2/6 Adding Docker's official GPG key and APT repository"

  export DEBIAN_FRONTEND=noninteractive
  log "Updating apt indexes and installing prerequisites..."
  run apt-get update -y || warn "apt-get update reported issues (continuing)."
  run apt-get install -y ca-certificates curl gnupg \
    || die "Failed to install prerequisites (ca-certificates curl gnupg)."

  run install -m 0755 -d /etc/apt/keyrings || die "Could not create /etc/apt/keyrings."

  # Re-download the key every run so a truncated/corrupt key self-heals.
  log "Fetching Docker GPG key -> ${KEYRING}"
  if ! run curl -fsSL "${DOCKER_GPG_URL}" -o "${KEYRING}"; then
    die "Could not download Docker's GPG key from ${DOCKER_GPG_URL}. Check network/proxy."
  fi
  run chmod a+r "${KEYRING}"

  local arch suite
  arch="$(dpkg --print-architecture)"

  # Try the host codename first, then the fallback suite.
  for suite in "${CODENAME}" "${FALLBACK_SUITE}"; do
    log "Trying Docker repo suite '${suite}' ..."
    printf 'deb [arch=%s signed-by=%s] https://download.docker.com/linux/debian %s stable\n' \
      "${arch}" "${KEYRING}" "${suite}" >"${APT_SOURCE}"

    # Validate the suite by updating ONLY the Docker source list.
    if run apt-get update -y \
        -o Dir::Etc::sourcelist="$(basename "${APT_SOURCE}")" \
        -o Dir::Etc::sourceparts="-" \
        -o APT::Get::List-Cleanup="0"; then
      DOCKER_SUITE="${suite}"
      if [[ "${suite}" != "${CODENAME}" ]]; then
        warn "Docker hasn't published the '${CODENAME}' suite yet — using '${suite}' (compatible)."
        warn "Re-run this script later to switch back to '${CODENAME}'."
      fi
      ok "Docker APT repository configured (suite: ${suite})."
      run apt-get update -y || true   # refresh full index now that the source is valid
      return 0
    fi
    warn "Suite '${suite}' not available from Docker's repo."
  done

  die "No usable Docker APT suite (tried '${CODENAME}' and '${FALLBACK_SUITE}'). Check connectivity."
}

# ---------------------------------------------------------------------------
# Step 3 — Install Docker Engine + plugins.
# ---------------------------------------------------------------------------
install_docker() {
  step "3/6 Installing Docker Engine, CLI, containerd, Buildx & Compose v2"
  export DEBIAN_FRONTEND=noninteractive
  run apt-get install -y \
    docker-ce docker-ce-cli containerd.io \
    docker-buildx-plugin docker-compose-plugin \
    || die "Docker package installation failed (see ${LOG_FILE})."
  ok "Docker CE and plugins installed."
}

# ---------------------------------------------------------------------------
# Step 4 — docker group + membership.
# ---------------------------------------------------------------------------
setup_group() {
  step "4/6 Configuring the 'docker' group"
  run groupadd docker 2>/dev/null || true   # idempotent; group may already exist

  if [[ -z "${TARGET_USER}" ]]; then
    warn "No target user resolved — skipping group membership. Add manually: usermod -aG docker <user>"
    return 0
  fi

  if id -nG "${TARGET_USER}" 2>/dev/null | tr ' ' '\n' | grep -qx docker; then
    ok "User '${TARGET_USER}' is already in the 'docker' group."
  else
    run usermod -aG docker "${TARGET_USER}" || die "Failed to add '${TARGET_USER}' to docker group."
    ok "Added '${TARGET_USER}' to the 'docker' group."
    warn "Group change takes effect in a NEW login session for ${TARGET_USER}."
    warn "That user should run 'newgrp docker' or log out/in before running setup-dev.sh."
  fi
  warn "Membership in 'docker' is effectively root-equivalent — restrict it on shared servers."
}

# ---------------------------------------------------------------------------
# Step 5 — Enable service, BuildKit, buildx builder.
# Fallback: use `service` when systemd is not the active init.
# ---------------------------------------------------------------------------
enable_service() {
  step "5/6 Enabling the Docker service and BuildKit/buildx"

  if command -v systemctl >/dev/null 2>&1 && systemctl is-system-running >/dev/null 2>&1; then
    run systemctl enable --now docker || die "Could not enable/start docker.service."
    ok "docker.service enabled and started (systemd)."
  else
    warn "systemd not active — falling back to 'service docker start'."
    run service docker start || warn "Could not start docker via 'service' (start it manually)."
  fi

  # BuildKit-explicit daemon config. NEVER clobber an existing daemon.json.
  run install -d -m 0755 /etc/docker || true
  if [[ -f "${DAEMON_JSON}" ]]; then
    log "${DAEMON_JSON} already exists — leaving it untouched (not overwriting your settings)."
    if grep -q '"buildkit"' "${DAEMON_JSON}" 2>/dev/null; then
      debug "buildkit key already present in daemon.json."
    else
      warn "daemon.json exists but doesn't mention buildkit; BuildKit is on by default in modern Docker anyway."
    fi
  else
    log "Writing ${DAEMON_JSON} with BuildKit enabled."
    printf '{ "features": { "buildkit": true } }\n' >"${DAEMON_JSON}"
    # Restart to apply, with a service fallback.
    run systemctl restart docker 2>/dev/null || run service docker restart 2>/dev/null || \
      warn "Could not restart Docker to apply daemon.json — restart it manually."
    ok "Wrote ${DAEMON_JSON} and restarted Docker."
  fi

  # Set up a named buildx builder (idempotent: use it if it already exists).
  if docker buildx version >/dev/null 2>&1; then
    if docker buildx inspect devbuilder >/dev/null 2>&1; then
      run docker buildx use devbuilder || true
    else
      run docker buildx create --use --name devbuilder || run docker buildx use devbuilder || true
    fi
    ok "buildx builder 'devbuilder' is active."
  else
    warn "buildx not callable yet — verify after Docker is fully up."
  fi
}

# ---------------------------------------------------------------------------
# Step 6 — Verification (mirrors the doc's "✅ Verify the whole host").
# ---------------------------------------------------------------------------
verify() {
  step "6/6 Verifying the host"
  local failures=0

  if docker version --format '{{.Server.Version}}' >/dev/null 2>&1; then
    ok "Docker Engine: $(docker version --format '{{.Server.Version}}')"
  else
    warn "Cannot reach the Docker daemon (is it running?)."; failures=$((failures+1))
  fi

  if docker compose version >/dev/null 2>&1; then
    ok "Compose v2: $(docker compose version --short 2>/dev/null || docker compose version | head -n1)"
  else
    warn "'docker compose' (v2) unavailable — ensure docker-compose-plugin is installed."; failures=$((failures+1))
  fi

  if docker buildx version >/dev/null 2>&1; then
    ok "buildx: $(docker buildx version | head -n1)"
  else
    warn "'docker buildx' unavailable — ensure docker-buildx-plugin is installed."; failures=$((failures+1))
  fi

  if run docker run --rm hello-world; then
    ok "'hello-world' container ran successfully."
  else
    warn "Could not run hello-world (check daemon + network)."; failures=$((failures+1))
  fi

  if [[ -n "${TARGET_USER}" ]]; then
    if id -nG "${TARGET_USER}" 2>/dev/null | tr ' ' '\n' | grep -qx docker; then
      ok "User '${TARGET_USER}' is in the 'docker' group."
    else
      warn "User '${TARGET_USER}' is NOT in the 'docker' group."; failures=$((failures+1))
    fi
  fi

  if [[ "${failures}" -eq 0 ]]; then
    ok "Host verification PASSED."
    return 0
  fi
  warn "Host verification finished with ${failures} issue(s) — see ${LOG_FILE}."
  return 1
}

# ---------------------------------------------------------------------------
# Final banner
# ---------------------------------------------------------------------------
finish() {
  step "Host preparation complete"
  log "Next: the developer runs the NON-root script from the project directory:"
  log "    ./scripts/setup-dev.sh"
  if [[ -n "${TARGET_USER}" ]]; then
    warn "${TARGET_USER} must start a fresh shell first ('newgrp docker' or log out/in) so docker works without sudo."
  fi
  ok "Done. 🚀"
}

# ---------------------------------------------------------------------------
# Main
# ---------------------------------------------------------------------------
main() {
  parse_args "$@"
  ensure_root "$@"
  init_logging
  resolve_target_user
  preflight

  if [[ "${MODE}" == "verify" ]]; then
    verify; exit $?
  fi

  remove_conflicts
  add_repo
  install_docker
  setup_group
  enable_service
  verify || true       # report, but don't hard-fail the whole run on a soft check
  finish
}

main "$@"
```
