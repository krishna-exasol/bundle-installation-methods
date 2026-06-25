# Download → verify → run

> **One-liner:** The same hosted-installer idea as a [script pipe](script-pipe.md), but the user downloads the script (or binary), checks its SHA256 against a published value, and only then runs it.

| | |
|---|---|
| **Category** | Script installer (integrity-hardened) |
| **Platforms** | All |
| **Prerequisites** | A shell + a hashing tool (`shasum`/`sha256sum`/`Get-FileHash`, all in-box) |
| **Handles** | acquire ✓ · **verify ✓** · place ✓ · configure ✓ · start ✓ · update ✓ · uninstall ✓ |
| **Maturity fit** | MVP (hardened) → Growth |
| **Trust model** | The user verifies a cryptographic hash (ideally signed) before executing — closes the "unseen code" gap of a raw pipe. |

This is the **#1-ranked script method**: it keeps the convenience of a hosted installer while removing the blind-execution risk.

---

## How it works

Three explicit steps instead of one piped line:

1. **Download** the installer asset from a **pinned release** (a tag, not a moving branch).
2. **Verify** its SHA256 against a checksum you publish next to the release — and, better, verify a **signature** over that checksum (cosign / GPG / `gh attestation`).
3. **Run** it only if the hash matches.

Because the asset is pinned and hashed, the bytes a user runs are exactly the bytes you released — reproducible and auditable.

---

## Example

```bash
# macOS / Linux
VER=v0.1.0
curl -fsSLO https://github.com/krishna-exasol/exasol-personal-ai/releases/download/$VER/install.sh
curl -fsSLO https://github.com/krishna-exasol/exasol-personal-ai/releases/download/$VER/install.sh.sha256
shasum -a 256 -c install.sh.sha256        # must print "install.sh: OK"
sh install.sh
```

```powershell
# Windows PowerShell
$ver = "v0.1.0"
irm "https://github.com/krishna-exasol/exasol-ai/releases/download/$ver/install.ps1" -OutFile install.ps1
$expected = (irm "https://github.com/krishna-exasol/exasol-ai/releases/download/$ver/install.ps1.sha256").Split(" ")[0]
if ((Get-FileHash install.ps1 -Algorithm SHA256).Hash -ieq $expected) { .\install.ps1 } else { throw "checksum mismatch" }
```

Stronger still — verify provenance instead of a bare hash:

```bash
gh attestation verify install.sh --repo krishna-exasol/exasol-personal-ai
# or, with cosign keyless:
cosign verify-blob --signature install.sh.sig --certificate install.sh.pem install.sh
```

Producing the checksum at release time:

```bash
sha256sum install.sh > install.sh.sha256   # attach both to the GitHub Release
```

---

## Pros

- **Integrity guaranteed** — the user runs exactly what you released; a corrupted/MITM'd download fails the check.
- **Reproducible & pinned** — tied to a version tag, not a mutable branch, so behavior can't silently change.
- **Auditable** — the asset is on disk; it can be read, scanned, archived before running.
- Keeps **all the orchestration power** of a script installer.
- Pairs naturally with **signing/provenance** for a real supply-chain story.

## Cons

- **More steps** — three commands vs one; lower conversion than a clean pipe, so most projects show the pipe first and document this as the "secure install."
- **Still runs a script** — verification proves *authenticity*, not *safety*; a malicious-but-authentic script is still malicious.
- Requires you to **publish and maintain** checksums (and keys) with every release.
- Slightly more to **document** and for users to get right (copy/paste of multiple lines).

## Security considerations

- Verify a **signature over the checksum**, not just the checksum — a hash served from the same compromised host buys little on its own. Use [sigstore/cosign](../cross-cutting/security.md) or GPG, or GitHub's build provenance (`gh attestation`).
- Pin to an **immutable tag or commit**, never `main`.
- Publish checksums in the **release notes / release assets**, ideally also in a second channel (your site) to reduce single-point compromise.
- See [Security](../cross-cutting/security.md) for the full supply-chain checklist.

## Approval & maintenance burden

Low–moderate: no third party, but every release must generate and attach a checksum (and signature). Automate it in CI (GoReleaser, a release workflow) so it never drifts from the artifact.

## Best for / Avoid when

**Best for:** the "secure install" path you advertise next to the convenience pipe; security-conscious users and enterprises; any public release past the throwaway-demo stage.

**Avoid when:** you're still iterating hourly off `main` (use the plain [pipe](script-pipe.md) until you cut tagged releases); or your users need a fully managed package (then [Winget](winget.md)/[Homebrew](homebrew.md)).

## See also

- [Script pipe](script-pipe.md) — the one-line convenience form
- [GitHub Releases (binaries)](github-releases-binaries.md) — where pinned, checksummed assets live
- [Versioning & pinning](../cross-cutting/versioning-pinning.md) · [Security](../cross-cutting/security.md) · [Comparison matrix](../02-comparison-matrix.md)
