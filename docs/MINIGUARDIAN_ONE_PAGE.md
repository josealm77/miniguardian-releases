# MiniGuardian — One-Page Summary

*A 30-second read. Full detail in `MINIGUARDIAN_SIMPLIFIED_GUIDE.md`.*

## What it is
A **security guard dog** that lives on one Linux machine. It watches everything,
locks down anything it can't prove safe, and keeps AI agents inside their box.

## The one idea
> **Don't trust the AI (or the attacker). Trust the locked-down container you
> put around it.**

## What it does (5 jobs)
1. **Watches** — files, processes, memory, network.
2. **Judges** — rates the system Calm → Suspicious → Hostile → Paranoid.
3. **Contains** — locks AI agents to what they're allowed to touch.
4. **Deceives** — escape attempts get fed a convincing fake world, all logged.
5. **Protects itself** — so nobody can quietly disarm it.

## Built honest (the kernel)
- A tiny **deterministic kernel ("Jayce")** runs checks 1000×/second — no
  randomness, fully replayable.
- A **judge** and an **enforcer** verify each other every tick; unverified =
  lock everything down.
- Everything lands in a **tamper-proof, hash-chained audit log**.

## AI containment (4 layers)
1. **Allowed?** — deny by default; only permitted actions proceed.
2. **In reality?** — escape attempts get a convincing fake "simulation."
3. **Deterministic** — token/time budgets stop runaway AI.
4. **Suspicious?** — catches prompt-injection and memory-poisoning.

## The AI Constitution (in the brain)
- Every agent has a **privilege level** (Normal → Solitary). Violations cut
  tokens and strip tools, telling the AI exactly what it did and lost.
- **Sabotage/framing** → straight to Solitary; the innocent victim is cleared.
- **Harm to human** is personality-aware: a *desktop* AI is killed instantly;
  a *robot/medical/vehicle* AI is contained (never killed mid-action).
- **Good behavior restores** the agent level by level.
- **Content guardians** (PromptGuard, memory-poison defense) keep the AI's
  *thinking* honest, not just its actions.

## Tools & governance
- **29 tools**: 11 read-only (auto-run), 18 that write/exec/delete/mutate/network/ingest
  (need **human approval**).
- Every call passes 5 gates: validation → proof check → capability → **human
  gate** → audit log.
- **mg-cli** wraps any AI tool in a workspace sandbox + supply-chain check.
- **TRust v2** is the governed command center; its **`mg-trust-tool` CLI** lets
  AI agents code safely through the same 29 governed tools (11 read-only +
  18 approval-gated). It also runs rich AI coding via **19 inline language
  adapters** (Python→Rust→Zig + the Apodeixis proof checker) and a **governed
  shell pipeline** for everyday CLI tools (`grep`, `cat`, ...) — each child
  time- and output-bounded, never `sh -c`.

## How it detects & protects
- Monitors processes, rootkits, memory, network, USB, browser, containers;
  plants **honeytokens** for high-confidence breach detection (a unique
  marker turning up somewhere is strong evidence, not a guarantee).
- Locks its own files, disguises itself, kills itself if peeked at, unfreezes
  via a separate sentinel, and auto-restores tampered system files.

## Bottom line
**MiniGuardian keeps untrustworthy things in a box they can't break out of —
and the box is built so nobody can quietly break into it either.**
