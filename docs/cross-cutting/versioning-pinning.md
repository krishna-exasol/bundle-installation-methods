# Versioning & pinning: from `latest` to immutable digests

An install is only as reproducible as the references it pins. This page covers
how to point at versions of images, git sources, and language packages — from the
loosest (`latest`, `main`) to the strictest (content digests, lockfiles) — and
how to give users control over which they get.

Examples use the catalog's case study: the Exasol Nano image, the JSON Tables git
ref, and `exasol-mcp-server==1.10.1`. See the
[case study](../case-studies/exasol-ai.md) and the
[comparison matrix](../02-comparison-matrix.md).

## Container images: tag vs semver vs digest

Three ways to reference an image, in increasing strictness:

| Reference | Example | Mutable? | Use for |
|-----------|---------|----------|---------|
| Rolling tag | `exasol/nano:latest` | Yes — moves anytime | Never in installers |
| Semver tag | `exasol/nano:8.34.0` | Mostly stable, but reassignable | Human-readable pinning |
| Digest | `exasol/nano@sha256:9f86d0…` | **No** — content-addressed | Reproducible installs |

- **`latest` is not a version.** It's whatever was pushed last. Two users running
  the same installer a week apart can get different software. Never bake `latest`
  into a shipped installer.
- **Semver tags** (`:8.34.0`) are readable and good enough for most users, but a
  tag is a *pointer* — a publisher can re-push `:8.34.0` over different bytes.
- **Digests are immutable.** `@sha256:…` names the exact content; if the bytes
  change, the digest changes. This is the strongest pin.

```yaml
# docker-compose.yaml — pin the digest, comment the human-readable tag
services:
  nano:
    image: exasol/nano@sha256:9f86d081884c7d659a2feaa0c55ad015a3bf4f1b2b0b822cd15d6c15b0f00a08  # nano 8.34.0
```

Practical pattern: **pin the digest, record the tag in a comment** so humans know
what it is and machines get exactly those bytes. Update the digest deliberately
when you bump versions. See [Docker Compose](../methods/docker-compose.md).

## Git sources: branch vs tag vs commit SHA

JSON Tables is consumed from its git repository (it shells out to a Rust `cargo`
engine at runtime — there's no wheel to pin). The same hierarchy applies:

| Ref | Example | Reproducible? |
|-----|---------|---------------|
| Branch | `@main` | No — moves with every push |
| Tag | `@v0.5.2` | Mostly — but tags can be re-pointed |
| Commit SHA | `@a1b2c3d…` | **Yes** — immutable |

```sh
# Loosest: tracks the branch HEAD (don't ship this)
pip install "git+https://github.com/exasol/json-tables.git@main"

# Better: a release tag
pip install "git+https://github.com/exasol/json-tables.git@v0.5.2"

# Strongest: a full commit SHA
pip install "git+https://github.com/exasol/json-tables.git@a1b2c3d4e5f6..."
```

Ship a **tag or SHA**, never `main`. A tag is the readable default; a SHA is the
truly immutable one.

## Language packages: pip / npm / cargo and lockfiles

Two layers matter: pinning your **direct** dependencies, and locking the **full
transitive tree**.

- **pip.** Pin exact versions with `==` and capture the resolved tree:

  ```text
  exasol-mcp-server==1.10.1
  pyexasol>=1,<2
  ```

  Use `pip-tools` / `pip freeze` or a lockfile (`requirements.txt` pinned, or
  `uv.lock` / `poetry.lock`) so the full graph is reproducible. See
  [pip/pipx/uvx](../methods/python-pip-pipx-uvx.md).
- **npm.** `package.json` ranges resolve, but `package-lock.json` (committed)
  pins the exact tree; `npm ci` installs strictly from the lock. See
  [npm/npx](../methods/npm-npx.md).
- **cargo.** `Cargo.toml` carries semver ranges; `Cargo.lock` pins exact
  versions and is committed for binaries/apps. See
  [other language managers](../methods/other-language-managers.md).

**Note on version range conflicts.** The case study's two tools can't share a
Python env precisely because of incompatible pins — JSON Tables needs
`pyexasol>=2.2,<3` while the MCP server needs `pyexasol>=1,<2`. Pinning made the
conflict *visible*, which is the point; the resolution was to isolate them in
separate containers rather than paper over it.

## Reproducibility

Reproducibility means: same inputs in, byte-identical (or at least
behaviorally-identical) result out, on any machine, any day. To get there:

- Pin **all** references to immutable forms (digests, SHAs, locked versions).
- Commit lockfiles to the repo.
- Avoid implicit `latest` / `main` / unbounded ranges anywhere in the chain.
- Record an SBOM per release so you can audit exactly what shipped (see
  [security](security.md)).

## Letting users pin

Don't hard-code versions the user can't override. Offer environment variables or
flags with safe defaults:

```sh
# install.sh respects overrides, defaults to a known-good pin
NANO_IMAGE="${NANO_IMAGE:-exasol/nano@sha256:9f86d081…}"
MCP_VERSION="${MCP_VERSION:-1.10.1}"
JSONTABLES_REF="${JSONTABLES_REF:-v0.5.2}"
```

```sh
NANO_IMAGE=exasol/nano:8.35.0 curl -fsSL https://example.com/install.sh | sh
```

This keeps the happy path reproducible while letting advanced users move forward
or roll back.

## Channels: stable / beta / nightly

For ongoing releases, expose a small number of **channels** rather than asking
users to track raw versions:

- **stable** — the default; well-tested, recommended for everyone.
- **beta** — release candidates for early adopters.
- **nightly** — bleeding edge, expect breakage.

Implement channels as moving aliases that *resolve to* an immutable pin at install
time (so a `stable` install still records the exact digest it got). Document which
channel a user is on and how to switch.

## Upgrade UX

- Make the version **visible** (`--version`, a banner, a label on the image).
- Make upgrade a **single, idempotent command** that re-pulls the pinned
  references and reconciles state — see
  [idempotency & uninstall](idempotency-uninstall.md).
- Don't auto-upgrade silently across breaking changes; respect the user's pin.
- Document the **downgrade/rollback** path (re-pin to the previous digest/SHA).
