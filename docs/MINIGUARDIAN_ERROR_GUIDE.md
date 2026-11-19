# 🚨 MINIGUARDIAN & TUI ERROR TROUBLESHOOTING GUIDE
*Exhaustive Reference for System Events, TUI Warnings, Audit Alerts, & Interpreter Errors*

---

## 📋 Table of Contents
0. [First step: ask MiniGuardian what it decided and why](#0-first-step-ask-miniguardian-what-it-decided-and-why)
1. [TUI Status & Socket Connection Errors](#1-tui-status--socket-connection-errors)
2. [Audit Trail & Process Gate Event Alerts](#2-audit-trail--process-gate-event-alerts)
3. [Ransomware & Storage Protection Alerts](#3-ransomware--storage-protection-alerts)
4. [Apodeixis Code Interpreter & Proof Errors](#4-apodeixis-code-interpreter--proof-errors)
5. [Robot & Fleet Command Fabric (RCF) Errors](#5-robot--fleet-command-fabric-rcf-errors)
6. [Stuck in PARANOID, fleet and removal plans (beta-37)](#6-stuck-in-paranoid-fleet-and-removal-plans-beta-37)
7. [Quick Remediation Commands](#7-quick-remediation-commands)

---

## 0. First step: ask MiniGuardian what it decided and why

Before re-running tests or guessing, run:

```
sudo mg-explain            # last 30 minutes (or: 2h, 90s, a unix time)
sudo mg-explain 2h --dig   # follow the evidence down to a cause
```

It prints, from what the daemon already records: every change of the threat
state **with the events that caused it**, audit decisions, held kills and
network containment, downloads waiting for a decision, deception and seal
state, and recent warnings. `--dig` follows that evidence through fixed,
read-only checks to a one-line conclusion per trigger; a "stopped at" line
names what is not recorded yet. It changes nothing. Needs `socat` and
`python3` (installed with the package).

From beta-37 the report starts with **at a glance**: if the state is not
normal, what is holding it and why (for example `kernel judge absent … strict
mode stays Paranoid until it is present`, or a remembered container pattern),
then one line per part: judge, containment, audit chain, ransomware vault,
attack memory, trust link, fleet, pentest window, removal plan, held kills.
Repeated entries are collapsed (`×30 since 09:07`) and common harmless
warnings carry a short `why:` note. `MiniGuardian is still starting (no
operator socket yet)` right after a restart means: run it again in a minute.

In the TRust terminal, `build`, `check` and `test` do the same for code: a
failure prints a fix summary (cause, file and line, what was ruled out, the
one command that verifies a fix); `fixes` lists past summaries.

## 1. TUI Status & Socket Connection Errors

| TUI Output / Error Message | Root Cause | Resolution |
| :--- | :--- | :--- |
| `threat_level: disconnected` | The `miniguard` daemon process is not running or the socket `/run/jayce-operator/miniguard.sock` is inactive. | Start the daemon: `sudo systemctl start miniguard` or `sudo miniguard --foreground`. |
| `Permission denied (os error 13)` | `trust_terminal` or `miniguard-status` was executed without `sudo`. Token `/run/jayce-operator/miniguard.sock.token` is `0600 root-only`. | Always run TUI tools with `sudo`: `sudo trust_terminal`. |
| `seal: unknown` | 1. Daemon has just booted (first 120s tick pending).<br>2. `/etc/miniguard/seal.hashes` is missing. | Wait two minutes after boot; if it stays `unknown`, run `sudo mg-seal --installed` (source builds: `sudo bash scripts/seal.sh`). |
| `seal: mismatch` | Binary hash drift: `/usr/local/bin/miniguard` or `/usr/local/bin/miniguard-sentinel` hash does not match `/etc/miniguard/seal.hashes`. | Reinstall the official package (`sudo apt install --reinstall ./miniguard_<version>_amd64.deb`, or `sudo dnf reinstall ./miniguard-<version>.x86_64.rpm`), then `sudo mg-seal --installed`. |
| `failed to bind port 19876: Address in use` | Another instance of `miniguard` is already running in background. | Stop existing service first (`sudo systemctl stop miniguard`) or run status query instead of launching second daemon. |

---

## 2. Audit Trail & Process Gate Event Alerts

| Audit Log Alert Pattern | Meaning & Severity | Action Taken by Kernel |
| :--- | :--- | :--- |
| `[ESCALATE] process_gate FREEZE_EVENT pid=X detection=stale-heartbeat` | **HIGH**. Process `X` or supervisor thread stopped responding to 5s pulse heartbeats. | Sentinel logs freeze event and attempts systemctl unit restart if configured. Press `c`/`r` in TUI to clear feed once resolved. |
| `[ESCALATE] process_gate SEAL_VIOLATION daemon=true sentinel=false` | **CRITICAL**. High-privilege binary hash mismatch detected on periodic tick. | Activates `paranoid` self-protection mode. System locks immutable flags. Re-deploy official release binaries. |
| `[ESCALATE] shroud proto_anomaly: connection to IP:PORT` | **MEDIUM**. Unexpected outbound TCP/UDP network connection attempt outside active whitelist. | Logged in forensic audit chain. IP added to behavioral tracking feed. |
| `[WARN] shadow_backup RESTORE_EVENT path=/etc/X` | **HIGH**. Protected system file modification detected on disk. | MiniGuardian automatically restores original baseline file from `ShadowBackup`. |
| `[WARN] void_pit QUARANTINE_EVENT pid=X` | **HIGH**. Malicious process `X` moved to isolated Void Pit cgroup. | Process execution suspended and network access revoked. |

---

## 3. Ransomware & Storage Protection Alerts

| Ransomware Monitor Metric | Status | Meaning & User Guidance |
| :--- | :--- | :--- |
| `Ransomware  OFF` | **INACTIVE** | `fanotify_init` failed to initialize. Re-deploy latest musl binaries with tiered kernel fallbacks. |
| `Ransomware  ON  15 pids  682 events  0 high` | **ACTIVE (CALM)** | **Normal browser/system activity**. High event counts come from Chrome, LevelDB, or IndexedDB caching. **`0 high`** means zero threat! |
| `Ransomware  ON  X pids  Y events  3 high` | **HIGH ALERT** | **Suspicious bulk-encryption detected!** High write-rate entropy burst detected across processes. |

### A program stopped responding: MiniGuardian paused it (beta-27)

MiniGuardian keeps a copy of a file in your protected folders (chosen with
`sudo mg-setup`; workstation default: Documents, Desktop, Pictures) just
before it is overwritten. A program that overwrites 50 or more existing files
there within 5 seconds is **paused** at once (ransomware does exactly that).
Real package managers and backup tools, and your terminal shell itself, are
never paused. A root process doing this is paused and held for your decision
(it appears in the dashboard's approvals); on drone and vehicle profiles it is
only reported, so the control plane keeps running. Tools that legitimately rewrite many
files in place — whole-project formatters, code generators, some test suites —
can be paused.

```bash
sudo mg-op ransomware list                 # what was paused, with its pid
sudo mg-op ransomware explain <pid>        # every check the rule made, and which one decided
sudo mg-op ransomware resume <pid>         # it was yours: let it continue
sudo mg-op ransomware restore-since 30 /home/<you>/Documents/project   # put files back (only under that folder)
```

A paused program's binary is remembered: the same file is refused from any
path afterwards ("Operation not permitted"). If that was your own tool:
`sudo mg-op immunity list`, then `sudo mg-op immunity release <id>`.

### "profile not chosen — run sudo mg-setup"

MiniGuardian does not guess what the machine is for. `sudo mg-setup` asks
(workstation / server / drone / vehicle, how strict, management IPs for a
server, which folders get pre-write copies) and writes
`/etc/miniguard/profile.toml`; your `miniguard.toml` is not changed. Re-run it
any time; it only changes what you change.

### Why didn't MiniGuardian act?

`sudo mg-explain 15m` has a **near misses** section: detectors that came at
least halfway to acting (value / threshold). If the daemon stops starting,
the **crash trail** section shows the last startup stage it reached, the files
it was loading, and the crash lines from the journal.

---

## 4. Apodeixis Code Interpreter & Proof Errors

| Interpreter Error Message | Security Category | Explanation & Fix |
| :--- | :--- | :--- |
| `Unsafe block requires trust` | Sandbox Enforcement | Code attempted to run an `unsafe {}` block while running under `Untrusted` scope. Enclose logic in explicit `trust {}` block (outside sandboxes). |
| `Cannot escalate trust inside a sandbox` | Containment Guard | `trust {}` block was attempted inside a `sandbox {}` block. Sandboxed code runs isolated and cannot escalate privileges. |
| `Step budget limit exceeded` | Starvation Prevention | Snippet exceeded CPU step limit (`max_steps`), preventing infinite loops from starving the system. |
| `Call stack depth limit exceeded (max 128)` | Stack Overflow Guard | Recursive calls or nested closures exceeded 128 frames. Restructure recursion into iterative loops. |
| `Precondition verification failure in function 'X'` | Contract Verification | Function `requires` predicate evaluated to `false`. Ensure caller passes arguments meeting the function's pre-flight contracts. |
| `Postcondition verification failure in function 'X'` | Contract Verification | Function `ensures` predicate evaluated to `false`. Check internal calculation logic in function `'X'`. |
| `Unproven theorem blocks execution` | Formal Logic | Theorem proof check failed. Add formal tactics (`trivial`, `refl`, `induction`) to complete the proof. |

---

## 5. Robot & Fleet Command Fabric (RCF) Errors

| RCF / Fleet Error Message | Root Cause | Operator Action |
| :--- | :--- | :--- |
| `RCF mission rejected: thermal threshold exceeded` | Robot CPU/motor temperature exceeds safety limit. | Allow unit to cool down before submitting new actuation missions (`robot status`). |
| `RCF mission rejected: low battery reserve` | Unit battery is below minimum threshold (e.g. < 15%). | Return unit to charging dock or swap power module. |
| `RCF mission proposal rejected: unproven theorem` | Robot mission script failed formal Apodeixis safety proof. | Prove safety invariant using the Apodeixis verifier (`apodeixis eval`). |

---

## 6. Stuck in PARANOID, fleet and removal plans (beta-37)

| What you see | Cause | What to do |
| :--- | :--- | :--- |
| PARANOID for a long time after boot; `mg-explain` says `kernel judge absent (reason: lost)` | The kernel judge (a small VM) sent no telemetry in time. Before beta-37 the wait was 10 s; a slow or busy machine needs longer (18 s was measured on a laptop). In strict mode MiniGuardian stays Paranoid while the judge is absent. | Update to beta-37 or later (the wait is 60 s). The judge is launched once per daemon start: `sudo systemctl restart miniguard` relaunches it. If QEMU cannot run on the machine, set `[kernel_vm] judge_mode = "best-effort"`. |
| `fleet  off — …` in `mg-explain` | This machine is not enrolled in a fleet. | Optional. To join: `sudo mg-fleet init` on the first machine, `sudo mg-fleet add <name>` there, `sudo mg-fleet join <file>` on the other one, then `sudo systemctl restart miniguard` on both. See `TRUST_COMMAND_MANUAL.md` §1a. |
| `fleet  on (node …) — no peers yet` | Enrolled, but no other machine found yet. | Both machines need to be on the same local network, with TCP 19877 and UDP 19878 allowed between them, or add a peer by address: `sudo mg-fleet peer add <ip>`. |
| `fleet  FAIL … suspected` | Two or more of your other machines agree that one machine went silent, changed its key, replayed old messages or reported traffic its neighbours did not see. | Check that machine from the outside: `sudo mg-op fleet:glance` shows each finding. No machine acts on another; the response is yours. If you reinstalled it on purpose (new key), accept it with `sudo mg-op fleet:accept <node> key=<new key>` (or `build=<new build>` after an update), using the values `fleet:glance` shows. |
| `removal plan  WAIT N confirmed malware` | MiniGuardian confirmed rooted malware, froze it and blocked its file. | Read the `removal plans` section of `mg-explain`: reinstall the package if a system file was replaced, move an unknown file to quarantine, remove the startup entries it lists, then end the frozen process and change passwords. MiniGuardian runs none of these steps itself. If the plan reports signs of kernel-level access, check the machine from a USB boot or reinstall. |
| Docker builds or containers reported as rooted malware or as mounting `/dev` | False alarms fixed in beta-34 to beta-37. | Update. Real cases (your `/etc`, `/root`, the Docker socket mounted into a container) are still caught. |

---

## 7. Quick Remediation Commands

```bash
# 1. Clear TUI Audit Feed & Telemetry Counters:
# Inside 'sudo miniguard-status', press key 'c' or 'r'

# 2. Re-Seal Installed Binaries & Hashes (package install):
sudo mg-seal --installed
#    (from a source build: sudo bash scripts/seal.sh)

# 3. Dump Full Forensic Log for Diagnostic Ingestion:
sudo trust_terminal
trust@MiniGuardian$ forensic --json

# 4. Check Kernel & Service Watchdog Status:
sudo systemctl status miniguard
```

---
*MiniGuardian Error Guide — Keep this manual handy during TUI operation & fleet testing!* 🛡️⚡
