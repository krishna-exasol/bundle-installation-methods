# Other language managers: cargo / go / gem

> **One-liner:** Install a tool through its language's own installer — `cargo install` (Rust), `go install` (Go), or `gem install` (Ruby) — typically building from source from the language registry.

| | |
|---|---|
| **Category** | Language package managers (per-ecosystem) |
| **Platforms** | macOS, Linux, Windows (wherever the toolchain runs) |
| **Prerequisites** | `cargo`: Rust toolchain (rustc + cargo). `go`: Go toolchain. `gem`: Ruby + RubyGems. Plus a C/linker toolchain for crates/gems with native deps. |
| **Handles** | acquire ✓ / verify ? (registry checksums; no signing by default) / place ✓ (into a per-language bin dir) / configure ✗ / start ✗ / update ? (manual rebuild) / uninstall ✓ |
| **Maturity fit** | MVP → Growth (the native way to ship a single-language tool to that ecosystem's developers) |
| **Trust model** | crates.io / pkg.go.dev (proxy + checksum DB) / rubygems.org. Checksums and (Go) a transparency log; **little central curation**; no mandatory artifact signing. |

## How it works
Each command pulls source (or, for gems, often source + a build step) from its registry and produces an executable on PATH.

- **`cargo install <crate>`** downloads the crate from **crates.io**, **compiles it from source** with the local Rust toolchain, and drops the binary in `~/.cargo/bin`. A Rust compiler is mandatory; there is no prebuilt-binary fast path through cargo itself. This is the natural fit for the **Rust half of `json-tables`** — but note it installs *the Rust binary only*, not the Python wrapper that ships alongside it.
- **`go install <module>@<version>`** fetches the module via the Go module proxy, **builds it from source**, and installs to `$GOBIN` (default `~/go/bin`). Versions are git tags/pseudo-versions; `@latest` or an explicit tag selects them. A Go toolchain is required.
- **`gem install <gem>`** downloads from **rubygems.org**; pure-Ruby gems install as-is, while gems with C extensions compile at install time (needing headers + a compiler). Binaries land in the active Ruby's bin dir.

All three install **one language's component cleanly and nothing else**. For this **bundle** they are, at best, a piece of the puzzle: `cargo install` can deliver the `json-tables` Rust CLI, but it cannot bring up the Exasol Nano database (a container) or the Python `exasol-mcp-server`. There is no cross-language orchestration here — each manager knows only its own ecosystem.

## Example
```bash
# Rust — builds the json-tables CLI from source (needs the Rust toolchain)
cargo install json-tables
cargo install json-tables --version 0.3.1     # pin a version
cargo install --git https://github.com/org/json-tables --tag v0.3.1   # from git
cargo uninstall json-tables

# Go — fetch + build a Go tool from its module path
go install github.com/org/some-tool@latest
go install github.com/org/some-tool@v1.4.0    # pin a tagged version
# (uninstall = remove the binary from $GOBIN)

# Ruby — install a gem (compiles C extensions at install time if any)
gem install some-tool
gem install some-tool -v 2.1.0
gem uninstall some-tool

# Reality check for the bundle: cargo installs ONLY the Rust binary.
# The DB container and the Python MCP server are out of scope for these tools.
```

## Pros
- Native, idiomatic path for each ecosystem's developers; one short command.
- Installs the exact tool from its canonical registry with version pinning.
- `cargo`/`go install` can build straight from a git tag — handy before any registry release.
- Self-contained binaries (especially Go) are easy to relocate and run.
- No packaging/approval gate: publish to the registry and users can install immediately.

## Cons
- **Build-from-source by default** (cargo always; go always; gems with native ext): users need the full toolchain and pay compile time and disk.
- **No central curation** beyond crates.io / pkg.go.dev / rubygems.org — discovery and trust are on the user; namespaces are first-come.
- Each manager covers exactly one language — it cannot install the DB or a different-language sibling. Useless as a whole-bundle installer.
- Weak built-in upgrade story: `cargo install` re-runs to upgrade (no `update` subcommand for installed binaries in older toolchains); go/gem similar.
- No service start, no config, no DB provisioning.

## Security considerations
These registries verify **download checksums** but do not mandate **artifact signing**. crates.io and rubygems.org are open-publish (typosquatting/name-confusion risk); Go's module proxy adds a **checksum database / transparency log** (`sumdb`) that detects after-the-fact tampering, a meaningful trust improvement. Because cargo/go build from source, install-time **build scripts** (`build.rs`) and gem `extconf.rb`/native compiles run arbitrary code on your machine — vet sources, pin versions, and prefer installing from a known git tag or a vendored lockfile. Keep `Cargo.lock`/`Gemfile.lock` committed for reproducibility. See [Security](../cross-cutting/security.md).

## Approval & maintenance burden
- **Approval:** none — self-service publish to each registry; users install freely.
- **Maintenance:** tag releases, keep `Cargo.lock`/`go.mod`/`Gemfile.lock` current, and provide prebuilt binaries *outside* these managers (GitHub Releases) if you want to spare users the toolchain. Native dependencies add per-platform build burden.

## Best for / Avoid when
**Best for:** delivering a single-language developer tool — e.g. the `json-tables` Rust CLI via `cargo install` — to people who already have that toolchain. **Avoid when:** end users lack the compiler/toolchain; you need a prebuilt binary with no build step; or you need the whole multi-component bundle (DB + Rust CLI + MCP server) installed together — a single-language manager fundamentally cannot do that.

## Real-world examples
- `cargo install ripgrep`, `cargo install cargo-edit`, `cargo install bat` — Rust CLIs from crates.io.
- `go install github.com/golangci/golangci-lint/cmd/golangci-lint@latest`, `go install honnef.co/go/tools/cmd/staticcheck@latest`.
- `gem install bundler`, `gem install jekyll`, `gem install rails`.

## See also
- [Comparison matrix](../02-comparison-matrix.md)
- [Python: pip / pipx / uvx](./python-pip-pipx-uvx.md)
- [npm & npx](./npm-npx.md)
- [Version managers (asdf / mise / proto)](./version-managers.md)
- [Security](../cross-cutting/security.md)
