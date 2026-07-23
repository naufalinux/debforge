# Prompt: Generate "Container-First Fullstack Dev Environment" Documentation

> Copy everything below the line into your AI coding agent. Fill in the `{{PLACEHOLDERS}}` first (or leave them — the agent has sensible defaults).

---

## ROLE

You are a **senior platform / DevOps engineer and technical writer**. You write documentation that a mixed-skill team actually follows without getting stuck. You favor clarity, correctness, and reproducibility over cleverness.

## OBJECTIVE

Produce **one self-contained Markdown document** that walks a cross-platform team through setting up a **container-first fullstack web development environment**, where **Debian 13 "trixie" (stable)** is the reference host/server OS.

The environment must satisfy four goals, and you should reinforce them throughout:
1. 🌐 **Cross-platform dev team standard** — identical workflow on Windows, macOS, and Linux.
2. ♻️ **Reproducible** — anyone, any machine, same result; pinned and version-controlled.
3. 💻 **"Works on my machine"** — a new hire is productive from a single clone + one command.
4. 🚀 **"Works in prod"** — dev/prod parity so nothing surprising happens at deploy time.

## REFERENCE STACK (concrete but swappable)

Use this as the illustrative stack unless placeholders say otherwise. Clearly mark every stack-specific part as **swappable** so teams can substitute their own:
- Frontend: `{{FRONTEND := Vite + React + TypeScript}}`
- Backend/API: `{{BACKEND := Node.js + TypeScript (Fastify/Express)}}`
- Database: `{{DB := PostgreSQL}}`
- Reverse proxy / edge: `{{PROXY := Caddy (or Traefik/nginx)}}`
- Orchestration (local): Docker Engine + Docker **Compose v2**

Keep all *principles* stack-agnostic; keep all *commands* concrete and runnable.

## HARD REQUIREMENTS

- Container-first: **everything runs in containers**; the host installs only Docker + Git + an editor. No language runtimes on the host.
- Debian trixie host setup must install **Docker CE from Docker's official APT repository** (not the distro `docker.io` package), enable **BuildKit/buildx**, add the user to the `docker` group, and note **cgroup v2** and **iptables-nft** realities on trixie.
- Cross-platform onboarding paths for:
  - 🪟 **Windows** via **WSL2** (Debian distro inside WSL) — stress keeping the repo *inside* the Linux filesystem, not `/mnt/c/...`, for performance and correct file events.
  - 🍎 **macOS** via Docker Desktop **or** Colima/Rancher Desktop — note bind-mount performance and VirtioFS.
  - 🐧 **Linux** native.
- Reproducibility guardrails: pin base images **by digest**, commit **lockfiles**, ship a `.dockerignore`, use **multi-stage builds**, run as a **non-root user**, and standardize line endings via `.gitattributes` (`* text=auto eol=lf`).
- Dev/prod parity: shared base images, `docker-compose.yml` + `docker-compose.override.yml` (dev) + a prod compose/target, **12-factor** config via env vars, `.env.example` committed and real `.env` git-ignored, and a note on **secrets** (never commit; use env/secret managers).
- Dev ergonomics: hot reload via bind mounts, `depends_on` + **healthchecks**, `docker compose exec`, log tailing, and a **task runner** (`Makefile` or `just`) that wraps common commands.

## REQUIRED STRUCTURE

Produce these sections in order. Start with a Markdown **table of contents** (anchor links).

1. **📖 Overview & Philosophy** — why container-first; how it delivers the 4 goals; a one-diagram mental model (ASCII/Mermaid) of frontend ↔ api ↔ db ↔ proxy.
2. **⚡ Quickstart (TL;DR)** — the impatient path: clone → `cp .env.example .env` → `make up` → open URL. Put this near the top.
3. **✅ Prerequisites** — what each OS needs before starting.
4. **🐧 Host Setup on Debian trixie** — exact APT commands for Docker CE, buildx, group, verification.
5. **🖥️ Cross-Platform Onboarding** — separate, clearly-labeled subsections for 🪟 / 🍎 / 🐧.
6. **🗂️ Project Structure** — annotated repo tree.
7. **📦 Dockerfiles** — multi-stage, non-root, pinned; separate dev vs prod targets.
8. **🧩 Compose Files** — base + dev override + prod; explain override merge behavior.
9. **🔐 Environment & Secrets** — `.env` strategy, `.env.example`, config precedence, secret handling.
10. **🔁 Daily Dev Workflow** — up/down, logs, exec, rebuild, running migrations/tests inside containers, hot reload.
11. **♻️ Reproducibility Guardrails** — digests, lockfiles, `.dockerignore`, `.gitattributes`, BuildKit cache.
12. **🚀 Dev/Prod Parity & Deployment** — build once/promote, healthchecks, resource limits, what changes between dev and prod.
13. **🛠️ Task Runner Reference** — the `Makefile`/`justfile` with each target explained.
14. **🧯 Troubleshooting** — table of symptom → cause → fix (see gotchas below).
15. **📋 New-Hire Checklist** — copy-paste checkbox list to reach "productive" state.

## GOTCHAS YOU MUST EXPLICITLY COVER

Bake these into Setup and Troubleshooting (with the fix, not just the warning):
- 🔑 **UID/GID mismatch** on bind mounts → files owned by root; solve with a build-arg UID/GID or matching non-root user.
- ↩️ **Line endings (CRLF/LF)** breaking shell scripts on Windows → `.gitattributes` + note on editor config.
- 🐌 **Slow bind mounts / broken file-watching** on macOS and Windows-`/mnt/c` → keep code in the native/WSL FS; VirtioFS/`:cached`.
- 🔌 **Port conflicts** and how to remap.
- 🏷️ **Never use `latest` tags** → pin by tag+digest.
- ⏰ **WSL2 clock drift** and Docker context selection.
- 🧱 Debian trixie specifics: official Docker repo vs `docker.io`, cgroup v2, iptables-nft.

## STYLE & QUALITY BAR

- ✍️ Tone: friendly, precise, encouraging — like a great onboarding buddy. Explain the **why** in one or two sentences before the **how**.
- 😀 Emojis: use them as **visual anchors** on headings and callouts, and a consistent legend: 💡 tip · ⚠️ warning · 🪟 Windows · 🍎 macOS · 🐧 Linux · ✅ verify. Don't overdo it inside prose.
- 🧾 Every command block: fenced with a language tag, copy-pasteable, **idempotent where possible**, and followed by a short **✅ verify** step (expected output or a check command).
- 📐 Use proper heading hierarchy, tables for comparisons/troubleshooting, and Mermaid or ASCII for diagrams.
- 🔤 Define jargon on first use; assume a competent dev who is **new to containers**.
- Keep it **complete and runnable** — a reader following top to bottom ends with a working environment. Prefer real, correct commands over pseudo-code.

## CONSTRAINTS (what NOT to do)

- ❌ No deprecated `docker-compose` (v1, hyphenated) syntax — use `docker compose`.
- ❌ No host-installed language runtimes in the workflow.
- ❌ No `latest` / unpinned images in examples.
- ❌ No committing real secrets; `.env` must be git-ignored.
- ❌ Don't assume Docker Desktop on Linux; native Engine is the reference.

## OUTPUT FORMAT

Output **only the final Markdown document** — no preamble, no explanation of your choices, no surrounding commentary. It should be ready to save as `README.md` / `docs/dev-environment.md` and committed as-is. If you make an assumption to fill a placeholder, note it inline in the doc as a `> 💡 Note:` callout rather than asking me.
