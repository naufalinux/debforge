# 🐳 DebForge

**DebForge** is a container-first fullstack development environment, forged for Debian.
It gives a team one reproducible workflow — from a developer's laptop to production — built
entirely on Docker Engine + Compose v2, with **Debian 13 "trixie"** as the reference OS.

Four goals drive every decision documented here:

- ♻️ **Reproducible** — pinned, version-controlled, same result on any machine.
- 💻 **Works on my machine** — clone the repo, run one command, you're productive.
- 🚀 **Works in prod** — dev and prod build from the same images; only config differs.
- 🧩 **One team standard** — everyone (and every server) runs the same container recipe.

---

## 📚 Documentation

| Doc | Use it when... |
|---|---|
| [`docs/dev-environment-debian.md`](docs/dev-environment-debian.md) | Your whole team runs Debian — laptops and servers alike. **This is the reference edition.** |
| [`docs/dev-environment.md`](docs/dev-environment.md) | Your team is mixed Windows / macOS / Linux and needs a cross-platform onboarding path (Debian remains the reference *server* OS). |
| [`scripts/README.md`](scripts/README.md) | You want to run the setup automation rather than follow the docs by hand. |

---

## ⚡ Quickstart (Debian)

```bash
# 1. Prepare the host — once per machine (installs Docker CE, buildx, Compose v2)
sudo ./scripts/setup-host.sh

# 2. Activate your new docker-group membership in this shell
newgrp docker

# 3. Configure + build + start + verify your project's stack
./scripts/setup-dev.sh
```

See [`scripts/README.md`](scripts/README.md) for all options (`--verify`, `--down`,
`--no-up`, `--debug`, …), and [`docs/dev-environment-debian.md`](docs/dev-environment-debian.md)
for the full walkthrough — philosophy, Dockerfiles, Compose files, secrets handling,
daily workflow, reproducibility guardrails, and troubleshooting.

---

## 🗂️ Repository Structure

```text
DebForge/
├── README.md                          # you are here
├── LICENSE                            # GPL-3.0
├── .gitignore                         # keeps .env and local logs out of git
├── docs/
│   ├── dev-environment-debian.md      # 📖 reference: Debian-only edition
│   └── dev-environment.md             # 📖 cross-platform edition (Windows/macOS/Linux)
└── scripts/
    ├── README.md                      # automation usage, options, troubleshooting
    ├── setup-host.sh                  # 1️⃣ host prep — requires root, once per machine
    └── setup-dev.sh                   # 2️⃣ user + container prep — no root, once per clone
```

---

## 🤝 Contributing

This repo currently holds environment documentation and setup automation, not application
code. When you scaffold an actual project from these docs (Dockerfiles, Compose files,
`.env.example`, a `Makefile`), keep the same guardrails: pin images by digest, commit
lockfiles, never commit `.env`, and keep dev/prod parity intact.

---

## 📄 License

Licensed under the [GNU General Public License v3.0](LICENSE).
