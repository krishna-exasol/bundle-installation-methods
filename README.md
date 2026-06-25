<div align="center">

# 📦 Bundle Installation Methods

### Every practical way to install &amp; distribute a multi-component software bundle — with how each works, an example, and honest **pros &amp; cons**.

**[🌐 Open the interactive site →](https://krishna-exasol.github.io/bundle-installation-methods/)**

<sub>The site has the full comparison matrix, pros/cons cards for all 20 methods, and a featured breakdown of why major tools lead with <code>curl | sh</code> / <code>irm | iex</code>.</sub>

![Methods](https://img.shields.io/badge/methods-20-3fb950) &nbsp;
![Docs](https://img.shields.io/badge/docs-30%20files-58a6ff) &nbsp;
![Scope](https://img.shields.io/badge/scope-vendor--neutral-bc8cff) &nbsp;
![License](https://img.shields.io/badge/license-MIT-d29922)

</div>

---

A **bundle** is a product made of several parts — a database, a server, a CLI — that must end up working together on a user's machine. There is no single best way to ship it; there's a trade-off between **shipping fast with low friction** (script pipes, Docker) and **trust, polish &amp; discoverability** (signed installers, OS package managers). This repo catalogs every option so you can pick deliberately.

> Written while bundling Exasol components (Nano / Personal + MCP Server + JSON Tables) into a one-command install — that appears as a [case study](docs/case-studies/exasol-bundle.md), but the catalog applies to any multi-component product.

---

## 🚀 Start here

| | |
|---|---|
| 🌐 **Interactive site** | **<https://krishna-exasol.github.io/bundle-installation-methods/>** — matrix + pros/cons cards + the "why `irm`" feature |
| 🧭 **Just need to choose?** | [Decision Guide](docs/01-decision-guide.md) — a decision tree + per-scenario picks |
| 📊 **Comparing options?** | [Comparison Matrix](docs/02-comparison-matrix.md) — all 20 methods, scored side by side |
| 📚 **New to the problem?** | [Overview](docs/00-overview.md) — what "bundle install" means + the judging axes |

---

## ⭐ The headline: why everyone uses `curl \| sh` / `irm \| iex`

The most common "Get started" command in modern developer tools is a single line that pipes a hosted install script into the shell. `irm | iex` is just the **PowerShell-native equivalent** of `curl | sh`.

```bash
curl -fsSL https://example.com/install.sh | sh      # macOS / Linux
```
```powershell
irm https://example.com/install.ps1 | iex            # Windows PowerShell
```

Shipped this way by **Claude Code, OpenAI Codex, rustup, Homebrew, Deno, Bun, uv, Ollama, nvm, Tailscale, k3s, Docker** and many more — because it's **one line, zero prerequisites, cross-platform, adaptive, fully scriptable, and instant to publish**. Full analysis: **[Script pipe →](docs/methods/script-pipe.md)**

---

## 🗂️ The 20 methods

| # | Method | Platforms | One-liner | Best stage |
|---|--------|-----------|-----------|------------|
| 1 | [Script pipe (`curl\|sh`, `irm\|iex`)](docs/methods/script-pipe.md) | All | Pipe a hosted installer into the shell | **MVP** |
| 2 | [Download → verify → run](docs/methods/download-verify-run.md) | All | Same, but checksum-verified first | MVP+ |
| 3 | [Homebrew](docs/methods/homebrew.md) | macOS, Linux | `brew install` via a tap, later core | Growth → Mature |
| 4 | [Winget](docs/methods/winget.md) | Windows | Official Windows Package Manager | Mature |
| 5 | [Scoop](docs/methods/scoop.md) | Windows | No-admin developer installs | Growth |
| 6 | [Chocolatey](docs/methods/chocolatey.md) | Windows | Enterprise Windows automation | Growth → Mature |
| 7 | [Linux native (deb/rpm)](docs/methods/linux-native-packages.md) | Linux | `apt`/`dnf` from your signed repo | Mature |
| 8 | [Snap / Flatpak](docs/methods/snap-flatpak.md) | Linux | Sandboxed, self-contained apps | Growth → Mature |
| 9 | [pip / pipx / uvx](docs/methods/python-pip-pipx-uvx.md) | All | Install/run a Python entry point | MVP → Growth |
| 10 | [npm / npx](docs/methods/npm-npx.md) | All | Install/run a Node entry point | MVP → Growth |
| 11 | [cargo / go / gem](docs/methods/other-language-managers.md) | All | Native install for that ecosystem | Niche |
| 12 | [Docker run / pull](docs/methods/docker-run-pull.md) | All | A single container image | MVP |
| 13 | [Docker Compose](docs/methods/docker-compose.md) | All | Multi-container stack from a file | **MVP (multi)** |
| 14 | [Docker socket bootstrap](docs/methods/docker-socket-bootstrap.md) | All | A container that installs the stack | CI/demo only |
| 15 | [Helm / Kubernetes](docs/methods/helm-kubernetes.md) | K8s | `helm install` a chart | Mature (cloud) |
| 16 | [Native installers (.pkg/.msi/.deb/.rpm)](docs/methods/native-installers.md) | Per-OS | Double-click platform installers | Mature |
| 17 | [GitHub Releases (binaries)](docs/methods/github-releases-binaries.md) | All | Download a prebuilt asset | MVP → Growth |
| 18 | [Version managers (asdf/mise/proto)](docs/methods/version-managers.md) | All | Pluggable multi-tool managers | Niche (devs) |
| 19 | [IaC &amp; cloud marketplace](docs/methods/iac-cloud.md) | Cloud | Terraform/OpenTofu, AMIs, one-click | Mature (cloud) |
| 20 | [Source build](docs/methods/source-build.md) | All | `git clone &amp;&amp; make` | Fallback |

Scores and rankings: **[Comparison Matrix →](docs/02-comparison-matrix.md)**

---

## 🧩 Cross-cutting topics

- 🔐 [Security](docs/cross-cutting/security.md) — pipe-to-shell trust, checksums, code signing, supply-chain &amp; SBOM
- 🏷️ [Versioning &amp; pinning](docs/cross-cutting/versioning-pinning.md) — `latest` vs tags vs digests, reproducibility
- ♻️ [Idempotency &amp; uninstall](docs/cross-cutting/idempotency-uninstall.md) — re-runnable installs &amp; clean removal
- 🛣️ [Maturity roadmap](docs/cross-cutting/maturity-roadmap.md) — MVP → public → mature distribution path

## 📎 Case study &amp; glossary

- 🧪 [Exasol bundle](docs/case-studies/exasol-bundle.md) — why `curl | sh` + Docker Compose was chosen, the `pyexasol` conflict, and the host-launcher nuance
- 📖 [Glossary](docs/glossary.md) — idempotent, SBOM, digest, tap, notarization, …

---

## ✅ TL;DR recommendation

| You are… | Use |
|----------|-----|
| Shipping an MVP/demo fast, multi-component | **Script pipe** that drives **Docker Compose** |
| Hardening that MVP for early adopters | **Download → verify → run** with a **pinned release + SHA256** |
| Going mainstream on Windows | **Winget** (+ Scoop / Chocolatey as secondary) |
| Going mainstream on macOS/Linux | **Homebrew** (tap → core), native **deb/rpm** |
| Cloud / Kubernetes native | **Helm** and/or **IaC modules / marketplace images** |

The golden rule: **offer 2–3 channels, not 10** — one zero-friction path for everyone, one native path per OS you care about, and automate releases to all of them.

---

<div align="center">
<sub>Related pages · <a href="https://krishna-exasol.github.io/exasol-ai/installation-methods.html">Exasol AI install methods</a> · <a href="https://krishna-exasol.github.io/exasol-personal-ai/architecture.html">Exasol Personal AI architecture</a></sub>

<br><br>

License: <a href="LICENSE">MIT</a>

</div>
