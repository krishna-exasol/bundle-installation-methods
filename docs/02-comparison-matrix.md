# Comparison Matrix

All methods, side by side. Scores are **1–5** (5 = best on that axis) and are judged for the common case of distributing a **multi-component product** to a broad audience. Your weighting will differ by audience and stage — see the [Decision Guide](01-decision-guide.md).

> **Friction** = fewer steps/prereqs is higher. **Trust** = built-in integrity/signing is higher. **Orchestr.** = how much setup logic it can run. **Ship-now** = how fast *you* can publish today. **Low-maint.** = less ongoing upkeep per release.

## Master table

| # | Method | Platforms | Friction | Prereqs | Orchestr. | Trust | Ship-now | Low-maint. | Best stage |
|---|--------|-----------|:-:|:-:|:-:|:-:|:-:|:-:|------------|
| 1 | [Script pipe](methods/script-pipe.md) | All | 5 | 5 | 5 | 2 | 5 | 4 | **MVP** |
| 2 | [Download → verify → run](methods/download-verify-run.md) | All | 3 | 5 | 5 | 4 | 5 | 3 | MVP+ |
| 3 | [Homebrew](methods/homebrew.md) | mac/Linux | 4 | 3 | 3 | 4 | 3 | 3 | Growth→Mature |
| 4 | [Winget](methods/winget.md) | Windows | 4 | 4 | 3 | 5 | 2 | 3 | Mature |
| 5 | [Scoop](methods/scoop.md) | Windows | 4 | 3 | 3 | 3 | 4 | 3 | Growth |
| 6 | [Chocolatey](methods/chocolatey.md) | Windows | 4 | 3 | 4 | 4 | 3 | 3 | Growth→Mature |
| 7 | [Linux native (deb/rpm)](methods/linux-native-packages.md) | Linux | 4 | 4 | 4 | 5 | 2 | 2 | Mature |
| 8 | [Snap / Flatpak](methods/snap-flatpak.md) | Linux | 4 | 3 | 3 | 4 | 3 | 3 | Growth→Mature |
| 9 | [pip / pipx / uvx](methods/python-pip-pipx-uvx.md) | All | 4 | 3 | 2 | 3 | 5 | 4 | MVP→Growth |
| 10 | [npm / npx](methods/npm-npx.md) | All | 4 | 3 | 2 | 3 | 5 | 4 | MVP→Growth |
| 11 | [cargo / go / gem](methods/other-language-managers.md) | All | 3 | 2 | 2 | 3 | 5 | 4 | Niche |
| 12 | [Docker run / pull](methods/docker-run-pull.md) | All | 4 | 3 | 2 | 4 | 5 | 4 | MVP (single) |
| 13 | [Docker Compose](methods/docker-compose.md) | All | 4 | 3 | 4 | 4 | 5 | 4 | **MVP (multi)** |
| 14 | [Docker socket bootstrap](methods/docker-socket-bootstrap.md) | All | 4 | 3 | 5 | 1 | 5 | 3 | CI/demo only |
| 15 | [Helm / Kubernetes](methods/helm-kubernetes.md) | K8s | 2 | 1 | 5 | 4 | 3 | 3 | Mature (cloud) |
| 16 | [Native installers](methods/native-installers.md) | Per-OS | 4 | 5 | 4 | 5 | 1 | 2 | Mature |
| 17 | [GitHub Releases (binaries)](methods/github-releases-binaries.md) | All | 3 | 4 | 2 | 4 | 5 | 4 | MVP→Growth |
| 18 | [Version managers (asdf/mise)](methods/version-managers.md) | All | 3 | 2 | 2 | 3 | 4 | 3 | Niche (devs) |
| 19 | [IaC & cloud marketplace](methods/iac-cloud.md) | Cloud | 2 | 1 | 5 | 5 | 2 | 2 | Mature (cloud) |
| 20 | [Source build](methods/source-build.md) | All | 1 | 1 | 4 | 3 | 5 | 3 | Fallback |

## One-line pros / cons

| Method | Biggest pro | Biggest con |
|--------|-------------|-------------|
| [Script pipe](methods/script-pipe.md) | One line, no prereqs, full control | Runs unseen code (trust) |
| [Download → verify → run](methods/download-verify-run.md) | Same power + integrity check | More steps; still runs a script |
| [Homebrew](methods/homebrew.md) | Beloved by mac/Linux devs; clean upgrades | Tap upkeep; core review bar is high |
| [Winget](methods/winget.md) | In-box, trusted, managed updates | Manifest PR review + signed installer |
| [Scoop](methods/scoop.md) | No-admin dev installs; easy own bucket | Niche; portable-app conventions |
| [Chocolatey](methods/chocolatey.md) | Enterprise Windows automation | Community moderation; PowerShell upkeep |
| [Linux native (deb/rpm)](methods/linux-native-packages.md) | Native, signed, system-integrated | Per-distro packaging + repo hosting |
| [Snap / Flatpak](methods/snap-flatpak.md) | Sandboxed, self-contained, auto-update | Confinement friction; bigger artifacts |
| [pip/pipx/uvx](methods/python-pip-pipx-uvx.md) | Trivial for Python devs; uvx ephemeral | Only the Python part; needs Python/uv |
| [npm/npx](methods/npm-npx.md) | Trivial for Node devs; npx zero-install | Only the Node part; needs Node |
| [cargo/go/gem](methods/other-language-managers.md) | Native for that ecosystem | Builds from source; one language only |
| [Docker run/pull](methods/docker-run-pull.md) | Reproducible, isolated, one image | Single container ≠ a whole bundle |
| [Docker Compose](methods/docker-compose.md) | Real multi-service local stack | Needs a wrapper for host checks/config |
| [Docker socket bootstrap](methods/docker-socket-bootstrap.md) | Portable "do everything" container | Socket mount = root on host ⚠️ |
| [Helm/Kubernetes](methods/helm-kubernetes.md) | K8s-native, declarative, scalable | Assumes a cluster; overkill for laptops |
| [Native installers](methods/native-installers.md) | Most trusted, double-click familiar | Per-OS packaging + signing certs |
| [GitHub Releases](methods/github-releases-binaries.md) | Pinned, signed assets; CI-friendly | Just files — no orchestration alone |
| [Version managers](methods/version-managers.md) | Great for devs juggling versions | Needs the manager; not for end users |
| [IaC & cloud](methods/iac-cloud.md) | Declarative cloud deploys at scale | Cloud-only; heavy for local use |
| [Source build](methods/source-build.md) | Universal fallback; full transparency | Needs full toolchains; slowest |

## How to use these scores

1. Decide your **audience** (end users vs developers vs ops) and **stage** (MVP vs mature).
2. Re-weight the columns: an MVP weights *Friction / Ship-now / Orchestration*; a mature consumer product weights *Trust / Low-maint. / discoverability*.
3. Pick the top **primary**, then add **one native path per OS** you care about. See the [Decision Guide](01-decision-guide.md) and [Maturity roadmap](cross-cutting/maturity-roadmap.md).

For the worked example behind the bundle's choice (script pipe → Docker Compose), see the [Exasol case study](case-studies/exasol-ai.md).
