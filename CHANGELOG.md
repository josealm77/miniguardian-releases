# MiniGuardian Changelog

All notable user-facing changes by beta release.

## 0.9.1-beta-39 — Fri Nov 20 2026

**`mg-why` now explains problems with Linux itself, not only with your code.**

- **System errors get a cause and a next step.** When a command fails because of the system rather than a program bug — a driver that won't load, a full or read-only disk, another install holding the package lock, a missing or failed systemd service, a port already in use, a name that can't be looked up, "Permission denied" or "Operation not permitted" — `mg-why` (and `why` / `suggest` in the TRust terminal) now says what went wrong and what to try, instead of "no known fix".
- **It reads the kernel's own log when you let it.** Some reasons only the kernel knows: kernel lockdown, AppArmor, seccomp, out-of-memory kills. Run `sudo mg-why -- <command>` and it reads the kernel log lines from while your command ran, names the exact reason and shows the lines. Without `sudo` it tells you when the kernel log may hold the answer. AI agents never receive those lines.
- **A `sudo` run lands in your own history,** so `mg-why last` and the terminal's `suggest` find it.
- **Add your own rules** in `~/.config/miniguard/system/*.toml` (yours are checked first).
- **A new device doesn't work?** If a webcam, headset or other new device does nothing, run `sudo mg-why -- modprobe <driver>` (for most webcams: `uvcvideo`). If the kernel refuses the driver, a reboot loads it.

## 0.9.1-beta-38 — Fri Nov 20 2026

**No more frozen editors and builds, downloads that always get checked, and a terminal that suggests fixes.**

- **Editors, git and builds are no longer frozen as ransomware.** When something overwrites many files in your protected folders at once, MiniGuardian still freezes it immediately. It now also compares the overwritten files with the copies it took just before each write. If they still hold the same kind of content (text stays text, an image keeps its header), the program is resumed automatically and the decision is logged. If they turned into random-looking data, lost their file-type header or disappeared, it stays frozen for your decision. The ransomware drill is still caught.
- **The file vault switches itself back on** a minute after a stall, instead of staying off until a restart. While it is off, a high write rate alone no longer freezes anything (that froze `ld`, test builds and `git clone`).
- **Downloads:** Chrome's in-progress files no longer leave a download approval that cannot be approved or rejected. Files larger than about 100 MB are now checked (they were skipped because of a memory limit). Every decision on a downloaded program or archive is kept on disk and listed by `sudo mg-explain`.
- **A lost kernel judge recovers by itself.** If the judge stops after start-up, MiniGuardian restarts itself after 2 minutes to start a fresh one (at most 3 times an hour), instead of staying in PARANOID until you restart it. `mg-explain` lists these as deliberate restarts.
- **Packages can add system groups again.** Installing a package that adds a group or system account (for example Docker's `docker` group) is no longer undone. Only additions of plain system entries during a package install are kept; any edit of an existing account, a new root account, a new member of a group or a real password is still rolled back. To keep another change on purpose: `sudo mg-op shadow accept /etc/group`.
- **Antibodies you sign travel to your other machines.** When MiniGuardian contains an attack it learns its pattern. `sudo mg-op campaign antibody export <id>` turns one into a proposal; you sign it offline with `miniguard-license antibody-sign` and copy the bundle to `/etc/miniguard/antibodies.json` on each machine. Only your signature is accepted, every antibody expires, and anything carrying a user name or home-folder path is refused. On another machine a match is reported and watched — it never blocks anything by itself. `sudo mg-op campaign feed` lists them.
- **The dashboard has a fleet page** (key `f`): each of your machines, its status as its peers see it, its last heartbeat and what the others report. The key list at the bottom now fits any window width.
- **Quieter, more truthful start-up.** File-guard snapshots and recovery backups now work (their folders were read-only). The false "no kernel lockdown", "no TPM", "shroud install failed" and network-filter warnings are gone, older AI-behaviour records are accepted again, and `mg-explain` dates old crash reports so they are not mistaken for new ones.
- **TRust terminal:** new commands `history`, `alias`, `write` / `append`, `edit`, `playbook` (record and replay, asking before each step), `sec` (security status on one screen), `verify` (installed programs against the seal), `approve` (what waits for you — it lists, it never approves) and `suggest` (the next step for your last failure). `why` now says what to try for common errors in Python, JavaScript, Rust, Go, Java, Ruby and C. Old duplicate names (`explain`, `diag`, `metrics`, …) tell you which command to use, and the built-in `git status` works on packed repositories. Commands that change files or settings work only when a person is typing, never for AI agents.

## 0.9.1-beta-37 — Wed Nov 18 2026

**Slower machines no longer get stuck in PARANOID, and a developer pentest mode.**

- **The kernel judge gets enough time to start.** On slower or busy machines the judge took longer than MiniGuardian waited for it (18 seconds measured on a laptop, against a 10-second limit). It was then counted as missing, and in strict mode MiniGuardian stayed in PARANOID all day, which made it freeze ordinary installs and builds far more often. The wait is now 60 seconds.
- **Docker image builds are recognised as containers,** so `docker build` / `docker compose build` steps are no longer reported as rooted malware.
- **The dashboard shows which version is installed,** in the corner of the MINI GUARDIAN box.
- **`sudo mg-explain` starts with "at a glance":** what is holding the current state and why, then one OK/FAIL line per part (judge, containment, audit chain, ransomware vault…). Repeated entries are collapsed ("×30 since 09:07"), and common harmless warnings carry a short "why" note. Without a Jayce sidecar the trust link now shows N/A instead of DOWN.
- **Fleet: your machines watch each other.** `sudo mg-fleet init` on one machine creates your fleet; `sudo mg-fleet add <name>` makes an enrollment file for each other machine and `sudo mg-fleet join` enrolls it. Enrolled machines find each other on your local network by themselves, and only machines carrying your fleet's certificates can join. Every machine then sends a signed heartbeat once a minute with its status. If one goes silent while still connected (its defences killed?), signs with a different key, replays old messages, or reports different traffic than its neighbour sees, the others flag it. A machine is marked suspected only when two others agree, so one compromised machine cannot get a healthy one blamed, and no machine ever acts on another. See it with `sudo mg-op fleet:glance` or the fleet line in `sudo mg-explain`. Allow TCP 19877 and UDP 19878 between your machines.
- **A removal plan for confirmed rooted malware.** MiniGuardian already freezes confirmed malware and blocks its file from running again. `sudo mg-explain` now also shows how to get rid of it: reinstall the package if a system file was replaced, move an unknown file to quarantine (never deleted), the startup entries that would bring it back with the command for each, then end the frozen process and change passwords. If there are signs of kernel-level access, the plan says plainly to check from outside the system (USB boot) or reinstall. MiniGuardian runs none of these steps itself.
- **AI sessions leave a trail they cannot erase.** Every file change a governed AI session makes is recorded where the AI cannot change or delete it, for the coming code-map tool to show which code an AI wrote.
- **`mg-explain` explains more:** what is holding a SUSPICIOUS state (for example a remembered container pattern), a fleet line, only this window's ransomware incidents (older ones counted), and a clear "still starting" message right after a restart. Machines installed from source with `seal.sh` now get `mg-explain`, `mg-op`, `mg-pentest` and `mg-fleet` too.
- **`mg-pentest` (developer feature, needs a licence from Jayce Automata Research).** Opens a time-limited window in which named pentest tools may attack named target machines from this computer. The tools can reach only those targets (plus their AI model's API); every other protection still applies to them, and what MiniGuardian would have blocked is recorded. It needs a licence signed for this machine and your approval on a red pop-up. Without a licence the command does nothing.

## 0.9.1-beta-36 — Sun Nov 15 2026

**No false "rooted malware" after updates or for container apps.**

- **After a system update, running services are no longer called rooted malware.** When an update replaces a program that a service is still running (for example Python under a system service), the old copy shows as deleted. MiniGuardian used to treat that as a planted binary, and could jump to PARANOID. It now recognises a program the package manager replaced. A deleted program anywhere else is still flagged.
- **Programs inside Docker and other containers** are no longer compared against your host's own program list, which they were never part of. Containers are still watched by the container guard.
- **Slow-scan reports cover the whole cycle,** so any remaining slowness can be traced to the exact step.

## 0.9.1-beta-35 — Sun Nov 15 2026

**Fewer false alarms on machines with GPUs, Docker and package upgrades.**

- **GPU containers are recognised.** A container using your NVIDIA GPU (for example Ollama) is no longer reported as mounting sensitive host files. Docker's own "blank out this path" mounts are recognised too. Mounting real system files or other devices into a container is still caught.
- **`apt upgrade` is never interrupted by MiniGuardian.** During an upgrade, the package manager briefly stages new scheduled-job files under temporary names. MiniGuardian used to move one of those away mid-upgrade, which could stall the upgrade. Those temporary names are never run by the system, so they are now left alone. The final file is still checked.
- **Restarting Docker containers no longer looks like an attack.** A container restart (new virtual network card) was reported as ARP spoofing over and over. Docker's own container-start helper was reported as rooted malware. Normal system threads were reported as hidden processes. All three are fixed, and real cases are still caught.
- **Slow-scan reports name the culprit.** When a scan cycle runs slow, the diagnostic report now says which check took the time.

## 0.9.1-beta-34 — Sun Nov 15 2026

**Fixes two false alarms that could keep MiniGuardian in HOSTILE/PARANOID on a normal desktop.**

- **Docker containers no longer look like an attack.** Every normal container was reported as "mounts sensitive host path /dev/…", because the check read the disk a file lives on instead of the path being mounted. It now looks at the real host path. A container's own files and its volumes are normal. Mounting your `/etc`, `/root`, `/boot`, `/dev`, your whole disk, or the Docker socket into a container is still caught, and those real cases are now caught more reliably than before.
- **Firefox is no longer reported as a broken sandbox on Pop!_OS and Ubuntu.** There, Firefox's content processes are protected by a seccomp filter rather than a separate namespace. Only a content process with neither is flagged.
- If MiniGuardian on your machine kept cycling between normal, hostile and paranoid after installing Docker, this release is the fix.

## 0.9.1-beta-33 — Sun Nov 15 2026

**MiniGuardian now remembers a persistent attacker — even when it changes its IP, tools and timing.**

- **Campaign memory for outside attackers.**
  - MiniGuardian recognises an attack by *what it goes after and in what order*: credentials, persistence, a command-and-control server. It does not go by the IP or the tool.
  - It never forgets with time. An attacker who waits, switches address or splits the work across several agents is still the same campaign.
  - Your own devices, your LAN and your fleet peers are never treated as attackers.
- **It learns from what a contained attacker keeps trying.** Once a campaign is contained, every new move it makes becomes an *antibody*: the shape of the move, with names and addresses removed. The next attacker of the same family is caught on first contact, even with no obvious hostile sign. Steps taken only by programs verified by your package manager (`ssh`, `aws`, `python3`) can never match a learned pattern on their own.
- **Quiet toward the attacker, loud toward you.**
  - A recognised attacker is quarantined silently, so it gets nothing to learn from.
  - You get one **RED ALERT** pop-up per attack and a red banner in the TUI until you press `n`.
  - When watching ends, the attacker is also put on the immunity list, which you can inspect and verify.
- **Starts in shadow mode.** Out of the box it only reports what it *would* do: a yellow "ATTACK DETECTED — shadow mode" notice, and nothing blocked. Switch with `[campaign] mode = "live"` when you're happy with what it reports.
- **You are always in control.** `sudo mg-op campaign list | explain <id> | release <id>` and `sudo mg-op campaign antibodies`. A release removes everything that campaign caused, and stays released.
- **Containers covered.** Attacks that land inside a Docker container are traced back to the attacker's address.
- **For security testers:** `[campaign] treat_as_outside = ["<test machine IP>"]` treats one LAN test machine as an outside attacker, for running your own pentest. It is shown in yellow in the TUI while set.
- **Fixed:** confirming a governed migration in the TUI (`m` twice) froze the whole screen until the daemon answered. It now shows progress at once and stays usable.
- Plus the beta-32 review follow-ups:
  - more dangerous command forms are red;
  - your own build tools' held kills need their name typed;
  - a credential or persistence write is never green.

## 0.9.1-beta-32 — Sat Nov 14 2026

**Dangerous approvals are impossible to miss — and can't be approved by reflex.**

- **Red means dangerous, and only that.** An approval turns red only for destructive commands, hidden code (decoded or downloaded-and-run), your credentials, things that run again after a restart, root or sandbox escape, and remote control. Everything else uses the normal style — red used to be on every pop-up.
- **Every red item says why** in plain words ("Touches your credentials: ~/.ssh/id_rsa", "Runs code you cannot see"), and shows hidden text decoded.
- **No reflex approvals.** A red item is approved only by typing its name. Approve all, groups, the timed window and auto-approve rules never approve a red item.
- **Colours by what Enter does** on the approvals page: every row in one colour — green (Enter is fine), yellow (look first), red (destructive; it blinks and shows its reasons). Held kills of your own build tools (`ld`, `cc`, `cargo`, test programs) are red.
- **A Deny that stops the AI session says so** ("Deny — stops the AI session") and asks once more.
- After 10 approvals in 5 minutes, a red prompt reminds you to read it.
- **Fewer pop-ups:** Python dependency folders (`.venv`, `site-packages`) no longer flood the approval queue.
- `mg-cli session approve <id> --confirm <name>` for dangerous items from a terminal.

## 0.9.1-beta-31 — Sat Nov 14 2026

**Check your machine for known-vulnerable and tampered software.**

- **Host supply-chain scan** (`mg-supply host-scan`, see `docs/HOST-SUPPLY-SCAN.md`):
  - finds known-vulnerable packages across your projects, installed tools and system packages, from an offline advisory database (OSV);
  - finds tampered system files (`dpkg --verify` / `rpm -V`);
  - writes a local report only, with licence attribution — nothing is sent anywhere;
  - runs on demand or on an opt-in weekly/daily timer; `--clean` removes the report, summary and database.
  - A scan with new findings shows one notice, one audit entry, a line in the TUI and a section in `mgx`.
- **AI Constitution in the TUI.** The Developmental page shows the AI Constitution with an owner bail key (Tab to select, `b` twice = one level up). Every privilege override is audited.
- **Fewer passwords.** A pop-up for an item you already approved or denied elsewhere now says "already handled" instead of asking for your password.
- **Licences.** Third-party notices are regenerated from the exact dependencies and installed with both the .deb and the .rpm (`/usr/share/doc/miniguard`).
- **Fixed:** `apodeixis tokens` and `apodeixis ast` now exit with an error code when the input does not parse (they exited 0, so scripts could not tell).

## 0.9.1-beta-30 — Fri Nov 13 2026

**Security update. Please install it.**

- **Security fix.** On beta-29 and earlier, a local program could reach MiniGuardian's built-in tools with root rights through the MCP broker. Every tool call now runs as the user who made it, approvals included, and only that user (or root) can approve it.
- **Leak guard for AI agents.** Before a governed AI session publishes anything (`git push`, `gh gist create`, `gh release create|upload`, `gh repo create --push`), MiniGuardian reads what would leave:
  - credentials and secrets files refuse it before you are even asked;
  - your personal information (your home path, plus anything you list under `[leak_guard] personal`) is flagged on the pop-up.
  - See `docs/LEAK-GUARD.md`.
- **Open plugins, hardened.**
  - Anything a plugin starts ends with the run, even if it detaches.
  - Rejected or abandoned installs are cleaned up, and at most 5 can wait per user.
  - A running plugin's files are never removed by a newer version.
- **Safer installers and helpers.** The opencode and browser installers never work as root inside your home folder. The approval pop-up only ever opens MiniGuardian's own programs.
- **Cleaner packages.** No build-machine paths inside the binaries, and all known dependency advisories fixed (TLS, QUIC, async runtime).
- **Fixed:**
  - the opencode firewall plugin was never installed by the .deb (it showed as inactive in the TUI);
  - a command that only mentions a push in quotes is no longer stopped;
  - refused AI-session network limits now cool down after 10 minutes instead of lasting the whole session.

## 0.9.1-beta-29 — Thu Nov 12 2026

**Build your own MiniGuardian plugins, in any language, safely.**

- **Open plugins.**
  - `mg-plugin new hello --lang python` (or node, rust, apodeixis) creates a plugin that runs inside MiniGuardian's sandbox.
  - It can't touch your files, the network or other programs directly. It asks, and MiniGuardian checks each request against what you approved at install.
  - Reads inside approved folders just work; writes, commands and network ask you.
  - AI agents can use installed plugins as tools.
  - Start with `docs/PLUGIN-QUICKSTART.md` or the new **Open Plugin Developer Manual**.
- **`mg-why -- <command>` explains why any program failed,** in any language. It names the file and line, the cause and the processes involved, with secrets masked. It works outside MiniGuardian too, and you can teach it a new language with a small text file.
- **Internet access through a monitored doorway.**
  - Plugin commands (`npm install`, `pip`, `cargo build`, your program) reach the internet only through MiniGuardian's proxy. Hosts you approved at install pass; a new site pauses for your OK.
  - Every connection is listed in `mg-plugin run` output and in `mg-explain`.
  - Programs that try to go around the proxy have no network at all.
- **AI agents can look things up (opt-in).** With `[session_egress] enabled = true`:
  - a governed agent's commands get the same proxy, with one pop-up per new site per session;
  - one dev-server port for previews, monitored for leaks and for connections from other AI sessions;
  - every `git push` asks you first.
- **Confidential data.** Mark folders as confidential. A plugin that reads them, or reads keys or `.env` files, must ask you before sending anything off the machine.
- **Supply Chain Sentinel v2** checks Python and Node dependencies and git contents, and blocks plugins whose dependencies can't be verified.
- **Fixed:**
  - a program running as you could stop the MiniGuardian daemon while a plugin action was running;
  - plugin commands could not start at all;
  - the ransomware rule could silently freeze AI agents and builds;
  - a long approved build no longer holds up every other approval.

## 0.9.1-beta-28 — Tue Nov 10 2026

- **The kernel judge is now included, free in the beta.** MiniGuardian runs as root, so on its own it would be the single highest authority on your machine. The kernel judge is an independent copy of its decision engines running in its own small, isolated virtual machine; it must agree with every decision, so taking over root alone is no longer enough to change what MiniGuardian decides. If the judge is missing or disagrees, MiniGuardian stays in its strictest mode. Earlier package installs had no judge, which is why the dashboard showed "judge absent".
- The judge's virtual machine is locked down: it runs as an unprivileged user inside QEMU's own security sandbox, with no network and no display. QEMU is installed automatically as a dependency (about 30–60 MB if you don't have it).

## 0.9.1-beta-27 — Sat Nov 07 2026

**Attack immunity: what MiniGuardian stops once, it stops at the first step next time** — after a restart, and when the same thing comes back under a different name or path.

- **Ransomware.** Files in your protected folders are copied just before they are overwritten. A program overwriting 50 or more existing files there within 5 seconds is paused immediately — including scripts that start one small helper per file, and programs that detach themselves first. Put files back with `sudo mg-op ransomware restore-since <minutes> <folder>`. The paused program's file is remembered and refused from then on.
- **Malicious programs.** A binary MiniGuardian confirmed as malicious is refused from any path, even renamed or copied.
- **Downloads and scheduled jobs.** A download you rejected is quarantined at once if it shows up again (even renamed); a scheduled job MiniGuardian held is held on sight if it is planted again under another name.
- **Floods.** During a connection flood, unknown addresses are limited to a small number of new connections each; existing connections, known UDP traffic and your management addresses always pass (IPv4).
- **Choose your deployment.** `sudo mg-setup` asks what the machine is for (workstation, server, drone, vehicle) and sets the numbers; the Ubuntu/Debian installer can ask for you. Until a profile is chosen, the dashboard says so. Your existing `miniguard.toml` is never rewritten.
- **New command `mg-op`** for operator actions: `sudo mg-op ransomware list | resume <pid> | explain <pid>`, `sudo mg-op immunity list | release <id>`.
- **Easier to investigate:** `mg-explain` now shows what came close to triggering but did not ("near misses"), and if the daemon cannot start, how far it got and why.
- **Heads-up:** tools that rewrite many existing files in your protected folders very quickly (whole-project formatters, code generators, some test suites) can be paused. `sudo mg-op ransomware resume <pid>` lets them continue; see the Error and Recovery Guide.
- Fixed: a start-up crash loop after a saved-state reload; a dashboard hazard that lit at random; false "not in baseline" alarms after system updates.

## 0.9.1-beta-26 — Fri Nov 06 2026

- Fixed: after every start MiniGuardian reported its kernel programs as tampered with and reloaded them. The check now verifies them the right way, and the false warnings and reloads are gone.
- When something pauses (freezes) the MiniGuardian daemon, the evidence now lists the root programs that could have done it (terminal shells and debugging tools) instead of unrelated system services.
- The AI behaviour posture now has one consistent name, and every change of it is recorded in the audit log with its cause.
- `mg-explain`: new `--until=TIME` option to look at one short moment in the past; `--dig` no longer treats MiniGuardian itself as a suspect after a freeze.

## 0.9.1-beta-25 — Fri Nov 06 2026

- New `mg-explain` command: `sudo mg-explain [since] [--dig]` explains what MiniGuardian decided and why — every threat-state change with the events that caused it, audit decisions, held kills, downloads waiting for you, seal state and recent warnings. `--dig` follows the evidence through fixed, read-only checks to a one-line conclusion. It changes nothing.
- The approvals page now says when auto-approve is turned off in the configuration, and pressing `w` explains how to turn it on, instead of a message that disappeared.
- The code analyzer no longer reports normal test vocabulary ("dummy", "stub") in test code.

## 0.9.1-beta-24 — Fri Nov 06 2026

- Fixed: on Fedora 44 the daemon did not start (the status page showed "disconnected"): systemd refused to set up the service under SELinux. If you installed beta-23 on Fedora, update to this release.
- Fixed: the kernel hardening settings file was missing from the packages, so installs from a package ran without it. It is now installed with the package.

## 0.9.1-beta-23 — Thu Nov 05 2026

- TRust terminal: commands that printed nothing now work, or say plainly that they are not available in this build. The help menu is reorganised by topic.
- TRust terminal: new commands that show real data only — host and daemon status, the brain's state and recent decisions, AI governance counts; notes, todo, timers, focus and saved sessions; project scaffolding, dependencies, workspace overview and a Markdown reader; web fetch/search and weather (run as you, refused while MiniGuardian is in a threat state, private host-only history that needs your password to view); a local network overview.
- TRust terminal: the code analyzers print a short summary instead of thousands of lines, with many false alarms removed.
- Forensics: every change of MiniGuardian's threat state is now recorded in the tamper-evident audit log **together with its cause** — the events that drove the decision, strongest first — so a HOSTILE or PARANOID moment can be explained afterwards instead of guessed at. The `forensic` report shows the live state, why it is being held (for example the kernel judge's start-up latch), progress back to calm, the last real event, and whether MiniGuardian's kernel programs are in place.
- TRust terminal: `ada` shows the recent state changes with their full causes and how long ago they happened; `forensic` / `forensics` open the incident review from the audit log.
- TRust terminal, a fix workflow that explains failures instead of re-running everything: `build`, `check` and `test` print a **fix summary** under any failure — the cause, the file and line, what was ruled out (such as known environment-only failures), the suggested direction, and the one command that verifies a fix. It also says when a fix needs the owner (sensitive files, deploy steps, deletions).
- TRust terminal: `fixes` keeps those summaries in a compressed history — `fixes repeats` shows causes that keep coming back, `fixes timings` the slowest parts of a build and anything built twice, `fixes disk` build folders and free disk space (cleaning always asks first). `test` runs only the tests for what changed since the last passing run (`test --all` for everything). Scripts can produce the same summary with `trust_terminal --explain-failure`.
- TRust terminal: the window layout follows the window width (it stayed narrow when maximised).
- Fixed: a brief false PARANOID state after the daemon restarted.
- Fixed: a routine system update held the dashboard at HOSTILE; a changed system file re-alerted on every scan; every web connection raised a certificate alert; false "audit chain tamper" warnings.
- Kills of processes MiniGuardian cannot verify are now held for your decision, with the process's network cut off meanwhile.
- New scheduled-job coverage (cron folders, user crontabs, at jobs, new timers); a suspicious scheduled job is set aside and you are asked, instead of it being deleted.
- The approvals page explains each request in plain English: who is asking, what exactly, and what Approve and Reject do.
- Kernel hardening settings are applied again, and the daemon uses about half the CPU it did.

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
