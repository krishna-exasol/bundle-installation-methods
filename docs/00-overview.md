# Overview: what "bundle install" means and how to judge a method

## The problem

A **bundle** is a product made of several moving parts that must end up working together on a user's machine. Examples:

- a database + an API server + a CLI
- a runtime + a language server + an extension
- the running case study here: **Exasol Personal/Nano + MCP Server + JSON Tables**

The goal of an installation method is to take a user from *"I have nothing"* to *"it's running and I can use it"* with the **fewest steps, least friction, and acceptable trust**, on the platforms your users actually run.

A good install method answers all of these:

1. **Acquire** the artifacts (binaries, images, scripts, packages).
2. **Verify** they are authentic and uncorrupted.
3. **Place** them where the system can find them (PATH, services, containers).
4. **Configure** them (ports, credentials, config files).
5. **Start / register** them (daemons, containers, shell completions).
6. **Allow upgrade and uninstall** later.

Methods differ enormously in how many of these they handle for you, and at what cost.

## The evaluation dimensions

Every method in this catalog is scored on the same axes. Keep these in mind when reading any method page.

| Dimension | Question |
|-----------|----------|
| **Friction** | How many commands / prerequisites before it works? |
| **Prerequisites** | What must already be installed (a package manager, Docker, a runtime)? |
| **Platforms** | Windows, macOS, Linux, cloud, all? |
| **Orchestration power** | Can it do checks, config, multi-component startup, helpers — or only drop a file? |
| **Trust / security** | Does the user run unseen code? Is there signing / checksum verification? |
| **Reliability** | How often does it "just work" across environments? |
| **Approval & maintenance burden** | Do you need a registry, review, signing certs, ongoing upkeep? |
| **Update & uninstall** | Is upgrading and removing first-class? |
| **Reproducibility** | Can a user pin an exact version and get a byte-identical install? |
| **Time-to-ship** | How fast can *you* publish it today? |

## The central tension

There is no single best method — there is a **trade-off curve** between two things:

- **Time-to-ship & friction** (good early): script pipes, Docker.
- **Trust, polish & discoverability** (good later): signed native installers, OS package managers.

Early on you optimize for *shipping and low friction*. As adoption grows you add *trust and discoverability*. Most successful projects **offer several methods at once** and let the user pick — see how the [maturity roadmap](cross-cutting/maturity-roadmap.md) sequences them.

## Categories

The 20 methods fall into a handful of families:

- **Script installers** — a hosted script does everything ([script pipe](methods/script-pipe.md), [verified](methods/download-verify-run.md)).
- **OS package managers** — [Homebrew](methods/homebrew.md), [Winget](methods/winget.md), [Scoop](methods/scoop.md), [Chocolatey](methods/chocolatey.md), [deb/rpm](methods/linux-native-packages.md), [Snap/Flatpak](methods/snap-flatpak.md).
- **Language package managers** — [pip/pipx/uvx](methods/python-pip-pipx-uvx.md), [npm/npx](methods/npm-npx.md), [cargo/go/gem](methods/other-language-managers.md).
- **Containers** — [run/pull](methods/docker-run-pull.md), [Compose](methods/docker-compose.md), [socket bootstrap](methods/docker-socket-bootstrap.md), [Helm](methods/helm-kubernetes.md).
- **Native & cloud** — [native installers](methods/native-installers.md), [GitHub Releases](methods/github-releases-binaries.md), [version managers](methods/version-managers.md), [IaC/cloud](methods/iac-cloud.md), [source build](methods/source-build.md).

Next: **[Decision Guide →](01-decision-guide.md)**
