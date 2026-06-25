# Glossary

Short definitions of terms used throughout this catalog. Alphabetical.

**AMI (Amazon Machine Image)** — A snapshot used to boot EC2 instances on AWS;
one way to ship a pre-baked bundle as a cloud image.

**Authenticode** — Microsoft's code-signing technology for Windows executables
and installers (`.exe`/`.msi`), verified with a code-signing certificate.

**BOM (Bill of Materials)** — A list of the parts that make up a product; see
*SBOM* for the software-specific form.

**Channel** — A named release stream (e.g. `stable`, `beta`, `nightly`) that a
user subscribes to; typically a moving alias that resolves to a fixed version.

**Confinement (snap)** — The sandboxing mode of a Snap package (`strict`,
`classic`, or `devmode`) controlling how much of the host it may access.

**console_script / entry point** — A Python packaging mechanism that installs a
named command-line executable wired to a function in the package.

**cosign / Sigstore** — Tooling for signing and verifying artifacts (images,
blobs) using short-lived, identity-bound certificates instead of long-lived keys.

**Digest** — A content hash (e.g. `@sha256:…`) that names the exact bytes of an
image or blob; immutable, unlike a tag.

**distroless** — A minimal container base image containing only the application
and its runtime — no shell, no package manager. Can't `RUN apt` at build time.

**Formula** — A Homebrew package definition (a Ruby file) describing how to
download, build, and install a piece of software.

**Gatekeeper** — The macOS security feature that checks an app is signed and
notarized before allowing it to run.

**GPG** — GNU Privacy Guard; used here to create and verify detached signatures
over release artifacts and to sign package repositories.

**host.docker.internal** — A special DNS name that lets a container reach a
service running on the host machine; used by the case study's companion
containers to connect to a database running on the host.

**Idempotent** — A property of an operation that produces the same result no
matter how many times it runs; an idempotent installer is safe to re-run.

**Lockfile** — A file pinning the exact resolved versions of every (transitive)
dependency, e.g. `package-lock.json`, `Cargo.lock`, `uv.lock`, `poetry.lock`.

**Manifest** — A metadata file describing a package or deployment (e.g. a Winget
manifest, a Kubernetes manifest, a container image manifest).

**notarization** — Apple's process of uploading software to its notary service
for an automated malware scan, required for distribution outside the App Store.

**nuspec** — The XML manifest describing a NuGet/Chocolatey package (id, version,
dependencies, files).

**OIDC (OpenID Connect)** — An identity layer over OAuth2; used in keyless signing
so a CI job or user can authenticate to obtain a short-lived signing certificate.

**OpenTofu / Terraform** — Infrastructure-as-code tools that provision cloud
resources from declarative configuration; one way to deploy a bundle to the
cloud.

**Portal (flatpak)** — A sandboxed interface that lets a Flatpak app request
specific host resources (files, devices) without broad access.

**Provenance** — Verifiable metadata describing how, where, and from what inputs
an artifact was built; the basis of supply-chain trust (see *SLSA*).

**SBOM (Software Bill of Materials)** — A machine-readable inventory of every
component and version in a piece of software; common formats are CycloneDX and
SPDX.

**Sidecar** — A secondary container that runs alongside a main one to provide
supporting functionality (proxy, adapter, helper).

**SLSA** — *Supply-chain Levels for Software Artifacts*; a framework grading the
trustworthiness of build provenance (Build track levels L0–L3).

**Tag** — A human-readable, mutable pointer to an image or git commit (e.g.
`:latest`, `:8.34.0`, `v1.2.0`); a publisher can re-point it, unlike a digest.

**Tap** — A third-party Homebrew repository of formulae you can add with
`brew tap`, letting a project ship via Homebrew without upstream review.

**Wheel** — A pre-built Python distribution format (`.whl`) that installs without
a build step; absent a wheel, pip may need a compiler or source toolchain.
