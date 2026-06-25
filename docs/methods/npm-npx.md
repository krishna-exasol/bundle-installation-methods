# Node: npm install -g / npx

> **One-liner:** Distribute a Node CLI through the npm registry — install it globally with `npm install -g`, or run it once without installing via `npx`.

| | |
|---|---|
| **Category** | Language package manager (Node.js / npm registry) |
| **Platforms** | macOS, Linux, Windows (anywhere Node.js runs) |
| **Prerequisites** | Node.js + npm (npx ships with npm ≥5.2). A network path to the npm registry. |
| **Handles** | acquire ✓ / verify ✓ (integrity hash / lockfile) / place ✓ (bin on PATH) / configure ✗ / start ? (only if the CLI self-starts) / update ✓ / uninstall ✓ |
| **Maturity fit** | MVP → Growth (standard way to ship a Node CLI) |
| **Trust model** | npm registry; `package-lock.json` integrity (SRI) hashes; optional npm provenance/attestations via Trusted Publishing. No mandatory signing of every package. |

## How it works
npm packages declare a `bin` field mapping command names to scripts. Two delivery modes:

- **`npm install -g <pkg>`** installs into a **single global prefix** shared by every globally installed package and links the `bin` entries onto PATH. Because all globals share one tree, version conflicts and "global pollution" are common — two tools wanting incompatible versions of a shared dependency, or a global install shadowing a project-local one. Globals also frequently require elevated permissions unless the prefix is user-owned.
- **`npx <pkg>`** fetches the package (if not cached), runs its `bin` **once**, and leaves nothing globally linked. It is the Node analogue of Python's `uvx`: great for one-off invocations, scaffolders, and CI. By default npx will download-and-run a package that isn't installed, which is convenient but means a typo can execute an unintended package — pass `--yes`/`--no` deliberately and pin versions.

Both verify download **integrity** against the SRI hash recorded in `package-lock.json` (or the registry's `dist.integrity`), so corruption/tampering in transit is detected.

For this **bundle**, npm/npx are only relevant if a component ships as a Node package — e.g. a JavaScript MCP client/wrapper or a Node-based launcher. The Python `exasol-mcp-server`, the Exasol Nano DB container, and the Rust `json-tables` core are **not** npm packages. A Node CLI published here could *orchestrate* the others (shelling out to `uvx`, `docker`, the Rust binary), but npm itself installs only the Node component.

## Example
```bash
# Global install of a Node CLI (shared global prefix; may need a user-owned prefix)
npm install -g exa-bundle-cli
exa-bundle-cli --help
npm update -g exa-bundle-cli
npm uninstall -g exa-bundle-cli

# Run once without installing — nothing left behind
npx exa-bundle-cli init
npx exa-bundle-cli@1.2.0 init        # pin the exact version for this run
npx --yes exa-bundle-cli init        # skip the install prompt explicitly

# A Node launcher can orchestrate the other components, but doesn't package them:
#   npx exa-bundle-cli up   ->   docker run exasol/nano ; uvx exasol-mcp-server ; ./json-tables ...
```

## Pros
- Ubiquitous on developer machines that already have Node; one short, familiar command.
- `npx` runs tools with **no persistent install** — ideal for scaffolders and CI steps.
- Integrity (SRI) hashes and `package-lock.json` give reproducible, verified installs.
- Trivial cross-platform publishing from a single `npm publish`.
- Optional npm **provenance** ties a published artifact to its source build.

## Cons
- **Node.js must already be installed** — not a turnkey path for non-developers.
- `npm install -g` shares one global tree: version conflicts and "global pollution" are routine; globals can shadow project-local versions.
- npx auto-running an uninstalled package can execute the wrong thing on a typo (name-confusion).
- Only fits components actually published as npm packages — irrelevant to the Python/DB/Rust parts.
- Installs the Node component only; cannot bring up the DB + Rust CLI + server together.

## Security considerations
The npm registry is open-publish and has a long history of typosquatting, dependency-confusion, and compromised-maintainer incidents. Mitigations: commit a `package-lock.json` and install with `npm ci` for byte-for-byte reproducibility; pin exact versions (avoid floating ranges for security-sensitive tools); prefer packages publishing **provenance/attestations**; run `npm audit` and consider `--ignore-scripts` to block install-time lifecycle scripts (a common malware vector). With npx, always pin `pkg@version` in untrusted or automated contexts. See [Security](../cross-cutting/security.md).

## Approval & maintenance burden
- **Approval:** none to publish (self-service `npm publish`); none for users to install.
- **Maintenance:** keep `dependencies` and the lockfile current, respond to `npm audit` advisories, and manage the `engines` Node range. Native addons (node-gyp) add per-platform build/prebuild burden, similar to native wheels in Python.

## Best for / Avoid when
**Best for:** shipping a Node-based CLI, scaffolder, or thin orchestrator/launcher to developers; `npx` for one-shot or CI use. **Avoid when:** your audience has no Node runtime and wants turnkey setup; the component isn't a Node package; or you need the whole multi-component bundle (DB + Rust CLI + server) provisioned in one step.

## Real-world examples
- `npx create-react-app`, `npx create-next-app`, `npx degit` — run-once scaffolders.
- `npm install -g typescript`, `vercel`, `pnpm`, `@angular/cli` — globally installed CLIs.
- Many MCP servers ship as npm packages launched via `npx <server>` from MCP client config.

## See also
- [Comparison matrix](../02-comparison-matrix.md)
- [Python: pip / pipx / uvx](./python-pip-pipx-uvx.md)
- [Other language managers (cargo / go / gem)](./other-language-managers.md)
- [Version managers (asdf / mise / proto)](./version-managers.md)
- [Security](../cross-cutting/security.md)
