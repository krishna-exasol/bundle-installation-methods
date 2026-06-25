# Bundle Installation Methods

A comprehensive, vendor-neutral reference on **how to distribute and install a software bundle** — a product made of several components (a database, a server, a CLI, …) that a user should be able to set up in as few steps as possible.

It catalogs **every practical installation method**, with how each works, a runnable example, and an honest list of pros and cons, plus cross-cutting guidance on security, versioning, and choosing the right path as a project matures.

> Written while bundling Exasol components (Nano / Personal + MCP Server + JSON Tables) into a one-command install. The Exasol bundle appears as a running [case study](docs/case-studies/exasol-bundle.md), but the catalog itself applies to any multi-component product.

---

## How to read this

1. **New to the problem?** Start with the [Overview](docs/00-overview.md) — what "bundle install" means and the dimensions every method is judged on.
2. **Need to pick one?** Go to the [Decision Guide](docs/01-decision-guide.md) — a decision tree and per-scenario recommendations.
3. **Comparing options?** The [Comparison Matrix](docs/02-comparison-matrix.md) ranks all methods side by side.
4. **Researching a specific method?** Open its page under [`docs/methods/`](docs/methods).

---

## The methods at a glance

| # | Method | Platforms | One-liner | Best maturity stage |
|---|--------|-----------|-----------|---------------------|
| 1 | [Script pipe (`curl \| sh`, `irm \| iex`)](docs/methods/script-pipe.md) | All | Pipe a hosted installer into the shell | **MVP** |
| 2 | [Download → verify → run](docs/methods/download-verify-run.md) | All | Same, but checksum-verified before running | MVP (hardened) |
| 3 | [Homebrew](docs/methods/homebrew.md) | macOS, Linux | `brew install` via a tap, later core | Growth → Mature |
| 4 | [Winget](docs/methods/winget.md) | Windows | Official Windows Package Manager | Mature |
| 5 | [Scoop](docs/methods/scoop.md) | Windows | Developer-friendly Windows installs | Growth |
| 6 | [Chocolatey](docs/methods/chocolatey.md) | Windows | Enterprise Windows automation | Growth → Mature |
| 7 | [Linux native packages (deb/rpm)](docs/methods/linux-native-packages.md) | Linux | `apt`/`dnf` from your repo | Mature |
| 8 | [Snap / Flatpak](docs/methods/snap-flatpak.md) | Linux | Sandboxed, self-contained Linux apps | Growth → Mature |
| 9 | [Python: pip / pipx / uvx](docs/methods/python-pip-pipx-uvx.md) | All | Install/run a Python entry point | MVP → Growth |
| 10 | [npm / npx](docs/methods/npm-npx.md) | All | Install/run a Node entry point | MVP → Growth |
| 11 | [Other language managers (cargo/go/gem)](docs/methods/other-language-managers.md) | All | `cargo install`, `go install`, … | Niche |
| 12 | [Docker run / pull](docs/methods/docker-run-pull.md) | All | A single container image | MVP |
| 13 | [Docker Compose](docs/methods/docker-compose.md) | All | Multi-container stack from a compose file | **MVP (multi-component)** |
| 14 | [Docker socket bootstrap](docs/methods/docker-socket-bootstrap.md) | All | A container that installs the stack | CI/demo only |
| 15 | [Helm / Kubernetes](docs/methods/helm-kubernetes.md) | All (K8s) | `helm install` a chart | Mature (cloud) |
| 16 | [Native installers (.pkg/.msi/.deb/.rpm/GUI)](docs/methods/native-installers.md) | Per-OS | Double-click platform installers | Mature |
| 17 | [GitHub Releases (raw binaries)](docs/methods/github-releases-binaries.md) | All | Download a prebuilt binary/asset | MVP → Growth |
| 18 | [Version managers (asdf/mise/proto)](docs/methods/version-managers.md) | All | Pluggable multi-tool version managers | Niche (devs) |
| 19 | [IaC & cloud marketplace](docs/methods/iac-cloud.md) | Cloud | Terraform/OpenTofu, AMIs, one-click | Mature (cloud) |
| 20 | [Source build (clone + build)](docs/methods/source-build.md) | All | `git clone && make` | Fallback |

Full criteria and rankings are in the [Comparison Matrix](docs/02-comparison-matrix.md).

---

## Cross-cutting topics

- [Security](docs/cross-cutting/security.md) — pipe-to-shell trust, checksums, code signing, supply-chain & SBOM
- [Versioning & pinning](docs/cross-cutting/versioning-pinning.md) — `latest` vs tags vs digests, reproducibility
- [Idempotency & uninstall](docs/cross-cutting/idempotency-uninstall.md) — re-runnable installs and clean removal
- [Maturity roadmap](docs/cross-cutting/maturity-roadmap.md) — MVP → public → mature distribution path

## Case study

- [Exasol bundle](docs/case-studies/exasol-bundle.md) — why `curl | sh` + Docker Compose was chosen, the `pyexasol` conflict, and the host-launcher nuance for Exasol Personal

## Glossary

- [Glossary](docs/glossary.md) — terms used throughout (idempotent, SBOM, tap, digest, …)

---

## TL;DR recommendation

| You are… | Use |
|----------|-----|
| Shipping an MVP/demo fast, multi-component | **Script pipe** that drives **Docker Compose** |
| Hardening that MVP for early adopters | **Download → verify → run** with a **pinned release + SHA256** |
| Going mainstream on Windows | **Winget** (+ Scoop/Chocolatey as secondary) |
| Going mainstream on macOS/Linux | **Homebrew** (tap → core), native **deb/rpm** |
| Cloud / Kubernetes native | **Helm** and/or **IaC modules / marketplace images** |

License: [MIT](LICENSE).
