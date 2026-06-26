# Script pipe (`curl | sh`, `irm | iex`)

> **One-liner:** Host an install script at a URL and pipe it straight into the shell, so the whole setup is a single copy-paste line on any machine.

| | |
|---|---|
| **Category** | Script installer |
| **Platforms** | All — `curl … \| sh` (macOS/Linux), `irm … \| iex` (Windows PowerShell) |
| **Prerequisites** | A shell (always present). No package manager, runtime, or registry account. |
| **Handles** | acquire ✓ · verify ⚠️ (only if you add checksums) · place ✓ · configure ✓ · start ✓ · update ✓ (re-run) · uninstall ✓ (if you ship one) |
| **Maturity fit** | **MVP** (and kept forever as the zero-friction path) |
| **Trust model** | The user runs code they have not read, fetched over HTTPS from a domain you control. Trust = your domain + TLS. |

This is the method most modern CLIs lead with, and the one chosen for the [Exasol bundle](../case-studies/exasol-ai.md).

---

## How it works

`curl`/`irm` downloads a script from a URL; the shell executes it from stdin without ever writing it to disk:

- **macOS / Linux:** `curl -fsSL <url> | sh` — `curl` streams the script, `sh` runs it.
- **Windows:** `irm <url> | iex` — `irm` (`Invoke-RestMethod`) downloads, `iex` (`Invoke-Expression`) runs it. PowerShell is in-box on Windows, so nothing else is needed.

The script then does whatever it wants on the host: detect OS/arch/libc, download the right artifact (a binary, image, or in our case start a Docker Compose stack), set up PATH, write config, register a service, print next steps. **It is a full program, not just a download** — that is the whole point.

```
 user types one line
        │
   curl / irm  ──HTTPS──►  your-domain.com/install.sh   (a script you control)
        │
       sh / iex  ──►  detect platform · fetch artifacts · configure · start · report
```

---

## Example

The Exasol bundle ships exactly this:

```bash
# macOS / Linux
curl -fsSL https://raw.githubusercontent.com/krishna-exasol/exasol-personal-ai/main/install.sh | sh
```
```powershell
# Windows PowerShell
irm https://raw.githubusercontent.com/krishna-exasol/exasol-ai/main/install.ps1 | iex
```

A minimal, well-behaved installer skeleton:

```sh
#!/usr/bin/env sh
set -eu                                   # fail fast
need() { command -v "$1" >/dev/null || { echo "need $1"; exit 1; }; }
need docker
os="$(uname -s)"; arch="$(uname -m)"      # adapt to the platform
# … download the right artifact / start the stack …
docker compose -f "$dir/compose.yaml" up -d --build
echo "Installed. Next: …"
```

> The flags matter: `curl -fsSL` = **f**ail on HTTP errors, **s**ilent, **S**how real errors, **L** follow redirects. Without `-f`, a 404 HTML page can get piped into your shell.

---

## Why the major players use it

This pattern is the default "Get started" command for a striking share of modern developer tools:

| Tool | Command (abridged) |
|------|--------------------|
| **Claude Code** | `curl -fsSL https://claude.ai/install.sh \| bash` · `irm https://claude.ai/install.ps1 \| iex` |
| **OpenAI Codex CLI** | script/`npm` bootstrap, same one-line pattern |
| **Rust (rustup)** | `curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs \| sh` |
| **Homebrew** | `/bin/bash -c "$(curl -fsSL …/install.sh)"` |
| **Deno** | `curl -fsSL https://deno.land/install.sh \| sh` · `irm https://deno.land/install.ps1 \| iex` |
| **Bun** | `curl -fsSL https://bun.sh/install \| bash` · `irm bun.sh/install.ps1 \| iex` |
| **uv / Astral** | `curl -LsSf https://astral.sh/uv/install.sh \| sh` · `irm https://astral.sh/uv/install.ps1 \| iex` |
| **Ollama** | `curl -fsSL https://ollama.com/install.sh \| sh` |
| **nvm**, **Starship**, **pnpm**, **Tailscale**, **k3s**, **Docker**, **Oh My Zsh** | all ship a `curl \| sh` (and usually an `irm \| iex`) one-liner |

They converge on it for concrete reasons:

1. **One command, any machine — zero prerequisites.** The user does not need to first install a package manager, a runtime, or sign into a registry. A shell is always there. This is the single biggest driver of "time to first success," which is what a landing page optimizes for.
2. **Cross-platform parity.** A `*.sh` and a `*.ps1` cover Windows, macOS, and Linux with the same UX. `irm | iex` is simply the **PowerShell-native equivalent** of `curl | sh` — `Invoke-RestMethod` downloads, `Invoke-Expression` runs — and PowerShell ships in Windows, so it needs nothing extra.
3. **The script is adaptive.** It detects OS, CPU arch (`x86_64`/`arm64`), and libc, then fetches exactly the right artifact. One URL serves everyone; the logic lives server-side in a file you own.
4. **It can do the whole job.** PATH edits, shell completions, service/daemon registration, prerequisite checks, config files, multi-component startup, "what to do next" output — none of which a bare package download or `docker run` can do.
5. **Instant publish and update — full control.** No store review, no manifest PR, no signing pipeline before you can ship. You push the script and it is live. You own the URL, the branding, and the install experience end to end.
6. **Trivial in CI / Dockerfiles / provisioning.** One line drops into a `RUN`, a CI step, or a setup doc.

In short: it is the **lowest-friction, fastest-to-ship, fully-controllable** install path — which is exactly why it wins for MVPs and stays as the universal fallback even after package managers are added.

---

## Pros

- **Lowest possible friction** — one copy-paste line, no prerequisites.
- **Cross-platform** with one script per shell family.
- **Adaptive** to OS/arch at runtime.
- **Full orchestration** — checks, config, startup, helpers, uninstall.
- **Ship and update instantly**, no third-party approval.
- **You own the funnel** — single branded URL, easy to document.

## Cons

- **Trust:** the user runs code they have not read. This is the headline objection (see [Security](../cross-cutting/security.md)).
- **No built-in integrity check** unless you add one (hence [download → verify → run](download-verify-run.md)).
- **Network required** at install time; corporate proxies/AV can interfere.
- **Idempotency, upgrade, and uninstall are your responsibility** — a package manager gives these for free.
- **Harder to audit/pin** than a versioned package; pointing at `main` means the command's behavior can change under users.
- **Not discoverable** — users won't `search` for it like a package; they must hit your docs.

## Security considerations

- Always serve over **HTTPS** from a domain you control; use `curl -fsSL` (the `-f` prevents an error page being executed).
- Offer a **read-before-run** form: `curl -fsSL <url> -o install.sh; less install.sh; sh install.sh`.
- Publish a **SHA256** and, ideally, sign the artifacts the script downloads (see [download → verify → run](download-verify-run.md) and [Security](../cross-cutting/security.md)).
- Keep the script **small, readable, and idempotent** so auditing is realistic.
- The common "the server can sniff `curl` and serve different bytes to `| sh`" worry is real in theory but not unique to this method — the deeper point is to **pin and verify** what gets executed.

## Approval & maintenance burden

Effectively zero to start: host one or two scripts. Ongoing burden is keeping them **idempotent**, **platform-aware**, and **in sync with releases** — and ideally moving from a branch URL to a **pinned, checksummed release asset**.

## Best for / Avoid when

**Best for:** MVPs and demos; multi-component products that need real setup logic; CLIs that want a friction-free "Get started"; anything you want live today on every OS.

**Avoid when:** your audience forbids running remote scripts (locked-down enterprise); you need app-store discoverability or managed auto-updates (use [Winget](winget.md)/[Homebrew](homebrew.md)); the install must be fully offline.

## See also

- [Download → verify → run](download-verify-run.md) — the hardened, checksum-verified variant
- [Docker Compose](docker-compose.md) — what the bundle's script actually drives
- [GitHub Releases (binaries)](github-releases-binaries.md) — where the script should fetch pinned, signed artifacts
- [Security](../cross-cutting/security.md) · [Idempotency & uninstall](../cross-cutting/idempotency-uninstall.md) · [Comparison matrix](../02-comparison-matrix.md)
