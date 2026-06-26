# Bundle Installation Methods

Every practical way to install and distribute a multi-component software **bundle** — a product made of several parts (a database, a server, a CLI) that must end up working together on a user's machine. Each method here comes with **how it works, an example, and honest pros &amp; cons**.

There is no single best method — there's a trade-off between **shipping fast with low friction** (script pipes, Docker) and **trust, polish &amp; discoverability** (signed installers, OS package managers). Use this site to pick deliberately.

[Compare all methods :material-table:](02-comparison-matrix.md){ .md-button .md-button--primary }
[How to choose :material-sign-direction:](01-decision-guide.md){ .md-button }

---

## ⭐ Why major players lead with `curl | sh` and `irm | iex`

!!! tip "The one-liner pattern"
    The most common "Get started" command in modern developer tools is a single line that pipes a hosted install script into the shell. `irm | iex` is just the **PowerShell-native equivalent** of `curl | sh` — `Invoke-RestMethod` downloads, `Invoke-Expression` runs — and PowerShell ships in Windows, so it needs nothing else.

=== "macOS / Linux"

    ```bash
    curl -fsSL https://example.com/install.sh | sh
    ```

=== "Windows PowerShell"

    ```powershell
    irm https://example.com/install.ps1 | iex
    ```

Shipped this way by **Claude Code, OpenAI Codex, rustup, Homebrew, Deno, Bun, uv, Ollama, nvm, Tailscale, k3s, Docker** and many more.

| Tool | Get-started command (abridged) |
|------|--------------------------------|
| Claude Code | `curl -fsSL https://claude.ai/install.sh \| bash` · `irm https://claude.ai/install.ps1 \| iex` |
| OpenAI Codex CLI | one-line script / `npm` bootstrap — same pattern |
| Rust (rustup) | `curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs \| sh` |
| Homebrew | `/bin/bash -c "$(curl -fsSL …/install.sh)"` |
| Deno | `curl -fsSL https://deno.land/install.sh \| sh` |
| Bun | `curl -fsSL https://bun.sh/install \| bash` |
| uv (Astral) | `curl -LsSf https://astral.sh/uv/install.sh \| sh` |
| Ollama | `curl -fsSL https://ollama.com/install.sh \| sh` |

**Why they converge on it:** one line, zero prerequisites · cross-platform parity · adaptive to OS/arch · does the whole job (PATH, services, config, multi-component startup) · ships &amp; updates instantly · trivial in CI. The honest trade-off is **trust** — the user runs code they haven't read — which the [verified variant](methods/download-verify-run.md) and [security guide](cross-cutting/security.md) address.

[Read the full breakdown :material-arrow-right:](methods/script-pipe.md){ .md-button }

---

## Explore

<div class="grid cards" markdown>

-   :material-rocket-launch: **Start fast (MVP)**

    A [script pipe](methods/script-pipe.md) that drives [Docker Compose](methods/docker-compose.md) — zero registry, cross-platform, full setup logic.

-   :material-shield-check: **Harden it**

    [Download → verify → run](methods/download-verify-run.md) with a pinned release + SHA256, plus the [security](cross-cutting/security.md) checklist.

-   :material-microsoft-windows: **Windows users**

    [Winget](methods/winget.md), [Scoop](methods/scoop.md), [Chocolatey](methods/chocolatey.md).

-   :material-apple: **macOS / Linux users**

    [Homebrew](methods/homebrew.md), native [deb/rpm](methods/linux-native-packages.md), [Snap/Flatpak](methods/snap-flatpak.md).

-   :material-language-python: **Language ecosystems**

    [pip / pipx / uvx](methods/python-pip-pipx-uvx.md), [npm / npx](methods/npm-npx.md), [cargo / go / gem](methods/other-language-managers.md).

-   :material-kubernetes: **Cloud / Kubernetes**

    [Helm](methods/helm-kubernetes.md) and [IaC / marketplace images](methods/iac-cloud.md).

</div>

---

## The 20 methods

| # | Method | Platforms | Best stage |
|---|--------|-----------|------------|
| 1 | [Script pipe (`curl\|sh`, `irm\|iex`)](methods/script-pipe.md) | All | **MVP** |
| 2 | [Download → verify → run](methods/download-verify-run.md) | All | MVP+ |
| 3 | [Homebrew](methods/homebrew.md) | macOS, Linux | Growth → Mature |
| 4 | [Winget](methods/winget.md) | Windows | Mature |
| 5 | [Scoop](methods/scoop.md) | Windows | Growth |
| 6 | [Chocolatey](methods/chocolatey.md) | Windows | Growth → Mature |
| 7 | [Linux native (deb/rpm)](methods/linux-native-packages.md) | Linux | Mature |
| 8 | [Snap / Flatpak](methods/snap-flatpak.md) | Linux | Growth → Mature |
| 9 | [pip / pipx / uvx](methods/python-pip-pipx-uvx.md) | All | MVP → Growth |
| 10 | [npm / npx](methods/npm-npx.md) | All | MVP → Growth |
| 11 | [cargo / go / gem](methods/other-language-managers.md) | All | Niche |
| 12 | [Docker run / pull](methods/docker-run-pull.md) | All | MVP |
| 13 | [Docker Compose](methods/docker-compose.md) | All | **MVP (multi)** |
| 14 | [Docker socket bootstrap](methods/docker-socket-bootstrap.md) | All | CI/demo only |
| 15 | [Helm / Kubernetes](methods/helm-kubernetes.md) | K8s | Mature (cloud) |
| 16 | [Native installers](methods/native-installers.md) | Per-OS | Mature |
| 17 | [GitHub Releases (binaries)](methods/github-releases-binaries.md) | All | MVP → Growth |
| 18 | [Version managers (asdf/mise/proto)](methods/version-managers.md) | All | Niche (devs) |
| 19 | [IaC & cloud marketplace](methods/iac-cloud.md) | Cloud | Mature (cloud) |
| 20 | [Source build](methods/source-build.md) | All | Fallback |

See the full scored [Comparison matrix](02-comparison-matrix.md), then the [Decision guide](01-decision-guide.md).

---

!!! note "Case study"
    This catalog grew out of bundling Exasol components (Nano / Personal + MCP Server + JSON Tables) into a one-command install. See [why `curl | sh` + Docker Compose was chosen](case-studies/exasol-ai.md).
