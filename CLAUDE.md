# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

DebForge is **documentation and setup automation for a container-first fullstack dev
environment**, not an application itself. There is no `apps/`, `docker-compose.yml`, or
`Dockerfile` in this repo — those are *illustrated* inside the docs as the reference
layout a team scaffolds on top of this. Don't go looking for application source; if asked
to "add a feature," clarify whether the request is about the docs/scripts here or about a
downstream project scaffolded from them.

The repo has three parts:
- `README.md` — top-level orientation and quickstart pointer.
- `docs/dev-environment-debian.md` and `docs/dev-environment.md` — the actual product:
  long-form reference guides.
- `scripts/setup-host.sh` and `scripts/setup-dev.sh` — automation that implements what
  the docs describe, plus `scripts/README.md` documenting them.

`.artifacts/` is a static, generated snapshot/export of the above for external sharing —
it is **not canonical**; edit the real files at the repo root/`docs/`/`scripts/`, never
`.artifacts/`.

## The two docs are near-duplicates by design — keep them in sync

`docs/dev-environment-debian.md` (Debian-only reference edition) and
`docs/dev-environment.md` (cross-platform Windows/macOS/Linux edition) share the same
section structure, the same example Dockerfiles/Compose files, and the same host-setup
steps for Debian trixie. The cross-platform doc simply adds Windows (WSL2) and macOS
(Docker Desktop/Colima) onboarding subsections around the same core.

**When editing shared content (host setup steps, Dockerfile/Compose examples,
troubleshooting table, env/secrets strategy), apply the change to both files.** Only the
"Onboarding" section and framing genuinely diverge between them. Use
`container-dev-env-doc-prompt.md` (the original generation prompt) as the source of truth
for required structure, tone, and hard constraints (e.g. no `docker-compose` v1 syntax, no
`latest` tags, no host-installed language runtimes) if regenerating or substantially
rewriting either doc.

## The two scripts are a deliberate root/non-root split

- `scripts/setup-host.sh` — **requires root** (auto-elevates via `sudo` if not). Changes
  machine-wide state: APT repo/packages, the `docker` group, the systemd service,
  `/etc/docker/daemon.json`. Run once per machine.
- `scripts/setup-dev.sh` — **refuses to run as root**. Configures the developer's own
  `.env`/`UID`/`GID` and brings up the Compose stack. Run once per clone (and again to
  restart/verify/tear down the stack).

Do not merge these or add root-requiring steps to `setup-dev.sh` — the split exists
specifically to avoid root-owned files on bind mounts (the UID/GID gotcha both docs and
scripts call out repeatedly).

Both scripts share the same conventions; new scripts or edits should follow them:
- `set -Eeuo pipefail` + an `ERR` trap (`on_err`) that reports the failing line number.
- A shared logging pattern: `log`/`warn`/`ok`/`err`/`step`/`debug` helpers, timestamped,
  tee'd to a log file (`/var/log/debforge-setup-host.log` for the host script,
  `<repo>/setup-dev.log` for the dev script), with graceful fallback if the preferred log
  location isn't writable.
- A `run CMD...` wrapper for external commands that logs failures without leaking noisy
  output to the console.
- Every step is idempotent — safe to re-run — and every destructive/system-changing step
  has a documented fallback (e.g. trixie→bookworm APT suite fallback, systemctl→service,
  make→raw `docker compose`).
- `--debug` (shell tracing + verbose logs), `--help`, and mode-specific flags
  (`--verify`, `--no-up`/`--down` on the dev script) are part of the expected CLI surface —
  extend this pattern rather than inventing a new flag style.

## Common commands

```bash
# Host prep (once per machine, needs root — auto-elevates)
sudo ./scripts/setup-host.sh
sudo ./scripts/setup-host.sh --verify        # re-check an already-prepared host, no changes
sudo ./scripts/setup-host.sh --user alice    # add a specific user to the docker group

# Activate the new docker-group membership in the current shell
newgrp docker

# Per-clone dev setup (must NOT be run as root)
./scripts/setup-dev.sh                       # configure .env/UID-GID + build + start + verify
./scripts/setup-dev.sh --no-up               # only write .env / UID-GID
./scripts/setup-dev.sh --verify              # only run health + endpoint checks
./scripts/setup-dev.sh --down                # stop and remove the stack (keeps volumes)
./scripts/setup-dev.sh --debug               # trace every command

# Quick syntax check when editing either script (no test suite/CI in this repo)
bash -n scripts/setup-host.sh
bash -n scripts/setup-dev.sh
```

There is no build, lint, or automated test tooling in this repo (no CI config, no
shellcheck invocation committed) — validating a script change means running it (or its
`--verify`/`--debug` mode) against a real or throwaway host/clone.
