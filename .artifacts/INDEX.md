# 📦 DebForge — Session Artifacts Export

Exported: 2026-07-23

This hidden directory is a **snapshot export** of everything produced in this workspace so
far, packaged as plain Markdown. It mirrors the live repo structure but is a separate,
static copy — editing files here has no effect on the real project. The live, canonical
files are `README.md`, `docs/`, and `scripts/` at the repo root; this directory exists so
the content can be reviewed, archived, or shared as pure Markdown outside the repo's
normal file layout.

## Contents

| Path | Mirrors | Notes |
|---|---|---|
| [`root/README.md`](root/README.md) | `../README.md` | Repo root README (project overview, quickstart, structure). |
| [`root/gitignore.md`](root/gitignore.md) | `../.gitignore` | Not Markdown natively — wrapped in a fenced code block. |
| [`docs/dev-environment-debian.md`](docs/dev-environment-debian.md) | `../docs/dev-environment-debian.md` | Reference edition: Debian-only environment doc. |
| [`docs/dev-environment.md`](docs/dev-environment.md) | `../docs/dev-environment.md` | Cross-platform edition (Windows/macOS/Linux). |
| [`scripts/README.md`](scripts/README.md) | `../scripts/README.md` | Setup-scripts usage guide. |
| [`scripts/setup-host.md`](scripts/setup-host.md) | `../scripts/setup-host.sh` | Root host-prep script, wrapped in a fenced `bash` block. |
| [`scripts/setup-dev.md`](scripts/setup-dev.md) | `../scripts/setup-dev.sh` | Non-root user/container-prep script, wrapped in a fenced `bash` block. |
| [`memory/INDEX.md`](memory/INDEX.md) | — | This workspace's persistent memory. **Currently empty** — no memory records exist. |
| [`response-history.md`](response-history.md) | — | Chronological export of the assistant's response text for every turn in this session, extracted from the session transcript. |

## Why this exists

Requested as a one-time export of the assistant-authored artifacts in this workspace
(the DebForge docs, scripts, and root README) plus any saved memory, consolidated into
plain Markdown under a dedicated hidden directory (`.artifacts/`) at the repo root.

## Not included

- `container-dev-env-doc-prompt.md` — the user's own original spec/prompt file, not an
  assistant-authored deliverable, so it wasn't copied into this export.
- Anything git-ignored (`.env`, `*.log`) — none existed at export time.

## Caveat: workspace directory name

At the time of this export, an attempt to rename the repo's root directory from
`dev-env` to `DebForge` did not persist — the platform reverted it back to
`/home/gnu/work/dev-env` (the session's bound working directory). All file *contents*
inside it (including the "DebForge" branding in the docs/README/scripts) are intact and
correct; only the top-level directory's name on disk stayed `dev-env`. This export was
generated from the current, real path.
