# Homebrew (custom tap & homebrew-core)

> **One-liner:** Distribute the bundle as a Homebrew formula — either from your own GitHub-hosted "tap" or, after a high review bar, from the official homebrew-core repository.

| | |
|---|---|
| **Category** | OS-level package manager (third-party) |
| **Platforms** | macOS (Intel & Apple Silicon), Linux (Homebrew on Linux / "Linuxbrew") |
| **Prerequisites** | Homebrew installed; Ruby (vendored with brew); a GitHub repo for a tap; for core: a notable, stable, releasable project |
| **Handles** | acquire ✓ / verify ✓ (sha256 + optional GPG) / place ✓ / configure ? / start ? (via `brew services`) / update ✓ / uninstall ✓ |
| **Maturity fit** | Growth (custom tap) → Mature (homebrew-core) |
| **Trust model** | Tap: you sign nothing centrally; users trust your GitHub org. Core: vetted by Homebrew maintainers, served over Homebrew's CDN/bottles. |

## How it works
A Homebrew **formula** is a Ruby class that declares a source URL, a `sha256` checksum, dependencies, and an `install` method. Formulae live in **taps** — a tap is simply a GitHub repository named `homebrew-<name>` under any org/user. Running `brew tap org/name` adds that repo as a formula source; `brew install org/name/foo` then installs the `foo` formula from it. A **custom tap** is the fast path: you control it, you can ship immediately, and you can include "unusual" dependencies.

**homebrew-core** is the default tap that ships with Homebrew, so its formulae install with a bare `brew install foo` (no tap prefix). Getting in is a curation gate, not a technical one: the project must be **notable** (broadly: not too niche, with real usage), have a **stable tagged/versioned release** (no `HEAD`-only, no arbitrary git snapshots), pass an automated **`brew test`** block, build from source where feasible, and avoid dependencies Homebrew won't accept (e.g. forks, vendored copies of existing formulae, or closed binaries when an open path exists). Once merged, Homebrew CI builds **bottles** (precompiled binary tarballs) per OS/arch and serves them from its CDN, so most users never compile.

For a multi-component **bundle**, the honest caveat: a formula installs *one logical package tree*. The pip MCP server and the Python+Rust `json-tables` CLI map cleanly to formula dependencies (a `resource` for pip wheels, a Rust build via `depends_on "rust" => :build`). The Exasol Personal/Nano database, however, is a containerized product — Homebrew can install a launcher/wrapper and even declare `depends_on "docker"`, but it cannot run the container engine for the user. So the formula realistically ships the CLI tooling and a wrapper script that pulls and starts the DB image; Docker (or Colima/Podman) remains a runtime prerequisite.

## Example
```bash
# --- Custom tap: consumer side ---
brew tap exasol-bundle/tap          # adds github.com/exasol-bundle/homebrew-tap
brew install exasol-bundle/tap/exa-bundle
brew services start exa-bundle      # optional: run the launcher as a managed service

# --- Tap repo layout ---
# github.com/exasol-bundle/homebrew-tap
#   Formula/exa-bundle.rb

# --- Formula/exa-bundle.rb skeleton ---
class ExaBundle < Formula
  desc "Exasol Nano + MCP server + json-tables CLI launcher"
  homepage "https://github.com/exasol-bundle/exa-bundle"
  url "https://github.com/exasol-bundle/exa-bundle/archive/refs/tags/v1.2.0.tar.gz"
  sha256 "PUT_REAL_SHA256_HERE"        # brew fetch / shasum -a 256
  license "MIT"

  depends_on "python@3.12"
  depends_on "rust" => :build          # builds the json-tables Rust CLI
  depends_on "docker"                  # runtime engine for the Nano DB image

  # pip-installed MCP server pinned as a vendored resource
  resource "exasol-mcp-server" do
    url "https://files.pythonhosted.org/.../exasol_mcp_server-1.10.1.tar.gz"
    sha256 "PUT_REAL_SHA256_HERE"
  end

  def install
    system "cargo", "install", *std_cargo_args(path: "json-tables")
    venv = virtualenv_create(libexec/"venv", "python3.12")
    venv.pip_install resources                 # installs the MCP server wheel
    (bin/"exa-bundle").write_env_script libexec/"bin/exa-bundle", PATH: "#{libexec}/venv/bin:$PATH"
  end

  service do                                   # `brew services` integration
    run [opt_bin/"exa-bundle", "serve"]
    keep_alive true
  end

  test do
    assert_match "exa-bundle", shell_output("#{bin}/exa-bundle --version")
  end
end

# --- The bar to enter homebrew-core (consumer then just runs `brew install exa-bundle`) ---
brew bump-formula-pr   # tooling for version bumps once accepted
# Submit a PR to github.com/Homebrew/homebrew-core adding Formula/exa-bundle.rb.
# Reviewers require: notability, a stable tagged release, passing `brew test`,
# no unsupported/duplicate deps, build-from-source friendliness, and clean `brew audit --strict --new`.
```

## Pros
- One short command for users; `brew install`/`upgrade`/`uninstall` are universally known on macOS.
- Automatic checksum verification on every download; bottles served over a trusted CDN (for core).
- Built-in upgrade path (`brew upgrade`) and service management (`brew services`).
- A custom tap requires **zero approval** — ship the same day.
- Native to both macOS and Linux developer machines.

## Cons
- Not a system default — users must have Homebrew installed first.
- A formula installs one artifact tree; a multi-container bundle still needs Docker/Podman underneath, which Homebrew can declare but not operate.
- homebrew-core acceptance is slow and gated on notability — niche/internal tools are routinely declined.
- Custom taps get **no central trust signal**; users implicitly trust your GitHub org.
- Ruby DSL plus per-OS/arch bottle building adds maintenance overhead.

## Security considerations
Formulae pin a `sha256` for each downloaded archive, so tampering en route is detected. Trust still flows from *where the URL points* and *who controls the tap*. A custom tap is only as trustworthy as your GitHub org — there is no Homebrew-side review. Prefer release artifacts hosted on a stable, HTTPS endpoint (GitHub releases, PyPI). Avoid `curl | bash` inside `install`. For the DB image, pin the container by digest, not a floating tag. See [Security](../cross-cutting/security.md).

## Approval & maintenance burden
- **Custom tap:** low approval (none), moderate upkeep — bump `url`/`sha256` per release, keep deps current.
- **homebrew-core:** high approval bar (maintainer review, notability, `brew audit --strict`, passing tests), lower long-term upkeep since Homebrew CI builds and hosts bottles. Expect review iterations and possible rejection for niche tools.

## Best for / Avoid when
**Best for:** developer-facing CLI tooling on macOS/Linux; teams wanting a clean upgrade path; projects popular enough to eventually land in core. **Avoid when:** your audience is Windows-first; the product is fundamentally a multi-container service where Homebrew only wraps a launcher; or you need a system-default package on servers (use native packages instead).

## Real-world examples
- `awscli`, `gh`, `node`, `ripgrep` — homebrew-core formulae.
- HashiCorp, Heroku, and many vendors ship **custom taps** (`hashicorp/tap`, `heroku/brew`) for products not in core.
- Docker Desktop is distributed as a Homebrew **cask** (a related mechanism for GUI apps/binaries).

## See also
- [Comparison matrix](../02-comparison-matrix.md)
- [Linux native packages (.deb / .rpm)](./linux-native-packages.md)
- [Snap & Flatpak](./snap-flatpak.md)
- [Security](../cross-cutting/security.md)
- [Versioning & pinning](../cross-cutting/versioning-pinning.md)
