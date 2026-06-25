# Helm chart on Kubernetes

> **One-liner:** Package the bundle as a Helm chart — templated Kubernetes manifests with a `values.yaml` — and install it into a cluster with `helm install`; cluster-native and production-grade, but assumes you already run Kubernetes (overkill for a laptop).

| | |
|---|---|
| **Category** | Kubernetes-native packaging & release management |
| **Platforms** | Any Kubernetes cluster (cloud-managed EKS/GKE/AKS, on-prem, or local k3s/kind/minikube) with Helm 3 |
| **Prerequisites** | A running Kubernetes cluster, a configured `kubectl` context, and the `helm` CLI |
| **Handles** | acquire ✓ / verify ✓ (chart provenance + image digests) / place ✓ (PVCs) / configure ✓ (`values.yaml`) / start ✓ (ordered via probes/hooks) / update ✓ (`helm upgrade`) / uninstall ✓ (`helm uninstall`) |
| **Maturity fit** | Mature (Growth → Mature for teams already standardized on Kubernetes) |
| **Trust model** | Chart pulled from an **OCI or HTTP repo**; integrity via **Helm provenance** (`.prov` + GPG, `helm verify`); per-image **`@sha256` digests** + cosign; charts themselves are cosign-signable in OCI registries |

## How it works
A **Helm chart** is a versioned bundle of templated Kubernetes manifests. Templates render into concrete `Deployment`/`StatefulSet`, `Service`, `PersistentVolumeClaim`, `ConfigMap`, and `Secret` objects, with all the knobs surfaced in a single `values.yaml` the consumer overrides. `helm install` renders the templates and applies the resulting objects to the cluster as a tracked **release**; `helm upgrade` diffs and rolls out changes; `helm rollback` reverts to a prior revision; `helm uninstall` removes the release's objects.

For the case study, the three components become Kubernetes workloads: the Exasol Personal/Nano database as a **`StatefulSet`** with a `PersistentVolumeClaim` (stable identity + durable storage), the MCP server and json-tables as **`Deployment`s**, each fronted by a **`Service`** so they reach the DB by its in-cluster DNS name. Startup ordering is handled the Kubernetes way — **readiness/liveness probes** plus optional init-containers or Helm hooks — rather than Compose's `depends_on`. Kubernetes adds what a single host cannot: rescheduling on node failure, horizontal scaling, rolling updates, and resource quotas.

**The honest caveat:** Helm assumes a **cluster already exists**. For a developer who just wants the bundle on a laptop, standing up Kubernetes (even k3s/kind) plus learning Helm values is substantial overhead — [Docker Compose](./docker-compose.md) is the lighter, correct choice there. Helm earns its keep when the bundle must run as part of a larger, multi-node, multi-tenant, or production platform.

## Example
```bash
# Add a classic HTTP chart repo (or use an OCI registry — see below)
helm repo add exa-bundle https://charts.example.com/exa-bundle
helm repo update

# Inspect and override configuration via values
helm show values exa-bundle/exa-bundle > my-values.yaml   # edit DB size, image digests, replicas

# Install the whole bundle as one release, pinned to a chart version
helm install exa exa-bundle/exa-bundle \
  --version 1.4.2 \
  --namespace exa --create-namespace \
  -f my-values.yaml \
  --set exasol.image=exasol/nano@sha256:9f2c0b1e...c4   # pin the DB image by digest

# OCI-registry alternative (charts as OCI artifacts, cosign-signable):
helm install exa oci://ghcr.io/example/charts/exa-bundle --version 1.4.2 -n exa --create-namespace

# Verify a downloaded chart's provenance before installing
helm pull exa-bundle/exa-bundle --version 1.4.2 --verify   # checks the .prov signature

# Lifecycle
helm upgrade exa exa-bundle/exa-bundle --version 1.4.3 -f my-values.yaml -n exa
helm rollback exa 1 -n exa          # revert to revision 1
helm uninstall exa -n exa           # remove the release (PVCs may be retained per reclaim policy)
```

```yaml
# my-values.yaml (excerpt) — the bundle's three components, configurable in one place
exasol:
  image: exasol/nano@sha256:9f2c0b1e...c4
  persistence: { enabled: true, size: 20Gi }   # StatefulSet PVC
mcpServer:
  image: example/exasol-mcp@sha256:3a7d11ef...90
  replicas: 1
jsonTables:
  image: example/json-tables@sha256:b21c44aa...ff
```

## Pros
- **Cluster-native:** rescheduling, scaling, rolling updates, resource limits, and namespaces come for free from Kubernetes.
- **One configurable artifact:** `values.yaml` exposes every knob; `helm upgrade`/`rollback` give release management and easy reverts.
- **Verifiable supply chain:** Helm **provenance** (`.prov`/GPG) plus OCI chart cosign signing and per-image digests.
- Fits existing GitOps/CD pipelines (Argo CD, Flux) and multi-environment promotion.
- Strong for **multi-tenant / production** deployments where the bundle is one of many workloads.

## Cons
- **Assumes a cluster exists** — heavyweight and overkill for single-laptop or single-host installs.
- Helm templating (Go templates + YAML) has a real learning curve and is error-prone to author.
- More moving parts (CNI, storage classes, ingress, RBAC) to get a "simple" stack running.
- Operational expertise required; debugging spans pods, services, PVCs, and events.
- Startup ordering is probe/hook-based, not the simple `depends_on` of Compose.

## Security considerations
Verify chart integrity with **`helm verify` / `--verify`** (provenance `.prov` + GPG) or, for OCI charts, **cosign**; pin both the **chart version** and every container **image digest**. Scope the release with a dedicated **namespace**, least-privilege **RBAC**/ServiceAccounts, **NetworkPolicies**, and Pod Security Standards; avoid cluster-admin installs. Keep secrets in Kubernetes **Secrets** (ideally backed by an external secret store / sealed-secrets), not in committed `values.yaml`. Set resource requests/limits and run containers as non-root with read-only root filesystems where possible. Track an SBOM per image. See [Security](../cross-cutting/security.md).

## Approval & maintenance burden
- **Publisher:** author and version the chart, maintain `values.yaml` and templates across Kubernetes API changes, sign chart + images, and publish to an HTTP or OCI repo. Higher authoring/maintenance cost than a `compose.yaml`.
- **Consumer:** needs a cluster and Helm fluency; updates are `helm upgrade` (with `rollback` as a safety net). Ongoing burden is cluster operations plus keeping chart/image versions current — justified at platform scale, excessive for a laptop.

## Best for / Avoid when
**Best for:** teams already running Kubernetes, multi-node/HA or multi-tenant deployments, and GitOps-managed platforms where the bundle is one workload among many. **Avoid when:** the target is a developer laptop or a single self-hosted host — use [Docker Compose](./docker-compose.md) (optionally wrapped in a [script pipe](./script-pipe.md)) instead; spinning up a cluster just to run this bundle is disproportionate.

## Real-world examples
- **Bitnami** charts (PostgreSQL, Redis, Kafka) — the canonical pattern for packaging stateful services as Helm releases.
- **Prometheus / Grafana (`kube-prometheus-stack`)**, **Ingress-NGINX**, and **cert-manager** ship as widely used charts.
- Argo CD and Flux deploy Helm charts declaratively in GitOps pipelines across environments.

## See also
- [Comparison matrix](../02-comparison-matrix.md)
- [Docker Compose](./docker-compose.md)
- [Docker run / pull](./docker-run-pull.md)
- [Script pipe](./script-pipe.md)
- [Versioning & pinning](../cross-cutting/versioning-pinning.md)
- [Security](../cross-cutting/security.md)
