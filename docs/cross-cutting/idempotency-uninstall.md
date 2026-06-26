# Idempotency & uninstall

An installer is software users run under pressure, often more than once, and
eventually want gone. Two properties separate a trustworthy installer from a
fragile one: it is **safely re-runnable** (idempotent), and it offers a **clean,
documented uninstall**. This page covers both.

See also [versioning & pinning](versioning-pinning.md) for the upgrade path and
the [case study](../case-studies/exasol-ai.md) for a concrete `uninstall.sh`.

## Why installers must be idempotent

Users re-run installers constantly: after a failed first attempt, to upgrade, to
repair a half-broken state, or because they simply forgot they already ran it. An
**idempotent** installer produces the same end state no matter how many times it
runs, with no duplicate side effects.

A non-idempotent installer fails in characteristic ways:

- Appends the same line to `PATH` / shell rc files on every run.
- Tries to create a container/volume/service that already exists and errors out.
- Re-downloads and overwrites config the user has since customized.
- Starts a second copy of a service that's already running.

The fix is a consistent pattern: **detect existing state → reconcile to the
desired state → only then act.**

## Writing idempotent shell

```sh
set -euo pipefail

# Create a directory only if missing (mkdir -p is itself idempotent)
mkdir -p "$HOME/.exasol-ai"

# Add to PATH only if not already present
LINE='export PATH="$HOME/.exasol-ai/bin:$PATH"'
RC="$HOME/.bashrc"
grep -qxF "$LINE" "$RC" || printf '%s\n' "$LINE" >> "$RC"

# Reconcile a container instead of blindly creating it
if docker ps -a --format '{{.Names}}' | grep -qx exasol-mcp; then
  docker rm -f exasol-mcp >/dev/null
fi
docker run -d --name exasol-mcp ...
```

Principles:

- Prefer naturally idempotent primitives (`mkdir -p`, `ln -sf`,
  `docker compose up -d` which reconciles to desired state).
- Guard appends with a presence check (`grep -qxF … || …`).
- Make "already done" a success, not an error.

## Writing idempotent PowerShell

```powershell
$dir = "$env:LOCALAPPDATA\exasol-ai"
if (-not (Test-Path $dir)) { New-Item -ItemType Directory -Path $dir | Out-Null }

# Add to user PATH only if missing
$path = [Environment]::GetEnvironmentVariable('Path','User')
if ($path -notlike "*$dir*") {
  [Environment]::SetEnvironmentVariable('Path', "$path;$dir", 'User')
}
```

> Do **not** use `New-Item -Force` on an existing file — it truncates content.
> Use `Test-Path` guards instead.

## Clean uninstall

A bundle touches many places; a real uninstall has to reverse all of them.
Enumerate what was created and remove each:

- **Files & directories** — install dir, generated config, helper scripts,
  shell-completion files.
- **Containers** — `docker compose down` (and `--rmi local` if you also want the
  pulled images gone).
- **Volumes** — named volumes for data. **Handle deliberately** (see below).
- **Services / daemons** — stop and unregister launchd / systemd / Windows
  services.
- **PATH entries** — remove the lines/segments the installer added.
- **Networks** — any Docker networks the bundle created.

```sh
# uninstall.sh (sketch)
docker compose -f "$HOME/.exasol-ai/docker-compose.yaml" down   # stops + removes containers/network
# Keep the database volume by default — user data is precious
if [ "${PURGE_DATA:-0}" = "1" ]; then
  docker volume rm exasol-ai_db-data || true
fi
rm -rf "$HOME/.exasol-ai"
# Remove the PATH line we added
sed -i.bak '\#\.exasol-ai/bin#d' "$HOME/.bashrc"
```

## Keep user data by default

The single most important uninstall principle: **never delete the user's data
unless they explicitly ask.** Containers, binaries, and config are reproducible;
a database is not.

The case-study `uninstall.sh` follows this — it tears down containers, networks,
and files but **keeps the database volume by default**, only removing it behind an
explicit opt-in (`PURGE_DATA=1` / a `--purge` flag). Print exactly what was kept
and how to remove it:

```text
Removed: containers, network, helper scripts, PATH entry.
KEPT:    database volume 'exasol-ai_db-data' (your data).
         To delete it too:  PURGE_DATA=1 ./uninstall.sh
```

Make destructive purges loud, opt-in, and reversible-until-confirmed.

## Uninstall checklist

- [ ] Stop and remove all containers/services the bundle started.
- [ ] Remove Docker networks the bundle created.
- [ ] Remove installed files, helper scripts, and completions.
- [ ] Remove PATH / rc-file entries the installer added.
- [ ] **Keep data volumes by default**; gate deletion behind an explicit flag.
- [ ] Print a summary of what was removed and what was kept.
- [ ] Make uninstall itself idempotent — re-running after a partial removal must
      not error.
- [ ] Document the uninstall command in the same place as the install command.
