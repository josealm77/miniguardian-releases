# MiniGuardian Changelog

All notable user-facing changes by beta release.

## 0.9.1-beta-22 — Tue Oct 13 2026

- Added `mg-mcp-broker`, a small root sidecar that lets any MCP client (opencode included) use the governed tools without needing the daemon's secret key. It only answers processes that are part of a governed session, and it is installed and started as its own service.
- Added `tool_status`: when a tool waits for your approval, the agent can now ask for the result after you approve it, instead of only ever learning it had been queued.
- Fixed: the daemon's management socket and key file did not appear on the host, so the status TUI, the broker and socket commands could not reach the daemon.
- Fixed: desktop notifications and clipboard tools failed inside the daemon ("Cannot autolaunch D-Bus without X11 $DISPLAY") because a root system service has no access to your desktop session. They now go through the same desktop helper the approval pop-ups already use.
- Fixed: the `memory_search` and `memory_recall` tools always failed with "unknown command".
- Fixed: approved builds (`run_tests`, `lint`, `build`) could run forever without a time limit, and could not find `cargo`.
- Approved tools that fail now report the real error instead of "unknown status".

## 0.9.1-beta-21 — Mon Oct 12 2026

- Set `MINIGUARD_LOG_STDERR=1` to mirror the daemon's error log to stderr. It also prints the child's Landlock network profile, which is what you need to diagnose a governed session that can't reach a loopback port.

## 0.9.1-beta-20 — Mon Oct 12 2026

- Fixed a fresh install where governed identity provisioning failed: the restricted systemd service PATH has no `/usr/sbin`, so `groupadd`/`useradd`/`setfacl` were never found. They are now called by absolute path.
- Added the `MINIGUARD_LOG_STDERR` environment variable for debugging startup failures.

## 0.9.1-beta-19 — Mon Oct 12 2026

- Fixed the packaged default config: the plugin uid/gid did not match the governed identity, so a fresh install with both subsystems enabled exited fatally at startup.

## 0.9.1-beta-18 — Sun Oct 11 2026

- Fixed the systemd unit so the daemon can start on a fresh install. It was not allowed to write to cgroups, so governed identity provisioning could not create its own cgroup and the daemon died on startup.

## 0.9.1-beta-17 — Sun Oct 11 2026

- Governed identity is now enabled and provisioned by default on a fresh install.
- A tool call the daemon deceived about now exits 203 with a machine-readable `# MG_SYNTHETIC` marker instead of an ambiguous failure.
- Download guard: copies rather than moves the file it holds, clears the exec bits on the original, skips duplicates, applies a 7-day TTL and a 1 GiB total cap, and refuses in-progress `.part` downloads. This fixes the empty-hash race during a release download.

## 0.9.1-beta-16 — Fri Oct 09 2026

- Harden seal.sh: daemon and sentinel binaries are now installed chmod 750 before chattr +i, so unprivileged local users can no longer copy them for offline static analysis. User-facing helpers (mg-cli, miniguard-status) remain world-executable.
- Added the 2026-10-09 security evaluation report and preliminary results (judge coprocessor online in strict mode, 1050/1050 campaign pass, 170/170 benchmark pass).

## 0.9.1-beta-15 — Mon Oct 05 2026

- Security: governed sessions could bypass the network allowlist with MPTCP sockets (Landlock only covers plain TCP). A per-session seccomp guard now refuses MPTCP/SCTP so programs fall back to TCP, which is governed.
- Security: team-chat text could inject operator commands, and the chat bridge lent the root operator token to any local caller or web page. Messages now travel encoded, the bridge holds no token, and the web servers reject foreign web pages.
- opencode 2.x runs governed: a private per-session server replaces its shared background service (which crashed under mg-cli).
- MiniGuardian's opencode plugins work on opencode 1.x and 2.x.
- claude login, agy and ollama get the local ports they need in governed mode.
- Team chat: the chat window asks for your password once to send as you; approve/reject, long and multi-line messages and task titles with spaces work. Team chat stays off by default in this beta.

## 0.9.1-beta-14 — Thu Oct 01 2026

- Security: under `sudo mg-cli`, governed tools ran as root. The read tier (no approval) could read root-only files, approved writes left root-owned files, and an approved `run_command` ran as root. Governed tools now run as the desktop user: a governed action can do exactly what you can.
- Fixed: clicking an approval notification did nothing on COSMIC (the click was never recognized). Clicking now opens an Approve / Deny / Review all dialog; Approve and Deny ask for your password (polkit) and then apply.
- Fixed: `mg-cli session approve <id>` without `--pid` failed with "multiple sessions" because sockets of ended sessions were never removed. Dead sessions are now ignored and cleaned up.
- Fixed: text you paste into Claude Code (`~/.claude/paste-cache/`) was queued for review as if an agent had written it.

## 0.9.1-beta-13 — Thu Oct 01 2026

- Fixed: `mg-cli --bind-ports <port>` now also allows connecting back to that port. Landlock treats bind and connect separately, so a tool that started its own local server on the granted port (opencode on Fedora) hung retrying a denied loopback connect unless `--net-ports` repeated the port.
- Fixed: every editor/AI-tool save (write a temp file, rename it onto the real one) was scored as a risky delete; after 7, mg-cli queued every action for review. A rename inside the watched tree is now a move.
- Fixed: TRust terminal language adapters now run code (verified: Python, C, C++, Java, Rust, Perl, Bash; Go, Zig, PHP, Swift and Kotlin were corrected but not yet tested). Builds use a private temp directory and every evaluation has a 60 s limit. New `@<lang> <code>`, `@<lang> … @end` blocks, and `langs`.

## 0.9.1-beta-12 — Tue Sep 29 2026

- Fixed: a governed AI CLI that rebinds a fixed local TCP port for its own background service (reported: opencode on Fedora, beta-11) was denied by the network sandbox — permission denied on the specific port. `--allow-bind` only ever granted ephemeral port 0, which doesn't cover a tool asking for a specific port back. New `mg-cli --bind-ports <list>` grants explicit local TCP ports (e.g. `--bind-ports 49374`), independent of `--allow-bind`. Same loopback caveat as `--allow-bind`: Landlock matches by port number only.

## 0.9.1-beta-11 — Mon Sep 28 2026

- Fixed: on Fedora Workstation a sealed install (beta-5 to beta-10) could boot to a blank screen with a cursor after the first reboot. The seal no longer adds hidepid=2 to /etc/fstab unless asked (mg-seal --hidepid); it is the only seal step that changes how the machine boots. Upgrading removes the entry an earlier seal added (an administrator's own hidepid line is kept).

## 0.9.1-beta-10 — Sun Sep 27 2026

- Findings MiniGuardian checked and found normal no longer appear under "Active Threats". They are counted ("checked, normal") and recorded as ALLOW in the audit trail, so nothing that has been cleared looks like a problem. With nothing to report, the panel says "No active threats."
- New exposed-service monitor: programs other machines can reach (listening on 0.0.0.0, :: or a public address) are registered the first time they are seen and shown as "checked, normal" in the Network panel. They are flagged only on risk signs: a port taken over by another program, a program running from a temp folder or a deleted file, a listener with no owning process, a root program outside the system folders, or an attack-tool port.
- Fixed: an inbound port-scan check treated listening sockets as incoming connection attempts, which could report the machine's own services as a scan.

## 0.9.1-beta-9 — Sun Sep 27 2026

- The ransomware monitor no longer flags MiniGuardian's own daemon and sentinel. Both write their audit log and state continuously and disguise their names as kernel threads (e.g. "rcu_gp"), so they showed up as a kernel thread doing mass file modification. They are now recognised by their service cgroup and binary, never by name.

## 0.9.1-beta-8 — Sun Sep 27 2026

- The "library hijack" false positive for programs that unpack a library to their own temp folder (Python espeak-ng/TTS wheels, PyInstaller apps) is fixed for real. The daemon runs with a private /tmp (systemd PrivateTmp=yes), so its checks looked in the wrong /tmp; paths are now resolved as the program sees them (/proc/<pid>/root). Libraries the program deletes after loading are judged by the file itself (/proc/<pid>/map_files). Group-writable files count as private when the group is the owner's own private group (the Ubuntu/Pop!_OS default). Libraries owned by another user, writable by others, or loaded by root are still flagged High.

## 0.9.1-beta-7 — Sun Sep 27 2026

- Upgrades over a sealed install are reliable: the services are stopped first, every file the installed package owns is unlocked (from the package manager's own file list), and the read-only /usr/local/bin mount is removed before unpacking. If resealing fails afterwards, the services are still started.
- Every install/upgrade step is logged to /var/log/miniguard-install.log.
- Uninstall is complete: immutable directories are unlocked before files are deleted; the seal's /etc/fstab hidepid entry is removed and /proc remounted without it (an administrator's own hidepid line is kept); browser integration files are removed. mg-seal --unseal also unlocks directories and removes every stacked mount; the new --unseal --full also reverts hidepid. hidepid detection recognises "hidepid=invisible".

## 0.9.1-beta-6 — Sun Sep 27 2026

- False positive fixed: a program loading a library it unpacked into its own private temp folder (Python speech/TTS wheels such as espeak-ng, PyInstaller apps) drove the host to Hostile as a "library hijack". Such loads are now logged at Low and no longer raise the threat level. Libraries in shared or other-writable temp folders, loads by root, and anything in /dev/shm are still flagged High.
- Public README: copy-paste install commands with the real version.

## 0.9.1-beta-5 — Sat Sep 26 2026

- Approvals: desktop "Review" pop-up opens one TUI approvals screen (miniguard-status --approvals) for daemon tool calls, unregistered tools and every mg-cli session; approve/reject need root (a governed agent can no longer approve its own queued actions); similar items collapse (xN); AI-tool housekeeping files are audited instead of queued; opt-in auto-approve (operator rules, earned trust, timed window).
- Governed CLI agents can no longer reach the X server, session D-Bus, Wayland or PipeWire (mount-namespace masking + Landlock abstract-unix scoping).
- False positives fixed: fresh daemon going Hostile (sandbox_escape, USB baseline, own-component detection, route baseline, gdbus); cargo builds with a /tmp target dir no longer flagged as library hijack (narrow rustc + proc-macro + same-owner exemption).
- Status TUI kernel page redesigned: live Jayce kernel engine metrics from the daemon's in-process engines (new read-only kernel_port socket command) with ISD/PEH/SHCM flags decoded; QEMU serial telemetry feeds the RCF/TPL harness panels (marked unverified); lineage checked against the build.
- Docs: Jayce kernel engine names corrected (Impossible State Detector, Predictive Execution Horizon, Self-Healing Causality Mesh).

## 0.9.1-beta-4 — Fri Sep 25 2026

- recursive_illusion.rs: RealityVerdict::LoopDetected was defined but never constructed — an agent probing indefinitely just cycled the same 7 fake layers forever with no escalation. Now wired: sustained probing (>10 attempts, >3 awakenings) overrides the verdict with a distinct response.
- illusion_bubble.rs: fixed a real blacklist bypass (quote-splitting, e.g. `cat /e''tc/shadow`) that previously fell through as Allow; added per-call variation to the ~30 static fake-response fallbacks so repeated probing no longer gets byte-identical output every time.
- New: cumulative session-risk escalation (mg_cli_governor.rs) — a session accumulating risky verdicts now loses auto-trust for the rest of its own run (threshold tuned to 7), on top of the existing non-blocking operator-review queue for individual risky actions.
- New: AGENTS.md now states an explicit, permanent rule — enforcement decisions are never LLM-reasoned, only deterministic (rule/proof-checked mission/counter-threshold/state machine). Audited: true everywhere in the live daemon already; a guardrail script now checks for regressions.
- mg_fanotify_gate.rs (experimental --fanotify-gate): added a 200ms fail-open timeout around its one synchronous mission call, so a hung evaluation can no longer block a governed child's write indefinitely.
- Fixed two real test-suite bugs: frame_guard.rs/wasm_detector.rs scanned the real system /tmp unconditionally, causing false positives on ordinary pre-existing files; and miniguard/tests/integration.rs's 14 tests needed root by design (SO_PEERCRED enforcement) and are now correctly marked as such instead of failing.

## 0.9.1-beta-3 — Sun Sep 20 2026

- AI Constitution privilege ladder now actually gates tool calls (was tracked but never enforced); human-operator override and the tick-loop violation queue fixed to survive the next brain sync instead of being silently wiped.
- auto_govern "migrate" mode now really kills and relaunches an ungoverned AI process under mg-cli (was silently a no-op notify).
- Sentinel no longer leaks its identity via a literal process name; process masking now discovers this machine's own live kernel-thread names instead of a static, PID-parity-predictable list.
- seal.sh's mg-cli self-signing no longer fails with a permission error on the staging directory handoff.
- New PromptGuard attack patterns, write-time secret scanning for AI memory, a real license-signature check in mg-Collaboration, and a fix for a trust_terminal persistence crash loop.

## 0.9.1-beta-2 — Sat Sep 19 2026

- Rebuilt from the current tree: sentinel no longer restarts with the daemon (Wants= instead of Requires=), accurate daemon uptime and health readiness, kernel vitals panel in miniguard-status, hardened terminal launcher.
- Ship the standalone trust_terminal (miniguard --terminal starts it).
- Ship mg-cli's signed integrity manifest; without it mg-cli refuses to run.
- Explicit root:root ownership for every file.

## 0.9.1-beta-1 — Tue Sep 15 2026

- Add staged UID-scoped governed identity DNS/UDP firewall profile.

## 0.9.0-1 — Wed Sep 02 2026

- Sidecar beta RPM package (free tier, up to 3 devices).
- Ships miniguard, miniguard-status, miniguard-sentinel, mg-cli, mg-trust, mg-mem, mg-supply, mg-governance-audit, mg-chat family, and systemd units.
- Robotics saddle, co-host, and >3-device clustering are licensed out of this build.
