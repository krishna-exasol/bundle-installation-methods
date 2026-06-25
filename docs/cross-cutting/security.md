# Security: trust, verification, and least privilege for bundle installers

Installing a bundle means running someone else's code on your machine and giving
it network and (often) data access. This page covers the trust questions every
install method raises, the *honest* mitigations for each, and a checklist you can
apply before shipping or before running an installer.

The themes here cut across every method in the catalog — see the
[comparison matrix](../02-comparison-matrix.md) for how individual methods score
on trust, and the [script-pipe](../methods/script-pipe.md) and
[Docker Compose](../methods/docker-compose.md) pages for method-specific notes.

## The pipe-to-shell trust problem

The canonical install one-liner is:

```sh
curl -fsSL https://example.com/install.sh | sh
```

The objection is real: **you run code you never read.** The bytes go straight
from the network into a shell. If the server is malicious or compromised, you've
executed arbitrary code as your user.

What is *honestly* true and false about the common criticisms:

- **"You run unseen code."** True. This is the core cost and it is not nothing.
- **"The server can detect `curl` and serve different bytes than a browser
  sees."** Technically true but mostly a *myth in practice*. A server can branch
  on `User-Agent` or on whether the connection is a pipe, and a careful attacker
  could serve benign content to anyone inspecting and malicious content to anyone
  piping. The mitigation is simple: **download first, read, then run a local
  file** (below). The detection trick does not survive that workflow, because the
  bytes you read are the bytes you run.
- **"`curl | sh` is uniquely dangerous."** Mostly false. Downloading and running
  *any* installer, binary, `.deb`, container image, or `npm` package is the same
  trust decision: you are trusting a publisher and a transport. A signed `.pkg`
  or a Homebrew formula running an arbitrary `install` block is not categorically
  safer than a script you actually read. The real variables are **transport
  integrity**, **publisher authentication**, and **whether you can verify what
  you got**.

### Honest mitigations

1. **Always use HTTPS/TLS.** This authenticates the *server* and prevents an
   on-path attacker from tampering with the bytes. It does **not** prove the
   server itself is honest, only that you reached the real host uncorrupted.
2. **Read before you run.** Prefer a two-step flow over a blind pipe:

   ```sh
   curl -fsSL https://example.com/install.sh -o install.sh
   less install.sh        # inspect it
   sh install.sh
   ```
3. **Pin and verify.** Publish a checksum (and ideally a signature) for the
   script and document how to check it (below).
4. **Fail closed.** A good installer uses `set -euo pipefail`, checks every
   prerequisite, and aborts rather than half-installing.

## Checksums (SHA256)

A checksum lets a downloader confirm the bytes are exactly what the publisher
released — it catches corruption and naive tampering, but **only if the checksum
itself is delivered through a trusted channel** (a signed release, a separate
host, an OS package index).

Publish:

```sh
sha256sum install.sh > install.sh.sha256
# or: shasum -a 256 install.sh
```

Verify:

```sh
sha256sum -c install.sh.sha256      # Linux
shasum -a 256 -c install.sh.sha256  # macOS
```

On Windows: `Get-FileHash install.ps1 -Algorithm SHA256`.

A bare checksum on the same server as the artifact stops accidental corruption
but not a server compromise (an attacker who replaces the artifact replaces the
checksum too). Signing closes that gap.

## Code signing

Code signing binds an artifact to an identity using a key the OS or a trust store
already trusts. Unlike a checksum, a valid signature survives a compromised
download host.

- **Apple notarization (macOS).** Since macOS Catalina, software distributed
  outside the App Store must be **signed with a Developer ID and notarized** —
  uploaded to Apple's notary service (via `xcrun notarytool`; `altool` was
  retired Nov 2023) for an automated malware scan. The resulting ticket should be
  **stapled** (`xcrun stapler`) so Gatekeeper can verify it offline; otherwise an
  offline machine may refuse to launch the app.
- **Windows Authenticode.** Sign `.exe`/`.msi` with a code-signing certificate
  (`signtool`). Helps clear SmartScreen reputation; EV or OV certs differ in how
  quickly reputation builds.
- **GPG (cross-platform).** Detached signatures (`gpg --detach-sign`) over
  release artifacts, verified with the publisher's known public key. Common for
  Linux tarballs and source releases.

## Package signing

OS and registry-level signing automates verification for the user:

- **Sigstore / cosign.** Keyless signing for container images and blobs. The
  signer authenticates over **OIDC** (GitHub, Google, Microsoft); **Fulcio**
  issues a short-lived (~10 min) certificate bound to that identity, signing
  happens with an ephemeral in-memory key, and the event is recorded in the
  **Rekor** transparency log. There is no long-lived key to steal. Verify with
  `cosign verify --certificate-identity ... --certificate-oidc-issuer ...`.
- **Helm provenance.** `helm package --sign` produces a `.prov` file; consumers
  run `helm install --verify` / `helm verify`. Note GnuPG 2 stores keys in
  `.kbx`, which Helm cannot read — export to legacy `pubring.gpg`. See
  [Helm](../methods/helm-kubernetes.md).
- **apt / yum GPG.** Distro repositories are signed; the package manager rejects
  unsigned or mismatched packages automatically. See
  [Linux native packages](../methods/linux-native-packages.md).

## Supply-chain attacks

Even a perfectly signed artifact can be malicious if the *source* was poisoned:

- **Typosquatting.** A package named `reqeusts` or `exasol-mcp-srv` that
  impersonates the real one. Pin exact names and versions.
- **Dependency confusion.** A public package shadows a private internal name,
  so the resolver pulls the attacker's version. Use scoped/namespaced names and
  configure registry priorities.
- **Compromised maintainers.** A hijacked account or a malicious co-maintainer
  ships a backdoored release. Pinning to a known-good version + checksum limits
  blast radius.

Defenses that scale:

- **SBOM** — a Software Bill of Materials listing every component and version, in
  **CycloneDX** or **SPDX** format. Lets you answer "am I affected by CVE-X?"
  quickly.
- **Provenance / SLSA.** SLSA v1.0's **Build track** grades how trustworthy an
  artifact's provenance is: L1 (provenance exists), L2 (hosted build + signed
  provenance), L3 (hardened, isolated build platform). GitHub's
  `gh attestation verify <artifact> --owner <org>` checks SLSA build provenance
  generated in CI; GitHub-hosted runners reach Build L2 by default and L3 with a
  reusable workflow.

## Least privilege at runtime

Verification gets the right bytes onto the machine; least privilege limits what
those bytes can do:

- **Bind to `127.0.0.1`, not `0.0.0.0`.** A service that only needs local access
  should not be reachable from the network. The case-study MCP server on `:4896`
  binds to localhost.
- **Don't mount `docker.sock` unless you truly need it.** Mounting the Docker
  socket into a container grants root-equivalent control of the host. Only the
  [socket-bootstrap](../methods/docker-socket-bootstrap.md) pattern needs it, and
  it's a deliberate trade-off, not a default.
- **Read-only by default.** Run containers with read-only root filesystems and
  least-needed capabilities; expose read-only data paths where the workload only
  reads. The case-study MCP server is configured read-only against the database.
- **Scope credentials.** Generate per-install secrets, store them with tight file
  permissions, and never bake credentials into images or scripts.

## Practical checklist

For **publishers**:

- [ ] Serve every artifact over HTTPS.
- [ ] Publish SHA256 checksums alongside each release.
- [ ] Sign releases (GPG / cosign) and platform installers (notarize on macOS,
      Authenticode on Windows).
- [ ] Generate an SBOM (CycloneDX or SPDX) per release.
- [ ] Emit build provenance (SLSA / `gh attestation`) from CI.
- [ ] Default services to `127.0.0.1`, read-only, least-capability.
- [ ] Make the installer `set -euo pipefail`, idempotent, and uninstallable.
- [ ] Document a read-before-run path, not only the blind pipe.

For **users**:

- [ ] Download, read, then run — don't pipe blindly for anything privileged.
- [ ] Verify the checksum/signature against a trusted source.
- [ ] Pin an exact version or digest (see
      [versioning & pinning](versioning-pinning.md)).
- [ ] Check what ports/sockets/volumes the installer exposes.
