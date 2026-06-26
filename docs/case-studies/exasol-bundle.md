# Case study (archive): the original Exasol AI bundles

!!! warning "Archived — superseded by `exasol-quickstart`"
    This page documents the **original `exasol-ai` and `exasol-personal-ai` bundles** — a `curl \| sh` / `irm \| iex` **script-pipe installer driving Docker Compose**, one per database (Nano and Personal).

    They have since been **unified into a single front door: [`exasol-quickstart`](recommended-approach.md)** (`pipx run exasol-quickstart`). For the current recommendation, start there.

    This page is kept for history — and because the **constraints it documents still drive the current design**: a dependency conflict that forces isolation, a distroless base, a runtime that shells out to Rust, and a database that connects *back* to the client.

This is the real problem that motivated the catalog: package an Exasol database
together with two companion tools into a **one-command install**. It's a useful
case study because it hits almost every hard constraint a bundle can — a
dependency conflict that forces process isolation, a distroless base that blocks
build-time baking, a runtime that shells out to a foreign toolchain, and a
database that connects *back* to the client. The method we chose was a
**script-pipe installer driving Docker Compose**.

## Problem

Bundle three things so a user goes from nothing to a running, queryable system
with one command:

1. **The database** — either **Exasol Nano** (a Docker image) or **Exasol
   Personal** (a host-installed database).
2. **Exasol MCP Server** — a pip package, `exasol-mcp-server==1.10.1`. Serves
   HTTP on **:4896**, gives an LLM/MCP client **read-only** access to the
   database.
3. **Exasol JSON Tables** — a Python CLI that, **at runtime, shells out to a Rust
   `cargo` ingest engine**. It needs the repo checked out and a Rust toolchain
   present; there is no published wheel.

## Constraints

The constraints, not preferences, drove every decision:

- **Hard dependency conflict.** JSON Tables requires `pyexasol>=2.2,<3`; the MCP
  Server requires `pyexasol>=1,<2`. These ranges **do not intersect**, so the two
  tools **cannot live in one Python environment**. They must run in **separate
  containers**. (See [versioning & pinning](../cross-cutting/versioning-pinning.md)
  on why pinning made this conflict explicit rather than silently broken.)
- **JSON Tables needs a Rust toolchain at runtime**, plus its source repo — it's
  not a self-contained binary.
- **Exasol Nano is distroless** — no shell, no package manager — so you can't
  `RUN apt`/`RUN pip` to bake tools onto it at build time.
- The install must be **cross-platform** and add **no prerequisites** beyond what
  users plausibly already have.

## Two architectures

Because the database comes in two forms, there are two bundles.

### (a) `exasol-ai` — Nano, fully containerized

Exasol Nano *is* a Docker image, so the whole bundle is a **3-service Docker
Compose** stack:

| Service | What it is | Notes |
|---------|-----------|-------|
| `nano` | Exasol Nano database image | The DB, in-container |
| `mcp` | `exasol-mcp-server==1.10.1` | HTTP :4896, read-only, own `pyexasol 1.x` env |
| `json-tables` | JSON Tables CLI + Rust `cargo` engine | Own `pyexasol 2.x` env, repo baked into image |

All three run as containers on a shared Compose network; the two tools reach the
database in-network. Clean and self-contained.

### (b) `exasol-personal-ai` — Personal on the host, tools in containers

**Exasol Personal is not a Docker image.** It's installed via a **host launcher
CLI** — `exasol install local` (macOS **Apple-Silicon only**) — and the database
**runs on the host**, not in a container.

So this bundle is **hybrid**:

- The database runs on the host via the launcher.
- The two companion tools still run as **containers**, connecting up to the host
  database via **`host.docker.internal:<dbPort>`**.
- Connection details are **discovered at install time** from `exasol info --json`
  plus the launcher's `secrets.json` (host, port, credentials), then injected
  into the containers.

This split — host DB, containerized tools, dynamic discovery of the connection —
is exactly the kind of orchestration a bare compose file can't do on its own.

## Method selection

We chose a **[script-pipe installer](../methods/script-pipe.md)** that drives
**[Docker Compose](../methods/docker-compose.md)**:

```sh
curl -fsSL https://example.com/install.sh | sh      # macOS / Linux
```
```powershell
irm https://example.com/install.ps1 | iex           # Windows
```

Why this method:

- **Zero extra prerequisites** — needs only Docker, which the target users have.
- **Cross-platform** — one approach covers macOS, Linux, and Windows.
- **Ships today** — no registry, no approval, no signing cert required.
- **Orchestration power** — the script does what a bare compose can't:
  - prereq checks (Docker present and running, architecture, the Personal
    launcher for bundle b);
  - configuration (ports, generated secrets, the discovery step above);
  - multi-container startup via Compose;
  - **helper-script generation** (wrappers, an `uninstall.sh`).

See the [comparison matrix](../02-comparison-matrix.md) for how this scores
against alternatives.

## What we rejected (and why) for the MVP

- **Bare `docker compose -f <URL> up`.** Can't do host prereq checks, can't do the
  Personal connection-discovery step, can't generate helpers or an uninstall. Too
  little orchestration for a hybrid bundle.
- **OS/package managers (Homebrew, Winget, …).** Need registry approval *and*
  require the user to already have that manager. Neither fits a ship-today MVP;
  they're on the roadmap, not the critical path.
- **Building JSON Tables as a wheel.** There's no PyPI wheel, and it shells out to
  `cargo` at runtime — a wheel wouldn't capture the Rust toolchain dependency.
- **Baking tools onto the Nano image.** Nano is **distroless** — no shell to
  `RUN` anything. Build-time baking is a non-starter (recorded in project memory).

## Caveats

- **JSON Tables bulk ingest uses Exasol's HTTP transport, where the database
  connects *back* to the client.** That reverse connection is fragile across a
  host/container network boundary — the DB may not be able to reach a port inside
  a container, or may resolve the wrong address. A **host-mode fallback** (running
  the JSON Tables ingest directly on the host instead of in a container) is
  documented for when the reverse connection fails.
- The Personal bundle is **macOS Apple-Silicon only**, inheriting the launcher's
  platform limits.

## Roadmap

Following the staged path in the
[maturity roadmap](../cross-cutting/maturity-roadmap.md):

- **Now (MVP):** script-pipe over Docker Compose, both bundles.
- **Next (Growth):** **pin the Nano image by digest** and **JSON Tables by tag**
  (see [versioning & pinning](../cross-cutting/versioning-pinning.md)); publish a
  GitHub release with **SHA256 checksums** and signatures (see
  [security](../cross-cutting/security.md)).
- **Later (Mature):** **Homebrew** and **Winget** front doors for discoverability,
  once demand and the maintenance budget justify the approval cycles.
