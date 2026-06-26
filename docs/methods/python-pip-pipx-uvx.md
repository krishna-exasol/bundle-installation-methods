# Python: pip / pipx / uvx

> **One-liner:** Install or run a Python-packaged tool (here, the `exasol-mcp-server` entry point) from PyPI — globally with `pip`, in a dedicated isolated venv with `pipx`, or ephemerally and install-free with `uvx`.

| | |
|---|---|
| **Category** | Language package manager (Python / PyPI) |
| **Platforms** | macOS, Linux, Windows (anywhere CPython or uv runs) |
| **Prerequisites** | `pip`: a Python interpreter + pip. `pipx`: Python ≥3.8 + pipx. `uvx`: the `uv` binary installed (bundles its own Python download). |
| **Handles** | acquire ✓ / verify ✓ (hashes/lockfile) / place ✓ (entry-point on PATH) / configure ✗ / start ? (only if the tool self-starts) / update ✓ / uninstall ✓ |
| **Maturity fit** | MVP → Growth (the standard way to ship a Python CLI/server) |
| **Trust model** | PyPI registry; optional pinned hashes / `requirements.txt` lockfiles; PyPI Trusted Publishing + attestations for newer releases. No mandatory signing. |

## How it works
All three tools acquire a **wheel or sdist from PyPI** and expose its console-script entry points. They differ in *where the code lives* and *how long it stays*.

- **pip** installs into whatever environment is active — a user site, a virtualenv, or (dangerously) the system Python. It does **not** isolate the tool from other packages, so dependency conflicts between tools sharing one environment are common. `pip install --user exasol-mcp-server` puts the `exasol-mcp-server` script on PATH but mixes its deps with everything else in that interpreter.
- **pipx** creates **one dedicated virtual environment per tool** and symlinks only the entry points onto your PATH. The MCP server gets its own private dependency tree; upgrading or removing it never disturbs another tool. This is the recommended way to install a Python *application* (as opposed to a *library* you import).
- **uvx** (`uv tool run`) is **ephemeral**: each invocation resolves the package, builds a throwaway virtualenv in uv's cache, runs the command, and discards the environment. Nothing is permanently installed. A cold run takes about a second; cached re-runs are tens of milliseconds. `uv tool install` is the persistent counterpart equivalent to pipx.

For this **bundle**, all three cleanly handle the **pip package** (`exasol-mcp-server`). They do **not** install the Exasol Nano database (a container product) or the Rust half of `json-tables` — a Python installer only handles the Python component. Treat them as the right tool for one piece, not the whole bundle.

## Example
```bash
# pip — global-ish install into the active/user environment (least isolated)
python -m pip install --user exasol-mcp-server
exasol-mcp-server --help

# pipx — isolated, persistent install (recommended for a CLI/server)
pipx install exasol-mcp-server
pipx upgrade exasol-mcp-server
pipx uninstall exasol-mcp-server
pipx install "exasol-mcp-server==1.10.1"          # pin a version

# uvx — run ephemerally, nothing left installed
uvx exasol-mcp-server --help
uvx exasol-mcp-server@1.10.1 --help               # pin a version for this run
uv tool install exasol-mcp-server                # persistent equivalent of pipx

# The json-tables Python+Rust CLI: pip/pipx work only if it ships a wheel
# with the Rust extension prebuilt (maturin/abi3). Otherwise a Rust
# toolchain is needed at install time to compile the extension.
pipx install json-tables
```

## Pros
- Native to the Python ecosystem; PyPI is the canonical home for `exasol-mcp-server`.
- pipx isolation eliminates cross-tool dependency conflicts and gives clean upgrade/uninstall.
- uvx needs **zero persistent install** — ideal for CI, one-off runs, and trying a version.
- uv resolves and installs dramatically faster than pip and can fetch its own Python.
- Cross-platform with one publish step; version pinning is trivial (`==`, `@`).

## Cons
- **Python (or uv) must already be present** — not a turnkey path for non-developer end users.
- Bare `pip install` into the system interpreter risks breaking OS-managed Python ("externally managed environment").
- Packages with native extensions (the Rust side of `json-tables`) need prebuilt wheels per platform, or a compiler at install time.
- None of them start a service, write config, or manage the DB container — installation ≠ running.
- A language installer brings up **one** component; it cannot stand up DB + Rust CLI + server together.

## Security considerations
PyPI is open-publish, so the registry curates little: typosquatting and dependency-confusion are real. Pin versions and, for reproducible installs, pin **hashes** (`pip install --require-hashes -r requirements.txt`). Prefer publishers using PyPI **Trusted Publishing** (OIDC, no long-lived tokens) and **attestations**. pipx/uv install with the same trust as pip — isolation limits blast radius but does not vet the package. uvx running arbitrary `package@version` strings executes whatever that version resolves to, so pin in untrusted contexts. See [Security](../cross-cutting/security.md).

## Approval & maintenance burden
- **Approval:** none to publish to PyPI (self-service); none for users to install.
- **Maintenance:** publish wheels/sdist per release; for native extensions, build wheels for each OS/arch (cibuildwheel/maturin) or accept source builds. Keep dependency pins current and watch for advisories.

## Best for / Avoid when
**Best for:** distributing the `exasol-mcp-server` (and any pure-Python CLI) to a developer audience; pipx for a kept tool, uvx for ad-hoc/CI runs. **Avoid when:** your users have no Python/uv and expect a single turnkey installer; or you need the *whole* bundle (DB + Rust CLI + server) bootstrapped in one command — use an orchestrator or container approach for that.

## Real-world examples
- `pipx install` is the documented install path for many Python CLIs (e.g. `black`, `httpie`, `poetry`, `ansible`).
- `uvx ruff`, `uvx mypy`, and `uvx --from build pyproject-build` are common ephemeral invocations in CI.
- Numerous MCP servers are published to PyPI and launched via `uvx <server>` directly from MCP client config.

## See also
- [Comparison matrix](../02-comparison-matrix.md)
- [npm & npx](./npm-npx.md)
- [Other language managers (cargo / go / gem)](./other-language-managers.md)
- [Version managers (asdf / mise / proto)](./version-managers.md)
- [Security](../cross-cutting/security.md)
