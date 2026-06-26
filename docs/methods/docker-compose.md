# Docker Compose (multi-container stack)

> **One-liner:** Declare the whole bundle — database, MCP server, and json-tables — as services in a single `compose.yaml`, then bring the stack up with one `docker compose up -d`; this is the real backbone of the case-study installer.

| | |
|---|---|
| **Category** | Local multi-container orchestration (declarative, single-host) |
| **Platforms** | Any host with Docker Engine + Compose v2 (Linux, macOS & Windows via Docker Desktop / WSL2); Podman via `podman compose` |
| **Prerequisites** | A container engine and the Compose plugin (`docker compose`); registry access for the service images |
| **Handles** | acquire ✓ / verify ✓ (per-image digests/cosign) / place ✓ (volumes) / configure ✓ (env + mounts) / start ✓ (ordered, healthchecks) / update ✓ (`pull` + `up`) / uninstall ✓ (`down -v`) |
| **Maturity fit** | MVP → Growth (the sweet spot for multi-component **local** installs and demos) |
| **Trust model** | Each service references a registry image by **tag** or immutable **`@sha256` digest**; per-image cosign signatures + SBOM. The `compose.yaml` itself is plain text you ship and version. |

## How it works
Compose reads a declarative `compose.yaml` describing one or more **services** (containers), the **networks** that connect them, and the **volumes** that persist their data. `docker compose up -d` reconciles that file: it creates a private bridge network so services reach each other **by service name** (DNS), pulls images, creates named volumes, and starts containers. `depends_on` with `condition: service_healthy` enforces **startup order** — the MCP server waits until the database's healthcheck passes before it starts — solving the exact problem a single [`docker run`](./docker-run-pull.md) cannot.

For the case study, the three components map cleanly onto three services: `exasol-nano` (the database, owning a data volume), `mcp-server` (depends on the DB being healthy, exposes the MCP endpoint), and `json-tables` (the tables/data component, sharing the same network). Compose handles networking, dependency ordering, volume lifecycle, restart policies, and per-service config in one file — which is why it is the **natural fit and the actual backbone** of this bundle.

**The honest boundary:** Compose orchestrates *containers*, but it cannot do **host-level** work — it can't check that ports 8563/8080 are free, confirm the engine version, verify free disk, write host config, or print friendly remediation. That gap is why the shipped product wraps Compose in a thin [script pipe](./script-pipe.md): the script does the host preflight + config, then hands the actual stack lifecycle to `docker compose up -d`. Compose is the engine; the script is the ignition and dashboard.

## Example
```yaml
# compose.yaml — the bundle's three components as one declarative stack
name: exa-bundle

services:
  exasol-nano:                       # the database
    image: exasol/nano@sha256:9f2c0b1e...c4   # pin by digest for reproducibility
    ports: ["8563:8563"]
    volumes: ["exasol-data:/exa"]
    environment:
      EXASOL_PASSWORD: ${EXASOL_PASSWORD:?set in .env}
    healthcheck:                     # gates dependents until the DB is actually ready
      test: ["CMD", "exaplus", "-c", "localhost:8563", "-q", "SELECT 1"]
      interval: 10s
      timeout: 5s
      retries: 12
    restart: unless-stopped

  mcp-server:                        # MCP server — only starts once the DB is healthy
    image: exasol/mcp-server:latest  # the official published image
    depends_on:
      exasol-nano:
        condition: service_healthy
    ports: ["4896:4896"]
    environment:
      EXA_DSN: "exasol-nano:8563"    # reaches the DB by service name over the compose network
      EXA_USER: "sys"
      EXA_PASSWORD: ${EXASOL_PASSWORD:?set in .env}
      EXA_SSL_CERT_VALIDATION: "false"
    command: ["--host", "0.0.0.0", "--port", "4896", "--no-auth"]
    restart: unless-stopped

  json-tables:                       # json-tables data component (built from source: Python + Rust)
    image: example/json-tables@sha256:b21c44aa...ff
    depends_on:
      exasol-nano:
        condition: service_healthy
    environment:
      EXASOL_DSN: "exasol-nano:8563"
    restart: unless-stopped

volumes:
  exasol-data:                        # named volume survives container recreation
```

```bash
# Bring the whole bundle up (detached); Compose creates the network, volume, and ordered start
EXASOL_PASSWORD=changeme docker compose up -d

docker compose ps                    # status of all three services
docker compose logs -f mcp-server    # follow one service
docker compose pull && docker compose up -d   # update: re-pull pinned images, reconcile
docker compose down                  # stop + remove containers/network (keeps volumes)
docker compose down -v               # full uninstall, including the data volume
```

## Pros
- **Multi-service by design:** networking, service-name DNS, dependency order, volumes, and restart policy in one declarative file.
- **Healthcheck-gated ordering** (`depends_on: condition: service_healthy`) — the right primitive for "DB up before MCP."
- **Idempotent reconciliation:** re-running `up -d` converges to the declared state; updates are `pull` + `up`.
- The `compose.yaml` is **versionable, reviewable, and diffable** — infrastructure as a single readable artifact.
- Per-service images can each be **digest-pinned and cosign-verified**.
- Clean teardown with `down -v`; minimal host footprint beyond the engine.

## Cons
- **Single-host only** — no multi-node scheduling, failover, or autoscaling (that's [Kubernetes/Helm](./helm-kubernetes.md)).
- **No host-level capabilities** — port/disk/engine preflight and host config must be done by a wrapping [script pipe](./script-pipe.md); Compose alone cannot.
- `depends_on` waits for *health*, not for application-level readiness migrations — you still design healthchecks carefully.
- Requires the Compose v2 plugin; subtle differences across Docker Desktop, Engine, and `podman compose`.
- Secrets in `environment:`/`.env` are easy to leak into logs or VCS if not handled with care.

## Security considerations
Pin every service to an **immutable `@sha256` digest** and verify with **cosign** so the stack is reproducible and tamper-evident; floating `:latest` tags make "what's running" non-deterministic across hosts. Keep secrets out of the committed `compose.yaml` — use a gitignored `.env`, Docker/Compose **secrets**, or an external secret store, and avoid echoing them in logs. Scope published ports to `127.0.0.1:` when the service is local-only. Run services as non-root users and drop capabilities where possible. The Compose file is code: review it, and generate/track an SBOM per image. See [Security](../cross-cutting/security.md).

## Approval & maintenance burden
- **Publisher:** maintain the `compose.yaml`, keep each service's image digest current, document required env/volumes, and ship the wrapping installer script. No store review gate — distribution is just the file plus images in a registry.
- **Consumer:** `docker compose pull && up -d` to update; `down -v` to remove. Low ongoing toil for a single host. The main upkeep is keeping pinned digests fresh and re-running preflight when the host changes.

## Best for / Avoid when
**Best for:** multi-component **local** installs, developer environments, demos, and self-hosted single-host deployments — exactly this bundle. Pairs ideally with a [script pipe](./script-pipe.md) for host preflight/config. **Avoid when:** you need multi-node scheduling, HA, autoscaling, or cluster-native operations (use [Helm on Kubernetes](./helm-kubernetes.md)), or when you genuinely only ship one container ([docker run](./docker-run-pull.md) suffices).

## Real-world examples
- **GitLab, Sentry, Supabase, n8n, Immich** all ship a `docker-compose.yml` as their canonical self-hosted install.
- Most "one command to run the whole stack" developer projects bottom out in `docker compose up -d`.
- The case-study installer is precisely this pattern: a thin script does host preflight, then drives a multi-service `compose.yaml`.

## See also
- [Comparison matrix](../02-comparison-matrix.md)
- [Docker run / pull](./docker-run-pull.md)
- [Docker socket bootstrap](./docker-socket-bootstrap.md)
- [Helm on Kubernetes](./helm-kubernetes.md)
- [Script pipe](./script-pipe.md)
- [Versioning & pinning](../cross-cutting/versioning-pinning.md)
- [Security](../cross-cutting/security.md)
