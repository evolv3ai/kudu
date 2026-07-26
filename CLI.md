# CLI Mode

Kudu can run entirely from the command line — no GUI window is opened. This is useful for scripting, IT admin workflows, agent tooling, and scheduled tasks beyond the built-in scheduler.

```
kudu --cli <command> [subcommand] [options]
```

Running `kudu --cli --help` prints the same reference at any time.

## Contents

- [Invocation](#invocation)
- [Global options](#global-options)
- [JSON output contract](#json-output-contract)
- [Command reference](#command-reference)
- [Elevation](#elevation)
- [Exit codes](#exit-codes)
- [Prometheus metrics](#prometheus-metrics)
- [Daemon mode](#daemon-mode)
- [Scripting examples](#scripting-examples)

## Invocation

**Packaged build.** The Windows installer does not add Kudu to `PATH`; invoke the executable directly:

```powershell
& "$env:LOCALAPPDATA\Programs\kudu\Kudu.exe" --cli perf info --json
```

> **Windows elevation.** Packaged Windows builds ship with
> `requestedExecutionLevel: requireAdministrator` (see `electron-builder.yml`), so **every**
> invocation of the packaged `.exe` triggers a UAC prompt — including read-only scans that
> do not need administrator rights. For unattended use, either run from source (below) or
> drive the CLI from a Task Scheduler entry configured with *Run with highest privileges*.

**From source.** No manifest is applied, so the CLI runs unelevated with no UAC prompt:

```bash
npm install
npm run build
npx electron . --cli perf info --json
```

CLI mode does **not** take the single-instance lock (that lives inside `initGui()`), so CLI
invocations run fine alongside a running GUI or tray instance.

## Global options

| Flag | Description |
|------|-------------|
| `--json` | Emit machine-readable JSON on stdout |
| `--verbose` | Show detailed progress, timing, and debug info |
| `-q`, `--quiet` | Suppress all output except errors and the final result |
| `--all` | Select all items for action commands |
| `-h`, `--help` | Show help |
| `-v`, `--version` | Show version |

`--verbose` and `--quiet` are mutually exclusive.

## JSON output contract

With `--json`, **stdout carries nothing but a single JSON document.** Progress lines,
verbose diagnostics, and other human-facing chatter are written to **stderr**, so stdout can
be piped straight into a parser:

```bash
kudu --cli programs list --json | jq '.count'      # stdout: pure JSON
kudu --cli programs list --json 2>/dev/null        # discard progress entirely
kudu --cli programs list --json 2>progress.log     # keep progress separately
```

Errors are structured too, rather than free text:

```json
{ "error": "invalid_usage", "usage": "kudu --cli perf <info|disk-health|kill> [pid]" }
{ "error": "not_found", "type": "service", "name": "nope" }
```

Commands that cannot complete without elevation report it as **data with exit code 0**, not
as a failure — check the flag rather than the exit status:

```json
{ "totalBootMs": 0, "entries": [], "available": false, "needsAdmin": true }
```

## Command reference

Availability below was verified on Windows 11 running **unelevated**. Commands marked
*admin* return `needsAdmin: true` or exit `3` without elevation. Registry, services,
debloat, drivers, and privacy commands are Windows-oriented.

Commands marked ⏱ walk the filesystem and can run for many minutes on a large volume —
give them a generous timeout in scripts, or expect no output at all.

### File cleaners

The original flag-based form is still supported and is the default when no command is given.

```
scan  [--system] [--browser] [--app] [--gaming] [--recycle-bin] [--all]
clean [--system] [--browser] [--app] [--gaming] [--recycle-bin] [--all]
```

| Flag | Description |
|------|-------------|
| `--system` | System temp files, caches, logs, crash dumps |
| `--browser` | Browser caches (Chrome, Edge, Brave, Firefox, …) |
| `--app` | Application caches (Discord, VS Code, npm, …) |
| `--gaming` | Game launcher caches, GPU shader caches, redistributables |
| `--recycle-bin` | Windows Recycle Bin |
| `--all` | All categories (default when none specified) |

Without `--clean` (or the `clean` command) this is a dry run — nothing is deleted. Targets
flagged `needsAdmin` in `rules/` are skipped when unelevated rather than failing the run.

JSON shape:

```json
{
  "scan": {
    "categories": ["system", "browser"],
    "results": [
      {
        "category": "system",
        "subcategory": "User Temp Files",
        "itemCount": 42,
        "totalSize": 104857600,
        "items": [{ "path": "…", "size": 1024, "lastModified": 1700000000000 }]
      }
    ],
    "totalItems": 42,
    "totalSize": 104857600
  },
  "clean": { "totalCleaned": 104857600, "filesDeleted": 40, "filesSkipped": 2, "errors": [] }
}
```

The `clean` key is present only when cleaning was requested.

### Inventory and diagnostics

| Command | Description | Unelevated |
|---------|-------------|:----------:|
| `perf info` | CPU model, cores/threads, total memory, OS version, hostname | ✅ |
| `perf disk-health` | Per-device S.M.A.R.T. health (model, type, size, temperature) | ✅ |
| `perf kill <pid>` | Kill a process by PID | depends on target |
| `programs list` | Installed programs — publisher, version, size, install location, uninstall strings | ✅ |
| `startup list` | Startup items — command, registry location, source, enabled state, impact | ✅ |
| `startup boot-trace` | Boot time trace | *admin* |
| `disk drives` | Drives with total/free/used bytes | ✅ |
| `disk analyze <drive>` | Disk usage breakdown (e.g. `disk analyze C`) — ⏱ long-running | ✅ |
| `disk file-types <drive>` | File-type breakdown for a drive — ⏱ long-running | ✅ |
| `services scan` | Windows services — name, display name, description, state | ✅ |
| `drivers scan` | Old/unused driver packages | *admin* |
| `history list` | Scan history | ✅ |
| `metrics` | Current metrics in Prometheus text format | ✅ |

### Maintenance

| Command | Description | Unelevated |
|---------|-------------|:----------:|
| `registry scan` | Scan for registry issues | ✅ |
| `registry fix [--all]` | Fix found registry issues | HKCU ✅ / HKLM *admin* |
| `debloat scan` | Detect removable preinstalled apps | ✅ |
| `debloat remove <pkg,…>` / `--all` | Remove packages (comma-separated) | *admin* |
| `leftovers scan` | Uninstall leftovers, grouped by category | ✅ |
| `leftovers clean` | Remove found leftovers | per-path; skips what it cannot touch |
| `network scan` | DNS cache, Wi-Fi profiles, ARP cache — **counts only, not entries** | ✅ |
| `network clean [--all]` | Flush selected network caches | *admin* |
| `updates check` | Available updates via winget / Chocolatey / Scoop / npm | ✅ |
| `updates run <id,…>` / `--all` | Install updates | *admin* |
| `drivers check-updates` | Check for driver updates | ✅ |
| `drivers clean <name,…>` / `drivers update [--all]` | Remove or update driver packages | *admin* |
| `services disable <name>` / `services manual <name>` | Change service start type | *admin* |
| `startup disable\|enable\|delete <name>` | Manage a startup item | HKCU ✅ / HKLM *admin* |
| `history clear` | Clear scan history | ✅ |
| `restore-point create [description]` | Create a system restore point | *admin* |

> `network scan` reports **how many** DNS/ARP entries exist, not what they are. The platform
> layer can enumerate established connections and DNS entries
> (`PlatformNetwork.getEstablishedConnections`, `getDnsCacheEntries`), but that data is not
> currently exposed through the CLI.

### Security

| Command | Description | Unelevated |
|---------|-------------|:----------:|
| `malware scan` | YARA-based threat scan — ⏱ long-running; exits `7` when threats are found | not verified |
| `malware quarantine <path>` | Quarantine a detected file | depends on path |
| `malware delete <path>` | Delete a detected file | depends on path |
| `privacy scan` | Audit privacy/telemetry settings | ✅ |
| `privacy apply [--all]` | Apply recommended privacy settings | *admin* |
| `cve list` | Known CVE vulnerabilities — **requires a cloud API key**; exits `1` without one | n/a |

### Configuration and service management

| Command | Description |
|---------|-------------|
| `config get [key]` | Show settings (e.g. `config get cloud.apiKey`) |
| `config set <key> <value>` | Update a setting |
| `service install` | Install as a systemd service — **Linux only** |
| `service uninstall` | Remove the systemd service — **Linux only** |
| `service status` | Show service status — **Linux only** |

On Windows and macOS the `service` commands exit `2` and point you at Task Scheduler / NSSM
and launchd respectively.

## Elevation

Kudu detects elevation (`net session` on Windows, `uid === 0` elsewhere) but never
self-elevates. Behaviour when a privileged operation is attempted without rights:

- Read-only scans **succeed** and simply omit inaccessible targets.
- Structured results set `needsAdmin: true` and still exit `0`.
- Hard-blocked operations exit `3` (`Permission denied`).

Because the graceful path is built in, the whole inventory and scanning surface is usable
unelevated — the packaged Windows manifest, not the code, is what forces UAC.

## Exit codes

| Code | Meaning |
|------|---------|
| `0` | Success |
| `1` | General error |
| `2` | Invalid arguments |
| `3` | Permission denied (needs elevation) |
| `4` | Partial success (some operations failed) |
| `5` | Nothing found (scan returned zero items) |
| `6` | Unknown command |
| `7` | Threats/issues found requiring attention |

## Prometheus metrics

Print metrics for a `node_exporter` textfile collector:

```bash
kudu --cli metrics
kudu --cli metrics --json    # JSON array of metric objects
```

Run a persistent HTTP endpoint:

```bash
kudu --cli metrics-server              # default port 9100
kudu --cli metrics-server --port 9200
```

Endpoints: `/metrics` (Prometheus text) and `/health` (`{"status":"ok"}`). Metrics include
`kudu_info`, `kudu_system_uptime_seconds`, `kudu_system_cpu_usage_percent`, and
`kudu_system_memory_{total,used}_bytes`.

> **The server binds `0.0.0.0`**, exposing system metrics to the local network. Restrict it
> with a firewall rule, or bind it to loopback, before running it on an untrusted network.

Unlike `--daemon`, the metrics server needs no cloud API key — making it the practical way
to expose live system state to local tooling without paying Electron start-up cost per query.

## Daemon mode

```bash
kudu --daemon
kudu --daemon --api-key <key>     # store the key, then start
```

Headless cloud agent: sends telemetry and health reports, responds to remote commands, and
auto-updates. No GUI, no tray. It **requires a cloud API key** and exits `1` without one —
set it via `--api-key` or `kudu --cli config set cloud.apiKey <key>`. Heartbeats are logged
every five minutes; `SIGTERM`/`SIGINT` shut down gracefully.

## Scripting examples

```bash
# Dry-run everything, nothing deleted
kudu --cli scan --all

# Clean system junk only
kudu --cli clean --system

# Feed inventory straight into jq
kudu --cli programs list --json 2>/dev/null | jq -r '.programs[].displayName'

# Branch on threats found
kudu --cli malware scan --json > scan.json
[ $? -eq 7 ] && echo "threats detected — review scan.json"

# Detect an elevation-limited result rather than guessing from exit status
kudu --cli startup boot-trace --json | jq -e '.needsAdmin == true' >/dev/null \
  && echo "re-run elevated for boot trace"

# Scheduled task / cron
kudu --cli clean --all --quiet
```
