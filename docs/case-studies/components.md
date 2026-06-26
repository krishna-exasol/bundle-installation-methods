# The components — what each piece is

The [Exasol bundle](exasol-ai.md) combines a **database** with two companion tools. This page summarizes what each piece actually is — language, packaging, how it installs/runs, and the one constraint that matters for bundling. The database comes in two forms (**Nano** or **Personal**), so four components are described.

| Component | Role | What it is | Installs as | Key port(s) |
|-----------|------|-----------|-------------|-------------|
| **Exasol Nano** | Database | Core SQL engine, single-node | Docker image (~124 MB) | 8563 (SQL), 8443 (UI) |
| **Exasol Personal** | Database | Launcher that provisions a real Exasol DB | Go binary (host CLI) | 8563 (SQL, dynamic locally) |
| **Exasol MCP Server** | LLM access | MCP server over the DB | pip package / Docker | 4896 (HTTP) |
| **Exasol JSON Tables** | JSON→SQL | Ingest + query JSON as SQL | Python + Rust (source) | — (CLI) |

---

## Exasol Nano

**The Core Exasol SQL engine packaged as a single-node Docker image** (~124 MB, arm64 + amd64; also a native Linux `.run` and a native macOS/Apple-Silicon build). Built for dev, testing, edge, and embedded analytics.

- **Run:** `docker run --rm -it --shm-size=512mb -p 8563:8563 -v exanano-data:/exa exasol/nano:latest`
- **Connect:** `127.0.0.1:8563` (TLS, self-signed cert), user `sys` / password `exasol`; Web UI on `8443`; data persists in the `/exa` volume.
- **Distroless image:** the image itself contains only the `/controller` entrypoint + the DB engine — no shell, apt, or python. The runtime environment is materialized at first start by `/controller`, which is why you can't bake tools onto the image at build time.
- **Stack system (dev-source / roadmap — _not in the public image yet_):** Nano's source has a `--provision-stacks <list>` system with built-in **python**, **java**, **rust**, **jupyter**, **marimo**, and `mcp-server` stacks. **The published `exasol/nano:latest` does not support it yet** — verified by testing: the flag is silently appended to the DB parameters and ignored. So treat stacks as a promising *future*; today you extend Nano with **sidecar containers** (see the [Nano case study](nano-jsontables-mcp.md)).
- **Why it matters here:** *once stacks ship in the public image*, the whole bundle could live inside a single Nano container. Until then, MCP runs as the published `exasol/mcp-server:latest` sidecar next to Nano.

Source: [hub.docker.com/r/exasol/nano](https://hub.docker.com/r/exasol/nano)

---

## Exasol Personal

**A launcher CLI — not a database image.** A single static **Go binary** (`exasol`) that *provisions* a full Exasol database, either locally or in the cloud. Positioned as "the Analytics Database for Agentic AI — free for personal use."

- **Install:** `curl …/installer.sh | sh` → binary at `~/.local/bin/exasol`.
- **Targets (presets):** `local` (a managed VM, **macOS Apple-Silicon only**) and cloud — `aws` / `azure` / `exoscale` / `stackit` — provisioned via **OpenTofu** (downloaded and cached on demand).
- **Local DB:** runs inside a VM on the host, reachable at `127.0.0.1:<dynamic dbPort>`, user `sys` / `exasol`. Connection details are discoverable via `exasol info --json` plus the launcher's `secrets.json`.
- **Lifecycle:** `install` / `init` / `deploy`, `start` / `stop` / `destroy`, `info` / `status`, `config`, and a built-in `connect` SQL shell (`-c` / `-f`, `--json` / `--csv`).
- **Why it matters here:** because the DB lives on the host (not a container), the companion tools must reach *up* to it — which is what makes that bundle a hybrid (host DB + containerized tools) rather than a clean all-container stack.

Source: [github.com/exasol/exasol-personal](https://github.com/exasol/exasol-personal)

---

## Exasol MCP Server

**A Model Context Protocol server that exposes an Exasol database to LLM / agent clients** (Claude Desktop, etc.). Clean Python package, the easiest piece to bundle.

- **Package:** `exasol-mcp-server` (v1.10.1), Python 3.10–3.13, built on **FastMCP**. Published to **PyPI** and as a **Docker image**.
- **pyexasol constraint:** `>=1,<2`.
- **Run:** stdio mode (`exasol-mcp-server`, for desktop clients) or HTTP mode (`exasol-mcp-server-http --host 0.0.0.0 --port 4896 --no-auth`, default port 8000) exposing `/mcp` (+ `/health`).
- **Connection:** via env vars `EXA_DSN`, `EXA_USER`, `EXA_PASSWORD`, `EXA_SSL_CERT_VALIDATION`, plus SaaS / OAuth / BucketFS options.
- **Safe by default:** metadata browsing and keyword search are on; **read queries, writes, profiling, and BucketFS are off by default**, and destructive ops require user elicitation. Also ships bundled "skills" (domain-knowledge resources) installable via `exasol-install-skills`.

Source: [github.com/exasol/mcp-server](https://github.com/exasol/mcp-server)

---

## Exasol JSON Tables

**A tool to ingest JSON/NDJSON into Exasol as a SQL-queryable relational table family, then query and reshape it** — ingest → query → reshape, all over one stable relational contract. The hardest piece to package.

- **Languages:** **Python + Rust.** Python CLI `exasol-json-tables` (`ingest`, `ingest-and-wrap`, `wrap`, `describe`, `structured-results`); a Rust crate (`json_to_parquet`, on `exarrow-rs 0.12.0`) does the JSON→Parquet→Exasol ingest.
- **pyexasol constraint:** `>=2.2,<3` (Python 3.10+).
- **The packaging crux:** there is **no PyPI wheel and no prebuilt binary** — the Python CLI **shells out to `cargo run` at ingest time**, so the repo checkout *and* a Rust toolchain must be present at runtime.
- **Bulk import:** uses Exasol's **HTTP transport, where the database connects *back* to the client** — fragile across a host/container network boundary (the bundle's main caveat).
- **Query surface:** a session-scoped SQL preprocessor enables JSON-native syntax (dotted paths, array joins, `TO_JSON(...)`) over the wrapper schemas.

Source: [github.com/exasol-labs/exasol-json-tables](https://github.com/exasol-labs/exasol-json-tables)

---

## The two constraints that shape everything

1. **Dependency conflict.** JSON Tables needs `pyexasol >=2.2,<3`; MCP Server needs `>=1,<2`. The ranges don't overlap, so the two tools **cannot share one Python environment** — they must be isolated (separate containers, or separate venvs/stacks).
2. **JSON Tables' Rust-at-runtime coupling.** No wheel + `cargo run` at ingest means the runtime must carry a Rust toolchain — which Nano's `rust` stack happens to provide.

See the [Exasol bundle](exasol-ai.md) case study for how these drove the choice of a script-pipe installer over Docker Compose.
