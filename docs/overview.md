# MiniGuardian — Overview

*Status as of 2026-09-19, re-checked against the source 2026-09-26. The
2026-09-19 evidence is in
`docs/reports/claims-audit-2026-09-19/claims-audit-frontdoor.tsv`. Numbers carry
the date they were measured.*

## What it is

MiniGuardian is a security daemon for a single Linux machine. It watches
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
| Enforcement | eBPF and iptables blocking, kill, quarantine, restore from shadow copies | Implemented. By default it only blocks in Hostile/Paranoid; Calm and Suspicious observe |
| Antifreeze sentinel | Separate process that un-freezes a stopped daemon (SIGCONT) and restarts it after repeated freezes | Implemented |
| Sealing | Immutable binaries and a hash manifest, re-checked about every 2 minutes | Implemented |
| `mg-cli` governor | Runs an AI tool under Landlock: file sandbox plus **default-deny TCP** (kernel 6.7+), a proof-checked action gate, a fake-response layer for escape attempts, a dependency check before launch, and (under sudo, CLI tools) no access to the desktop: X server, session D-Bus and Wayland sockets are hidden so the agent cannot fake pointer or keyboard input | Implemented (desktop isolation added 2026-09-26) |
| Human approval | Queued actions (tool calls, unregistered tools, risky file actions) raise a desktop pop-up whose only button, **Review**, opens a password-protected approval terminal; approval always needs sudo | Implemented 2026-09-26, **not yet verified on a live desktop** |
| Governed tools | 29 built-in tools; reading is automatic, writing, running, deleting, network and memory ingest wait for human approval | Implemented |
| Content scanners | Prompt-injection guard, memory-poisoning scanner, tool-call gate, exposed on a read-only local socket | Implemented (see limits) |
| Identity firewall | UID-scoped UDP policy for the governed account | Implemented, **off by default** (shadow mode first) |
| AI envelope (tagging, per-tool egress, GUI exec interception) | Detects AI tools and tags them | **Detection only**; the enforcement parts are planned |
| Jayce kernel | 44 deterministic engines, boots in QEMU from `artifacts/kernel/jayce-kernel.iso` | Implemented as source and a QEMU image |
| Kernel as a "judge" | The daemon launches the kernel in QEMU and cross-checks it | **Partial**: optional, off on this host, and cannot start under the shipped systemd unit (see Limits) |
| Kernel vitals in the TUI | Live kernel counters on the `k` page of `miniguard-status` | Implemented; read from a serial log and marked unverified |
| TRust Smart Terminal (the TRust terminal) | Command shell with Prompt Guard on every input, 18 inline languages plus Apodeixis, and the governed tool system | **Partial.** The shell, analyzers and tools work; 94 of the 157 catalogued commands are not implemented here, and are marked as such (see below). Standalone `trust_terminal` binary (v0.1.0, `miniguard --terminal` starts it) and linked into the daemon for `term:` commands |
| Cosmic Map | 2D fleet observatory | **Not available in this beta**: `miniguard --map` prints a "not available" message; the `cosmic_map` crate still builds a standalone binary that is not installed |
| Master Console, fleet intelligence, swarm immunity | Cross-host dashboard, rule distribution, shared attack signatures | **Future plan.** Nothing is deployed (hosting needs paid infrastructure). A console prototype exists as a separate project outside this repo; the daemon has no client for it |
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
   tampered or unpinned dependencies.
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
not the registry or those handlers. Result, measured by running every catalog name in a
sandbox: 23 commands execute natively, 28 are ordinary host programs, 12 are documented
subcommand forms, and **94 are catalogued but not implemented**; 92 of those have
working handlers in the SDK. `commands` lists the catalog with the unavailable ones
marked, and running one prints a clear message (exit 127). The earlier "200+ commands" figure was the SDK's count
(226), not this build's.

**Limits:** the `SmartTerminalV2` wrapper (`docs/design/TRUST_TERMINAL_V2.md`) is used only by an
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
- **The kernel judge is optional.** Without it the daemon is its own authority
  and labels itself "self-attested". Under the shipped unit
  (`MemoryDenyWriteExecute=yes`) QEMU's default accelerator cannot start, so
  enabling `[kernel_vm]` needs a decision (hardware acceleration, a separate
  unit, or relaxing that setting). Do not enable it without setting
  `judge_mode = "best-effort"`, because the unset default is `strict` and a
  failed launch would latch the paranoid state.
- **Trust from the kernel sidecar fails open for malformed replies** (a tracked
  design tension in `docs/SECURITY_FEATURES_CATALOG.md` §6.3).

## Numbers you can check

| | Value | How to check |
|---|---|---|
| Kernel engines | 44 | `ENGINE_COUNT` in `jayce_kernel_src/src/kernel/engines/mod.rs` |
| Governed tools | 29 (11 read-only, 18 approval-gated) | `ToolRegistry` in `trust_terminal/src/terminal/smart_terminal/tool_system.rs` |
| Terminal commands | 157 unique catalog names (158 entries; `swarm` twice): 23 native, 28 host programs, 12 subcommand forms, 94 not implemented (measured 2026-09-19) | `docs/reports/claims-audit-2026-09-19/terminal-command-status.tsv` |
| Inline languages | 18, plus Apodeixis | `trust_terminal/src/terminal/smart_terminal/builtin.rs` |
| Tests | about 2,260 `#[test]` functions across the workspace (counted 2026-09-19) | `grep -r '#\[test\]'` |
| Daemon unit tests | 1341 passed, 0 failed, 2 ignored (2026-09-19) | `cargo test -p miniguard --lib` |
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
| `mg-session-hub`, hook bridge | Route tool hooks to the right governed session |
| `mg-collab`, `mg-chat*` | Multi-editor collaboration and governed chat |
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
Details are in `docs/AI_CONTAINMENT_MANUAL.md` §13 and `AGENTS.md`.

## Where to read next

The full index is [`docs/README.md`](README.md).

- Using the AI containment layer: `docs/AI_CONTAINMENT_MANUAL.md`,
  `docs/GOVERNED_TOOLS_REFERENCE.md` (tools and the human approval flow)
- The dashboard and the terminal: `docs/TUI_USER_GUIDE.md`,
  `docs/TRUST_COMMAND_MANUAL.md`
- Everything the daemon can do: `docs/SECURITY_FEATURES_CATALOG.md`
- Design specs: `docs/design/`; history: `AGENTS.md`, `CHANGELOG.md`
- Papers (historical, kept as written): `docs/papers/`
- Superseded documents (not maintained): `docs/archive/`

## License

Proprietary, source-available under the Jayce Automata Research Source License
v1.0 (`LICENSE`).
