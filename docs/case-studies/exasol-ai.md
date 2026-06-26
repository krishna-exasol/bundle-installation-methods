# Case study (archive): `exasol-ai` — Nano + MCP + JSON Tables

!!! warning "Archived — superseded by `exasol-quickstart`"
    `exasol-ai` was the original **Nano-based** bundle: a `curl \| sh` / `irm \| iex` **script-pipe installer driving a 3-service Docker Compose** stack. It has since been folded into the unified [`exasol-quickstart`](recommended-approach.md) front door (the **Docker / Nano** path). Kept for history — its constraints still drive the current design.

    **Repo:** [github.com/krishna-exasol/exasol-ai](https://github.com/krishna-exasol/exasol-ai)

## The goal

Package three things so a user goes from nothing to a running, queryable system with **one command**:

1. **Exasol Nano** — the database, *a Docker image*.
2. **Exasol MCP Server** — `exasol-mcp-server==1.10.1`; HTTP on **:4896**, read-only LLM/MCP access.
3. **Exasol JSON Tables** — a Python CLI that shells out to a **Rust `cargo` ingest engine** at runtime (no published wheel).

## The architecture — fully containerized

Because Nano *is* a Docker image, the whole bundle is a **3-service Docker Compose** stack on a shared network:

| Service | What it is | Notes |
|---------|-----------|-------|
| `nano` | Exasol Nano database image | the DB, in-container |
| `mcp` | `exasol-mcp-server==1.10.1` | HTTP :4896, read-only, own `pyexasol 1.x` env |
| `json-tables` | JSON Tables CLI + Rust `cargo` engine | own `pyexasol 2.x` env, repo baked into the image |

All three run as containers on one Compose network; the tools reach the database **in-network**, so JSON Tables' bulk ingest (which has the DB connect *back* to the client) works without any `host.docker.internal` plumbing.

## Constraints that shaped it

- **Hard dependency conflict** — JSON Tables needs `pyexasol >=2.2,<3`; MCP needs `>=1,<2`. The ranges don't intersect, so the two tools **can't share one Python environment** → **separate containers**. (See [versioning & pinning](../cross-cutting/versioning-pinning.md).)
- **JSON Tables needs a Rust toolchain at runtime** plus its source repo — not a self-contained binary.
- **Nano is distroless** — no shell or package manager — so you **can't `RUN apt`/`pip`** to bake tools onto it at build time.

## Method: script-pipe → Docker Compose

```bash
curl -fsSL https://example.com/install.sh | sh      # macOS / Linux
```
```powershell
irm https://example.com/install.ps1 | iex           # Windows
```

A bare `docker compose -f <URL> up` can't do prerequisite checks, generate helper/uninstall scripts, or wire configuration — so a thin [script-pipe installer](../methods/script-pipe.md) drives [Docker Compose](../methods/docker-compose.md). See the [comparison matrix](../02-comparison-matrix.md).

## What we rejected (and why)

- **Bare compose-from-URL** — too little orchestration (no checks, helpers, or uninstall).
- **Baking tools onto the Nano image** — Nano is distroless; there's no shell to `RUN` anything.
- **A JSON Tables wheel** — none exists, and it shells out to `cargo` at runtime.
- **Package managers (Homebrew/Winget)** — need registry approval *and* the user to already have them.

## Where it went

`exasol-ai` is now the **Docker / Nano path of [`exasol-quickstart`](recommended-approach.md)** — published containers (`exasol/nano` + `exasol/mcp-server`) plus a JSON Tables sidecar, behind one Python command. See also the focused write-up: [Nano + JSON Tables + MCP](nano-jsontables-mcp.md).
