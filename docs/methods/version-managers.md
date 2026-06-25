# Version managers: asdf / mise / proto

> **One-liner:** Pluggable multi-tool version managers that install and switch between *versions* of many languages and CLIs per project — powerful for developers, but a layer on top of the language installers, not a turnkey bundle installer.

| | |
|---|---|
| **Category** | Multi-tool version manager (plugin-based) |
| **Platforms** | macOS, Linux; Windows (mise/proto have native Windows support; asdf is shell-based and Unix-oriented) |
| **Prerequisites** | The manager itself installed + shell integration (PATH shims). Plugins/backends for each tool; often the underlying language toolchain for source builds. |
| **Handles** | acquire ✓ / verify ? (per-plugin; varies) / place ✓ (shims on PATH) / configure ? (`.tool-versions`/`mise.toml`) / start ✗ / update ✓ (`upgrade`) / uninstall ✓ |
| **Maturity fit** | Growth → Mature **for developer environments**; not aimed at end users |
| **Trust model** | Inherits each plugin/backend's trust (asdf plugins are arbitrary git repos; mise/proto wrap registries + asdf plugins). Reproducibility via committed `.tool-versions` / `mise.toml`. |

## How it works
These tools manage **multiple versions of multiple tools** behind PATH **shims**. A per-directory manifest pins versions, and the manager activates the right one when you `cd` in.

- **asdf** is the original: a thin core plus **plugins** (each an arbitrary git repo) that know how to list, install, and expose versions of one tool. Versions are pinned in a `.tool-versions` file. Plugins commonly compile from source or download official binaries — asdf delegates the actual acquisition to the plugin.
- **mise** (formerly rtx) is a faster, Rust-built, asdf-compatible successor. It reads `.tool-versions` *and* its own `mise.toml`, ships a curated registry of backends, can reuse asdf plugins, and adds env-var and task management. `mise use` records a tool+version into the manifest and installs it.
- **proto** (from the moon/proto project) is another Rust-built, plugin-based manager (WASM plugins) focused on speed and a single `.prototools` manifest.

All three solve **"which version of tool X is active here"** across many tools at once. Crucially, they do **not** invent a new acquisition mechanism — under the hood a plugin/backend still uses `cargo`, `go`, prebuilt release binaries, etc. They need to be **installed first**, plus shell integration, before any tool is available.

For this **bundle**, a version manager could pin, say, a Python version (for `exasol-mcp-server`) and a Rust version (to build `json-tables`) consistently across a dev team — genuinely useful. But it **cannot stand up the Exasol Nano database container**, write config, or start a server. It manages tool *versions*, not a running multi-component system. It is at best one supporting layer in a developer setup, not an installer an end user runs.

## Example
```bash
# --- mise: pin Python + Rust for the repo, then install ---
mise use python@3.12          # writes to mise.toml / .tool-versions and installs
mise use rust@1.79
mise install                  # install everything the manifest pins
mise exec -- cargo install json-tables   # build the Rust CLI under the pinned toolchain
mise upgrade                  # bump installed versions

# --- asdf: equivalent flow via plugins ---
asdf plugin add python
asdf plugin add rust
asdf install python 3.12.4
asdf set python 3.12.4        # records in .tool-versions (older asdf: `asdf local`)

# --- proto: single .prototools manifest ---
proto install python 3.12
proto install rust 1.79
proto use                     # install everything in .prototools

# Reality check: none of these start the Exasol Nano DB container or the
# MCP server — they only ensure the right LANGUAGE versions are present.
```

## Pros
- One tool to pin and switch **many** languages/CLIs per project — eliminates "works on my machine" version drift.
- Committed `.tool-versions` / `mise.toml` / `.prototools` make a team's toolchain reproducible.
- Auto-activation on `cd` removes manual version juggling.
- mise/proto are fast (Rust) and support Windows; mise adds env + task management.
- asdf's plugin model means almost any tool can be added.

## Cons
- **The manager must be installed and shell-integrated first** — a meaningful prerequisite, and shims can confuse tooling that expects real binaries on PATH.
- Built **for developers juggling versions**, not for end users who just want the bundle running.
- Still delegates to the underlying installer (source builds, toolchain needs) — it doesn't remove those costs.
- Cannot provision a database container, write config, or start services — only language/tool versions.
- A version manager cannot bring up the DB + Rust CLI + server together; it is a supporting layer, not a bundle installer.

## Security considerations
Trust is inherited from each plugin/backend. **asdf plugins are arbitrary git repositories** that run install scripts on your machine — vet the plugin source and pin it; there is no central signing. mise and proto curate a registry of backends (and proto sandboxes plugins as WASM), which narrows but does not eliminate the risk; reused asdf plugins carry asdf's trust profile. Pin exact versions in the manifest and commit it so installs are reproducible and reviewable. Be cautious with `mise trust`/auto-activation of untrusted repo configs, which can execute project-defined env/tasks. See [Security](../cross-cutting/security.md).

## Approval & maintenance burden
- **Approval:** none to adopt; each developer installs the manager themselves.
- **Maintenance:** keep the manager and plugins/backends updated; bump pinned versions in the manifest; verify plugins still work across OS/arch. Low per-release burden once a team standardizes, but it is ongoing dev-environment hygiene rather than a product distribution channel.

## Best for / Avoid when
**Best for:** standardizing the *toolchain versions* (Python, Rust, Node) a dev team uses to build/run the bundle's components consistently. **Avoid when:** your audience is end users wanting a one-command install; you need the DB container, config, and services provisioned (these don't do that); or you want a signed, curated distribution channel.

## Real-world examples
- `asdf` with `asdf-nodejs`, `asdf-python`, `asdf-ruby` plugins on developer machines.
- `mise` adopted as a faster asdf replacement, often with `mise.toml` committed to repos for env + tasks.
- `proto` used in moon-based monorepos to pin per-project tool versions via `.prototools`.

## See also
- [Comparison matrix](../02-comparison-matrix.md)
- [Python: pip / pipx / uvx](./python-pip-pipx-uvx.md)
- [npm & npx](./npm-npx.md)
- [Other language managers (cargo / go / gem)](./other-language-managers.md)
- [Security](../cross-cutting/security.md)
