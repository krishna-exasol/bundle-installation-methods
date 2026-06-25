# Scoop

> **One-liner:** A script-based, user-scope Windows package manager that installs apps from JSON manifests in "buckets" into your home directory, with no admin rights and no central approval for buckets you control.

| | |
|---|---|
| **Category** | OS package manager (Windows), user-scope |
| **Platforms** | Windows 10/11 (PowerShell 5.1+). Works without admin; some apps offer an optional global (admin) install. |
| **Prerequisites** | PowerShell with `RemoteSigned` execution policy; Git is recommended (and required to add buckets). No admin needed. |
| **Handles** | acquire ✓ / verify ✓ / place ✓ / configure ? / start ✗ / update ✓ / uninstall ✓ |
| **Maturity fit** | MVP / Growth |
| **Trust model** | Manifests pin a `hash` per download; the `main`/`extras` buckets are community-moderated, but **any third-party bucket you publish has no central review** — trust is whatever the bucket owner provides. |

## How it works
Scoop installs into the user's profile (`~\scoop\apps`, with shims in `~\scoop\shims` added to PATH) so it never needs administrator rights and never touches the registry or `Program Files` by default. Each app is a **JSON manifest** describing a download URL, a `hash`, what to extract or shim, and optional pre/post-install scripts. Manifests are grouped into **buckets**, which are just Git repositories of `.json` files.

Out of the box Scoop knows the `main` bucket (CLI tools) and you can add curated ones like `extras`, `versions`, `nonportable`, and `java`. Crucially, you can also add *your own* bucket from any Git URL. Adding a bucket is a client-side action with no gatekeeper — if you control the repo, you control the package, and users opt in by running `scoop bucket add`.

Install resolves a manifest, downloads the artifact, verifies it against the manifest `hash`, extracts it under `~\scoop\apps\<name>\<version>`, and creates shims for declared binaries. Updates are `scoop update`; because installs are versioned directories, rollbacks and side-by-side versions are cheap. Manifests can run `installer`/`post_install` script blocks, which is where light configuration can happen — but Scoop favors portable, no-installer apps, so service registration and heavy setup are awkward.

## Example
```powershell
# User side: install Scoop, add the publisher's bucket, install the bundle
Set-ExecutionPolicy -Scope CurrentUser RemoteSigned
Invoke-RestMethod -Uri https://get.scoop.sh | Invoke-Expression

scoop bucket add exasol https://github.com/exasol/scoop-bucket
scoop install exasol/exasolai

scoop update exasolai
scoop uninstall exasolai
```

Publisher responsibilities. Create a Git repo (e.g. `exasol/scoop-bucket`) containing one JSON manifest per app:

```json
{
  "version": "1.0.0",
  "description": "Exasol Nano + MCP Server + JSON Tables CLI bundle",
  "homepage": "https://example.com/exasolai",
  "license": "Proprietary",
  "architecture": {
    "64bit": {
      "url": "https://downloads.example.com/ExasolAI-1.0.0-x64.zip",
      "hash": "sha256:8f3a...REAL_HASH...c21d"
    }
  },
  "bin": "exasolai.exe",
  "post_install": [
    "& \"$dir\\setup\\install-mcp.ps1\""
  ],
  "checkver": "github",
  "autoupdate": {
    "architecture": {
      "64bit": { "url": "https://downloads.example.com/ExasolAI-$version-x64.zip" }
    }
  }
}
```
No PR to a central repo is required for your own bucket; `checkver`/`autoupdate` let bots bump versions and recompute hashes automatically.

## Pros
- No admin rights and no registry pollution — everything lives under the user profile, trivially removable.
- You own your bucket: ship a new version by pushing a commit, with zero external approval or review latency.
- `checkver`/`autoupdate` automate version bumps and hash updates from CI or a bot.
- Versioned install directories make rollback and side-by-side versions easy.
- Lightweight for CLI tools — ideal for the Rust JSON Tables CLI and other portable binaries.

## Cons
- Smaller, more developer-centric user base than winget/Chocolatey; end users may have never heard of it.
- A self-hosted bucket has **no independent vetting** — security rests entirely on the publisher, and users must trust your repo when they `bucket add`.
- Built for portable apps; bundling a database engine plus a pip package plus a Rust CLI into one Scoop manifest means leaning on `installer`/`post_install` scripts, which Scoop only loosely supports and does not orchestrate well.
- No real service lifecycle: Scoop will not register or start a Windows service for you.
- Getting into the official `extras` bucket *does* require a PR and review; only your private bucket is approval-free.

## Security considerations
Every manifest pins a per-download `hash` (sha256 by default), so a tampered artifact is rejected before extraction. The official `main`/`extras` buckets are community-moderated on GitHub. However, a third-party bucket is just a Git repo: there is no central review of your manifests, so the trust model for your bundle is "trust the publisher's repo." Mitigate by signing your artifacts, serving downloads over HTTPS, and keeping the bucket repo access-controlled. Scoop runs in user scope, which limits blast radius compared to admin-level installers, but a malicious `post_install` script still runs with the user's privileges.

## Approval & maintenance burden
For your own bucket: essentially zero approval — create the repo, write the manifest, tell users to `scoop bucket add`. Maintenance is a commit per release, or fully automated via `checkver`/`autoupdate`. For inclusion in the official `extras` bucket, expect a GitHub PR and maintainer review with manifest-style feedback. Overall the lowest publishing friction of the three Windows managers, at the cost of reach and independent vetting.

## Best for / Avoid when
**Best for:** developer tooling and portable CLIs (like the JSON Tables binary) where you want fast, admin-free, self-published distribution. **Avoid when:** you target non-technical Windows users, need a Windows service installed/started, or require third-party security vetting before users trust the install.

## Real-world examples
Scoop's own `main` and `extras` buckets carry tools like `gh`, `ripgrep`, `fzf`, `nodejs`, `python`, and `7zip`; many projects (e.g. Hugo, Neovim, Starship) publish their own buckets for early or portable builds.

## See also
- [Comparison matrix](../02-comparison-matrix.md)
- [winget](winget.md)
- [Chocolatey](chocolatey.md)
- [Native installers](native-installers.md)
- [Security](../cross-cutting/security.md)
