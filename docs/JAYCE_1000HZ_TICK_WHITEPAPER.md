# Why 1,000 Hz "Just Works": The Deterministic Tick Architecture of the Jayce Kernel

**Title:** The Jayce Kernel at 1 kHz — Why a Fixed 1,000 Hz Tick Is Not a Problem (and Why It Is on General-Purpose Kernels)

**Subject:** Jayce Kernel v0.4 (canonical tree), Mini Guardian daemon, Apodeixis verification
**Scope:** A code-verified explanation of why the Jayce Kernel sustains a deterministic 1,000 Hz tick cadence with bounded, budgeted engine execution — and why that same cadence is impractical as a *security substrate* on a general-purpose kernel.
**Method:** Every architectural claim below is verified against the canonical kernel tree shipped in the Mini Guardian workspace on the desktop (`jayce_kernel_src/`) and the host-side kernel port (`jayce_kernel/`). File:line citations are given throughout.

---

## Abstract

General-purpose kernels (Linux, Windows, BSD) run scheduler ticks in the 100–250 Hz range, and even when their timers can fire faster, the *guarantees* are absent: scheduling is preemptive, latency is unbounded, page faults and dynamic allocation inject jitter, and there is no compile-time mechanism to say "this engine must run exactly every N ticks, within a hard budget, and this is provable." A 1,000 Hz tick on such a substrate is therefore not a throughput problem — it is a *certainty* problem.

The Jayce Kernel inverts that relationship by construction. It makes the 1,000 Hz tick the fixed heartbeat of the machine and then removes every source of nondeterminism around it. This paper describes, with source references, the five structural reasons the cadence holds:

1. The clock itself is a **real PIT channel running at exactly 1,000 Hz** — not a software timer.
2. The tick rate is a **constitutional constant** of the Immutable Spine, enforced at compile time.
3. Engine firing is **mathematically determined by a sequence number and power-of-two dividers**, so engine scheduling is arithmetic, not policy.
4. The tick path is **single-threaded with zero-cost state access** (`KernelCell`) and no heap allocation, so the per-tick cost is flat and predictable.
5. Every engine is **budgeted** (time, memory, messages) and the whole configuration is **fingerprinted and proof-checked**, so the cadence cannot drift silently.

The paper also records the precise corrections a source audit makes to commonly-repeated claims about the system (engine count, the role of `KERNEL_LOCK`, and the scope of "no wall-clock" determinism).

---

## Table of Contents

1. [The Question, Framed Honestly](#1-the-question-framed-honestly)
2. [Layer 1 — The Clock: A Real PIT at Exactly 1,000 Hz](#2-layer-1--the-clock-a-real-pit-at-exactly-1000-hz)
3. [Layer 2 — The Immutable Spine: Tick Rate as Constitution](#3-layer-2--the-immutable-spine-tick-rate-as-constitution)
4. [Layer 3 — Arithmetic Scheduling: `seq`, Dividers, and the AND Fast Path](#4-layer-3--arithmetic-scheduling-seq-dividers-and-the-and-fast-path)
5. [Layer 4 — The Single-Threaded Tick Substrate and `KERNEL_LOCK`](#5-layer-4--the-single-threaded-tick-substrate-and-kernel_lock)
6. [Layer 5 — Budgets, Mode Scaling, and the Safety-Dirty Short-Circuit](#6-layer-5--budgets-mode-scaling-and-the-safety-dirty-short-circuit)
7. [The Host Port: Mini Guardian Advancing the Tick at 1 kHz](#7-the-host-port-mini-guardian-advancing-the-tick-at-1-khz)
8. [Verification: Apodeixis, Lineage Hashes, Fingerprints](#8-verification-apodeixis-lineage-hashes-fingerprints)
9. [Measured Evidence and Operational Results](#9-measured-evidence-and-operational-results)
10. [Why a Normal Kernel Cannot Offer the Same](#10-why-a-normal-kernel-cannot-offer-the-same)
11. [Source-Audit Corrections to Common Claims](#11-source-audit-corrections-to-common-claims)
12. [Conclusion](#12-conclusion)
13. [Appendix A — Source Map](#appendix-a--source-map)
14. [Appendix B — Engine Firing Examples](#appendix-b--engine-firing-examples)

---

## 1. The Question, Framed Honestly

The question "is 1,000 Hz too much for a kernel?" is usually answered with raw throughput, and that is the wrong metric. Modern CPUs can trivially service 1,000 interrupts per second; even Linux supports `CONFIG_HZ=1000`. So the honest question is not *Can hardware do it?* but *Can the behavior be made deterministic enough to be useful?*

A compression engine, a fraud detector, a liveness monitor, or a safety chain acquires its power not from firing often but from firing **on a schedule that never surprises the rest of the system**. On a general-purpose kernel:

- the scheduler decides *when* your loop runs, and it is preemptive;
- allocation, page faults, and cache misses happen *when they happen*;
- high-resolution timers fire with microsecond-to-millisecond jitter, unbounded in the worst case;
- nothing at compile time guarantees your loop even *exists* next boot.

That is what "1,000 Hz is too much for a normal kernel" really means: at 1 kHz, jitter of even a few milliseconds means a *majority* of your scheduled firings are wrong, so the rate provides no benefit — only load.

---

## 2. Layer 1 — The Clock: A Real PIT at Exactly 1,000 Hz

The tick is not synthesized in software; it is a hardware interrupt from the x86 Programmable Interval Timer, configured once at boot. From `jayce_kernel_src/src/kernel/arch/x86_64/timer.rs`:

```rust
const PIT_INPUT_HZ: u32 = 1_193_182;   // PIT oscillator base frequency
const PIT_FREQUENCY_HZ: u32 = 1000;    // 1 kHz

pub fn init_timer_1khz() {
    let divisor = PIT_INPUT_HZ / PIT_FREQUENCY_HZ;   // 1193
    // Channel 0, lobyte/hibyte, mode 2 (rate generator)
    cmd.write(0b0011_0100);
    ch0.write((divisor & 0xFF) as u8);
    ch0.write((divisor >> 8) as u8);
}
```

Key architectural facts:

- **Mode 2 (rate generator)** reloads the count automatically, so the interrupt cadence is a property of the hardware, not of any software re-arming path. There is no "missed re-arm" failure mode.
- **The divisor is integer-fixed**: 1,193,182 / 1,193 = 1,000 Hz. There is no drift correction, because drift correction is what introduces jitter; the rate is exact by construction.
- The interrupt handler is the entire heartbeat of the machine (`handle_timer_interrupt`):

```rust
pub fn handle_timer_interrupt() {
    TICKS.fetch_add(1, Ordering::Relaxed);
    crate::time::on_timer_tick();                    // advance the canonical tick counter
    crate::engines::heartbeat::mark_from_timer();    // heartbeat engine
    crate::engines::telemetry::tick(ticks() as u32); // telemetry engine
}
```

Everything else in the kernel descends from this one counter. The tick *sequence* — a monotonic `u64` — is the kernel's only notion of time; the watchdog, the truth chain, the predictive horizon, and the UI cadence are all measured in ticks, never in wall-clock seconds inside the kernel core.

---

## 3. Layer 2 — The Immutable Spine: Tick Rate as Constitution

The rate is not a config option. It is a constitutional constant in the Immutable Spine, a module that contains *no mutable state whatsoever*. From `jayce_kernel_src/src/kernel/spine/invariants.rs`:

```rust
/// PIT timer frequency in Hz.
pub const TIMER_FREQ_HZ: u64 = 1_000;
```

and the module's own contract (verbatim from the source header):

> "These are the immutable architectural limits that define what Jayce IS. No value here may change at runtime. No mutable state lives in this module. Violations are caught at compile time — not at runtime, not with panics."

The Spine module (`src/kernel/spine/mod.rs`) states it plainly: "The Immutable Spine — constitutional law of the Jayce kernel. This module contains only constants and read-only verification helpers."

This means:

- **The 1,000 Hz rate cannot drift** — not by a capability grant, not by a config file, not by a shim call, not by a compromise of any runtime component. There is literally no write path to it.
- **Downstream cadences are derived constants, not independent decisions**: `RENDER_INTERVAL_TICKS = 1_000` (UI may not render more often than once per second), `PAGE_ROTATE_INTERVAL_TICKS = 5_000` (page rotation at 5 Hz), plus fixed `MAX_TASKS`, `MAX_REGIONS`, and per-engine `ENGINE_BUDGETS`. Because these are all in one immutable module, the entire system's rhythm is auditable in one place.
- **Enforcement is compile-time**: `spine::init()` calls `invariants::verify_boot_invariants()` at boot, and the invariants module additionally contains compile-time `const _: () = assert!(...)` checks (e.g., the host spine asserts the budget array length and divider properties at compile time).

A 1,000 Hz loop is sustainable precisely because nothing at runtime is allowed to change what "1,000 Hz" means.

---

## 4. Layer 3 — Arithmetic Scheduling: `seq`, Dividers, and the AND Fast Path

This is the heart of the answer to "why is 1,000 Hz cheap." Engine firing is computed from a *sequence number* and a *divider*, not from a scheduler's whim.

The canonical kernel (from `jayce_kernel_src/src/kernel/engines/mod.rs`):

```rust
fn should_tick(mode: VitalMode, seq: u64, full_div, constrained_div, minimal_div) -> bool {
    let mut div = match mode { Full => full_div, Constrained => constrained_div, Minimal => minimal_div };
    if strict_security_mode() { div = div.saturating_mul(2).max(2); }
    if div <= 1 { return true; }
    if div.is_power_of_two() { (seq & (div - 1)) == 0 } else { (seq % div) == 0 }
}
```

And the host-side port (`jayce_kernel/src/spine.rs`) keeps the identical contract:

```rust
pub fn should_tick(seq: u64, div: u64) -> bool {
    if div <= 1 { return true; }
    if div.is_power_of_two() { (seq & (div - 1)) == 0 } else { (seq % div) == 0 }
}
```

Why this is the right mechanism for a high tick rate:

- **Power-of-two dividers collapse to a single AND instruction.** Every engine's cadence (every 2nd, 4th, 8th, 16th, 64th tick) is one masked comparison. Scheduling twenty engines per tick costs a handful of arithmetic operations — nanoseconds, not microseconds.
- **It is a pure function of `seq`.** Given the sequence number and the mode, the firing pattern of every engine for the entire future is fully determined up front. There is no queue, no priority inversion, no preemption to analyze.
- **Non-power-of-two dividers still work** via modulo, so cadences like "every 3 ticks" are expressible — the fast path simply does not apply. (The compile-time spine guards in the host port assert that the tick-mode constants are exactly `0,1,2`, and the budget table is statically sized to the engine count.)
- **Strict-security mode is a total multiplier**: every divider doubles under `strict_security_mode()`, halving effective engine load while keeping the cadence lawful — an example of the arithmetic path being *policy by construction*.

The full round-robin is `tick_all()` (`engines/mod.rs`), which advances `ENGINE_SEQ`, runs each engine through `tick_tracked!` or `tick_scaled_tracked!`, and records liveness via `engine_status::note_engine_tick(id)` so a watchdog can later prove that every engine actually *did* fire on its schedule.
---

## 5. Layer 4 — The Single-Threaded Tick Substrate and `KERNEL_LOCK`

A 1,000 Hz loop is only sustainable if the state it touches does not *fight* it. The kernel's answer is radical single-threading:

**Inside the kernel core, state access is zero-cost.** All kernel state lives in `KernelCell<T>` (`jayce_kernel_src/src/kernel/kernel_cell.rs` and its host mirror), an `UnsafeCell` wrapper with no locks, no atomics, and an unsafe access API. The documented contract:

> "KernelCell instances are only accessed from the kernel's single tick thread. Violating this is undefined behavior."

Because there is exactly one thread (the tick thread) mutating state, the per-tick cost of "state access" is a raw pointer dereference — no mutex acquire, no contention, no cache-line ping-pong. That is a *structural* answer to the concurrency problem, not an optimization: there is nothing to serialize at 1 kHz because there is no concurrency inside the tick.

**At the daemon boundary, the lock is `KERNEL_LOCK`.** The Mini Guardian daemon is multi-threaded (tick thread, socket server tasks, gossip tasks), so it cannot inherit the kernel's single-thread assumption. `jayce_kernel/src/lib.rs`:

```rust
/// Global serialization for all kernel-state access from the daemon side.
/// ... The lock restores the single-threaded contract at the port boundary.
/// Every `jk::` call from a daemon thread MUST be wrapped in with_kernel_lock(...).
pub static KERNEL_LOCK: std::sync::Mutex<()> = std::sync::Mutex::new(());

pub fn with_kernel_lock<R>(f: impl FnOnce() -> R) -> R {
    let _guard = KERNEL_LOCK.lock().unwrap_or_else(|poisoned| poisoned.into_inner());
    f()
}
```

Punctuation matters here:

- The kernel core is **lock-free by design** — `KernelCell` has no cost.
- The **lock exists only where the single-threaded contract is violated** — at the daemon port boundary. It buys back exactly the semantics the kernel core always had.
- The lock is exercised continuously (tick advancement, kernel bridge reads, forensic reports, gossip handoffs all run through `with_kernel_lock`), which is precisely why the race conditions that used to appear under chaos testing are gone: chaos stress now exercises the *same* serialization path every production thread uses.

This is the correct separation of concerns, and it is why "1,000 Hz plus a multi-threaded daemon" does not regress into data races or out-of-bounds panics: the single-threaded core guarantees linearizable state, and the boundary lock makes cross-thread access indistinguishable from the core's own access pattern.

---

## 6. Layer 5 — Budgets, Mode Scaling, and the Safety-Dirty Short-Circuit

A high tick rate is only useful if *no single tick* can run away. Every engine carries hard budgets:

- **In the canonical kernel**, `ENGINE_BUDGETS` is a const array with per-engine `time` / `mem` / `msgs` ceilings (e.g., heartbeat: 4 time units / 8 messages per tick; scheduler: 12 time units / 32 messages), enumerated in `spine/invariants.rs`. "Any attempt to expand a budget … is a compile error, not a policy decision" (per the architecture document).
- **In the host port**, `jayce_kernel/src/spine.rs` defines `EngineBudget { max_us, max_msgs, full_div, constrained_div, minimal_div }` for the 22 host-engine set, with dividers chosen by role:
  - Always-on, latency-sensitive: **NETWORK** (div 1), **VITALS** (1/1/2), **WATCHDOG** (1/1/1), **PIGGYBACK** (1/1/1).
  - Periodically refreshed: **TRUTH** (1/4/16), **HARDWARE_INFO** (1/16/64), **RECOVERY** (1/4/16), **CLUSTER** (1/4/8).
  - Budget ceilings: e.g., RECOVERY may spend up to 1000 µs/tick; most engines are capped at 100–500 µs.
Two further mechanisms keep average tick cost low:

- **Three tick modes.** `Full` → `Constrained` → `Minimal` scale the dividers of every engine coherently (the same `div` values in the table), so under load the kernel reduces *who runs* — never *how any engine runs*. Watchdog, truth, constitutional, self-healing and impossible-state engines remain div-1/div-2 in all modes because they are the safety floor.
- **The safety-dirty short-circuit.** A global `SAFETY_DIRTY` flag is set by any state-modifying engine and cleared after the safety chain runs. When it is clear, the expensive snapshot-compare stage of the safety analysis chain is *skipped entirely* (`engines/mod.rs`): "When clear, the safety chain can skip expensive snapshot capture + comparison." Idle ticks therefore approach the pure cost of `should_tick` arithmetic + stream pokes, which is what makes 1,000 empty ticks per second essentially free.

Together these mean the tick loop's worst case is bounded by the sum of engine budgets (all static numbers), and its *common* case is a handful of always-on engines plus arithmetic — the definition of a substrate that can run at 1 kHz indefinitely.

---

## 7. The Host Port: Mini Guardian Advancing the Tick at 1 kHz

On bare metal, the PIT drives the tick. In the Mini Guardian "co-host" deployment the daemon advances the *same* logical tick counter at the *same* cadence, so kernel-time units remain identical on host and metal. From `miniguard/src/main.rs`:

```rust
// Kernel time: advance the canonical tick counter at the kernel's 1 kHz
// cadence (TIMER_FREQ_HZ = 1_000). live_ticks() must behave like the real
// kernel's clock — a 1 ms tick count — or watchdog/PEH thresholds tuned in
// kernel-tick units would be meaningless on the host.
std::thread::spawn(|| loop {
    jk::with_kernel_lock(|| jk::time::on_timer_tick());
    std::thread::sleep(std::time::Duration::from_millis(1));
});
```

Design points that make this loop meaningful even though it runs atop a host OS:

- **It is a logical tick, not a wall-clock timer.** The daemon's `live_ticks()` is kernel time; host clock is used only where the real kernel would use hardware — and never in the scheduling path.
- **The 1 ms sleep is deliberately an *upper* bound on cadence, not a precision claim.** Because all engine decisions are integer functions of `seq`, a tick that lands slightly early or late does not corrupt state; the daemon's own co-host scheduling (`tick_scheduler.rs`) assigns 1 ms slots to its foreground monitors (PromptGuard, memfd, loopback, audit chain, etc.), so host behavior matches kernel behavior: budgeted stages, fixed cadence.
- **The tick is the system heartbeat end-to-end.** The co-host paper says it directly: unlike agents that "run asynchronously and depend on host operating system clocks," Mini Guardian relies on the kernel's "1,000 Hz logical tick as the system's heartbeat." Telemetry (`JAYCE_TLM`) is emitted on a fixed tick cadence carrying a SHA-256 engine-lineage fingerprint, so the daemon can verify each tick that it is running the *same* engine code as the canonical kernel build.

This is why the 1,000 Hz answer is portable: the archetype is the hardware PIT tick, and the daemon reproduces its *semantics* (a monotonic, integer, lock-serialized tick count) on the host.

---

## 8. Verification: Apodeixis, Lineage Hashes, Fingerprints

A cadence is only trustworthy if something proves it is still the cadence you think it is. Three verification mechanisms converge on the 1,000 Hz claim:

1. **Compile-time constitutional checks.** The Spine exposes no runtime write path; `assert!` constants verify table shapes (budget array length equals engine count; tick-mode values are 0/1/2). A rate change requires a source change, and a source change *is the evidence* — there is no silent drift.

2. **Engine-lineage fingerprints.** The daemon crate embeds `ENGINE_LINEAGE` — a SHA-256 over the canonical engine sources computed by `build.rs` — and the kernel's telemetry carries the identical value. A judge/enforcer agreement check compares the two on a fixed cadence: a mismatch means enforcer and judge disagree about what code they run, and the daemon fails closed. The `jayce_kernel` crate even recomputes the digest in a test (`engine_lineage_recomputes_to_embedded_value`) to catch drift between `build.rs` and the sources it hashes.

3. **Spine fingerprints.** The host spine computes an FNV-1a32 `SPINE_FINGERPRINT` from the engine-count and budget-table constants at compile time, then exposes `verify_fingerprint(expected)` — runtime tampering detection against a stored value. This mirrors the ex-post-facto question "is the cadence config that built this binary the config I approved?" with a numeric answer.

4. **Apodeixis proof checking.** The formal-verification mission substrate checks bounded-effect missions with invariant/step-budget proofs. Its scheduling is arranged so proof work is measured against the tick budget and "proof checking never stalls the kernel's 1,000 Hz tick cycle" (architecture document). Requests are checked *within* tick budgets (e.g., the RCF mission bytecode is allocated a fixed number of ticks and terminated on exhaustion), so verification binds to the same arithmetic cadence as everything else rather than running as an uncontrolled background process.

The combined effect: rate, tables, budgets, and proof load are all either compile-time-fixed or compared against fingerprints at runtime on schedule. There is no anonymous source of timing drift.
The Jayce Kernel attacks the actual requirement: **a fixed, budgeted, provable firing cadence**. The 1,000 Hz rate is then not a risk but the property that makes the entire security model legible — a watchdog that fires every tick, a truth chain that updates every tick, a telemetry line that arrives every 500 ticks, all expressed in a single shared timebase.