# MiniGuardian TUI User Guide

The MiniGuardian status TUI (`miniguard-status`) is a live threat dashboard that
renders three windows:

| # | Window | Open with |
|---|--------|-----------|
| 0 | **Main dashboard** | default view |
| 1 | **AI Safety** | press `a`, or launch with `--ai` |
| 2 | **Tool Governance (Governance Center)** | press `g` |
| 3 | **Developmental Governance** | press `e` |
| 4 | **Tool Approvals** | press `t` |
| 5 | **Kernel Judge** | press `k` |

The header and footer render on every window. This guide explains every metric
you will see in each window and what it means.

---

## Global header & footer (all windows)

The header (`MINI GUARDIAN`) carries the system's overall posture:

- **`CALM` / `SUSPICIOUS` / `HOSTILE` / `QUARANTINED`** — the daemon's global
  threat state, color-coded (green / yellow / red / magenta). This is the
  single most important number: it is the aggregate of every detector and the
  level at which active enforcement escalates.
- **`Uptime: 1h 5m`** — how long the daemon has been running (d/h/m/s).
- **`N detected  N kills  N blocks  N restores`** — lifetime counters:
  threats detected, enforcer kills (processes terminated), blocks (IPs/exec
  denied), and restores (system files rolled back to baseline).

Footer: `q: quit  a: AI  g: governance  e: developmental  t: approvals  k:
kernel judge  v: validate  c/r: clear  [d] mute`.

---

## 1. Main window (page 0)

### Engine Status

The five core "engine" subsystems and their live counters. PEH, ISD and
SHCM are safety-analysis engines of the Jayce kernel
(`jayce_kernel_src/src/kernel/engines/`): proprietary code of Jayce
Automata Research, designed and written by Jose Almendarez, under the Jayce
Automata Research Source License v1.0 (see `LICENSE`). The daemon runs the
canonical kernel sources through the `jayce_kernel` port.

| Metric | Meaning |
|--------|---------|
| **PEH** `Active`/`Inactive` | Predictive Execution Horizon — projects hazards 8–20 ticks ahead from the trend of scheduler starvation, latency overruns, capability denials and timing jitter. Is it running? |
| **PEH `N hazards`** | number of raised hazard bits (crash risk, deadlock risk, runaway engine, dangerous actuator, policy violation, timing anomaly, invariant divergence) |
| **PEH `N blocks`** | enforcement blocks issued by PEH |
| **PEH `N quarantine`** | quarantines issued by PEH |
| **ISD** `Active`/`Inactive` | Impossible State Detector — checks kernel counters for states that cannot legitimately happen (timing quantile inversion, capability counter reversal, signal accounting breach, memory budget breach, engine flow contradiction, policy order breach). Is it running? |
| **ISD `N impossible`** | impossible states detected |
| **ISD `N freeze`** | freeze-and-contain actions taken |
| **ISD `N quarantine`** | quarantines issued by ISD |
| **Shadow** `Active` | Constitutional Shadow — is active (has ≥1 active constraint)? |
| **Shadow `N constraints`** | number of active constitutional constraints applied |
| **Shadow `N violations`** | constitutional constraint violations |
| **SHCM `N breaks`** | Self-Healing Causality Mesh — causal-chain breaks detected (tick chain, capability without signal, effect without parent, policy/prediction/binding chain). On a break SHCM freezes the tick, reconstructs the cause or repairs state |
| **Illusion** `Calm/Suspicious/Hostile/Quarantined` | current state of the illusion (deception) bubble |
| **Illusion `N probes`** | number of times something probed the illusion layer |
| **Ransomware ON/OFF** | is the ransomware behavioral monitor enabled? |
| **Ransomware `N high-risk`** | high-risk ransomware events |
| **Ransomware `FS N writers`** | currently active filesystem-writing PIDs (ransomware-like activity) |
| **Ransomware `N ops`** | total ransomware-related operations observed |
| **Ransomware `Last alert:`** | text of the most recent ransomware alert (or `none`) |
| **Seal `LK=` `SB=` `TPM=` `manifest=` `daemon=` `sentinel=`** | host tamper-posture line: kernel lockdown, Secure Boot, TPM presence, seal-manifest present, daemon+sentinel immutability (green when fully sealed, yellow otherwise) |
| **Proof `judge-graded / self-attested / degraded`** | honesty label for how enforcement proof is obtained. `judge-graded` = the kernel coprocessor judge is aligned; `self-attested` = daemon is sole authority (no coprocessor); `degraded` = judge was lost |
| **Proof `judge=N present=true`** | judge agreement gauge (0 = none, 1 = trust, 2 = latched distrust) and whether the coprocessor is present |
| **Proof `absent:<reason>`** | why the judge is absent (shown only when not `none`) |
| **Proof `Attest yes/no (daemon=ok/miss sentinel=alive/dead)`** | the daemon's own weak self-attestation: whether attested, daemon matches seal, sentinel alive |

### DDoS Detection

| Metric | Meaning |
|--------|---------|
| **Severity** | DDoS severity: `None / Low / Medium / High / CRITICAL` |
| **Sources** | distinct attacker sources, with a normalized bar (sources/128) |
| **Mitigated** | events mitigated |
| **Void Pit** | packets swallowed by the void pit (dropped black-hole) |
| **In-flight** | packets currently in flight through the guard |

### Network & Process

| Metric | Meaning |
|--------|---------|
| **Blocked IPs** | total blocked IPs, and the first few shown (red) |
| **Sandboxed** | PIDs currently sandboxed, plus pending kills |
| **DNS Anomaly** | DNS anomaly count (red when >0 — e.g. tunneling/exfil) |
| **Env Mode** | kernel environment mode: `normal / degraded / essential / error` |
| **Caps Denied** | capability denials (attempts blocked by the capability broker) |
| **Bound. Viol** | filesystem/net boundary violations (red when >0) |

### Active Threats

A live feed of events, each row: `[SEVERITY] timestamp PID source  description`.
Severity is color-coded (`LOW`…`CRITICAL`). PID is omitted when 0 (system-wide
event). `source` is the detecting module (e.g. `network_monitor`,
`ebpf_enforcer`). The panel title shows `(N)` current events or `(N) [+M old]`
with scrolled-off events. Use `←/→` to focus this panel and `↑/↓` to scroll.

### Telemetry

| Metric | Meaning |
|--------|---------|
| **CPU** | CPU load % (bar + number) |
| **Memory** | memory load % |
| **Load** | system load average |
| **Tick time p50 / p95 / p99** (per 5 s cycle) | how long one main-loop cycle's work takes, in ms (percentiles over the last 4096 cycles). This is work duration, not timing jitter — the loop runs every 5 s, so ~100 ms is ~2% of the period. Green below 1 s, yellow above 1 s, red above 2.5 s (half the period). Cycles over 40 ms also write a per-stage `TICK BREAKDOWN` line to the sealed log ring (`logs`). Busy hosts (many processes, e.g. a browser) raise it. |
| **Sched Starve** | scheduler starvation events (red when >0) |

### Memory Graph

| Metric | Meaning |
|--------|---------|
| **Tiers: Hot / Warm / Cold** | page-tier counts across the memory hierarchy |
| **Compress** | page compression ratio (e.g. `4.2x`) |
| **Patterns** | memory access pattern count |
| **Risk [bar] N/10** | memory risk score 0–10 (green/yellow/red/magenta) |
| **Forecasts** | per-PID predictive rows: `PID <hazard> → <action>` (e.g. `crash_risk → restrict`; restrict=yellow, quarantine=red, adjust=cyan) |

### Audit Trail

A live log of governance verdicts, each row `[VERDICT] source  detail`:

- **`[ALLOW]`** (green) — action permitted
- **`[BLOCK]`** (red) — action denied
- **`[ESCALATE]`** (yellow) — action escalated to a human/supervisor
- **`[TRACK]`** (cyan) — noise logged but not enforced (e.g. proto anomaly,
  cert anomaly, connection flood, new destination IP)

The panel title shows `(N)` current entries or `(N) [+M old]`. Use `←/→` to
focus this panel and `↑/↓` to scroll.

---

## 2. AI Safety window (page 1) — press `a`

### AI Containment

Shows how AI tools/editors are governed:

| Metric | Meaning |
|--------|---------|
| **Governed:** | AI tools/editors running under `mg-cli` governance (or `none`) |
| **Detected:** | AI processes observed via the `/proc` scan (or `none`) |
| **Envelope:** | per-tool containment envelope tags with `pid`/`kind`/`fs:yes|no`
  (filesystem-pin status); `scanning…` until the envelope is ready; `none` when empty |
| **Allowed** | governed actions allowed |
| **Apodeixis Blocked** | proof-checked Apodeixis missions that vetoed the action |
| **Illusion Deceived** | actions redirected to a synthetic (illusion) response instead of the real system |
| **Illusion Rejected** | actions rejected outright by the illusion bubble |
| **File Monitor Blocked** | per-file monitor interceptions |
| **Ledger entries** | size of the authoritative AI-action ledger |
| **Ledger head** | hash/preamble of the latest ledger entry (16 chars) |

### Last Governed Action / Behavioral Assessment

| Metric | Meaning |
|--------|---------|
| **Last event** | most recent governed event, e.g. `ollama -> Allow (ollama run llama3)` or `(no governed actions observed yet)` |
| **Last workspace** | workspace of the last governed action |
| **Last event time** | time of the last event (or `never`) |
| **Behavioral score `N% (posture)`** | machine-behavior risk score and posture
  (`Calm/Suspicious/Hostile/Quarantine`) for the detected AI processes |
| **Indicators** | detected behavioral indicators with confidence, e.g.
  `ShutdownResistance (95%), GoalDrift (70%)` (or `none`) |
| **Assessment** | a plain-language multi-signal assessment, or `No behavioral assessment yet` |

### Self Integrity

The daemon auditing *its own* integrity (the one self-focused panel):

| Metric | Meaning |
|--------|---------|
| **Self integrity: clean / elevated / critical** | the daemon's self-integrity level (0=clean, 1=elevated, ≥2=critical) |
| **Events** | number of self-integrity events |
| **Last deviation** | most recent self-deviation detail (e.g. `FilelessInjection …`)
  and how long ago, or `none` |

### AI Memory Defense

Runs the memory-poison / compaction defenses on AI context:

| Metric | Meaning |
|--------|---------|
| **AI memory status** | content verdict: `Clean / Suspicious / Drifting / Poisoned / Critical` (or `awaiting telemetry`) |
| **Action** | defensive action for the last scan (e.g. `withhold`) |
| **Scans flagged** | number of scans that flagged content |
| **Quarantined** | items quarantined |
| **Context drift** | % of AI context that is external content, with a bar
  (red >70%, yellow >50%) |
| **Context budget** | total context tokens, `OVER BUDGET` flag, and drift-tick counter |
| **Compaction target** | tokens to preserve / drop on compaction |
| **Output schema** | last output-schema validation result (`valid` / `invalid (N total)` / `no validation yet`) |
| **Replay entries** | model-replay entries recorded |
| **Last flagged scan** | detail of the last flagged scan and age, or `none` |

Privacy/overlay line:

| Metric | Meaning |
|--------|---------|
| **Privacy/Overlay: vpn=on (wg0) overlays=tun0** | VPN active + interfaces, overlay networks |
| **relay-safe** | whether the host is exempt from beacon/C2 classification |
| **self-block refusals** | times the daemon refused to block itself |
| **Environment** | desktop compositor + session (`de_compositor`, `de_session`) and any
  known third-party desktop issues — present so external desktop bugs are not
  mistaken for guardian events |

---

## 3. Tool Governance window (page 2) — press `g`

### Governed Tools

The managed AI tools + migration:

- **`no governed sessions registered`** — empty state.
- **`• tool`** (green bullets) — governed tools list.
- **`tool[N] kind fs:yes`** (dark gray) — containment envelope tag per tool.
- **`[m] migrate to governed: tool[N] kind fs:no (selected)`** (cyan) — an
  ungoverned PID you could migrate; `↑/↓` selects, `m` opens confirmation.
- **`[MIGRATE ARMED] pid N — press m to CONFIRM`** (yellow) — confirm arms the
  migration (terminates + relaunches the tool under governance).
- **`last: pid N: ok`** — result of the last migration.

### Plugin Governance (sealed | harness)

**Left — sealed daemon plugins:** each row shows plugin name + state
(`ACTIVE`/`INACTIVE`/`?`), e.g. `intent-classifier`, `secret-detector`, with a
detail line `v<version> exec=<binary>`. State derives from the signed manifest
being present AND the exec binary existing.

**Right — harness scanners & manifest summary:**
- `Harness scanners:` — e.g. `ACTIVE miniguard-firewall.js` (opencode plugin
  installed) and `ACTIVE mg-content-scan` (`/usr/local/bin`).
- `manifest summary:` — the signed manifest fields for a selected plugin:
  `plugin`, `version`, `schema`, `capabilities` (e.g. `prompt_eval, mem_scan`),
  and `exec_sha256` (truncated hash pin).

### Governance Validation

`Governance Validation` — press `v` (`[v] run full suite`) to fire real checks
at the running system. Idle text explains the suite. When running it shows
`RUNNING…`, then one row per check as `{PASS|FAIL|INDET} {name} {detail}`, then
a bold `summary: N pass, N fail, N indeterminate`. The eight checks:

| Check | Meaning |
|-------|---------|
| `eval socket reachable` | pings the tokenless eval socket |
| `prompt guard allows benign` | a benign prompt must pass the prompt guard |
| `prompt guard detects injection` | a prompt-injection must be detected/withheld |
| `live classify: destructive -> INT_HARM` | destructive text must classify as harm |
| `live classify: benign -> INT_SAFE` | benign text must classify as safe |
| `provenance chain audit` | independent audit over the action ledger; `INDET` if indeterminate, `FAIL` if refuted records exist |
| `plugin manifest + hash pin` | installed plugin binary hash must match the manifest `exec_sha256` |
| `supply sentinel: dependency integrity + policy` | `mg-supply` audit; `WARN`→INDET, `QUARANTINED`→FAIL |

---

## 4. Developmental Governance window (page 3) — press `e`

Per-agent behavior records from the conditioning layer: how each governed AI
tool has behaved across sessions and what autonomy it has earned. An agent
appears after its first governed `mg-cli` session. Pressing `e` again
refreshes the list.

**List view** — one row per agent: `AGENT`, `SESS` (total sessions), `ACTS`
(total actions), `BLK` (total blocked), `LEARN%` (learning score; green above
90%, yellow above 70%), `CLEAN` (consecutive clean actions), `CAPS`
(`baseline`, or `+N` capabilities earned). `↑`/`↓` select, `Enter` opens the
detail view.

**Detail view** (`Esc` goes back to the list):
- Agent ID, tool, workspace.
- **Behavior Summary** — total sessions, actions and blocks, learning score,
  consecutive clean actions and sessions.
- **Block Reasons** — each distinct reason with its count (`×N reason`).
- **Capability State** — each capability as `baseline` or `EARNED`:
  `workspace_write`, `workspace_exec` (always available),
  `outside_workspace_read`, `network_allowlisted`, plus `max_context_tokens`,
  `parallel_sessions` and `operator_prompt_threshold` (lower = more autonomy).
- **Earning Progress** — how many more clean actions/sessions until the next
  capability is earned, or `EARNED`.

In the detail view, **`r` resets that agent's record to baseline** (the
daemon's `conditioning_reset`: totals, block reasons and learning score). This
is different from `r` everywhere else, which clears the audit trail.

## 5. Tool Approvals window (page 4) — press `t`

The one place to decide everything waiting for a human. Open it directly with
`miniguard-status --approvals` — that is what clicking a MiniGuardian approval
notification runs (under `sudo -k`). Pressing `t` again refreshes, and the list
also refreshes itself while the page is open.

Rows are marked by kind:

| Kind | What it is | Decided through |
|---|---|---|
| `call` | a write/exec/delete/network call queued by the daemon's tool system (`mg-trust-tool`, `mg-mcp-bridge`, …) | daemon `tool:approve` / `tool:reject` |
| `file` | a risky file action in a governed `mg-cli` session | that session's socket (as root) |
| `sess` | a tool call queued inside a governed `mg-cli` session (hook bridge) | that session's socket (as root) |
| `xN` | an unregistered tool a governed AI tried to call (N = repeats) | daemon `toolreq` |

Each row: request id, tool (or file action), kind, requesting agent (or
`session <pid>`) and time, preview (or path).

**Groups.** Session file actions from the same session, with the same action,
in the same directory collapse into one line — `▸ ×14  delete  file … /dir/
(14 items)`. `Enter`/`x` on a group approve/reject all of its items; `o`
expands it (`▾`) to show each item as `└ fv-N`, where `Enter`/`x` act on that
one item.

**What never reaches this list:** an AI tool's own housekeeping in a
governed session (session transcripts, caches, backups, lock and temp files —
e.g. `~/.claude/sessions/`, `~/.claude/projects/`, `~/.claude/.claude.json.tmp.*`,
`.bash_history-*.tmp`) is audited but not queued. Its settings, hooks,
skills, commands, agents, `.mcp.json` and `.claude.json` are never treated
that way. The title shows `N pending` and
the auto-approve posture (`auto-approve off`, or the number of rules, earned
trust, and a running `WINDOW Nm left`).

| Key | Effect |
|-----|--------|
| `↑` / `↓` | select a request |
| `Enter` | **approve** (a `×N` group: all N) |
| `x` | **reject** (a group: all of it; a rejected `file` item ends that governed session, like a hard block) |
| `o` | expand / collapse the group under the cursor |
| `s` | **approve-global** (`xN` only) — admit the tool and persist it to `[ai_gate] extra_allowed_tools` |
| `w` | open / close the **timed auto-approve window** (needs `[auto_approve] enabled = true`; see `GOVERNED_TOOLS_REFERENCE.md`) |

The same actions are available on the auth socket (`tool:pending|approve|reject
<id>`, `toolreq ...`, `autoapprove status|on [min]|off`) and, for session items,
with `sudo mg-cli session review`.

## 6. Kernel Judge window (page 5) — press `k`

A read-only view of the Jayce kernel. The daemon runs the kernel's engines
in-process all the time (the `jayce_kernel` port, ticked at 1 kHz), so this
page always has live kernel metrics, even with no QEMU window or kernel VM. An
attached kernel adds its own telemetry on top:

- **The kernel is running in its QEMU window** (`jayce-kernel-0.3/build.sh`,
  the usual harness). The window tees its serial telemetry to a log, and every
  panel reads the newest frame from that log, marked **UNVERIFIED**.
- **The daemon runs the kernel as its judge** (`[kernel_vm]`). Panels show the
  daemon's authenticated values, which win over the serial log whenever
  present.

From top to bottom:

- **Jayce kernel** strip: where the numbers come from, kernel uptime, frame
  count, handshake state (`VERIFIED` only with a present, aligned, unlatched
  judge and both fingerprints), the peace-audit score as a bar, ISD/PEH status
  (`clear` or `N flagged`), and the proof grade.
- **Kernel engines — daemon, live in-process (self-attested)**: always shown.
  It reads the daemon's `kernel_port` socket command about once a second:
  - Impossible State Detector: checks, hits, freeze and quarantine actions.
    The mask is decoded into the six kernel checks plus three host-only ones:
    dead pid holding fd, zombie with active connections, orphaned locks.
  - Predictive Execution Horizon: forecasts, mitigations, blocks,
    quarantines, and the attacker playbook.
  - Self-Healing Causality Mesh: breaks and observations, per-chain break
    counts, and the decoded break mask.
  - Constitutional Shadow, illusion state, and the truth chain.
  - Doorman, network verdicts, void pit, watchdog, burn pit, service
    supervisor, piggyback bus, and the automata monitor.

  A counter that can signal trouble is yellow when non-zero and red when it
  rose since the last poll. These are the daemon's own engines, so they are
  self-attested, never judge-graded. An older daemon without `kernel_port`
  shows a "rebuild and reseal" notice.
- **Kernel vitals**: only when a kernel is attached (QEMU serial log or the
  daemon's kernel VM); see below.
- **Harness**: Robot Command Fabric (active programs, ticks, compiles+sims,
  completed; the serial-command acknowledgement exists only on the daemon
  path) and the Temporal Proof Ledger (records, latest violation with its RCF
  legend, chain head).
- **Judge / lineage**: the authority chain, judge presence/mode/agreement and
  the absence reason, and the kernel's engine-lineage (`l=`) and judge-protocol
  (`p=`) fingerprints compared with the ones this `miniguard-status` build was
  compiled against: `matches this build`, `DIFFERS from this build`, or `not
  observed`. A match from a serial log is not verification — anyone who can
  write the log can forge it. `Daemon engines` compares the daemon's own engine
  lineage with this TUI build; `DIFFERS` means the daemon and TUI were built
  from different kernel sources (rebuild and reseal both).

The page never executes kernel harness actions. It observes evidence only.

### Kernel vitals panel (middle of the `k` page)

Live counters the kernel exports on its serial port (one `JAYCE_TLM:` frame every
500 ms; all values are running totals, so the panel shows a **rate** and a short
**trend line** rather than the raw total alone). Fields: security counters
(capability / IPC-endpoint / shim denials, endpoint misses, boundary violations,
service restarts, burn-pit entries), scheduler (preemptions, starvation events,
run-queue depth), safety engines (peace-audit score, impossible-state mask, PEH
hazard mask, memory-map flag), Robot Command Fabric and proof-ledger counters.
On wide terminals (120+ columns) it uses three columns: safety engines, security
counters, and scheduler + RCF + ledger.

Under the safety engines, the two masks are spelled out with the kernel's own
flag names. **Impossible State Detector** (`isd_m`): timing quantile inversion,
capability counter reversal, signal accounting breach, memory budget breach,
engine flow contradiction, policy order breach. **Predictive Execution Horizon**
(`peh_h`): crash risk, deadlock risk, runaway engine, dangerous actuator, policy
violation, timing anomaly, invariant divergence. A clear mask shows `✓ all N
checks clear`; each raised flag shows as a red `✗` line. A unit test keeps these
tables in step with the kernel sources. SHCM does not export a mask in
telemetry, so it is not listed here.
Fields the kernel adds later still appear under "Other (new kernel fields)".

Colours: a security counter that rose in the last ~10 s is **red**; a non-zero
counter that is not rising, or a non-zero mask, is yellow; zero/steady is green.
Trend lines: counters draw the per-frame increase, levels draw their own
min-to-max (a flat non-zero level is drawn at mid height, "steady").

The title says where the frames came from:

| Title | Source | Trust |
|---|---|---|
| `daemon coprocessor` | frames the daemon's kernel VM forwards over the socket (`kernel_vm`) | verified path (not yet forwarded by the daemon — see AGENTS.md) |
| `serial log — UNVERIFIED, display only` | a log file written by a QEMU window | anyone who can write the file can forge frames; monitoring only, never evidence |
| `STALE — frames stopped` | no frame for 5 s (kernel paused/exited/log gone) | — |

To feed the panel from your QEMU window, just start it the usual way
(`jayce_kernel_src/build.sh`): the GUI launch now also tees the kernel's serial
output to `/tmp/jayce-kernel-serial.log` (`JAYCE_KERNEL_SERIAL_LOG` to change it;
the log is truncated at each launch and grows about 1.4 KB/s). The map's telemetry
pipe still receives every frame — QEMU's chardev `logfile=` is a pure tee. Point
the TUI at another file with `miniguard-status --kernel-log <path>`. The TUI runs
as root, so it treats that file as untrusted: it refuses symlinks and files owned
by anyone other than root or you, and reads only bounded chunks.
Identity-firewall and workspace-ACL switches remain independent configuration
gates and cannot be enabled from this view.

---

## Authority Required modal (any window)

If the substrate demands a human authorization decision, a red modal overlays:

- **`⚠ AUTHORITY REQUIRED`** — red bold title.
- The pending **authority summary** (what decision is needed).
- **`Enforcement continues automatically — this needs a human decision.`**
- **`Suggested: g-page fleet rules · console web Revoke/Suspend`**
- `[space] acknowledge · [d] mute ding · q quits`

Pressing `space` acknowledges and dismisses it.

---

## Keyboard shortcuts

| Key | Effect |
|-----|--------|
| `q` | quit |
| `Esc` | quit — except on the Developmental Governance page, where it closes the agent detail view |
| `a`, `A` | toggle AI Safety window (page 1); from any other page, open it |
| `g`, `G` | toggle Governance Center (page 2); from any other page, open it |
| `e`, `E` | open or refresh Developmental Governance (page 3) |
| `t`, `T` | open or refresh Tool Approvals (page 4) |
| `k`, `K` | open or refresh Kernel Judge evidence (page 5) |
| `c`, `C`, `r`, `R` | (except `r` in the agent detail view, see page 3) clear the audit trail + live feed and wipe transient counters — but NOT the daemon's cumulative lifetime counters (Engine Status, DDoS, blocked IPs, kills/blocks/restores, telemetry). Those keep their values, and the live feed refills on the next tick. See note below. |
| `v`, `V` | run the live governance validation suite (gov page) |
| `m`, `M` | arm / confirm governed migration of the selected ungoverned PID (gov page) |
| `↑` / `↓` | scroll validation results / move governed-tools selection (gov page) |
| `←` / `→` | switch focus between Active Threats and Audit Trail (main page) |
| `↑` / `↓` | scroll the focused feed panel (main page) |
| `↑` / `↓`, `Enter` | select an agent / open its detail (Developmental page) |
| `r` | reset the selected agent to baseline (Developmental detail view only) |
| `↑` / `↓`, `Enter`, `x`, `s`, `w` | select / approve / reject / approve-global (`xN` only) / timed auto-approve window (Tool Approvals page) |
| `space` | acknowledge the Authority Required modal |
| `d` | mute/unmute the authority-modal terminal ding |
| (resize) | immediate redraw / layout reflow |

To get back to the main dashboard, press the key of the page you are on if it
toggles (`a`, `g`), or `a` then `a` from any other page.

**Refresh:** the TUI re-fetches status automatically every tick. The interval
defaults to 1000 ms; change it with `--interval <ms>`. (There is no manual
refresh key — the page updates itself.)

**Why readings come back after `c`/`r`:** `c`/`r` clears the daemon's recent
audit history plus the TUI's local live-feed buffers (Active Threats, Audit
Trail) and resets transient counters (ransomware, SHCM breaks, illusion
probes, and the AI page's "last event" fields). It does **not** reset the
daemon's cumulative lifetime counters — Engine Status, DDoS, blocked IPs,
sandboxed PIDs, kills/blocks/restores, and the telemetry/memory graphs are
authoritative daemon state that `c`/`r` intentionally does not touch. Those
values keep their true lifetime readings, and the live feed refills on the
next 1-second tick with whatever is *currently* active. So if threats are
genuinely ongoing, the feed repopulates — that's by design, not a bug.

## CLI options

| Flag | Effect |
|------|--------|
| `--ai` | open the AI Safety window at startup |
| `--approvals` | open the Tool Approvals window at startup (what clicking an approval notification runs) |
| `--demo` | simulate escalating attack scenarios |
| `--interval <ms>` | refresh interval (default 1000) |
| `--kernel-log <path>` | kernel serial log shown (unverified, display-only) as "kernel vitals" on the `k` page; default `$JAYCE_KERNEL_SERIAL_LOG`, else `/tmp/jayce-kernel-serial.log` |
| `--socket <path>` | socket path (default `/run/jayce-operator/miniguard.sock`) |
| `--compare`, `--assess`, `--products` | non-TUI text modes (stdout), not windows |
