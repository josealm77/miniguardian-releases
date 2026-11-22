# MiniGuardian — Overview

*Status as of 2026-09-19, re-checked against the source 2026-09-26,
2026-11-13 (beta-31), 2026-11-19 (beta-37), 2026-11-20 (beta-38, beta-39) and 2026-11-21 (beta-40). The
2026-09-19 evidence is in an internal claims-audit report (not published;
reviewers can ask for it). Numbers carry
the date they were measured.*

## What it is

MiniGuardian is a security daemon for a Linux machine (from beta-37, your
machines can also watch each other as a fleet). It watches
processes, files, memory and network activity, rates the host as
**Calm → Suspicious → Hostile → Paranoid**, and can block, quarantine or
restore things when the rating rises. Its distinguishing feature is
**AI containment**: a wrapper (`mg-cli`) that runs an AI coding tool inside a
kernel sandbox, gates what it may do, and asks a human before anything
destructive happens.

The design idea is that the AI, or an attacker, is not the trust boundary. The
locked-down container around it is.

It is built on the **Jayce kernel**, a small deterministic kernel written in
Rust. The daemon uses the kernel's engine code in-process (a host-side port), and
the kernel can also run on its own in QEMU.

## Status at a glance

| Area | What it does | Status |
|---|---|---|
| Host protection daemon | Process, file, memory, network and browser-surface detectors feeding a threat "brain" | Implemented |
| Enforcement | eBPF and iptables blocking, kill, quarantine, restore from shadow copies | Implemented. By default it only blocks in Hostile/Paranoid; Calm and Suspicious observe. From beta-40 IPv6 too: remembered attackers' /64 (and /48 after three) and IPv6 flood throttling in the XDP program; flood mode not yet tested against a real flood |
| Antifreeze sentinel | Separate process that un-freezes a stopped daemon (SIGCONT) and restarts it after repeated freezes | Implemented |
| Sealing | Immutable binaries and a hash manifest, re-checked about every 2 minutes | Implemented |
| `mg-cli` governor | Runs an AI tool under Landlock: file sandbox plus **default-deny TCP** (kernel 6.7+), a proof-checked action gate, a fake-response layer for escape attempts, a dependency check before launch, and (under sudo, CLI tools) no access to the desktop: X server, session D-Bus and Wayland sockets are hidden so the agent cannot fake pointer or keyboard input | Implemented (desktop isolation added 2026-09-26) |
| Human approval | Queued actions (tool calls, unregistered tools, risky file actions) raise a desktop pop-up with **Approve**, **Deny** and **Review**; approving a session's file action asks for your password (pkexec), and the TUI's approvals page decides the same queue. From beta-31 a pop-up whose item was already decided asks no password | Implemented, in daily use |
| Dangerous approvals | Destructive commands, hidden code, credentials, persistence, root/sandbox escape and remote control are red, list their reasons, and are approved only by typing the item's name; no group, window or rule approves them | Implemented (beta-32) |
| Governed tools | 52 built-in tools; reading is automatic, writing, running, deleting, network and memory ingest wait for human approval | Implemented; all 52 tested end to end from beta-40 (network, desktop and push parts run outside a governed session) |
| Content scanners | Prompt-injection guard, memory-poisoning scanner, tool-call gate, exposed on a read-only local socket | Implemented (see limits) |
| Identity firewall | UID-scoped UDP policy for the governed account | Implemented, **off by default** (shadow mode first) |
| AI envelope (tagging, per-tool egress, GUI exec interception) | Detects AI tools and tags them | **Detection only**; the enforcement parts are planned |
| Jayce kernel | 44 deterministic engines, boots in QEMU from `artifacts/kernel/jayce-kernel.iso` | Implemented as source and a QEMU image |
| Kernel as a "judge" | The daemon launches the kernel in QEMU (as `nobody`, seccomp, no network) and every decision must agree with it | Implemented; on in the shipped config (`[kernel_vm] judge_mode = "strict"`) |
| Kernel vitals in the TUI | Live kernel counters on the `k` page of `miniguard-status` | Implemented; read from a serial log and marked unverified |
| TRust Smart Terminal (the TRust terminal) | Command shell with Prompt Guard on every input, 18 inline languages plus Apodeixis, and the governed tool system | **Partial.** The shell, analyzers and tools work; 42 of the 144 catalogued commands are not implemented here, and are marked as such (see below; counted from the code 2026-11-20). Standalone `trust_terminal` binary (v0.1.0, `miniguard --terminal` starts it) and linked into the daemon for `term:` commands. From beta-40 the interactive terminal hands lines it does not own to bash with the terminal attached (full shell syntax, full-screen programs, no time/output limit) after the same prompt guard and containment check; the agent path is unchanged |
| Cosmic Map | 2D fleet observatory | **Not available in this beta**: `miniguard --map` prints a "not available" message; the `cosmic_map` crate still builds a standalone binary that is not installed |
| Attack immunity | Every contained attack is remembered and refused at its first step next time, after a restart and under a new name or path (`sudo mg-op immunity list`) | Implemented (beta-27) |
| Campaign memory | Recognises an outside attacker by what it goes after and in what order, across IP, tool and timing changes; learns antibodies from a contained attacker (`sudo mg-op campaign list`) | Implemented (beta-33); **shadow mode by default** (reports only) until `[campaign] mode = "live"` |
| Rooted-malware removal plan | For confirmed rooted malware, `mg-explain` lists the steps to remove it (reinstall, quarantine, startup entries, kernel-level warning) | Implemented (beta-37); suggestions only, MiniGuardian runs none of them |
| `mg-explain` | What the daemon decided and why, starting with an "at a glance" health list | Implemented (beta-25; at a glance beta-37) |
| AI-edit trail | Every file change a governed AI session makes is recorded where the AI cannot change it | Implemented (beta-37); the code-map tool that reads it is not shipped yet |
| `mg-pentest` | A time-limited window in which named pentest tools may reach named targets from this machine | Implemented (beta-37); developer feature, needs a licence signed by Jayce Automata Research for the machine and an owner approval |
| Download guard | Executable downloads are quarantined until you release them | Implemented; from beta-38 files over ~100 MB are checked too and every decision is kept on disk |
| Open plugins | Plugins in any language in a sandbox; every action checked against what you approved at install (`mg-plugin`) | Implemented (beta-29) |
| Egress proxy | Plugin commands (and, opt-in, AI sessions) reach the internet only through MiniGuardian's proxy; new sites ask | Implemented (beta-29) |
| `mg-why` | Explains why a program failed, in any language, secrets masked | Implemented (beta-29); next steps for common errors in 7 languages and `suggest` in the terminal (beta-38); Linux system errors and, as root, the kernel log (beta-39) |
| Owner-signed antibody feed | Attack patterns you sign travel to your other machines; a match is reported and watched, never blocks by itself | Implemented (beta-38); bundles are copied by hand (fleet distribution not yet built) |
| Leak guard | Before a governed AI session publishes, credentials are refused and personal information is flagged | Implemented (beta-30) |
| Host supply-chain scan | `sudo mg-supply host-scan`: projects, tools and system packages against public advisories; system files against their packages; local report only | Implemented (beta-31) |
| Your own fleet | `mg-fleet` enrolls your machines; they find each other on the local network, exchange signed heartbeats and flag a machine that goes silent or changes (two must agree) | Implemented (beta-37); **not yet tested across two machines** |
| Master Console, fleet intelligence, swarm immunity | Cross-host dashboard, rule distribution, shared attack signatures | **Future plan.** Nothing is deployed (hosting needs paid infrastructure). A console prototype exists as a separate project outside this repo. The daemon can verify signed fleet rules (`[fleet]`, off by default) but has no console client |
| Bare-metal OS installer | Boot the kernel as the operating system | **Not shipped**: the installer script is a stub |
| Windows / macOS | | Not supported (Linux only) |

## How AI containment works

When you launch a tool through `mg-cli`, each step can stop the launch:

1. The command text goes past a **fake-response layer**. Commands on a blacklist
   (`rm -rf`, `curl`, `sudo`, reads of `/etc/shadow`...) get a plausible fake
   answer and no process is started, so an escape attempt looks like it worked
   while everything is logged.
2. A **privilege check** from the AI "constitution": an agent that has misbehaved
   has fewer tools and a smaller token budget.
3. A **proof-checked rule (Apodeixis mission)** decides allow / score / block for
   the action. If the proof fails, the action does not run.
4. A **risk score** maps to allow, audit, ask the operator, or block.
5. A **dependency check** (`mg-supply`) refuses to launch in a workspace with
   tampered or unpinned dependencies. It reads Rust, Python (`uv.lock`, hashed
   `requirements.txt`) and Node (`package-lock.json`) lockfiles; the launch
   check runs when the workspace has a `Cargo.toml`.
6. The tool starts inside **Landlock**: it can write only to its workspace and
   `/tmp`, reads outside the workspace are limited to system paths (`/usr`,
   `/etc`, `/proc`, …), and it cannot open TCP connections unless you allow
   specific ports (`--net-ports`, `--allow-dns`). The workspace is what you
   give it: `--workspace ~` (as in the documented aliases) makes your whole
   home directory the workspace. `--share-data` additionally grants the tool's
   own state directories (`~/.config`, `~/.local/share`, `~/.local/state`,
   `~/.cache`, and e.g. `~/.claude`) and read-only `~/.ssh` and `~/.gitconfig`.
   If the kernel cannot enforce Landlock, the launch is refused rather than
   silently unrestricted.
7. While it runs, a **file monitor** re-checks actions in the workspace.

Every tool call the AI makes goes through the same five gates: argument
validation, the proof-checked rule, a capability check, a **human approval**
step for anything that changes the system, and an audit record. The AI cannot
approve its own actions.

## How it protects itself

The daemon renames its process to look like a kernel worker, clears its command
line, forbids being traced (a tracer makes it kill itself and restart), and
keeps its binaries immutable with a hash manifest. The sentinel and the daemon
watch each other. Modified system files (`/etc/passwd`, `sudoers`, `crontab`,
`hosts`...) are restored from shadow copies, with the attacker's version kept as
evidence. Every decision is written to a hash-chained audit log.

## The TRust Smart Terminal

One product with several names: the TRust Smart Terminal, "the TRust terminal", the
`trust_terminal` crate and binary, and the `SmartTerminal` type. It began in the
Jayce SDK, where it lived inside `trust_core`; here it is its own crate
(`trust_terminal`), sharing `trust_core` for the prompt guard and AI-containment
engines. The port is nearly a straight copy plus a governance layer.

**Where it runs**

- **Standalone REPL.** `trust_terminal`, or `miniguard --terminal` (which hands over to
  it). Every input line goes through the Prompt Guard first.
- **Inside the daemon.** The daemon links the same crate. `term:` socket commands run
  through a daemon-hosted `SmartTerminal`, and the daemon's `ToolRegistry` serves the
  governed tools: `tool-list` publishes MCP schemas straight from the live registry,
  and approval requests queue in the daemon.
- **Governed tools from outside.** `mg-trust-tool` (CLI), `mg-mcp-bridge` (MCP clients)
  and `mg-cli` session sockets all call a governed registry; queued calls are approved
  by a human out of band — the desktop Review pop-up, the TUI's Tool Approvals page
  (`sudo miniguard-status --approvals`), or `sudo mg-cli session review` — always
  behind sudo.
- **Network filter.** The daemon's transparent proxy reuses the terminal's
  `covert_channel` rules to hard-deny reverse shells, `curl | sh` and exfiltration posts.

**What the port added to the SDK version:** the `covert_channel` hard-deny and soft-flag
rules, a 10-second taint window after a network fetch, and a fixed `PATH` for
`run_command` (the daemon runs with an empty environment).
**What the port removed:** background (`&`) job spawning and the external-program helpers
in `shell.rs` (`find_in_path`, `execute_external`), plus the SDK's four extra tool
families (action, browser, research, web: about 5,200 lines). No document or changelog
records why; it may have been deliberate hardening.

**The command layer was not ported.** In the SDK, the TRust Terminal is layered: a REPL
(`trust_cli`) over `trust_core`, which holds the parser, a capability-checked registry and
226 registered command handlers (60 modules under `commands/`). MiniGuardian carries the
Smart Terminal part (shell, analyzers, language adapters, tools) and the catalog of names, but
not the registry or those handlers. Many were ported in November 2026 on real state
only (no canned output). Counted from the code on 2026-11-20: 144 catalog names, of
which **42 are catalogued but not implemented**; the rest run natively, are ordinary
host programs, or are documented subcommand forms. `commands` lists the catalog with the unavailable ones
marked, and running one prints a clear message (exit 127). The earlier "200+ commands" figure was the SDK's count
(226), not this build's.

**Limits:** the `SmartTerminalV2` wrapper (internal design doc) is used only by an
example program. One known evasion is recorded in code, in the test
`known_gap_python_reverse_shell_is_not_caught_by_any_pattern`. `trust_terminal` is not in the
`seal.hashes` pin list.

## What it does not do

- **It does not stop a root attacker.** Root can defeat a userspace daemon.
  Kernel lockdown (integrity mode) and sealing raise the cost; they are not a
  guarantee. No system is "un-hackable".
- **Enforcement is conservative by default.** In Calm and Suspicious the daemon
  observes; blocking starts at Hostile.
- **Egress control needs kernel 6.7 or newer** for TCP port rules. UDP is
  governed only by the identity firewall, which is off by default.
- **The retrieval scanner needs several markers.** A single injected sentence
  in fetched content is not flagged by that scanner (a deliberate fix for false
  positives); the tool-call gate and the sandbox are what stop the resulting
  action.
- **The kernel judge has a boot latch.** In `strict` mode (shipped) the daemon
  stays Paranoid until the judge is up and agrees, about 75 seconds after
  start. From beta-37 it waits up to 60 seconds for the judge's first
  telemetry (slow machines need it; before, 10 seconds left them Paranoid). If
  QEMU cannot run on a machine, set `judge_mode = "best-effort"`;
  without a judge the daemon labels itself "self-attested".
- **Trust from the kernel sidecar fails open for malformed replies** (a tracked
  design tension in `docs/SECURITY_FEATURES_CATALOG.md` §6.3).

## Numbers you can check

| | Value | How to check |
|---|---|---|
| Kernel engines | 44 | `ENGINE_COUNT` in `jayce_kernel_src/src/kernel/engines/mod.rs` |
| Governed tools | 52 (19 read-only, 33 approval-gated; counted 2026-11-13) | `ToolRegistry::new()` in `trust_terminal/src/terminal/smart_terminal/tool_system.rs` |
| Terminal commands | 144 catalog names, 42 not implemented (counted 2026-11-20) | `COMMAND_CATALOG` and `NOT_IMPLEMENTED` in `trust_terminal/src/terminal/smart_terminal/command_reference.rs` |
| Inline languages | 18, plus Apodeixis | `trust_terminal/src/terminal/smart_terminal/builtin.rs` |
| Tests | about 2,260 `#[test]` functions across the workspace (counted 2026-09-19) | `grep -r '#\[test\]'` |
| Daemon unit tests | 2,505 passed, 2 failed, 26 ignored (2026-11-20, run inside a governed AI session; the 2 failures are health-server tests that open a TCP connection, which the session's sandbox refuses) | `cargo test -p miniguard --lib --release` |
| Status TUI tests | 25 passed (2026-09-19) | `cargo test -p miniguard-status` |
| Terminal tests | 196 unit + 11 v2 integration passed (2026-09-19) | `cargo test -p trust_terminal` |

## Components

| Binary | Role |
|---|---|
| `miniguard` | The daemon (runs as root under systemd) |
| `miniguard-sentinel` | Antifreeze and seal watchdog |
| `miniguard-status` | Terminal dashboard (`sudo miniguard-status`) |
| `mg-cli` | AI tool governor |
| `mg-trust-tool`, `mg-mcp-bridge` | Governed tools for scripts and MCP clients |
| `mg-supply` | Supply-chain check |
| `mg-why` | Explain why any program failed, in any language ([guide](EXPLAIN-A-FAILURE.md)) |
| `mg-plugin`, `mg-open-shell` | Build, install and run your own plugins in any language ([quickstart](PLUGIN-QUICKSTART.md)) |
| `mg-session-hub`, hook bridge | Route tool hooks to the right governed session |
| `mg-chat*` | Governed chat (`mg-collab`, multi-editor collaboration, is built from source but not in the packages) |
| `trust_terminal` | Standalone TRust command shell |

## Get started

```bash
cargo build --release --workspace
sudo bash scripts/seal.sh            # install, seal, verify (add --mount-ro for a read-only /usr/local/bin)
sudo miniguard-status                # dashboard; press k for the kernel page
```

`scripts/deploy.sh` is the wizard for choosing a mode (`sidecar`, `cohost`,
`saddle`), workload and posture. After rebuilding `mg-cli`, sign it (the build
signs it during `seal.sh` with your key) so its self-integrity check accepts it.
Details are in `docs/AI_CONTAINMENT_MANUAL.md` §13.

## Where to read next

The full index is [`docs/README.md`](README.md).

- Using the AI containment layer: `docs/AI_CONTAINMENT_MANUAL.md`,
  `docs/GOVERNED_TOOLS_REFERENCE.md` (tools and the human approval flow)
- The dashboard and the terminal: `docs/TUI_USER_GUIDE.md`,
  `docs/TRUST_COMMAND_MANUAL.md`
- Everything the daemon can do: `docs/SECURITY_FEATURES_CATALOG.md`
- Release history: `CHANGELOG.md`
- Papers (historical, kept as written): `docs/papers/`
- Design specs, operator notes and superseded documents are internal (not
  published); reviewers and partners can ask for them.

## License

Proprietary, under the Jayce Automata Research Source License v1.1
(`LICENSE`). Third-party components keep their own licences; see
`THIRD_PARTY_NOTICES.md` (installed in `/usr/share/doc/miniguard/`).
