# GitHub Releases Binaries (prebuilt assets)

> **One-liner:** Attach prebuilt, per-platform binaries (plus a `checksums.txt` and signatures) to a GitHub Release and let users — or a script — download the right asset directly from the release page or API.

| | |
|---|---|
| **Category** | Prebuilt binary distribution via a release/asset host |
| **Platforms** | Any OS/arch you cross-compile for (linux/amd64, linux/arm64, darwin/arm64, windows/amd64, …) |
| **Prerequisites** | A cross-compiling/release pipeline (often GoReleaser or a GitHub Actions matrix); `gh`/`curl` on the consumer side; optional `cosign`/checksums for verification |
| **Handles** | acquire ✓ / verify ✓ (checksums + provenance) / place ? (you/the script move it onto PATH) / configure ✗ / start ✗ / update ? (re-download / wrapper script) / uninstall ✗ (delete the file) |
| **Maturity fit** | MVP → Mature (cheap to start, scales to large projects) |
| **Trust model** | `SHA256SUMS`/`checksums.txt` for integrity; **Sigstore/cosign** signatures or **GitHub artifact attestations** (`gh attestation`) for provenance; HTTPS + GitHub's TLS for transport. |

## How it works
A GitHub Release is a tag plus a set of uploaded **assets** — arbitrary files, each with a stable download URL (`/releases/download/<tag>/<asset>`) and queryable via the Releases API and the `gh` CLI. The standard pattern is to build one self-contained binary (or a small tarball/zip) per `os/arch`, name them predictably (`exa-bundle_1.2.0_linux_amd64.tar.gz`), and upload them alongside a single `checksums.txt`. Tools like **GoReleaser** automate the cross-compile → archive → checksum → sign → upload flow in CI.

Consumers acquire assets three ways: click the release page, `curl` the direct URL, or `gh release download`. Crucially, **these release assets are the backbone that most "curl | sh" installers download** — a one-line script just resolves the OS/arch, fetches the matching asset, verifies the checksum, and moves it onto `PATH` (see [Script-pipe installers](./script-pipe.md)). Verification scales from a plain `sha256sum -c` up to **cosign** keyless signatures or **`gh attestation verify`**, which proves the binary was built by your GitHub Actions workflow.

For the **bundle**, the compiled Rust/Python `json-tables` CLI maps perfectly onto a single static binary per platform. The pip-based MCP server can be shipped as a frozen single-file executable (PyInstaller) so it needs no Python on the host. The **Exasol Personal/Nano** database is a container, not a binary, so the "binary" you release is realistically a small **launcher** that pulls the digest-pinned image — the release is great for the CLIs and bad for the DB engine itself.

## Example
```bash
# ---------- Producer: GoReleaser builds, checksums, and signs on tag ----------
# .goreleaser.yaml (excerpt): builds matrix + sboms + signs with cosign keyless
#   builds: [{goos: [linux,darwin,windows], goarch: [amd64,arm64]}]
#   checksum: {name_template: 'checksums.txt'}
#   signs:   [{cmd: cosign, signature: '${artifact}.sig', certificate: '${artifact}.pem'}]
goreleaser release --clean        # cross-compiles, writes checksums.txt, uploads assets
# Or generate a provenance attestation in the workflow:
#   - uses: actions/attest-build-provenance@v1
#     with: { subject-path: 'dist/*' }

# ---------- Consumer: download the right asset + verify integrity ----------
VER=1.2.0; OS=$(uname -s | tr A-Z a-z); ARCH=$(uname -m | sed 's/x86_64/amd64/')
BASE="https://github.com/example/exa-bundle/releases/download/v$VER"
curl -fsSLO "$BASE/exa-bundle_${VER}_${OS}_${ARCH}.tar.gz"
curl -fsSLO "$BASE/checksums.txt"
# Integrity: confirm the SHA256 matches the published list
sha256sum --ignore-missing -c checksums.txt

# Provenance (pick one):
cosign verify-blob --certificate-identity-regexp '.*' \
  --certificate-oidc-issuer https://token.actions.githubusercontent.com \
  --signature exa-bundle_${VER}_${OS}_${ARCH}.tar.gz.sig \
  exa-bundle_${VER}_${OS}_${ARCH}.tar.gz
gh attestation verify exa-bundle_${VER}_${OS}_${ARCH}.tar.gz --repo example/exa-bundle

# Place it on PATH yourself (no installer wraps this step):
tar -xzf exa-bundle_${VER}_${OS}_${ARCH}.tar.gz
sudo install -m 0755 json-tables exa-bundle /usr/local/bin/
```

## Pros
- Free, reliable, CDN-backed hosting that comes with the repo — no infrastructure to run.
- Self-contained binaries: no runtime/dependency resolution on the user's machine.
- First-class verification options: `checksums.txt`, cosign keyless signing, and GitHub build-provenance attestations.
- The release URL/API is the download source most script-pipe and package-manager formulas already consume.
- Trivial cross-platform reach via a CI build matrix or GoReleaser.

## Cons
- **Acquire-and-place only**: nothing configures services, manages config, starts daemons, or uninstalls — that is left to the user or a wrapper script.
- No automatic updates; users must re-download (or run a self-updating wrapper).
- You own the cross-compilation matrix and must keep it green for every target.
- Containerized components (the Exasol DB) do not fit the "single binary" model and need a launcher shim.
- Easy to ship assets **without** signatures/checksums, leaving consumers no way to verify.

## Security considerations
Always publish a `checksums.txt` and document how to verify it; better, add **provenance**: cosign keyless signatures or `actions/attest-build-provenance` so users can prove the artifact came from your specific workflow (`gh attestation verify`). Build in CI from a pinned, auditable workflow rather than uploading from a laptop. Note that release assets are **mutable** — an attacker with push/release rights (or a compromised token) can replace a file under the same URL, so signature/attestation verification, not just "it downloaded," is what protects users. Pin to an immutable tag and never to `latest` in automation. See [Security](../cross-cutting/security.md) and [Versioning & pinning](../cross-cutting/versioning-pinning.md).

## Approval & maintenance burden
No third-party approval — you publish on your own repo. Maintenance is the **release pipeline**: keeping the cross-compile matrix building, regenerating checksums/signatures, and writing release notes. Adopting GoReleaser + GitHub Actions makes each release a tag push, which is low ongoing effort. The hidden cost is the *consumer-side* tooling you also tend to ship (an install script, a package-manager formula) so users do not have to hand-place binaries.

## Best for / Avoid when
**Best for:** CLI tools and self-contained binaries (a great fit for the `json-tables` CLI); projects that want a free, verifiable download backend that script-pipe installers and Homebrew/Scoop formulas can point at; teams that already build in GitHub Actions. **Avoid when:** the product needs real install-time configuration, service registration, or auto-updates out of the box; the primary component is a container/daemon (use Compose/packages); or your users expect a double-click GUI installer.

## Real-world examples
- `kubectl`, `helm`, `gh`, `terraform`, and `ripgrep` all publish per-platform binaries + `checksums.txt` (often `.sig`/SBOMs) on releases.
- GoReleaser itself, `cosign`, and `syft` ship signed release assets with cosign + SLSA provenance.
- Most `curl … | sh` installers (rustup, k3s, nvm, Deno) ultimately resolve OS/arch and download a GitHub (or equivalent) release asset.

## See also
- [Comparison matrix](../02-comparison-matrix.md)
- [Script-pipe installers](./script-pipe.md)
- [Native OS installers](./native-installers.md)
- [Source build](./source-build.md)
- [Security](../cross-cutting/security.md)
- [Versioning & pinning](../cross-cutting/versioning-pinning.md)
