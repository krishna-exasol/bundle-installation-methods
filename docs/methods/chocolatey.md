# Chocolatey

> **One-liner:** A mature, PowerShell-driven Windows package manager whose packages are NuGet `.nupkg` archives wrapping install scripts, distributed via a moderated community feed or your own private feed.

| | |
|---|---|
| **Category** | OS package manager (Windows) |
| **Platforms** | Windows 7+/Server 2008+ with .NET Framework 4.8 and PowerShell. Most installs run elevated (admin). |
| **Prerequisites** | Admin PowerShell for the typical machine-wide install; .NET 4.8; the `choco` client (bootstrapped via a one-line script). |
| **Handles** | acquire ✓ / verify ✓ / place ✓ / configure ✓ / start ✓ / update ✓ / uninstall ✓ |
| **Maturity fit** | Growth / Mature |
| **Trust model** | Community feed packages are human + automated moderated and (since 2024) signed; checksums are required for remote downloads. Private/internal feeds have whatever trust you enforce. |

## How it works
A Chocolatey package is a NuGet package: a `.nuspec` metadata file plus a `tools\` folder containing PowerShell scripts — most importantly `chocolateyInstall.ps1` (and optionally `chocolateyUninstall.ps1`, `chocolateyBeforeModify.ps1`). The install script is full PowerShell, so it can download installers, verify checksums, run silent MSI/EXE installers, register services, write config, and add shims. This scripting power is exactly why Chocolatey handles configure and start where winget and Scoop do not.

The default source is the **Chocolatey Community Repository** (`community.chocolatey.org`). Publishing there means `choco pack` then `choco push`, after which the package enters a moderation pipeline: automated virus/scan and metadata validation, then human moderator review. Approval can take from a day to weeks depending on backlog and how clean the package is.

Alternatively — and commonly for vendors — you host an **internal/private feed** (a NuGet server, Nexus, ProGet, Artifactory, or just a folder/UNC share). Packages on your own feed bypass community moderation entirely; you own the trust boundary. Many enterprises ship the bundle this way for control and speed.

Because the package is a script, a Chocolatey package for the bundle can install the Exasol Nano engine, `pip install` the MCP Server into a managed environment, drop the JSON Tables Rust CLI, and wire them together in one `choco install` — the most capable of the three for true multi-component orchestration.

## Example
```powershell
# Bootstrap the client (admin PowerShell)
Set-ExecutionPolicy Bypass -Scope Process -Force
[System.Net.ServicePointManager]::SecurityProtocol = 3072
iex ((New-Object System.Net.WebClient).DownloadString('https://community.chocolatey.org/install.ps1'))

# User side
choco install exasolai -y
choco upgrade exasolai -y
choco uninstall exasolai -y

# From a private feed
choco install exasolai --source="https://nuget.internal.example.com/v3/index.json" -y
```

Publisher responsibilities. Author a `.nuspec` and a `chocolateyInstall.ps1`, then pack and push:

```powershell
# chocolateyInstall.ps1 (excerpt)
$ErrorActionPreference = 'Stop'
$packageArgs = @{
  packageName    = 'exasolai'
  fileType       = 'msi'
  url64bit       = 'https://downloads.example.com/ExasolAI-1.0.0-x64.msi'
  checksum64     = '8F3A...REAL_HASH...C21D'   # required for remote downloads
  checksumType64 = 'sha256'
  silentArgs     = '/qn /norestart'
  validExitCodes = @(0)
}
Install-ChocolateyPackage @packageArgs
# Then: install MCP pip package, place JSON Tables CLI, register service, etc.
```
```powershell
choco pack
choco push exasolai.1.0.0.nupkg --source="https://push.chocolatey.org/"   # community feed
# or push to your own NuGet/ProGet feed to skip moderation
```

## Pros
- Most capable orchestration of the three: arbitrary PowerShell means it can genuinely install, configure, and start a multi-component bundle in one command.
- Mature, widely deployed in enterprises; strong tooling (private feeds, `choco` config management, Chocolatey for Business features).
- Private feeds give you full control with no external review and instant publishing.
- Handles uninstall and upgrade lifecycle, including pre-modify hooks for services.
- Checksums are required for remote downloads on the community feed, closing the swapped-binary gap.

## Cons
- Community moderation latency is the worst of the three for first-time publishing — approval can take days to weeks.
- Power is a liability: a `chocolateyInstall.ps1` is arbitrary code, so package quality and safety depend heavily on the author; bad scripts can leave broken half-installs.
- Typically requires admin/elevation, raising the privilege blast radius compared to Scoop's user scope.
- More moving parts to maintain per release: nuspec versioning, install/uninstall scripts, checksum updates, and exit-code handling.
- The richest features (internal feeds at scale, CDN, package internalizer) are gated behind Chocolatey for Business licensing.

## Security considerations
Community-feed packages go through automated scanning (VirusTotal-style checks) plus human moderation, and remote downloads must declare a `checksum` that the install script verifies before execution. Since 2024 the community client and packages emphasize signing. Still, the install script runs as administrator with full PowerShell, so the effective trust unit is "the package author's code," not just a pinned binary — review the `tools\` scripts of any package you do not control. On a private feed you set the entire trust boundary: sign packages, restrict who can push, and pin checksums. The community install bootstrap itself executes a remote script, which organizations often replace with an internalized, audited bootstrap.

## Approval & maintenance burden
Community feed: pack, push, then pass automated validation and human moderation — budget for reviewer back-and-forth and a slow first approval. Each release is a new versioned package plus updated checksums and scripts. Private feed: no approval at all; push and it is live, at the cost of running and securing your own feed. Maintenance is heavier than winget/Scoop because you own install and uninstall logic as code that must stay correct across Windows versions.

## Best for / Avoid when
**Best for:** vendors that need genuine multi-component install + configure + service-start in one command, and especially enterprises shipping via a controlled private feed. **Avoid when:** you want admin-free user-scope installs (use Scoop), need the lowest publishing latency on the public feed, or cannot maintain PowerShell install/uninstall scripts over time.

## Real-world examples
The Chocolatey Community Repository distributes packages like `googlechrome`, `git`, `vscode`, `nodejs`, `python`, `7zip`, and `docker-desktop`; countless enterprises ship internal software through private ProGet/Nexus/Artifactory Chocolatey feeds.

## See also
- [Comparison matrix](../02-comparison-matrix.md)
- [winget](winget.md)
- [Scoop](scoop.md)
- [Native installers](native-installers.md)
- [Security](../cross-cutting/security.md)
