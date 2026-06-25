# Snap & Flatpak (sandboxed Linux apps)

> **One-liner:** Distribute the bundle as a self-contained, sandboxed Linux app — Snap (via the Snapcraft Store) or Flatpak (via Flathub) — that bundles its own runtime and runs under confinement.

| | |
|---|---|
| **Category** | Universal/sandboxed Linux package (cross-distro) |
| **Platforms** | Snap: Ubuntu and any distro with `snapd`. Flatpak: most Linux distros with `flatpak` (Flathub default remote) |
| **Prerequisites** | Snap: `snapd`, a Snapcraft Store account, `snapcraft.yaml`. Flatpak: `flatpak` + a runtime (e.g. Freedesktop SDK), `flatpak-builder`, a manifest |
| **Handles** | acquire ✓ / verify ✓ (store-signed) / place ✓ / configure ? (portals/interfaces) / start ✓ / update ✓ (auto for snap) / uninstall ✓ |
| **Maturity fit** | Growth → Mature (strong for GUI/desktop & cross-distro single-artifact apps) |
| **Trust model** | Store-mediated: Snapcraft Store / Flathub sign and serve; apps run **confined/sandboxed** and must request access via interfaces/portals. |

## How it works
Both formats ship an app **with its dependencies bundled** so it runs across distros, and both **sandbox** the app by default.

**Snap** packages are built from a `snapcraft.yaml` describing `parts` (build steps), `apps` (entry points/daemons), and `plugs` (capabilities like `network`, `docker`, `home`). Snaps use **confinement** levels: `strict` (full sandbox, only declared interfaces), `classic` (no sandbox — for tools that need full host access, and requires manual store review), or `devmode` (dev only). `snapcraft` builds the `.snap`; you push it to the **Snapcraft Store** across channels (`stable`/`candidate`/`beta`/`edge`). Snaps **auto-update** by default.

**Flatpak** apps are built from a manifest (JSON/YAML) listing a `runtime` (e.g. `org.freedesktop.Platform`), an `sdk`, and `modules` (sources built in a sandbox). `flatpak-builder` produces the app; you publish to **Flathub** (or any remote). Sandboxing is enforced and host access is mediated by **portals** (file chooser, network, etc.) plus `--filesystem`/`--share` permissions. Updates come via `flatpak update`.

**Critical bundle caveat:** Snap and Flatpak are designed to install **one packaged, sandboxed artifact** — typically a single app and its libraries. They are a poor fit for a **multi-container service stack**. The `json-tables` Python+Rust CLI can be bundled cleanly. The MCP server (pip) can be vendored into the part/module. But the Exasol Personal/Nano database is a **container** — and a sandboxed Snap/Flatpak cannot host a Docker/Podman daemon for you. At best the snap requests the `docker` interface or shells out to a host engine, which breaks the "self-contained" promise and runs into confinement limits. **Call this out plainly:** for this bundle, Snap/Flatpak realistically packages only the CLI/MCP layer; the database still needs Docker/Podman on the host. They shine for single GUI/CLI apps, not container orchestrations.

## Example
```bash
# ---------- Snap ----------
# snapcraft.yaml
name: exa-bundle
base: core24
version: '1.2.0'
summary: Exasol Nano launcher + MCP server + json-tables CLI
confinement: strict          # strict sandbox; use 'classic' only if host access is essential (manual review)
parts:
  json-tables:
    plugin: rust
    source: ./json-tables
  mcp-server:
    plugin: python
    source: ./mcp-server     # vendors the pip package + deps into the snap
apps:
  exa-bundle:
    command: bin/exa-bundle
    plugs: [network, network-bind, home, docker]   # 'docker' interface to reach a host engine
# Build, test, publish:
snapcraft                                 # builds exa-bundle_1.2.0_amd64.snap
sudo snap install --dangerous ./exa-bundle_1.2.0_amd64.snap   # local test
snapcraft login && snapcraft upload --release=stable exa-bundle_1.2.0_amd64.snap
# Consumer:
sudo snap install exa-bundle
sudo snap connect exa-bundle:docker docker:docker   # manual interface connection if needed

# ---------- Flatpak ----------
# org.example.ExaBundle.yaml (manifest)
#   app-id: org.example.ExaBundle
#   runtime: org.freedesktop.Platform   runtime-version: '24.08'
#   sdk: org.freedesktop.Sdk
#   finish-args: ["--share=network", "--filesystem=home"]
#   modules: [ json-tables (rust), mcp-server (python pip) ]
flatpak install org.freedesktop.Sdk//24.08 org.freedesktop.Platform//24.08
flatpak-builder --repo=repo --force-clean build-dir org.example.ExaBundle.yaml
flatpak build-bundle repo exa-bundle.flatpak org.example.ExaBundle   # single-file bundle
# Publish to Flathub: submit the manifest as a PR to github.com/flathub/flathub (reviewed).
# Consumer:
flatpak remote-add --if-not-exists flathub https://flathub.org/repo/flathub.flatpakrepo
flatpak install flathub org.example.ExaBundle
flatpak run org.example.ExaBundle
```

## Pros
- One artifact runs across many distros — bundled runtime sidesteps "dependency hell."
- **Sandboxed by default**: least-privilege confinement reduces blast radius of a compromise.
- Store-mediated distribution with signing, channels, and (for Snap) automatic updates.
- Clean install/uninstall with no leftover system state.
- Good for shipping a current version on distros with old system libraries.

## Cons
- **Built for a single sandboxed app, not a multi-container stack** — the Exasol DB still needs a host Docker/Podman engine the sandbox can't provide.
- Confinement fights system integration: reaching the host Docker socket, devices, or arbitrary paths requires interfaces/portals or `classic` confinement (manual review).
- Larger downloads (bundled runtimes) and extra disk per app/runtime.
- Snap auto-update and the proprietary, single Snapcraft Store backend are points of contention for some users; Flathub is more open but desktop-centric.
- Headless/server use is secondary to the desktop-app use case both ecosystems optimize for.

## Security considerations
The sandbox is the headline benefit: a `strict` snap or a portal-mediated Flatpak limits what a compromised app can touch. That same model is the friction — every host capability (network, home, the Docker socket) must be **explicitly granted**, and granting broad access (e.g. `classic` confinement or `--filesystem=host`) erodes the protection. Store signing covers integrity and provenance, but vendored dependencies inside the snap/flatpak are **your** responsibility to keep patched (the bundled runtime can age). Prefer minimal interfaces/`finish-args`; pin any container image you pull by digest. See [Security](../cross-cutting/security.md).

## Approval & maintenance burden
- **Snap (Snapcraft Store):** self-publish to `strict`/`stable` with no human gate; **`classic` confinement requires manual review/approval**. Upkeep: maintain `snapcraft.yaml`, rebuild per `base`, manage channels.
- **Flatpak (Flathub):** submission is a **reviewed PR** to the Flathub repo (app-id ownership, manifest quality, reasonable permissions). Self-hosting your own remote avoids review but loses Flathub's reach/trust. Runtimes need periodic version bumps.

## Best for / Avoid when
**Best for:** single self-contained GUI or CLI apps that must run across many distros with minimal host coupling; teams that want sandboxing and store-managed updates. **Avoid when:** your product is fundamentally a **multi-container service** (like this DB bundle) — the sandbox can't host the engine; or you need deep host/system integration, headless server fleets, or the smallest possible footprint (use native packages or plain Docker/Compose).

## Real-world examples
- **Snap:** `code` (VS Code), `slack`, `chromium`, and many Canonical-published apps; classic-confinement examples include `kubectl` and various dev tools.
- **Flatpak/Flathub:** OBS Studio, GIMP, Blender, and most Linux desktop apps ship via Flathub manifests.

## See also
- [Comparison matrix](../02-comparison-matrix.md)
- [Homebrew](./homebrew.md)
- [Linux native packages (.deb / .rpm)](./linux-native-packages.md)
- [Security](../cross-cutting/security.md)
- [Versioning & pinning](../cross-cutting/versioning-pinning.md)
