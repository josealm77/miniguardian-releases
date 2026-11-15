# Host supply-chain scan

`mg-supply host-scan` checks the software on this machine for known
vulnerabilities and for tampered system files. It writes a report. That is
all it does:
- it never blocks, upgrades or deletes anything;
- **it never sends anything anywhere**. The report stays on this machine.
  If you choose to report a finding to a maintainer, you do that yourself.

## What it checks

- **Your projects:** `Cargo.lock`, `package-lock.json`, `uv.lock`,
  `requirements.txt` under every home folder. Build output (`target/`,
  `node_modules/`) and tool caches are skipped.
- **Installed tools:** Rust tools (`~/.cargo/.crates2.json`), Python packages
  (`*.dist-info`) and global npm packages.
- **System packages (with `sudo`):**
  - the installed list, checked against your distribution's advisories
    (Ubuntu and Debian families);
  - every installed file, checked against what its package shipped
    (`dpkg --verify` / `rpm -V`). Configuration files you have edited are
    not counted.

It reads lockfiles and package metadata directly. It never runs `npm`,
`pip`, `cargo` or any package's code.

## Commands

| Command | What it does |
|---|---|
| `sudo mg-supply host-scan --refresh` | Downloads the newest advisory database, then scans. |
| `sudo mg-supply host-scan` | Scans with the database you already have. |
| `mg-supply host-scan` | Scans only your own home folder (no system packages). |
| `… --full` | Prints the whole report instead of the short summary. |
| `… --json` | The same report as JSON. |
| `… --out report.txt` | Also saves the report to a file. |
| `… --no-system` | Skips system packages. |
| `sudo mg-supply host-scan --schedule weekly` | Runs it automatically each week (or `daily`). Off until you turn it on. |
| `sudo mg-supply host-scan --schedule off` | Stops the schedule. |
| `sudo mg-supply host-scan --clean` | Deletes the saved report, the summary and the downloaded database, and says how much space was freed. |

**Files don't pile up.** Each scan replaces the previous report
(`/var/lib/jayce/miniguard/host-scan-report.txt`) and summary. Each refresh
replaces the database (`/var/lib/jayce/miniguard/advisories/`). `--clean`
removes all three.

## The report

The terminal shows a short summary: one line per vulnerable package version,
worst first, and how many places it was found. The full report (every
advisory and every location) is saved to
`/var/lib/jayce/miniguard/host-scan-report.txt` when run with `sudo`, or
printed with `--full`. The same lockfile copied into many folders is listed
once, with all its locations.

- **VULNERABLE:** a package version with a known advisory. It lists the
  advisory id and severity, the version that fixes it, and a command you
  could run to update. The command is shown, never run.
- **TAMPERED:** a system file that differs from what its package installed.
  It shows a reinstall command and **where you could report it** if you
  choose to: your distribution's security team.
- **No fix yet:** a known advisory with no fixed version released for
  your release. Listed so you know, but there is nothing to install yet, so
  it never raises a pop-up.
- **Third-party system packages:** packages built outside your distribution
  (Pop!_OS/System76 kernels, drivers, PPAs) use their own version numbers,
  which the distribution's advisories don't track. They are listed under
  Not checked; keep them current with your system updater.
- **Severity:** your distribution's own priority when it gives one,
  otherwise the advisory database's rating, otherwise one computed from the
  CVSS v3 score.
- **Unmaintained:** Rust crates whose advisory only says the crate is no
  longer maintained. There is no fixed version, so they are listed
  separately and never counted as vulnerable or shown as a pop-up.
- **Not checked:** anything that could not be checked, and why. For
  example: database missing or stale, a version that could not be compared,
  a file too large to read, or not run as root.
- **Attribution and licences:** every advisory source used, with its
  licence. Keep this section when you share a report.

## Data source and licences

Advisories come from [OSV](https://osv.dev), downloaded from
`osv-vulnerabilities.storage.googleapis.com`. OSV includes the GitHub
Advisory Database, RustSec, the PyPA advisory database, and Ubuntu and
Debian security data, each under its own licence. The report names the ones
it used. Fedora and other distributions without an OSV feed get the
integrity check only. Withdrawn advisories are ignored, and only advisories
for packages actually found on the machine are read, so a scan stays light
even though the downloaded database is large (the Ubuntu file alone is
several hundred MB).
