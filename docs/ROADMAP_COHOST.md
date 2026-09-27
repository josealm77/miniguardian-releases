# Roadmap: Cohost Mode — Enforcement Beneath the Operating System

**Status:** planned, not started. Paused until funded and staffed.
**Author:** Jose Almendarez, Jayce Automata Research. September 2026.

## The thesis

A root attacker inside an operating system can defeat anything that also runs
as root in that operating system: stop it, unseal it, or replace it.
MiniGuardian today runs as root on Linux, so against an attacker who already
holds root it can only **detect, slow down and keep evidence**. It cannot
**prevent**.

Enforcement survives a root attacker only when it runs **beneath** the
operating system, where OS root cannot reach it. The Jayce kernel is designed
for that position: a deterministic, bounded, proof-checked base layer, with
Linux as a guest behind the frozen Shim ABI and the Inner Shroud.

A related claim: the system as a whole can only be deterministic if its base
layer is. The kernel therefore allows no hardware entropy in its internal
state. It may read the environment (timers, sensors) as input, but the same
inputs must always produce the same outputs.

## Where things stand (September 2026)

| Piece | State |
|---|---|
| MiniGuardian daemon (Linux, sidecar) | Shipped as a beta. Host detection, AI containment through `mg-cli` (Landlock, default-deny network, human approval), sealed install, sentinel, audit chain. |
| Jayce kernel engines inside the daemon | Running. The daemon executes the canonical kernel engine sources at 1 kHz; the status TUI shows them live (`kernel_port`). |
| Jayce kernel, bare metal | Boots in QEMU and emits signed-lineage telemetry over serial. Multi-architecture core. |
| Judge / enforcer agreement | Implemented. The daemon compares its engines' lineage with the running kernel's and latches paranoid mode on silence or mismatch. Off by default (needs KVM under the shipped service). |
| Container "cohost" deploy (`deploy.sh --mode=cohost`) | Exists. Runs the daemon in a Docker container watching the host. **This does not resist host root** and is not the cohost mode this roadmap describes. |
| Jayce kernel hosting Linux as a guest | **Not built.** No hardware virtualization (VT-x/SVM, EPT) yet. The personality loader hands control to the personality instead of staying underneath it. |

## Phases

Each phase has an exit test run on a dedicated, isolated test machine, never
on a development machine.

### Phase A — Baseline: measure the current system against root
- **Build:** a test PC with the beta package, Secure Boot, `lockdown=integrity`
  and a sealed install.
- **Test:** a scripted root red-team. Stop both services, clear `chattr +i`,
  remount `/usr/local/bin`, tamper with the seal manifest, freeze the
  sentinel, detach eBPF programs, edit shadow-backed files, strip lockdown.
- **Exit:** a table of every attack marked *blocked*, *detected* or
  *undetected*, with the evidence each left behind. It gives an honest
  "before" measurement.

### Phase B — Judge beside Linux
- **Build:** enable the kernel judge under KVM with `judge_mode = "strict"`.
- **Test:** kill, pause or replace the judge; feed forged telemetry; alter the
  engine sources.
- **Exit:** each attempt latches paranoid mode and leaves an audit record.
  This proves detection, not survival.

### Phase C — Judge on separate hardware (first proof of the thesis)
- **Build:** the Jayce kernel on its own small board (x86 mini-PC or ARM SBC),
  connected to the protected machine over serial. The judge's verdict drives
  something OS root cannot fake: a hardware network cut or power relay.
- **Test:** the Phase A red-team, now as root on the protected machine.
- **Exit:** root on Linux can cut the link but cannot silence or fake the
  judge without being contained. A silent judge already fails closed.
- **Why first:** it tests the model without building a hypervisor, and it
  matches the kernel's robotics and embedded direction.

### Phase D — Jayce kernel beneath Linux
- **Build:** a VT-x hypervisor layer in the kernel (VMX root mode, VMCS,
  EPT, interrupt and device virtualization), running Linux as a guest behind
  the Shim ABI and the Inner Shroud, with the daemon inside the guest reporting
  to the kernel beneath it.
- **Test:** the Phase A red-team from guest root.
- **Exit:** guest root cannot disable enforcement, alter kernel state, or read
  kernel memory; every attempt is contained and recorded.

### Phase E — Verification and publication
- An independent security review of Phases C and D.
- The v0.6 kernel whitepaper, written from the source, with results from
  Phases A to D.
- A reproducible test kit, so reviewers can repeat the red-team.

## What it needs

| Role | Why |
|---|---|
| Systems/hypervisor engineer (x86 VT-x, EPT) | Phase D is specialised low-level work. |
| Embedded engineer | Phase C board bring-up, serial link, hardware kill path. |
| Security researcher / red-teamer | Designs and runs the root attack suites independently of the builders. |
| Founder / architect (Jose Almendarez) | Kernel design, determinism model, integration, review. |

**Equipment:** two or three disposable test PCs with TPM 2.0 and Secure Boot,
an x86 mini-PC and an ARM SBC for the judge, serial adapters, a
network-cut/relay module, and an isolated test network.

## Principles that do not change
- Civilian use only. The work targets people, businesses and critical
  infrastructure (water, power, health care, industrial control), never
  weapons or military use. Defence funding is declined on principle; the
  license's no-weaponization clause is non-waivable.
- Enforcement decisions stay deterministic: rules, proof-checked missions,
  counters. They are never decided by a language model (see `AGENTS.md`,
  "Governance decisions").
- Every claim ships with the test that proves it, and failures are published
  alongside successes.
- Testing happens on isolated, disposable machines only.
