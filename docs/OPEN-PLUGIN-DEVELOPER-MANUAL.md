# MiniGuardian Open Plugins — Developer Manual

*For anyone building plugins, tools or AI-assisted projects on MiniGuardian.
Covers what you are building on, how plugins work, the commands you'll use,
how approvals work, and how strong the protection actually is.*

> Command lists here come from the current source code. They are not copied
> from older manuals. Some TRust commands in the catalog have no handler in
> this build and are left out on purpose.

---

## 1. What you are building on

### MiniGuardian

MiniGuardian is a security daemon for one Linux machine. Its job:
**untrusted code, including AI agents and plugins, runs inside a box it
can't get out of, and nothing gets past the owner without them seeing it.**

It does five things:

1. **Watches:** files, processes, memory, network, USB, containers.
2. **Judges:** it rates the whole machine Calm → Suspicious → Hostile →
   Quarantined, and enforcement escalates with that level.
3. **Contains:** AI agents and plugins only touch what the owner allowed.
4. **Deceives:** escape attempts are handled by a deception layer and
   logged. This manual doesn't describe how it works, by design.
5. **Protects itself:**
   - sealed binaries and config;
   - a separate sentinel process;
   - a tamper-evident, hash-chained audit log.

Every decision that matters is **deterministic**, meaning rules and state
machines and never an AI's judgement. Anything AI-shaped can propose, but
only fixed rules and a human decide.

### The Jayce microkernel

Underneath is the **Jayce kernel**, a small deterministic kernel:

- **A fixed 1,000 Hz heartbeat.** Engines run on a fixed schedule worked out
  by arithmetic from the tick number, each within a hard budget of time,
  memory and messages. There is no randomness, and every run can be replayed.
- **An immutable "spine".** The tick rate and the engine schedule are
  compile-time constants. The configuration is fingerprinted and checked.
- **Kernel judge.** The kernel can run beside the daemon as a judge in a
  small VM. The two cross-check each other, and if the judge is absent or
  disagrees, the system locks down. The TUI shows this as `Proof:
  judge-graded` (judge aligned), `self-attested` (no judge) or `degraded`.
- **Apodeixis.** Apodeixis is Jayce's proof-checked rule language. Security
  rules ("missions") are written in it and checked before they run. You can
  also write plugins in it (§3).

Background: `docs/JAYCE_1000HZ_TICK_WHITEPAPER.md`, `docs/overview.md`.

### Where your code fits

```
  you / your AI agent
        │  mg-plugin run · TRust `plugin` · MCP tool plugin_<name>
        ▼
  MiniGuardian daemon ── approvals ──► the owner (pop-up / TUI)
        │  checks every action against what the owner approved
        ▼
  open plugin shell (signed)  ──►  YOUR PROGRAM (any language)
        no network, no files, not running as you;
        everything goes through "actions"
```

---

## 2. Open plugins in one page

A plugin is **your own program, in any language**. It runs inside
MiniGuardian's signed open-plugin shell. You don't sign anything, you don't
touch MiniGuardian's code, and you don't need `sudo`.

- **It is contained.** Your program has no network, can't read the owner's
  files, and doesn't run as the owner.
- **It asks for everything.** Reading or writing a file, fetching a URL and
  running a command are all *actions* it sends to MiniGuardian. Each one is
  checked against what the owner approved at install.
- **Reads inside approved folders are automatic.** Writes, commands and
  network show the owner a pop-up.
- **Every run is recorded:** who called it, every action and its outcome,
  and every network connection.
- **Humans and AI agents can both call it.** AI agents connected through
  MiniGuardian's MCP broker see each installed plugin as the tool
  `plugin_<name>`.

Start with `docs/PLUGIN-QUICKSTART.md`. A working plugin takes about 15
minutes.

---

## 3. Building a plugin

### Create, test, install

```bash
mg-plugin new hello --lang python     # or: node, rust, apodeixis
cd hello
mg-plugin test                        # stage → supply check → owner approves → run
mg-plugin install                     # a normal (non-dev) install
mg-plugin run hello '{"any":"json"}'
mg-plugin list | logs hello [-n 20] | remove hello
```

Every change gives the plugin a new fingerprint, and a new fingerprint
needs a new approval. Nobody's code runs without the owner seeing it.

### The manifest: `mg-plugin.toml`

```toml
name = "hello"
version = "0.1.0"
author = "Your Name"
api = 1
run = ["python3", "main.py"]   # a runtime on the machine, or a file you ship
files = ["main.py"]
# runtimes = ["node"]          # extra runtimes a mixed-language plugin calls

[[capability]]
tool = "read_file"
paths = ["~/mg-plugin-data/hello"]

[[capability]]
tool = "network_fetch"
hosts = ["api.example.com"]

[[capability]]
tool = "run_command"
commands = ["git", "npm"]      # program names only, no shell operators
paths = ["~/projects/hello"]   # folders it may run in (pass "cwd")
```

| Tools you can declare | Scope |
|---|---|
| `read_file list_files file_metadata grep search_code tree git_status` | `paths` (read) |
| `write_file append_file edit_file create_dir copy_file move_file delete_file` | `paths` (write) |
| `network_fetch` | `hosts` |
| `run_command` | `commands` + `paths` |

`~` always means the installing user's home. A path you didn't list is
denied, including a tool's default folder.

### The protocol

Your program talks to MiniGuardian in one JSON object per line on
stdin/stdout. Every message carries `"mgp": 1`.

| You receive | You send |
|---|---|
| `{"type":"request","id":"r1","input":…,"caller":{…}}` | `{"type":"log","level":"info","msg":"…"}` |
| `{"type":"action_result","id":"a1","ok":true,"output":"…"}` or `"ok":false,"denied":"why"` | `{"type":"action","id":"a1","tool":"read_file","args":{"path":"~/…"}}` |
| | `{"type":"answer","id":"r1","output":…}` (ends the run) |

Fields MiniGuardian doesn't recognise are ignored, so newer versions won't
break your plugin.

### Languages

- **Any language.** `run[0]` is either a binary in your plugin folder or a
  runtime program on the machine.
- **Runtimes approved everywhere:** `python3`, `node` and `apodeixis`.
- **Any other runtime** (`ruby`, `java`, …) adds one line to the owner's
  install pop-up. After that single approval it works for every plugin.
- **Mixed-language plugins** list the extra runtimes with `runtimes = [...]`.
- **Dependencies are checked at install** by the supply-chain sentinel.
  Vendor them, for example a `.mg-wheelhouse` for Python, or the install is
  blocked.

**Mixing languages at a C boundary** (ctypes, cgo, Rust FFI, Apodeixis FFI):
structs can be laid out differently on each side because of padding and
alignment. Keep field order and sizes identical, use `#[repr(C)]` in Rust,
and never send raw struct bytes between processes; use JSON, as the plugin
protocol does. `mg-why` points this out when a crash looks like a layout
mismatch.

### Apodeixis plugins

`mg-plugin new hello --lang apodeixis` creates a plugin whose logic is
deterministic, budgeted per step, and checked for information flow before it
runs:
- numeric input fields arrive as `signal("name")`;
- `emit_action(tool, target)` asks for one governed action.

Apodeixis is the best fit for policy-like plugins, where you want a proof
that the logic can't leak or run away. See `docs/APODEIXIS_LANGUAGE_GUIDE.md`.

### When something fails

- `mg-plugin` prints a hint under every denial (see the table in the
  quickstart).
- `mg-why -- <command>` explains any crash in any language: where the code
  broke, why, and which processes were involved. It works outside
  MiniGuardian too. See `docs/EXPLAIN-A-FAILURE.md`.
- **New languages for `mg-why`** are a small TOML file, no code
  (`docs/WRITE-A-LANGUAGE-FILE.md`).

---

## 4. Network, data and limits

### Network for plugins
- **`network_fetch`** fetches one URL from a host listed in `hosts`.
- **Commands** (`npm install`, `pip`, `cargo build`, your program) have no
  network of their own. They reach the internet only through MiniGuardian's
  egress proxy, which is set for them as `HTTPS_PROXY`.
- **Hosts:** the hosts you listed pass. A new host pauses the command for an
  owner pop-up.
- **Fail closed:** programs that ignore the proxy have no network.
- **Limits:**
  - HTTPS only;
  - never private or local addresses;
  - byte caps and connection-rate caps ask the owner before continuing;
  - proxied commands may run up to 15 minutes.

### Confidential data
The owner can mark folders as confidential (`[open_plugins] confidential`).
Secrets are also recognised on their own: `.env` files, SSH and cloud keys,
token-shaped content. Once a run reads any of these, **every** outbound step
asks the owner first, and the pop-up names what was read and where it would
go. From then on, action results carry `"label": "secret"`.

### What a plugin can never do
- Reach the network, the owner's files or other programs directly.
- Connect to local system sockets (D-Bus, MiniGuardian's own sockets).
- Approve its own installs or actions, from a script or an AI agent.
- Do anything new without a new install approval.

---

## 5. AI agents in your projects

Run any AI coding tool under governance:

```bash
sudo mg-cli --curated-dev --deterministic --share-data --workspace ~/project claude
```

- **Workspace only.** The agent works in its workspace. Its own credentials
  are hidden from the commands it runs.
- **Approvals.** Writes, deletes, commands and network go through the
  approval queue.
- **Internet for the agent** (`[session_egress] enabled = true`, off by
  default):
  - the agent's commands share a private network whose only way out is the
    proxy;
  - each new site asks once per session;
  - one dev-server port (`dev_port`, default 3000) is reserved for the
    session. `preview_port` shows it on your desktop after you approve it.
    Every connection is monitored and recorded: other AI sessions are
    refused, and output that looks like a secret ends the connection;
  - every `git push` asks first (HTTPS remotes with a repo-scoped token; SSH
    keys stay hidden);
  - **leak guard:** before you are asked, what a push, gist or release
    would publish is scanned. Credentials refuse it; your personal
    information is flagged on the pop-up (`docs/LEAK-GUARD.md`).
- **Review an agent's session:** `sudo mg-cli session review`.

---

## 6. Adding MiniGuardian to an existing project

You don't have to write a plugin to get protection. These work on a project
you already have:

| Goal | How |
|---|---|
| **Check your dependencies** | `mg-supply audit ~/my-project` checks Rust, Python and Node lockfiles and git dependencies against the supply-chain sentinel's catalog and reports the verdict per dependency. |
| **Let an AI tool work on it safely** | `sudo mg-cli --curated-dev --share-data --workspace ~/my-project claude` (or another agent). The agent stays inside the project folder, its writes, deletes, commands and network ask you first, and the supply check runs before it starts. |
| **Give your own scripts guard rails** | Call `mg-trust-tool` from build, deploy or maintenance scripts (`mg-trust-tool run_command "…"`, `write_file …`). Reads run at once; anything risky waits for your approval and is audited. |
| **Wrap risky automation as a plugin** | Turn a deploy script, data job or release step into an open plugin. It then runs only with the folders, commands and hosts you approved, and every run is recorded. |
| **Protect secrets in the repo** | List folders such as `secrets/` or `data/customers/` under `[open_plugins] confidential` in `/etc/miniguard/miniguard.toml`. Anything that reads them and then tries to send data out has to ask you first. |
| **Find security bugs in the code** | In the TRust terminal: `diagnose ~/my-project` (all analyzers), `bloodhound` (cross-file taint and dependency risks), `plumber` (auto-fixable issues), `linter <file>`. |
| **Understand failures fast** | `mg-why -- cargo test` (or any command, any language) names the file and line, the cause and the processes involved, with secrets masked. |
| **Encode team rules** | Write project policy in Apodeixis ("no new network hosts without review", "no writes outside `src/`") so it is proof-checked and deterministic. |
| **Know what happened** | `sudo bash scripts/mg_explain.sh 2h` or the TRust `forensic` command after anything odd. |

Most projects start with the first three rows. The others are worth adding
as the project grows.

---

### Building MiniGuardian in: programs, OS images and servers

Adding MiniGuardian to a program, an operating-system image or a server
fleet makes the whole thing harder to attack, **including through its
supply chain**. It does that in two layers.

**Layer 1: stop bad code getting in.**

| Where | What you add | What it stops |
|---|---|---|
| **Your program's build or installer** (Rust) | `mg_supply::embed::audit_project(dir, allow_fetch, catalog)` returns Allow / Warn / Block for the whole dependency graph | Archives that don't match their pinned checksums, missing or unverifiable dependencies, git dependencies changed under you (verified without running the repo's hooks or filters), local dependencies that drifted. Every verdict goes to an append-only, hash-chained catalog, so past audits can't be quietly rewritten. Offline and deterministic: no network, no guesswork. |
| **CI** | `mg-supply audit <dir>` as a required step | The same checks for Rust, Python and Node projects, with an exit code your pipeline can fail on |
| **Plugins and AI sessions** | Nothing extra: it runs at every plugin install and every `mg-cli` launch | A dependency that changed since the owner approved it needs a new approval |
| **OS images and servers** | The MiniGuardian daemon package | Sealed MiniGuardian binaries pinned by hash, signed updates, and package upgrades checked against the package database. New executables arriving in downloads are quarantined until the owner decides. |

**Layer 2: limit what bad code can do if it gets in anyway.**

A supply-chain attack that passes every check, for example a malicious
release whose checksums are genuine, still runs into the same walls as any
other untrusted code:

- **No network of its own.** It can only reach approved hosts through the
  egress proxy. Phoning home to a new server pauses for the owner.
- **No quiet theft.** After it reads secrets or confidential folders,
  anything outbound asks the owner first, naming what was read.
- **No reach beyond its scope.** Landlock folders, the seccomp deny list, no
  local system sockets, never root.
- **No second chance.** If it attacks once (ransomware, a rootkit, a
  dropper, persistence), the payload's fingerprint is refused at exec from
  then on, even after a restart and under a new name.
- **A record of everything.** Every action and connection is in the audit
  chain, so you can see exactly what the compromised package tried.

**Where to build it in:**
- **Applications:**
  - embed `mg_supply` in your updater or plugin loader;
  - send risky operations (writes, commands, network) through
    `mg-trust-tool` or the MCP broker, so a human approves them;
  - use `why_core` for crash reports that name the cause;
  - use Apodeixis for rules that must provably hold.
- **Operating-system images:** ship the daemon sealed (kernel lockdown,
  immutable binaries). Every program on the image then gets the exec gate,
  the ransomware vault, persistence and rootkit detection, the download
  guard and attack immunity, with no code changes.
- **Servers:** the same, plus the eBPF/XDP flood shield and IP blocks that
  survive restarts. Over fleet gossip, servers share blocked IPs, and the
  master console can broadcast immunity rules to every node.

**Honest limits.**
- Layer 1 catches dependencies that were changed, are missing or drifted.
  It can't tell that an honestly published release is malicious. That is
  what Layer 2 is for.
- MiniGuardian is proprietary, under the Jayce Automata Research Source
  License v1.1 (`LICENSE`). Its terms for building it in:
  - **Shipping MiniGuardian with your program:** you may ship the official
    signed packages, unmodified, with your program, or require them as a
    dependency. This is free for individuals, small organisations and
    non-commercial programs; larger companies selling a product with it need
    a commercial licence.
  - **Copying or embedding its code** (including the libraries above) still
    needs a written agreement.
  - **Plugins you write are yours,** to license and sell as you like.

### Remote maintenance (kiosks, servers, fleets)

**What works today:**
- Administrators reach machines over SSH as usual. MiniGuardian governs AI
  agents and untrusted code, not your own admin sessions.
- **Approvals need a person with the sudo password, not someone at the
  screen.** That person can work over SSH: `sudo miniguard-status
  --approvals`, or `sudo mg-cli session review` for an AI session.
- Management addresses listed in `mgmt_allowlist` are never blocked by the
  flood shield.
- Fleet nodes talk over mutual TLS with signed messages. Fleet rules from the
  master console are signed and checked against its pinned key.

**Partner builds only.** The following need a personal build from the
MiniGuardian owner, made for partners on request:
- an unattended kiosk/fleet profile (no approval pop-ups on public screens);
- approvals signed by each technician and sent over the fleet link;
- confidential diagnostics (logs, screenshots, forensic reports) sealed
  end-to-end for the operator;
- maintenance envelopes written in Apodeixis and proof-checked before use,
  with information-flow labels that keep confidential data on the sealed
  channel.

None of this is in the standard build. Contact Jayce Automata Research to
ask about one.

---

## 7. Ideas: what to build

- **Code-health bot:** read a repo, run `diagnose`-style checks, write a
  report into an approved folder. Reads only, so it never asks.
- **Dependency auditor:** list a project's lockfiles, check them against an
  advisory API (one approved host), flag risky versions.
- **Docs generator:** read source files, write Markdown docs to `docs/`.
  Every write is visible to the owner.
- **Release helper:** run `cargo build` / `npm run build` through
  `run_command`, then package the artifacts. Registries go through the proxy
  with one approval each.
- **Website builder and deployer:**
  - scaffold, build, preview locally;
  - deploy with a CLI over HTTPS to one approved host;
  - an AI agent can drive it, with every push and upload asking first.
- **Data pipeline for sensitive files:** process a confidential folder
  locally. The information-flow pop-up guarantees nothing leaves without the
  owner saying yes.
- **Research fetcher:** fetch pages from approved hosts, summarise them into
  notes. Good as an MCP tool for agents.
- **Backup verifier:** hash an approved folder and compare it with last
  week's manifest. Read-only and fully audited.
- **Policy as code:** write a team rule ("no secrets in commits", "no new
  network hosts without review") in Apodeixis, so it is proof-checked and
  can't leak.
- **Homelab monitor:** check service status pages on approved hosts and
  write a daily summary.
- **Multi-language tooling:** a Python front end calling a Rust core. List
  both runtimes and keep the boundary in JSON.

---

## 8. Command reference

### TUI: `sudo miniguard-status`

| Key | Action |
|---|---|
| `a` | AI Safety window (also `miniguard-status --ai`) |
| `g` | Tool Governance window |
| `e` | Developmental Governance window |
| `t` | **Tool Approvals**: everything waiting for a human (also `miniguard-status --approvals`) |
| `k` | Kernel Judge window |
| `v` | run the full governance validation suite (Governance window) |
| `m` (twice) | migrate a selected process under governance (Governance window) |
| `c` / `r` | clear the audit trail and threat feed |
| `d` | mute the authority sound |
| `q` | quit |

**In Tool Approvals (`t`):**

| Key | Action |
|---|---|
| `↑` `↓` | select |
| `Enter` | approve (a group: all of it) |
| `x` | reject |
| `o` | expand or collapse a group |
| `s` | approve-global (unregistered tools only) |
| `w` | open or close the timed auto-approve window (if enabled in config) |

The header shows the machine's state (`CALM` / `SUSPICIOUS` / `HOSTILE` /
`QUARANTINED`). The Engine Status panel shows the kernel engines, the
ransomware monitor, the seal and the proof label.

### TRust terminal

Commands with a handler in this build. Type `help <command>` for details.

| Area | Commands |
|---|---|
| **Plugins** | `plugin new`, `plugin test`, `plugin install`, `plugin run`, `plugin list`, `plugin logs`, `plugin remove` |
| **Explaining failures** | `why -- <command>`, `why last` (same as `mg-why`) |
| **Code analysis** | `diagnose` / `sniff [path]` (all analyzers), `bloodhound [path]` (cross-file taint), `plumber [path]` (auto-fixable issues), `linter` / `codesniffer <file\|code> [lang]` |
| **Languages & build** | `lang run\|build\|test\|dep\|fmt\|lint\|new` (14 languages), `python`, `node`, `ruby`, `perl`, `php`, `lua`, `r`, `haskell`, `bash`, `scaffold project\|module <name>`, `dep [list\|tree]`, `workspace`, `doc read\|toc\|search\|info <file>`, `jvc pack\|unpack` |
| **Files** | `ls`, `cd`, `pwd`, `cat`, `mkdir`, `rm` (`--force` for dirs), `cp`, `mv`, `stat`, `touch`, `find`, `grep`, `write` |
| **Git & web** | `git …`, `web fetch\|head <url>`, `web search <words>`, `web history` |
| **Security & forensics** | `forensic [today\|yesterday\|DATE] [origin …] [--json]` (needs sudo), `logs`, `guardian`, `shroud`, `illusion`, `landlock`, `seccomp`, `ebpf`, `lockdown` |
| **Kernel** | `kernel status\|vitals\|truth\|log\|heal\|services\|monitor\|spine\|watchdog\|isd\|shadow\|net\|net-inspect\|net-block <ip>\|net-unblock <ip>\|cluster\|burnpit`, `spine`, `vitals` |
| **System** | `sys-info`, `sys-mem`, `sys-cpu`, `monitor [n]`, `env` |
| **Productivity** | `timer`, `focus`, `note`, `session save\|restore\|list` |

### Forensics from a shell

```bash
sudo bash scripts/mg_explain.sh [30m|2h] [--dig]    # one digest of what happened and why
sudo mg-cli session review                          # an AI session's waiting items
```

---

## 9. How approvals work

1. **Something asks.** An AI agent, a plugin or a governed command asks for
   an action that writes, deletes, runs, uses the network or reaches a new
   host. The call does not run. It is queued (exit code `202`, with a
   request id).
2. **The owner is told.** A desktop pop-up appears. The pop-up itself
   approves nothing, so a forged click can't approve anything. Its
   **Review** button opens the approvals screen, which asks for the owner's
   password every time.
3. **A human decides, out of band.**
   - **Where:** the TUI (`t`), `sudo mg-cli session review` for AI sessions,
     or the operator socket.
   - **Who:** never the agent or the plugin. Processes inside a governed
     session, and processes the daemon started (plugins), are refused as
     approvers.
4. **The decision is recorded** in the hash-chained audit log, and the
   caller gets the result.

**What waits for approval:**
- write, exec, delete, system and network tool calls;
- plugin installs, including every new fingerprint;
- new runtimes;
- new network hosts (once per run or session);
- confidential-data uploads;
- byte and rate caps;
- `git push` from AI sessions;
- `preview_port`;
- downloads held by the download guard;
- held kills.

**How long:** pending items wait 30 minutes, then they are rejected.
Connections to a new host are held up to 90 seconds; after that, the next
attempt follows the owner's answer.

**Auto-approve** is optional and off by default. It covers operator rules,
earned trust, or a short timed window that the owner opens from the TUI
(`w`). It never applies to network fetches, commands or setenv.

---

## 10. How secure is it, honestly

### Layers a plugin action passes through
1. **Identity:** each action runs as the installing user, never as root, on
   its own thread.
2. **Filesystem:** Landlock limits it to the approved folders, plus
   read-only system folders.
3. **Syscalls:** the daemon's seccomp deny list applies. New namespaces,
   mounts, ptrace and similar are refused.
4. **Network:**
   - each action gets its own empty network namespace;
   - commands go through the egress proxy;
   - the proxy resolves hosts itself and never connects to private
     addresses;
   - local Unix sockets are refused.
5. **Approval:** a human, out of band, with a password.
6. **Information flow:** after confidential data is read, any outbound step
   asks.
7. **Supply chain:** dependencies are checked at install, and the plugin
   shell is signed and pinned by hash.
8. **Records:** every run, action and connection goes to the audit chain
   and the forensics digest.

### Limits you should know
- **Root beats userspace.** An attacker who already has root can defeat a
  userspace daemon. Kernel lockdown and sealing raise the cost; they are not
  a guarantee.
- **Approvals are only as good as the human.** Read the pop-up. Approving
  everything quickly turns the strongest layer into the weakest.
- **The proxy checks hosts, not content.**
  - It doesn't decrypt HTTPS.
  - A host you approve can receive whatever the plugin sends it, within the
    byte caps.
  - Approving a big shared CDN host approves more than one site.
- **Push detection for AI sessions is best effort.** A script can hide a
  `git push`. The real control is a repo-scoped token you can revoke. The
  leak guard scans the ordinary ways (git, `gh gist`, `gh release`,
  `gh repo create --push`); it reads text, not binaries.
- **Kernel features.**
  - Unix-socket blocking needs Landlock `RESOLVE_UNIX`.
  - TCP port rules need Landlock network support (kernel 6.7+).
  - On older kernels, some layers are skipped, and this is logged.
- **The agent process itself** (in an AI session) keeps its own HTTPS
  access to its model provider. Only the commands it runs are proxied.

### It gets harder to hit the more it's hit

MiniGuardian isn't a forensics tool that only tells you afterwards what
happened. It **acts during an attack and learns from it**:

- **It stops attacks as they happen.** Examples:
  - a burst of encrypting writes is frozen mid-run, and the files are
    restored from its vault;
  - escalating threat levels lock the machine down step by step;
  - held processes lose their network.
- **It remembers every attack, across restarts and updates.** It keeps an
  immunity ledger, IP blocks that survive a restart, and a saved attack
  memory.
- **A repeat is refused at its first step.** A payload that attacked once is
  fingerprinted. The next time it tries to start, the exec gate refuses it
  before it runs, even after a restart and under a new name or path.
- **Each attack makes the next one cost more.** Every attack it survives
  becomes a rule it enforces from then on.

The records (audit chain, `forensic`, `mg_explain.sh`) exist so you can see
*why* it acted. They are not the defence itself.

### Proven, not promised

These are measured results, from the *J.A.R. MiniGuardian Security and
Containment* report of November 2026.

**126 documented security features in 21 areas,** counted from
`docs/SECURITY_FEATURES_CATALOG.md`. Each one is built in the daemon, not on a
roadmap. The areas run from kernel hardening and an eBPF/XDP network shield
to AI containment, deception, ransomware recovery and attack immunity.

**Repeat drill (7 Nov 2026).** Each attack was run, the daemon was restarted,
and the same attack was run again under a new name or path:

| Attack | First run | Repeat after restart |
|---|---|---|
| Ransomware (200 test files) | Paused after about 50 files; all restored byte-for-byte | Renamed encryptor refused at exec; 0 files touched |
| Rootkit (root process posing as a kernel thread) | Detected and fingerprinted | Same payload refused at exec from a new path |
| Malicious download | Quarantined; the owner rejected it | Recognised as rejected before, even under an allowed name |
| Cron persistence | Held in quarantine | Held within 5 s under a new name |
| Privilege escalation | Not run yet: no safe, reversible test exists | — |

**Red-team campaign (beta-26):**
- 35 scenarios × 3 passes × 5 runs, across seeds 42, 7, 1337 and 2026.
- **525/525 passed, all 35 scenarios stable.** The verifier reported that
  integrity, reproducibility and verdict stability hold.
- **Decision benchmark:** 170/170 labelled decisions correct before and after
  the attack traffic. No benign action was denied, and the daemon didn't stay
  locked down once the attacks stopped.

**Threat drill (host attacks beyond AI escape):** 10 PASS, 1 SKIP, 0 FAIL.
Covered:
- droppers;
- `ld.so.preload`;
- `/etc/crontab` tampering;
- systemd and timer persistence;
- cron.daily;
- credential reads;
- a SIGSTOP on the daemon itself, undone by the sentinel;
- a rootkit posing as `kworker`, frozen in the void pit.

The skip was the user-crontab case, because the test machine already had a
crontab.

**Unit tests:** over 2,100 at the time of writing.

**Honest caveats from the same report:**
- All results come from one development machine. Package installs on other
  distributions weren't tested.
- Privilege escalation and the user-crontab path are untested live.
- Some newer pieces (the egress proxy for plugins and AI sessions) are
  covered by unit tests. Their live checks are still pending.

The aim is not "unhackable". It is this: **nothing reaches the owner's
files, network or secrets without either a rule they set or a decision they
made, everything leaves a record, and every attack it survives makes the
machine harder to attack again.**
