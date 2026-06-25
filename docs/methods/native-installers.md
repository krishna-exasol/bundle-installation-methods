# Native OS Installers (.pkg / .msi / .exe / .deb / .rpm GUI)

> **One-liner:** Ship one signed, double-clickable installer file per operating system that lays down all bundle components and runs setup logic, the way commercial desktop software is distributed.

| | |
|---|---|
| **Category** | OS-native packaged installer (single-file, double-clickable) |
| **Platforms** | macOS (`.pkg`/`.dmg`); Windows (`.msi`, `.exe`/MSIX); Linux desktop (one-shot `.deb`/`.rpm` opened in a GUI installer) |
| **Prerequisites** | Per-OS packaging tooling (`pkgbuild`/`productbuild`, WiX/Inno Setup/Advanced Installer, `dpkg-deb`/`rpmbuild`); **code-signing certificates** (Apple Developer ID + notarization, Windows Authenticode/EV); a build/signing pipeline |
| **Handles** | acquire ✓ / verify ✓ (signing) / place ✓ / configure ✓ (install scripts) / start ✓ (LaunchDaemon/Service) / update ? (needs an updater) / uninstall ? (varies by format) |
| **Maturity fit** | Growth → Mature (high setup cost; pays off once you have a stable desktop audience) |
| **Trust model** | Apple Developer ID signing + **notarization** (Gatekeeper), Windows **Authenticode** (ideally EV cert for instant SmartScreen reputation), GPG/Authenticode on Linux packages. |

## How it works
A native installer is a self-contained file the OS knows how to open. On **macOS** a `.pkg` is an archive plus a `Distribution.xml` and optional `preinstall`/`postinstall` scripts; you build it with `pkgbuild`/`productbuild`, **sign** it with a Developer ID Installer certificate, then **notarize** it (upload to Apple, which scans it and issues a ticket you "staple" to the file) so Gatekeeper allows it without scary prompts. On **Windows**, a `.msi` (built with WiX) is a declarative database of files, registry keys, services, and custom actions, while a `.exe` bootstrapper (Inno Setup, NSIS) is procedural; both should carry an **Authenticode** signature — an **EV** certificate earns immediate SmartScreen reputation, a standard OV cert builds it over time. On **Linux desktop**, a single `.deb`/`.rpm` opened in a graphical store works the same way as the package-manager path but as a one-shot file.

For the **bundle**, the installer's scripts do the heavy lifting: drop the compiled `json-tables` binary into `PATH`, install the MCP server (vendored Python runtime or a frozen executable so you do not depend on the user's Python), register a background service (a macOS LaunchDaemon or a Windows Service) and, for the **Exasol Personal/Nano** database, either bundle its launcher or run a post-install step that pulls the digest-pinned container image and writes config. The key trade-off: installers excel at first install but have **no built-in update or uninstall story** unless you add one (a Sparkle/WinSparkle updater, an MSI upgrade table, a generated uninstaller).

## Example
```bash
# ---------- macOS: build, sign, notarize, staple ----------
pkgbuild --root ./payload --identifier com.example.exa-bundle \
  --version 1.2.0 --scripts ./scripts exa-bundle-component.pkg
productbuild --distribution Distribution.xml --package-path . exa-bundle-1.2.0.pkg
# Sign the installer with a Developer ID Installer cert
productsign --sign "Developer ID Installer: Example Inc (TEAMID)" \
  exa-bundle-1.2.0.pkg exa-bundle-1.2.0-signed.pkg
# Notarize so Gatekeeper trusts it, then staple the ticket to the file
xcrun notarytool submit exa-bundle-1.2.0-signed.pkg \
  --apple-id you@example.com --team-id TEAMID --password "$APP_PW" --wait
xcrun stapler staple exa-bundle-1.2.0-signed.pkg

# ---------- Windows: build an MSI, sign with Authenticode ----------
candle.exe exa-bundle.wxs && light.exe -ext WixUtilExtension exa-bundle.wixobj -o exa-bundle.msi
signtool sign /fd SHA256 /tr http://timestamp.digicert.com /td SHA256 \
  /a exa-bundle.msi            # /a auto-selects the (ideally EV) code-signing cert
# Silent / unattended install (for IT deployment via Intune/SCCM)
msiexec /i exa-bundle.msi /qn /norestart ADDLOCAL=ALL

# ---------- Linux desktop: one-shot .deb opened in a store ----------
dpkg-deb --root-owner-group --build exa-bundle exa-bundle_1.2.0_amd64.deb
gpg --detach-sign --armor exa-bundle_1.2.0_amd64.deb     # optional detached signature
# user double-clicks the .deb -> GNOME Software / discover handles it
```

## Pros
- Familiar, trusted UX for non-technical users: download, double-click, click Next.
- One file delivers everything — binaries, the MCP server, services, and DB bootstrap — with no package manager required.
- Install scripts can do real setup: register services/daemons, write config, create users, pull the pinned DB image.
- Signing + notarization gives a strong, OS-enforced authenticity guarantee that end users actually see.
- Supports enterprise distribution: silent flags (`/qn`, `--unattended`) plug into Intune, SCCM, Jamf, MDM.

## Cons
- **Highest packaging cost** of any method: a separate installer and toolchain per OS, each tested independently.
- Code-signing is mandatory in practice and costs money and process — Apple Developer Program + notarization round-trips, an Authenticode (especially EV, hardware-token) certificate for Windows.
- **No native update mechanism**: you must bolt on Sparkle/WinSparkle/MSI upgrade logic, or users re-download manually.
- Uninstall completeness varies; leftover services, files, and registry keys are common without careful authoring.
- Slow iteration: a notarization/signing pipeline adds minutes-to-hours of latency to every release.

## Security considerations
Signing is the entire trust story and must be correct: on macOS, sign **and** notarize **and** staple — unstapled apps fail when offline; on Windows, always timestamp (`/tr`) so signatures outlive certificate expiry, and prefer an EV cert to avoid SmartScreen warnings on early downloads. Protect signing keys in an HSM or a locked-down CI signer; a leaked Developer ID or code-signing key lets attackers ship malware as you. Audit post-install scripts as privileged code (they run as root/SYSTEM); if they pull the DB image, pin it by **digest** (`@sha256:...`). Distribute over HTTPS and publish checksums for the raw download. See [Security](../cross-cutting/security.md).

## Approval & maintenance burden
No third-party store approval is required for self-hosted installers, but the **certificate and notarization machinery is the burden**: enrolling in the Apple Developer Program, procuring/renewing OV or EV Authenticode certs, and maintaining per-OS build agents (real macOS hardware or licensed CI). Each OS release is a separate artifact to produce, sign, smoke-test, and host. Adding auto-update is a second sub-project. Expect this to be the most operationally heavy distribution channel you run.

## Best for / Avoid when
**Best for:** desktop/end-user products with a non-technical audience; enterprise IT that deploys via MDM with silent flags; software where a polished, signed, double-click experience is part of the brand. **Avoid when:** your audience is developers comfortable with package managers; you are pre-product-market-fit and cannot absorb the signing/notarization overhead; the bundle is fundamentally a server/container stack (Compose or packages fit better); or you need frequent, low-friction releases.

## Real-world examples
- Docker Desktop ships a signed/notarized `.dmg` for macOS and a signed `.exe` for Windows.
- Zoom, Slack, 1Password, and Visual Studio Code distribute signed `.pkg`/`.msi`/`.exe` desktop installers with silent-install flags for enterprise rollout.
- GitHub Desktop and Postman use Squirrel/WinSparkle-style updaters layered on top of native installers to solve the missing-update problem.

## See also
- [Comparison matrix](../02-comparison-matrix.md)
- [GitHub Releases binaries](./github-releases-binaries.md)
- [Linux native packages](./linux-native-packages.md)
- [Source build](./source-build.md)
- [Security](../cross-cutting/security.md)
- [Versioning & pinning](../cross-cutting/versioning-pinning.md)
