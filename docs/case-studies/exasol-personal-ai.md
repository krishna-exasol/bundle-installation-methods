# Case study (archive): `exasol-personal-ai` — Personal + MCP + JSON Tables

!!! warning "Archived — superseded by `exasol-quickstart`"
    `exasol-personal-ai` was the original **Personal-based** bundle: the **host launcher** provided the database and the two tools ran in **containers** reaching up to it via `host.docker.internal`. It has since been folded into the unified [`exasol-quickstart`](recommended-approach.md) front door (the **macOS / Personal** path, currently *experimental*). Kept for history.

    **Repo:** [github.com/krishna-exasol/exasol-personal-ai](https://github.com/krishna-exasol/exasol-personal-ai)

## The goal

Same three pieces as [`exasol-ai`](exasol-ai.md) — database + MCP Server + JSON Tables in **one command** — but with **Exasol Personal** as the database.

## Why Personal changes everything

**Exasol Personal is not a Docker image.** It's installed via a **host launcher CLI** — `exasol install local` (macOS **Apple-Silicon only**) — and the database **runs on the host**, not in a container.

So this bundle is **hybrid**:

- The **database runs on the host** via the launcher.
- The two companion tools run as **containers**, connecting up to the host DB via **`host.docker.internal:<dbPort>`**.
- Connection details are **discovered at install time** from `exasol info --json` plus the launcher's `secrets.json` (host, port, credentials), then injected into the containers.

That split — host DB, containerized tools, dynamic connection discovery — is exactly the orchestration a bare compose file can't do on its own, which is why a [script-pipe installer](../methods/script-pipe.md) drove it.

## Constraints that shaped it

- The same **`pyexasol` conflict** as [`exasol-ai`](exasol-ai.md) (MCP `>=1,<2` vs JSON Tables `>=2.2,<3`) → separate environments.
- **JSON Tables needs a Rust toolchain** at runtime (no wheel).
- **macOS Apple-Silicon only** — inherited from the Personal launcher's platform limit.

## The defining caveat — ingest reverse-connection

JSON Tables' bulk ingest uses Exasol's **HTTP transport, where the database connects *back* to the client.** With the **DB on the host** and **JSON Tables in a container**, that reverse connection is fragile across the host/container boundary — the DB may not reach a port inside the container. A **host-mode fallback** (running JSON Tables ingest directly on the host) is documented for when it fails. `wrap` / `describe` / query (client→DB only) are unaffected.

> This is the key reason the unified tool's macOS path co-locates the tools **on the host** next to the DB — see [Personal + JSON Tables + MCP](personal-jsontables-mcp.md).

## Where it went

`exasol-personal-ai` is now the **macOS / Personal path of [`exasol-quickstart`](recommended-approach.md)** (`exasol-quickstart --base personal`) — *experimental, not yet validated end-to-end*. The fully-tested path today is the [Nano + Docker bundle](nano-jsontables-mcp.md).
