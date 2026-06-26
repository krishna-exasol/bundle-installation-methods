# Joining Exasol Personal + JSON Tables + MCP — the recommended method

Same three-way bundle, but with **Exasol Personal** as the database instead of [Nano](nano-jsontables-mcp.md). That one swap changes the best method, because **Personal is not a container — it's a host launcher** that provisions a real Exasol DB (a managed VM locally on macOS Apple-Silicon, or a cloud instance via OpenTofu).

!!! success "Recommendation"
    **A one-command [script-pipe installer](../methods/script-pipe.md) that co-locates everything on the host**: let the **Personal launcher** provide the database, then install **MCP Server** and **JSON Tables** as **isolated host environments** ([pipx](../methods/python-pip-pipx-uvx.md) for MCP, a dedicated venv + Rust toolchain for JSON Tables) — all talking to the DB over `127.0.0.1`.

    ```bash
    curl -fsSL https://example.com/install.sh | sh
    #  1. exasol install local            → DB on 127.0.0.1:<dbPort>
    #  2. exasol info --json               → discover dsn / port / password
    #  3. pipx install exasol-mcp-server   → MCP on :4896 (own venv)
    #  4. venv + cargo build json-tables   → CLI against 127.0.0.1 (own venv)
    ```

    Co-locating the tools with the DB on `localhost` is the key move — it makes JSON Tables' bulk ingest work and removes the host/container boundary entirely.

See [The components](components.md) for what each piece is.

---

## The one constraint that decides everything

JSON Tables' bulk **ingest uses Exasol's HTTP transport, where the database connects *back* to the client**. That reverse connection only works reliably when **JSON Tables shares a network with the database**:

- ✅ **Same host (`localhost`)** — DB on the host, JSON Tables on the host → trivial.
- ✅ **Same Docker network** — the all-container [Nano answer](nano-jsontables-mcp.md).
- ⚠️ **Across a host/container boundary** — DB on host, JSON Tables in a container reached via `host.docker.internal` → the DB often can't connect back into the container. Ingest is fragile; `wrap`/`describe`/query still work.

With Personal the database lives **on the host**, so the clean way to satisfy that constraint is to **put the tools on the host too**. This is the single biggest reason the Personal method differs from the Nano one.

> MCP Server has no such issue — it only makes **outbound** connections to the DB — so MCP could run in a container if you prefer. JSON Tables is the piece that must sit next to the database.

---

## Recommended architecture

```text
┌──────────────────────────── macOS host ─────────────────────────────┐
│                                                                      │
│  exasol launcher ── provisions ──►  Exasol Personal DB               │
│  (~/.local/bin)                     127.0.0.1:<dbPort>  sys/exasol    │
│                                          ▲         ▲                  │
│                                 localhost│         │localhost         │
│                      ┌───────────────────┴──┐  ┌───┴────────────────┐ │
│                      │ MCP Server (pipx venv)│  │ JSON Tables (venv  │ │
│                      │ exasol-mcp-server-http│  │  + Rust toolchain) │ │
│                      │ :4896                 │  │  cargo ingest      │ │
│                      └───────────────────────┘  └────────────────────┘ │
└──────────────────────────────────────────────────────────────────────┘
        connection (dsn/port/password) discovered via `exasol info --json`
```

### How the hard constraints are handled

1. **`pyexasol` conflict** (JSON Tables `>=2.2,<3` vs MCP `>=1,<2`): each tool gets its **own environment** — `pipx` puts MCP in an isolated venv; JSON Tables gets a separate venv. No shared interpreter, no clash — and no containers required.
2. **JSON Tables' Rust-at-runtime coupling**: install a host **Rust toolchain** (rustup) once; the venv builds the `cargo` ingest engine and runs it locally.
3. **Dynamic connection**: the local DB port is assigned at deploy time, so the installer reads it from **`exasol info --json`** (+ `secrets.json` for the password) and injects it into both tools — never hardcoded.

### Why a script pipe on top

The [script-pipe installer](../methods/script-pipe.md) is what turns four manual steps into one line: install the launcher, ensure a local deployment exists, discover the connection, set up the two isolated environments, and start MCP — plus a `run-json-tables` helper and an uninstaller. It needs no registry approval and works as the single "Get started" command. (Note: Personal's **local** mode is macOS Apple-Silicon only; the launcher itself is cross-platform for cloud targets.)

---

## Why not the alternatives

| Method | Verdict | Why |
|--------|---------|-----|
| **Host-co-located tools + script pipe** *(recommended)* | ✅ | Everything on `localhost` → ingest works; `pipx`/venv resolves the dependency conflict cleanly; no Docker or `host.docker.internal` needed. |
| **Host DB + two containers** (`host.docker.internal`) | ⚠️ Works, with a caveat | This is the existing `exasol-personal-ai` bundle. MCP is fine, but **JSON Tables ingest hits the reverse-connection problem** across the host/container boundary — so that bundle documents a *host-mode fallback* for ingest. The recommended method simply makes host-mode the **default** for JSON Tables. |
| **Hybrid: MCP container + JSON Tables on host** | ✅ Acceptable | Valid middle ground — MCP only needs outbound, so a container is fine; JSON Tables stays on the host where ingest works. Slightly less uniform than all-host. |
| **Native Nano stacks** | ❌ N/A | Personal isn't Nano and isn't a container, so there's no runtime stack system to extend. That's the [Nano case study](nano-jsontables-mcp.md), not this one. |
| **Cloud Personal + tools** | ⚠️ Limited | For a cloud DB, `wrap`/query/MCP work over the remote DSN, but **ingest's reverse connection generally can't reach your laptop**. Run JSON Tables on a host the DB can reach, or use **local** Personal for ingest-heavy work. |

---

## Relationship to the `exasol-personal-ai` bundle

The existing `exasol-personal-ai` bundle implements the **two-container** shape (host DB + MCP and JSON Tables containers via `host.docker.internal`) and already documents the host-mode fallback for ingest. The recommendation here is the natural refinement: **keep JSON Tables on the host by default** so ingest "just works," and optionally containerize only MCP. Same discovery (`exasol info --json`), same one-command UX — just the tool placement that the reverse-connection constraint argues for.

## Nano vs Personal — the unifying principle

| | Database is… | Best method | Why |
|---|---|---|---|
| **[Nano](nano-jsontables-mcp.md)** | a container with a stack system | extend Nano with **stacks** (built-in `mcp-server` + custom `json-tables`) | one runtime; localhost everywhere |
| **Personal** *(this page)* | a host launcher | co-locate tools **on the host** (pipx/venv) | tools share `localhost` with the host DB |

Both reduce to the same rule: **put JSON Tables wherever it shares `localhost` with the database** — because of the HTTP-transport reverse connection. MCP, being outbound-only, is flexible either way.

---

## In one sentence

Because **Exasol Personal puts the database on the host**, the best way to join Personal + JSON Tables + MCP is a one-line script-pipe installer that **installs both tools as isolated host environments next to that DB** (pipx for MCP, a venv + Rust for JSON Tables), so ingest works on `localhost` and the `pyexasol` conflict is resolved without containers.

**Related:** [Script pipe](../methods/script-pipe.md) · [pip / pipx / uvx](../methods/python-pip-pipx-uvx.md) · [Source build](../methods/source-build.md) · [The components](components.md) · [Nano + JSON Tables + MCP](nano-jsontables-mcp.md)
