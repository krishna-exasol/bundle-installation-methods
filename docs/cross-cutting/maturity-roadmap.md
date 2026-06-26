# Maturity roadmap: sequencing install methods as a project grows

No project should try to ship every install method on day one, and none should
stop at a single `curl | sh` forever. This page lays out a staged path — what to
offer at each stage, and what *you* must build to support it — so you add trust
and discoverability as adoption justifies the cost.

The guiding rule: **offer 2-3 channels at every stage, and automate releases to
all of them.** Users have different constraints (no Docker, no Homebrew, locked-
down Windows); meeting a few of them cheaply beats a single "perfect" method.

See the [overview](../00-overview.md) for the underlying trade-off curve and the
[comparison matrix](../02-comparison-matrix.md) for per-method scores.

## The three stages

### Stage 1 — MVP: ship today, zero approval

Optimize for **time-to-ship** and **lowest prerequisites**. You control the whole
release; nobody has to approve anything.

- **Script-pipe installer** — `curl … | sh` / `irm … | iex`. Does prereq checks,
  config, multi-container startup, and helper-script generation that a bare
  compose can't. See [script-pipe](../methods/script-pipe.md).
- **Docker / Docker Compose** — for users who'd rather read the compose file and
  run it themselves. See [Docker Compose](../methods/docker-compose.md) and
  [docker run/pull](../methods/docker-run-pull.md).

What you must build: a hosted script, a published image (or compose file), and a
documented uninstall. That's it.

### Stage 2 — Growth: add verifiability and reach

Now adoption justifies investment in **trust** and a couple of **package-manager**
front doors that don't need a long approval cycle.

- **Verified release** — GitHub Releases with **SHA256 checksums** and a
  signature; a "download, verify, run" path alongside the pipe.
- **pipx / uvx / npx** — one-shot runs for the CLI components without polluting
  the user's environment. See [pip/pipx/uvx](../methods/python-pip-pipx-uvx.md)
  and [npm/npx](../methods/npm-npx.md).
- **Homebrew tap** — your own tap (`brew install you/tap/bundle`) ships
  immediately, no upstream review. See [Homebrew](../methods/homebrew.md).

What you must build: a CI release pipeline that produces checksums + signatures +
SBOM (see [security](security.md)), published packages, and a tap repo.

### Stage 3 — Mature: native, signed, discoverable everywhere

Invest in the methods that require **registry approval, signing certificates, and
ongoing maintenance** — they pay off in discoverability and trust at scale.

- **Winget** and **Homebrew core** — discoverable in the default catalogs;
  require submission/review. See [Winget](../methods/winget.md).
- **Native signed installers** — notarized `.pkg`/`.dmg` (macOS), Authenticode
  `.msi`/`.exe` (Windows).
- **deb/rpm repositories** — signed apt/yum repos for Linux fleets. See
  [Linux native packages](../methods/linux-native-packages.md).
- **Helm chart** — for Kubernetes users, with provenance. See
  [Helm](../methods/helm-kubernetes.md).

What you must build: signing certs (Apple Developer ID, Windows code-signing,
GPG), registry submissions and their review cycles, a hosted package repo, and the
maintenance to keep all of it green.

## Stage → methods → what you must build

| Stage | Methods offered | What you must build / own | Approval needed? |
|-------|-----------------|---------------------------|------------------|
| **MVP** | Script pipe; Docker / Compose | Hosted script, published image, uninstall | None |
| **Growth** | Verified release + SHA256; pipx/uvx/npx; Homebrew tap | CI release pipeline, checksums + signatures + SBOM, package publish, tap repo | None (self-owned) |
| **Mature** | Winget; Homebrew core; native signed installers; deb/rpm repos; Helm | Signing certs, registry submissions, hosted package repo, ongoing upkeep | Yes (registries, notary) |

## Principles for sequencing

- **Never regress friction.** Adding Stage 2/3 methods must not break the Stage 1
  one-liner. Keep the simple path working.
- **Automate releases to every channel.** One tag push should produce the image,
  the checksummed release, the tap bump, and (later) the package submissions.
  Manual per-channel releases rot fast.
- **Add trust before reach.** Checksums and signatures (Stage 2) should land
  before you chase catalog placement (Stage 3) — discoverability without
  verifiability just distributes unverified bytes more widely.
- **Let adoption pull the next stage.** Don't build a deb repo for three Linux
  users. Move to Stage 3 methods when real demand (and the maintenance budget)
  exists.
- **Always offer 2-3 channels.** At each stage, give users a choice that covers
  the common "I don't have X" cases — no Docker, no Homebrew, locked-down OS.

The case study sits at Stage 1 today (script-pipe over Docker Compose) with a
Stage 2/3 roadmap of pinned digests, checksummed releases, and eventual
Homebrew/Winget — see the [case study](../case-studies/exasol-ai.md).
