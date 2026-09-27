# 🛡️ TRust Smart Terminal — Complete Command Reference Manual

> **Status (measured 2026-09-19).** This manual documents the command catalog inherited
> from the Jayce SDK's TRust Terminal, which registers 226 commands there. Running each of
> the 157 catalog names in a sandbox showed: **23 execute natively** in this build, **28 are
> ordinary host programs** (`ls`, `cat`, `git` and similar, which work when installed),
> **12 are hyphenated forms of `robot` / `drone` / `swarm` / `vehicle` subcommands** (write
> them with a space: `robot move`), and **94 are catalogued but not implemented here**. Every
> one of those 94 has a handler in the SDK's `trust_core` (or, for `drone-waypoint` and
> `swarm-offload`, is listed in the catalog only); the handlers were not ported. In the
> terminal, `commands` lists the catalog with these marked `[not available in this build]`,
> and running one prints a clear message (exit 127) instead of `spawn error`.
> Per-command table: `docs/reports/claims-audit-2026-09-19/terminal-command-status.tsv`.
> **Re-checked against the code 2026-09-26:** the catalog and the not-implemented list
> are unchanged. Rows below carry their status inline.

*Jayce Microkernel, IronPilot Robotics & MiniGuardian Security Substrate v0.9.0-beta*

---

## 📌 Table of Contents
1. [MiniGuardian Local Security & Forensics](#1-miniguardian-local-security--forensics)
2. [Microkernel & Engine Commands](#2-microkernel--engine-commands)
3. [Autonomous Robotics & Motion Control](#3-autonomous-robotics--motion-control)
4. [Aerial Drone Fleet Management](#4-aerial-drone-fleet-management)
5. [Autonomous Vehicles & Ground Fleets](#5-autonomous-vehicles--ground-fleets)
6. [Swarm & Fleet Coordination](#6-swarm--fleet-coordination)
7. [Hardware Bridges & Signal Processing](#7-hardware-bridges--signal-processing)
8. [Cognitive & ADA Commands](#8-cognitive--ada-commands)
9. [Security, Deception & Isolation](#9-security-deception--isolation)
10. [Orchestration & Parallel Universes](#10-orchestration--parallel-universes)
11. [Development, Build & Multi-Language Tools](#11-development-build--multi-language-tools)
12. [Filesystem & Navigation Commands](#12-filesystem--navigation-commands)
13. [Diagnostics, System & Telemetry](#13-diagnostics-system--telemetry)
14. [Productivity & Utility Commands](#14-productivity--utility-commands)

---

## 1. MiniGuardian Local Security & Forensics

These commands run locally inside `trust-terminal` and directly interface with the sealed `miniguard` daemon over `/run/jayce-operator/miniguard.sock`.

| Command | Usage | Description |
| :--- | :--- | :--- |
| **`forensic`** | `forensic [YYYY-MM-DD]` | Generates an **Executive Forensic Summary Report** by date. |
| **`forensics`** | `forensics` | Alias for `forensic`. |
| **`report`** | `report` | Alias for `forensic`. |
| **`forensic --json`** | `forensic --json` or `forensic raw` | Dumps the full, unformatted raw JSON daemon evidence payload. |
| **`logs`** | `logs` or `log` | Displays the sealed in-memory log ring from the MiniGuardian daemon. |
| **`diagnose`** | `diagnose [path]` | Runs all static security analyzers (**Bloodhound + Plumber + Linter**) on a target codebase. |
| **`sniff`** | `sniff [path]` | Alias for `diagnose`. |
| **`bloodhound`** | `bloodhound [path]` | Runs cross-file dependency graph security and taint analysis. |
| **`plumber`** | `plumber [path]` | Scans directory for auto-fixable code and configuration issues. |
| **`linter`** | `linter <file\|code> [lang]` | Multi-language linter for Rust, C, Go, Python, Bash, and JS. |
| **`codesniffer`** | `codesniffer [file]` | Alias for `linter`. |

---

## 2. Microkernel & Engine Commands

Direct control and telemetry interfaces into the Jayce Microkernel Substrate.

| Command | Usage | Description |
| :--- | :--- | :--- |
| **`kernel status`** | `kernel status` | Displays live microkernel ticks, thread health, and active engine states. |
| **`kernel truth`** | `kernel truth` | Inspects the HMAC-signed truth chain and cryptographic entry verifications. |
| **`kernel vitals`** | `kernel vitals` | CPU, memory, load average, and process jitter (p50/p95/p99) metrics. |
| **`kernel burnpit`** | `kernel burnpit` | Inspects disposed Void Pit artifacts and permanent isolations. |
| **`kernel cluster`** | `kernel cluster` | Compute cluster nodes, home network leverage, and offload task status. |
| **`kernel doorman`** | `kernel doorman [args]` | Doorman capability token verification and authorization checks. **[not available: no `kernel doorman` subcommand in this build]** |
| **`kernel illusion`** | `kernel illusion` | Inspects the Deception Stack status (`Hostile` vs `Calm`). |
| **`kernel shadow`** | `kernel shadow` | Inspects filesystem baseline constraints and shadow rollback posture. |
| **`kernel isd`** | `kernel isd` | Impossible State Detector (ISD) metrics and freeze counts. |
| **`kernel net`** | `kernel net` | eBPF network firewall status, blocked IPs, and active drops. |
| **`kernel net-block`**| `kernel net-block <IP>` | Manually block an IP address at the eBPF layer. |
| **`kernel net-unblock`**| `kernel net-unblock <IP>` | Unblock an IP address at the eBPF layer. |
| **`kernel net-inspect`**| `kernel net-inspect` | Inspect active socket connections and protocol anomalies. |
| **`kernel piggyback`**| `kernel piggyback` | Inspects background process leverage and thread scheduling. |
| **`kernel watchdog`** | `kernel watchdog` | Hardware watchdog timer and heartbeat health. |
| **`kernel spine`** | `kernel spine` | Microkernel spine timing dividers and fingerprint check (`0x2b1704c5`). |

---

## 3. Autonomous Robotics & Motion Control

Full IronPilot robotics engine commands for arms, rovers, bipeds, quads, and AMRs.

| Command | Usage | Description |
| :--- | :--- | :--- |
| **`robot list`** | `robot list` | Enumerate registered robotic units and active joint states. |
| **`robot status`** | `robot status <robot_id>` | Show battery, joint temperatures, end-effector state, and safety status. |
| **`robot move`** | `robot move <robot_id> <x> <y> <z>` | Dispatch a spatial motion vector with collision avoidance. |
| **`robot grasp`** | `robot grasp <robot_id> <force_n>` | Actuate end-effector gripper with closed-loop force feedback. |
| **`robot wait`** | `robot wait <robot_id> <ms>` | Insert micro-pause for sensor/camera synchronization. |
| **`robot mission`** | `robot mission <robot_id> <script>` | Execute a formal proof-checked, pedestrian-aware safety mission script. |
| **`robot estop`** | `robot estop <robot_id>` | Emergency stop — instantly latches hardware watchdog and freezes joints. |
| **`robot program`** | `robot program <robot_id> <prog_id>`| Attach and execute a registered Observatory OS robotics program. |

---

## 4. Aerial Drone Fleet Management

Command and monitor autonomous aerial drones and UAVs.

| Command | Usage | Description |
| :--- | :--- | :--- |
| **`drone list`** | `drone list` | Enumerate active aerial drones, flight modes, and battery telemetry. |
| **`drone status`** | `drone status <drone_id>` | Display altitude, pitch, roll, yaw, airspeed, and GPS/optical flow state. |
| **`drone arm`** | `drone arm <drone_id>` | Arm drone flight motors following pre-flight safety check. |
| **`drone disarm`** | `drone disarm <drone_id>` | Disarm flight motors (ground safety lock). |
| **`drone takeoff`**| `drone takeoff <drone_id> [alt_m]` | Command autonomous takeoff to specified target altitude. |
| **`drone land`** | `drone land <drone_id>` | Command precision autonomous landing sequence. |
| **`drone waypoint`**| `drone waypoint <id> <lat> <lon> <alt>`| Queue a navigation waypoint with geofence boundary validation. |
| **`drone rth`** | `drone rth <drone_id>` | Trigger Return-To-Home (RTH) failsafe mode. |

---

## 5. Autonomous Vehicles & Ground Fleets

Command and monitor autonomous ground vehicles (AGVs), rovers, and mobile platforms.

| Command | Usage | Description |
| :--- | :--- | :--- |
| **`vehicle list`** | `vehicle list` | Enumerate autonomous vehicles and active navigation tasks. |
| **`vehicle status`**| `vehicle status <vehicle_id>` | Show odometer, battery/fuel, speed, steering angle, and LIDAR state. |
| **`vehicle route`** | `vehicle route <id> <dest>` | Compute and execute deterministic optimal route across sector map. |
| **`vehicle speed`** | `vehicle speed <id> <max_mps>` | Set maximum speed limit and acceleration constraints. |
| **`vehicle mode`** | `vehicle mode <id> <mode>` | Toggle operating mode (`autonomous`, `manual`, `follow`, `convoy`). |
| **`vehicle teleop`**| `vehicle teleop <id>` | Launch low-latency teleoperation control pipeline. |

---

## 6. Swarm & Fleet Coordination

Coordinate multi-agent swarms, formations, and distributed edge compute.

| Command | Usage | Description |
| :--- | :--- | :--- |
| **`swarm list`** | `swarm list` | List active autonomous swarms and formation topologies. |
| **`swarm status`** | `swarm status <swarm_id>` | Display swarm unit count, centroid position, and dispersion index. |
| **`swarm formation`**| `swarm formation <id> <shape>`| Set swarm geometry (`grid`, `line`, `v-shape`, `orbit`, `ring`). |
| **`swarm sync`** | `swarm sync <swarm_id>` | Synchronize clock drift and spatial coordinates across swarm units. |
| **`swarm offload`**| `swarm offload <task_id>` | Distribute perception/compute workloads across local edge fleet nodes. |
| **`cluster drive`** | `cluster drive <policy>` | Set cluster drive policy (`RoboticsFleet`, `EdgeCompute`, `SwarmMesh`). **[not available in this build]** |

---

## 7. Hardware Bridges & Signal Processing

Interface with hardware fieldbuses (CAN, Serial, ROS2, EtherCAT, Modbus) and signal processing pipelines.

| Command | Usage | Description |
| :--- | :--- | :--- |
| **`bridge list`** | `bridge list` | Enumerate hardware bridge channels and bus baud rates. |
| **`bridge status`**| `bridge status <channel>` | Query packet throughput, frame drops, and bus error counters. |
| **`signal`** | `signal classify <stream>` | Classify raw sensor streams into normalized semantic events. |
| **`translator`** | `translator <audio\|sensor>` | Interpret audio, LIDAR, and sensor signals into deterministic semantics. **[not available in this build]** |
| **`translate`** | `translate <script\|stream>` | Decode scripts, runes, symbol systems, or binary protocols. **[not available in this build]** |

---

## 8. Cognitive & ADA Commands

Interface with ADA (Autonomous Deterministic Agent) and cognitive memory structures.

| Command | Usage | Description |
| :--- | :--- | :--- |
| **`ada`** | `ada <status\|query>` | Query and control ADA's cognitive state directly from TRust. **[not available in this build]** |
| **`nlmg`** | `nlmg <inspect\|query>` | Manipulate the Natural Language Memory Graph (NLMG). **[not available in this build]** |
| **`pattern`** | `pattern <list\|clear>` | Cognitive pattern memory inspection and management. **[not available in this build]** |
| **`drive`** | `drive` | Drive vector control and internal goal alignment. **[not available in this build]** |
| **`curiosity`** | `curiosity` | Control the curiosity exploration engine. **[not available in this build]** |
| **`hypersearch`** | `hypersearch <query>` | Execute semantic hypersearch across indexed codebases and docs. **[not available in this build]** |

---

## 9. Security, Deception & Isolation

| Command | Usage | Description |
| :--- | :--- | :--- |
| **`illusion`** | `illusion` | Control the Deception Stack and inspect intercepted MITRE probes. |
| **`chauffeur`** | `chauffeur` | Chauffeur inter-process message router control and isolation. **[not available in this build]** |
| **`shim`** | `shim` | Inspect ABI boundary isolation shims and capability denials. **[not available in this build]** |
| **`trust`** | `trust` | Boot Trust Engine posture and mode tuning. |
| **`sec`** | `sec` | Inspect security posture, run self-tests, and tune trust modes. **[not available in this build]** |
| **`boot`** | `boot` | Scaffold and validate boot-safe release projects and Secure Boot flow. **[not available in this build]** |

---

## 10. Orchestration & Parallel Universes

| Command | Usage | Description |
| :--- | :--- | :--- |
| **`map`** | `map` | Launches the graphical **Cosmic Observatory** GUI (SDK). `miniguard --map` is also stubbed in this beta. **[not available in this build]** |
| **`pulse`** | `pulse` | Launches the Jayce Pulse Deck telemetry interface. **[not available in this build]** |
| **`cbpu`** | `cbpu` | Runs Constitutionally-Bound Parallel Universe exploration. **[not available in this build]** |
| **`universe`** | `universe` | Manages universe branches, drift detection, and repair advice. **[not available in this build]** |
| **`observatory`**| `observatory` | Manages Observatory OS panels, themes, and extensions. **[not available in this build]** |
| **`autorun`** | `autorun` | Self-healing, auto-compiling, and auto-patching build loop. **[not available in this build]** |

---

## 11. Development, Build & Multi-Language Tools

| Command | Usage | Description |
| :--- | :--- | :--- |
| **`lang`** | `lang <run\|build\|test> [file]`| Language-aware runner for 14 languages (Rust, C, C++, Go, Python, Bash, JS, TS, etc.). |
| **`jvc`** | `jvc <pack\|unpack> <file>` | Deterministic JVC binary compressor and decompressor tool. |
| **`code`** | `code <prompt>` | Code generation pipeline. |
| **`patch`** | `patch` | Automated code patch pipeline control. |
| **`scaffold`** | `scaffold <template>` | Scaffolds new projects with microkernel-safe templates. **[not available in this build]** |
| **`build`** | `build` | Compiles workspace or Observatory SDK target artifacts. **[not available in this build]** |
| **`check`** | `check` | Runs `cargo check` in the current workspace. **[not available in this build]** |
| **`test`** | `test` | Runs `cargo test` in the current workspace. |

---

## 12. Filesystem & Navigation Commands

| Command | Usage | Description |
| :--- | :--- | :--- |
| **`ls`** | `ls [path]` | List directory contents with file metadata. |
| **`cd`** | `cd <path>` | Change working directory. |
| **`pwd`** | `pwd` | Print working directory. |
| **`cat`** | `cat <file>` | Display file contents. |
| **`mkdir`** | `mkdir <dir>` | Create a directory. |
| **`rm`** | `rm <path>` | Delete file or directory (`--force` required for dirs). |
| **`cp`** | `cp <src> <dst>` | Copy file. |
| **`mv`** | `mv <src> <dst>` | Move or rename file. |
| **`stat`** | `stat <path>` | Display path file metadata and permissions. |
| **`touch`** | `touch <file>` | Create an empty file or update timestamp. |
| **`find`** | `find <pattern>` | Find files matching a pattern. |
| **`grep`** | `grep <pattern> [file]`| Search file contents for a pattern. |
| **`write`** | `write <file> <text>` | Overwrite text to a file. |
| **`append`** | `append <file> <text>` | Append text to a file. **[not available in this build]** |
| **`tree`** | `tree [path]` | Show directory tree structure. |
| **`head`** | `head <file>` | Show first lines of a file. |
| **`tail`** | `tail <file>` | Show last lines of a file. |
| **`diff`** | `diff <file1> <file2>` | Show line differences between two files. |

---

## 13. Diagnostics, System & Telemetry

| Command | Usage | Description |
| :--- | :--- | :--- |
| **`sys-info`** | `sys-info` | Display TRust version, substrate release, and host OS metadata. **[not available in this build]** |
| **`sys-mem`** | `sys-mem` | Detailed memory utilization breakdown. **[not available in this build]** |
| **`sys-cpu`** | `sys-cpu` | CPU model, topology, and core load distribution. **[not available in this build]** |
| **`peace`** | `peace` | Run Peace Audit engine self-test. **[not available in this build]** |
| **`watchdog`** | `watchdog` | Query watchdog safety timers. **[not available in this build]** |
| **`telemetry`** | `telemetry` | Display telemetry subsystem metrics. **[not available in this build]** |

---

## 14. Productivity & Utility Commands

| Command | Usage | Description |
| :--- | :--- | :--- |
| **`help`** | `help [command]` | List commands or show detailed help for a specific command. |
| **`ask`** | `ask <question>` | Ask the terminal an advisory question. **[not available in this build]** |
| **`history`** | `history` | Display interactive shell command history. **[not available in this build]** |
| **`alias`** | `alias [name=cmd]` | Define or list shell command aliases. **[not available in this build]** |
| **`timer`** | `timer <secs>` | Set a session or kernel timer. **[not available in this build]** |
| **`note`** | `note <text>` | Add a developer session note. **[not available in this build]** |
| **`todo`** | `todo` | View and manage your workspace todo list. **[not available in this build]** |
| **`focus`** | `focus` | Track focus and active developer workflow time. **[not available in this build]** |

---

*(Manual generated for MiniGuardian / IronPilot Robotics & TRust Smart Terminal)*
