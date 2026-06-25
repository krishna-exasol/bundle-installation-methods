# Source Build (git clone + make/cargo/poetry)

> **One-liner:** Ship nothing prebuilt — users `git clone` the repository and compile each component themselves with the language's native build tool, the universal fallback that works anywhere a toolchain does.

| | |
|---|---|
| **Category** | Build-from-source (compile on the target machine) |
| **Platforms** | Anywhere the required toolchains run (Linux, macOS, Windows/WSL, BSD, exotic arches) |
| **Prerequisites** | `git` plus the **full toolchain for every component**: a C compiler/`make`, **Rust/`cargo`**, **Python + `poetry`/`pip`**, and a container engine for the DB; build-time system libraries/headers |
| **Handles** | acquire ✓ (clone) / verify ? (git tag/commit signatures) / place ? (manual `install`) / configure ? (env/config files) / start ✗ / update ? (`git pull` + rebuild) / uninstall ✗ |
| **Maturity fit** | MVP (day-one path for contributors) and **always available** as the universal fallback |
| **Trust model** | You read/build the actual source; integrity via **signed git tags/commits** and a pinned commit/tag; dependency integrity via lockfiles (`Cargo.lock`, `poetry.lock`) and checksums. |

## How it works
Building from source means the user fetches the repository at a specific tag or commit and invokes each component's native build system. The bundle is polyglot, so there is no single command: the **`json-tables`** component needs **Python and Rust** (a `poetry install` for the Python side and `cargo build --release` for the Rust extension/CLI), the **MCP server** is a Python project installed with `poetry`/`pip` into a virtualenv, and the **Exasol Personal/Nano** database is a container that is *pulled or built*, not compiled — so "from source" for the DB layer realistically means building/pulling its image and wiring it up, not recompiling the engine.

This path is the **universal fallback**: when no package, binary, or installer exists for a platform or architecture, source build still works because it depends only on toolchains, not on you having packaged for that target. It is also the **canonical developer/contributor workflow** — the same steps CI runs. The cost is that the user must supply and maintain a complete, version-correct build environment, and source build does almost nothing beyond producing artifacts: placing them on `PATH`, registering services, and starting daemons are all left to the user (or to a `make install` you provide).

## Example
```bash
# 1) Acquire a specific, verifiable revision (not the moving default branch)
git clone https://github.com/example/exa-bundle.git
cd exa-bundle
git checkout v1.2.0
git verify-tag v1.2.0          # verify the maintainer's GPG/SSH signature on the tag

# 2) Build the Rust + Python `json-tables` component
cargo build --release          # honours Cargo.lock; -> target/release/json-tables
( cd json-tables-py && poetry install )   # honours poetry.lock for the Python side

# 3) Build / install the Python MCP server into an isolated venv
( cd mcp-server && poetry install && poetry build )   # or: pip install .

# 4) The DB is a container — pull a digest-pinned image (not compiled)
docker pull exasol/personal@sha256:abc123…

# 5) Place artifacts yourself (source build won't do this for you)
sudo install -m 0755 target/release/json-tables /usr/local/bin/
make install     # if the repo provides a Makefile target to place files + units

# 6) Update later: re-fetch a tag and rebuild
git fetch --tags && git checkout v1.3.0 && cargo build --release
```

## Pros
- **Maximally portable** — works on any OS/arch with the toolchains, including ones you never packaged for.
- Full transparency: users build the exact source they can read and audit; nothing opaque is downloaded.
- Zero distribution infrastructure for the maintainer — the repo *is* the delivery mechanism.
- Natural fit for contributors and customization (patches, feature flags, custom build options).
- Lockfiles (`Cargo.lock`, `poetry.lock`) give reproducible dependency sets.

## Cons
- **Heavy prerequisites**: the user must install and maintain a full Rust + Python (+ C/`make` + container) toolchain — exactly what `json-tables` needs.
- Slow and failure-prone: long compiles, missing system headers, version skew, and platform-specific build breakage.
- Does **not** handle placement, service registration, startup, or uninstall unless you hand-author `make` targets.
- No verification unless you sign tags/commits and users actually check them.
- Builds pull transitive dependencies at build time — a broad supply-chain surface (crates, PyPI packages).

## Security considerations
The trust anchor is the revision you build: check out a **signed tag or pinned commit** and verify it (`git verify-tag` / `git verify-commit`), never the moving default branch in automation. Commit and respect **lockfiles** (`Cargo.lock`, `poetry.lock`) so dependency versions and hashes are fixed; consider `cargo --locked`, `poetry install --no-update`, and hash-checking to prevent silently pulling a different package. Building executes third-party `build.rs`/setup hooks and downloads many transitive deps, so the build host itself is part of the attack surface — build in a clean, isolated environment. Pin the DB container by **digest**. See [Security](../cross-cutting/security.md) and [Versioning & pinning](../cross-cutting/versioning-pinning.md).

## Approval & maintenance burden
No third-party approval — distribution is just a public repo with tags. The maintainer's burden shifts to **keeping the build green and documented**: clear, accurate build instructions; committed lockfiles; CI that builds every supported toolchain/OS so contributors are not the first to hit breakage; and `make`/script helpers for placement so users are not left with bare binaries. The *user's* burden is the highest of any method — they own the toolchain, the build time, and every build failure.

## Best for / Avoid when
**Best for:** contributors and developers; platforms/architectures you do not (yet) package for; users who need to patch or customize; and as the always-present fallback behind every other method. **Avoid when:** the audience is non-technical or cannot install Rust/Python toolchains; you need turnkey install with services started and clean uninstall; or fast, low-friction onboarding matters — in those cases ship [GitHub Releases binaries](./github-releases-binaries.md), a [native installer](./native-installers.md), or a [package manager](./linux-native-packages.md) built *from* this same source.

## Real-world examples
- `ripgrep`, `bat`, and most Rust CLIs document a `cargo build --release` (or `cargo install`) source path alongside prebuilt releases.
- Postgres, Redis, and Nginx ship the classic `./configure && make && make install` source build as their canonical reference build.
- Python projects across PyPI provide a `poetry install` / `pip install .` developer build that mirrors what their wheels are built from.

## See also
- [Comparison matrix](../02-comparison-matrix.md)
- [GitHub Releases binaries](./github-releases-binaries.md)
- [Python (pip / pipx / uvx)](./python-pip-pipx-uvx.md)
- [Native OS installers](./native-installers.md)
- [Security](../cross-cutting/security.md)
- [Versioning & pinning](../cross-cutting/versioning-pinning.md)
