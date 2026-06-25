# Windows Package Manager (winget)

> **One-liner:** Microsoft's first-party CLI package manager for Windows that installs apps from a community-curated manifest repository, calling the publisher's own signed installer under the hood.

| | |
|---|---|
| **Category** | OS package manager (Windows) |
| **Platforms** | Windows 10 1809+ and Windows 11 (ships with App Installer / Microsoft Store) |
| **Prerequisites** | A recent Windows build with the `winget` client (App Installer). No admin needed to run the client; individual installers may request elevation. |
| **Handles** | acquire ✓ / verify ✓ / place ✓ / configure ✗ / start ✗ / update ✓ / uninstall ✓ |
| **Maturity fit** | Growth / Mature |
| **Trust model** | Publisher ships a code-signed installer; manifest PRs are validated and human-moderated in `microsoft/winget-pkgs`; the manifest pins a SHA256 of the installer. |

## How it works
winget is a thin client over a *manifest* repository. The default source, `winget`, points at the open-source [`microsoft/winget-pkgs`](https://github.com/microsoft/winget-pkgs) repo, where each application is described by YAML manifests grouped by package identifier (e.g. `Exasol.ExasolAI`) and version. A manifest does not contain the binary; it records the publisher, license, an installer URL, the installer type (msi, msix, exe, zip, portable), and a `InstallerSha256` hash.

When a user runs `winget install`, the client resolves the identifier against the indexed source, downloads the installer from the URL in the manifest, verifies the downloaded bytes against the pinned SHA256, then executes the installer with the documented silent switches. Apps that register with Windows' Apps & Features (ARP) become upgradeable and uninstallable through winget without any extra manifest work.

Publishing means opening a pull request against `microsoft/winget-pkgs`. Automated validation (manifest schema, installer download, SmartScreen/static analysis, signature checks) runs first, followed by moderation. Once merged, the package is available to every winget user worldwide on the next index refresh.

winget covers acquire, verify, place, update, and uninstall well. It does **not** run post-install configuration or start services beyond whatever the installer itself does — for a bundle, all multi-component orchestration must live inside the installer payload.

## Example
```powershell
# User side
winget search ExasolAI
winget install --id Exasol.ExasolAI --silent --accept-package-agreements --accept-source-agreements
winget upgrade --id Exasol.ExasolAI
winget uninstall --id Exasol.ExasolAI

# Validate a manifest locally before opening a PR
winget validate --manifest .\manifests\e\Exasol\ExasolAI\1.0.0\
winget install --manifest .\manifests\e\Exasol\ExasolAI\1.0.0\   # local install test
```

Publisher responsibilities. Ship a single **code-signed** installer (MSI/MSIX preferred; signed EXE accepted) that internally lays down the Exasol Nano database, the MCP Server pip package, and the JSON Tables Python+Rust CLI, then author a manifest set:

```yaml
# Exasol.ExasolAI.installer.yaml (excerpt)
PackageIdentifier: Exasol.ExasolAI
PackageVersion: 1.0.0
Installers:
  - Architecture: x64
    InstallerType: msi
    InstallerUrl: https://downloads.example.com/ExasolAI-1.0.0-x64.msi
    InstallerSha256: 8F3A...REAL_HASH...C21D
    InstallerSwitches:
      Silent: /qn
ManifestType: installer
ManifestVersion: 1.6.0
```
Submit via `wingetcreate submit` or a hand-crafted PR to `microsoft/winget-pkgs`. The `wingetcreate` tool auto-computes the hash and scaffolds the three required manifest files (version, installer, defaultLocale).

## Pros
- First-party, preinstalled on modern Windows — zero bootstrap for most users.
- `winget upgrade --all` gives bundle updates a familiar, central UX.
- SHA256 pinning plus a required signed installer means a strong, verifiable trust chain.
- No hosting cost for the manifest; Microsoft serves the index. You still host the installer.
- Enterprise-friendly: works with Group Policy, can be mirrored via a private REST source.

## Cons
- You must produce and maintain a real Windows installer (MSI/MSIX). winget does not bundle disparate components for you — packaging Nano + a pip package + a Rust CLI into one installer is the hard part it does *not* solve.
- PR review latency is variable: simple version bumps can merge in hours, but new packages or anything tripping validation can sit for days awaiting a human moderator.
- The installer must be code-signed or it will fail SmartScreen/validation; that means buying and managing an OV/EV certificate.
- Every new version is another PR (or an automated submission you must wire up). Manifest schema versions change over time and occasionally require updates.
- No native post-install configuration or service-start hooks — those must live in your installer.

## Security considerations
The publisher's installer must be code-signed; unsigned or reputation-poor binaries are blocked by SmartScreen during validation. Each manifest pins `InstallerSha256`, so a tampered or swapped download is rejected by the client before execution. Manifests are reviewed by Microsoft moderators and the change history is public in `microsoft/winget-pkgs`. The trust chain is: signed installer (Authenticode) -> pinned hash in a reviewed manifest -> Microsoft-served index -> first-party client. You still control (and must secure) the installer's hosting URL.

## Approval & maintenance burden
Getting listed: produce a signed installer, author three manifest files, pass automated validation, and clear human moderation. Expect to handle reviewer feedback on metadata, license, and silent-install switches. Keeping it shipping: open a new PR per release (ideally automated with `wingetcreate` in CI), recompute hashes, and occasionally migrate to newer manifest schema versions. Lower ongoing burden than Chocolatey's install scripts, but you carry full responsibility for the installer itself.

## Best for / Avoid when
**Best for:** teams that already ship (or can build) a single signed Windows installer and want broad, first-party reach with central upgrades. **Avoid when:** you have not solved single-installer packaging, cannot code-sign, or need install-time orchestration/configuration that winget will not run.

## Real-world examples
Microsoft PowerToys, Microsoft.WindowsTerminal, Git.Git, Mozilla.Firefox, Python.Python.3, Docker.DockerDesktop, and thousands more are distributed through `microsoft/winget-pkgs`.

## See also
- [Comparison matrix](../02-comparison-matrix.md)
- [Scoop](scoop.md)
- [Chocolatey](chocolatey.md)
- [Native installers](native-installers.md)
- [Security](../cross-cutting/security.md)
