# Docker pull / docker run (single image)

> **One-liner:** Distribute one component as a single OCI image pulled from a registry and started with `docker run` — the simplest container delivery, but it ships exactly one container, not a multi-component bundle.

| | |
|---|---|
| **Category** | Single-container distribution (OCI image + container engine) |
| **Platforms** | Any host with Docker Engine / Podman / containerd (Linux, macOS & Windows via Docker Desktop or WSL2) |
| **Prerequisites** | A container engine (Docker / Podman) and registry access (Docker Hub, GHCR, ECR, a private registry) |
| **Handles** | acquire ✓ / verify ✓ (digests, cosign) / place ✓ (image layers) / configure ? (env/flags only) / start ✓ / update ? (re-pull + recreate, manual) / uninstall ✓ (`rm` + `rmi`) |
| **Maturity fit** | MVP (single service) — too thin for a real multi-component bundle |
| **Trust model** | Registry-hosted image referenced by **tag** (mutable) or **`@sha256` digest** (immutable); optional Sigstore **cosign** signatures and SBOM/provenance attestations |

## How it works
`docker pull` fetches an image — a stack of content-addressed layers plus a manifest — from a registry into the local store. `docker run` creates and starts a container from that image, optionally wiring in environment variables (`-e`), published ports (`-p`), and mounts (`-v`). The image is identified either by a **tag** (`exasol/nano:8.32`, which the publisher can re-point at any time) or by an **immutable digest** (`exasol/nano@sha256:…`, which always resolves to the exact bytes that were signed and tested).

The defining limitation for a *bundle*: `docker run` starts **one container**. The case-study product has three moving parts — the Exasol Personal/Nano database, the MCP server, and the json-tables component — that must share a network, start in dependency order, and persist data in volumes. A single `docker run` cannot express "start the DB, wait for it to be healthy, then start the MCP server pointed at it." You either collapse everything into one fat image (an anti-pattern that defeats independent updates and process isolation) or you graduate to an orchestrator. For multi-component installs, [Docker Compose](./docker-compose.md) is the natural next step; `docker run` is best for delivering or demoing a single component of the bundle.

## Example
```bash
# Pull by tag (convenient, but the tag is mutable — it can change under you)
docker pull exasol/nano:8.32

# Pull by immutable digest (reproducible; pins exact bytes)
docker pull exasol/nano@sha256:9f2c0b1e...c4

# (Optional) verify a Sigstore signature before running
cosign verify exasol/nano@sha256:9f2c0b1e...c4 \
  --certificate-identity-regexp '.*' \
  --certificate-oidc-issuer https://token.actions.githubusercontent.com

# Run a single component with a named volume, published port, and config via env
docker run -d \
  --name exasol-nano \
  -p 8563:8563 \
  -v exasol-data:/exa \
  -e EXASOL_PASSWORD=changeme \
  --restart unless-stopped \
  exasol/nano@sha256:9f2c0b1e...c4

# Inspect / update / remove
docker logs -f exasol-nano
docker pull exasol/nano:8.32 && docker rm -f exasol-nano && docker run -d ...   # manual update
docker rm -f exasol-nano && docker volume rm exasol-data                         # uninstall + data
```

## Pros
- **Lowest-friction container delivery:** one `pull` + one `run`, no extra tooling beyond an engine.
- **Reproducible** when you pin by `@sha256` digest — identical bytes everywhere.
- **Verifiable** with cosign signatures and SBOM/provenance attestations.
- Clean, isolated runtime; trivial uninstall (`rm`/`rmi`/`volume rm`).
- Works identically across Linux, macOS, and Windows hosts.

## Cons
- **Single container only** — cannot express a multi-service bundle, dependency order, or shared networking. This is disqualifying for the real product.
- No declarative state: ports, volumes, env, and restart policy live in shell flags that drift and are easy to mistype.
- **Updates are manual** — re-pull, stop, remove, recreate; no convergence logic.
- Tags are mutable by default; a careless `:latest` makes "what is actually running" non-deterministic.
- No host-level checks (free ports, disk, engine version) — those need a wrapping [script pipe](./script-pipe.md).

## Security considerations
Prefer **immutable digests** over floating tags so the running bytes match what you reviewed and signed. Verify images with **cosign** and inspect SBOM/provenance attestations before first run. Pulling from a public registry trusts that registry's integrity and the publisher's account hygiene; mirror critical images into a registry you control where supply-chain risk matters. Treat `-e SECRET=…` as visible in process listings and image history — prefer Docker secrets or env files with tight permissions. Avoid `--privileged` and never bind-mount the Docker socket into a workload container (see [docker socket bootstrap](./docker-socket-bootstrap.md) for why that is root-equivalent). See [Security](../cross-cutting/security.md).

## Approval & maintenance burden
- **Publisher:** build, tag, sign, and push images; maintain a digest changelog and (ideally) automated SBOM generation. No store review gate — you control the registry.
- **Consumer:** track new digests, schedule re-pulls, and recreate containers. Without an orchestrator this is per-host manual toil, which is exactly why a single-component image rarely stays the whole story for a bundle.

## Best for / Avoid when
**Best for:** shipping or demoing a **single** component, smoke-testing one service, CI base images, or the leaf node of a larger compose/Helm setup. **Avoid when:** the deliverable is a multi-component bundle needing orchestration, ordered startup, shared networks, or persistent multi-volume state — use [Docker Compose](./docker-compose.md) (local) or [Helm on Kubernetes](./helm-kubernetes.md) (cluster) instead.

## Real-world examples
- `docker run -d -p 6379:6379 redis@sha256:…` — single datastore for local dev.
- `docker run --rm -it ubuntu:24.04 bash` — ephemeral throwaway environment.
- Official images (`postgres`, `nginx`, `node`) are routinely demoed as one-liners, then promoted into Compose/Helm for real deployments.

## See also
- [Comparison matrix](../02-comparison-matrix.md)
- [Docker Compose](./docker-compose.md)
- [Docker socket bootstrap](./docker-socket-bootstrap.md)
- [Script pipe](./script-pipe.md)
- [Versioning & pinning](../cross-cutting/versioning-pinning.md)
- [Security](../cross-cutting/security.md)
