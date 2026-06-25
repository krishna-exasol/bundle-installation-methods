# Docker socket bootstrap (sock-mounting installer container)

> **One-liner:** Ship a small "bootstrap" container that mounts the host's `/var/run/docker.sock` and installs the rest of the stack from inside — convenient one-shot setup, but socket access is **root-equivalent on the host**, so treat it as CI/demo-only.

| | |
|---|---|
| **Category** | Container-driven installer (Docker-out-of-Docker via host socket) |
| **Platforms** | Linux hosts with a Docker daemon socket; macOS/Windows via Docker Desktop's socket forwarding (with caveats) |
| **Prerequisites** | A running Docker daemon and permission to mount `/var/run/docker.sock` into a container |
| **Handles** | acquire ✓ / verify ? (bootstrap image must itself be verified) / place ✓ / configure ✓ (script inside container) / start ✓ / update ? (re-run bootstrap) / uninstall ? (needs a teardown command) |
| **Maturity fit** | MVP / demos / CI only — **not** for untrusted or production hosts |
| **Trust model** | A registry-hosted bootstrap image (pin by **`@sha256` digest** + **cosign**) that is granted **full control of the host Docker daemon** via the mounted socket |

## How it works
The Docker daemon listens on a Unix socket, conventionally `/var/run/docker.sock`. Any process that can talk to that socket can create, start, and stop containers — including privileged ones that bind-mount the host filesystem. A **socket-bootstrap** installer packages the install logic (pull images, write a generated `compose.yaml`, run `docker compose up -d`, do host config) into a small container, then mounts the host socket into it: `-v /var/run/docker.sock:/var/run/docker.sock`. Inside, the bootstrap uses a Docker client (this is **Docker-out-of-Docker** — it drives the *host* daemon, it does not run a nested daemon) to stand up the database, MCP server, and json-tables services as **sibling** containers on the host.

The appeal is a single, dependency-light entry point: the user needs only Docker, runs one `docker run`, and the bootstrap does the rest — no local script interpreter, no cloning a repo, deterministic install logic shipped as an image. For the case study this can wrap the same multi-service [Compose](./docker-compose.md) stack the script-pipe installer would, just delivered as a container instead of a piped script.

**The security reality, stated plainly:** mounting the Docker socket grants the bootstrap container the **same power as root on the host**. A container with socket access can launch another container that mounts `/` and reads or writes any host file, escaping any isolation the container appeared to provide. There is no meaningful sandbox here. This is acceptable in **ephemeral CI runners and throwaway demo VMs** where the host is disposable and the bootstrap image is trusted and pinned. It is **not** appropriate for a user's laptop, a shared host, or any production machine — prefer a transparent [script pipe](./script-pipe.md) the user can read, or plain [Compose](./docker-compose.md), in those cases.

## Example
```bash
# One-shot installer: mount the host Docker socket so the bootstrap drives the host daemon.
# The bootstrap image is pinned by digest and (ideally) cosign-verified first.
cosign verify example/exa-bundle-bootstrap@sha256:7c1e9a4f...d2 \
  --certificate-oidc-issuer https://token.actions.githubusercontent.com \
  --certificate-identity-regexp '.*'

docker run --rm \
  -v /var/run/docker.sock:/var/run/docker.sock \    # ROOT-EQUIVALENT host access — see warning
  -v "$PWD/exa-data":/out \                          # where the bootstrap writes compose.yaml/config
  -e EXASOL_PASSWORD=changeme \
  example/exa-bundle-bootstrap@sha256:7c1e9a4f...d2 install

# Conceptually, inside the bootstrap container it runs something like:
#   docker compose -f /out/compose.yaml up -d        # creates DB + MCP + json-tables as SIBLING containers

# Teardown must be a first-class command (don't leave orphaned siblings/volumes):
docker run --rm \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v "$PWD/exa-data":/out \
  example/exa-bundle-bootstrap@sha256:7c1e9a4f...d2 uninstall
```

## Pros
- **Single dependency:** only Docker is required on the host — no local shell interpreter, repo clone, or language runtime.
- Install logic is **shipped as a versioned, signable image** (pin by digest), so the procedure is deterministic and reproducible.
- Good fit for **ephemeral CI**: one `docker run` stands up the full stack in a disposable runner.
- Can reuse the same multi-service [Compose](./docker-compose.md) definition, just delivered differently.

## Cons
- **Root-equivalent host access** via the socket — the single biggest drawback; effectively no isolation.
- **Opaque:** the user runs an image, not a readable script — harder to audit than a [script pipe](./script-pipe.md) they can inspect before piping.
- Sibling-container model means the bootstrap must carefully label/track what it created so **uninstall** can find and remove it (easy to orphan volumes/networks).
- Socket path and behavior differ on Docker Desktop (macOS/Windows) and rootless Docker; not uniformly portable.
- No real privilege reduction is possible while retaining socket access.

## Security considerations
**Mounting `/var/run/docker.sock` is equivalent to giving the container root on the host.** A malicious or compromised bootstrap image can start a privileged sibling that mounts `/` and takes over the machine. Mitigations reduce but never eliminate this: pin the bootstrap by **`@sha256` digest**, **cosign-verify** it and check its SBOM before running, run only on **disposable** hosts (CI/demo), and consider a **socket proxy** (e.g. an allow-listing proxy that exposes only the specific Docker API calls the installer needs) instead of the raw socket. Mounting the socket **read-only does not help** — the Docker API is fully exploitable over a read-only socket. For end-user installs, prefer a transparent script pipe or plain Compose. See [Security](../cross-cutting/security.md).

## Approval & maintenance burden
- **Publisher:** build, sign, and digest-pin the bootstrap image; implement and test idempotent `install`/`update`/`uninstall` subcommands; clearly document the socket-mount risk and restrict the pattern to CI/demo guidance.
- **Consumer:** must accept granting host-root to the image. Updates re-run the bootstrap; uninstall depends entirely on the publisher having shipped a working teardown path.

## Best for / Avoid when
**Best for:** ephemeral CI pipelines, throwaway demo VMs, and internal tooling on disposable hosts where convenience outweighs isolation and the image is trusted. **Avoid when:** installing on a user's laptop, a shared/multi-tenant host, or any production system — use a transparent [script pipe](./script-pipe.md) over [Compose](./docker-compose.md), or [Helm on Kubernetes](./helm-kubernetes.md) for clusters, where privileges can be scoped properly.

## Real-world examples
- CI providers and tools (e.g. Docker-based test harnesses, `testcontainers`, build agents) routinely mount the socket inside runners.
- Watchtower and similar "manage other containers" tools mount the socket — and are commonly cited as cautionary examples of the privilege they hold.
- Socket-proxy projects (allow-listing Docker API proxies) exist precisely because raw socket mounts are root-equivalent.

## See also
- [Comparison matrix](../02-comparison-matrix.md)
- [Docker Compose](./docker-compose.md)
- [Docker run / pull](./docker-run-pull.md)
- [Script pipe](./script-pipe.md)
- [Versioning & pinning](../cross-cutting/versioning-pinning.md)
- [Security](../cross-cutting/security.md)
