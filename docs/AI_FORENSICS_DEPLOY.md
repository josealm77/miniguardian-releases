# AI forensics — deploy note

Property of Jayce Automata Research LLC. Branch `ai-forensics` (worktree
`~/Desktop/MiniGuardian.worktrees/ai-forensics`), not merged, not sealed, not deployed.
Four security review rounds, all findings fixed or listed under "Known limits" below.

## What this branch adds

- `miniguard/src/ai_forensics.rs`:
  - itemised, append-only AI containment violations in `/var/lib/jayce/miniguard/ai-violations/`
    (text fields capped at 4 KiB; a cut field keeps its full length and sha256);
  - the engine's AI events pulled into `/var/lib/jayce/miniguard/ai-log/` by a worker thread every
    5 s (labelled `engine-reported`, the file owner's uid, `pulled_at`, and the `day` it is filed
    under — today or yesterday only, whatever `t` the engine wrote);
  - dumps that move the live log into JVC archives in `ai-log/archive/` (`ai-log-<id>.<n>.jvc`,
    one per 4 MiB; never delete), recorded in the audit chain with each part's file sha256;
  - closed days packed with JVC (Generic mode, 4 MiB whole-line pieces) after a byte-for-byte round
    trip; the plain file is removed only after the archive is synced to disk.
- Reads of the engine's folder (user-controlled) are hardened: real folders owned by the operator or
  root and not group/world-writable, regular files owned by the operator, no symlinks, size and
  line caps, per-tick byte/file budgets, a 32 MiB live-store cap.
- A user-planted packed trace is never decoded by the daemon: a confined helper
  (`miniguard --ai-trace-unpack <id>`, run as nobody with no new privileges, memory/CPU/file caps,
  a wall clock) does it; one lookup searches at most 64 parts within 30 s.
- Answers keep rows as raw lines under 8 MiB / 2000 rows (newest by arrival), so a planted flood
  can't exhaust the daemon's 256 MiB address space.
- One line in `brain.rs::ai_record_violation` (records each violation; counts unchanged).
- `ai_*` commands on the existing root-only management socket (nothing new listens).
- `trust_terminal`: `forensics origin ai`, `forensics ai.<kind>`, `forensics ai.violations`,
  `forensics ai --archive <file>`, `forensics ai trace <id>`, `forensics ai dump DUMP`, `--json`;
  control characters, C1, bidi overrides and in-field newlines are shown escaped, never sent.

## Built binaries (release, musl) — branch head e51bdca

| binary | sha256 |
|---|---|
| `target/x86_64-unknown-linux-musl/release/miniguard` | `44a45001edfe0de2ef673e8f441d3cb72cc0228a4c93f7810026b583f9546f09` |
| `target/x86_64-unknown-linux-musl/release/trust_terminal` | `2286c646ff9ca9c154894d0607ce1b9097c6fe847488c45dec369400e8ceb3f8` |

Engine (mg-llm-core, branch `frontal-lobe`, head 51f784f) writing the events:
`target-verified/mg-llm-core-51f784ff`
(sha256 `1473c64b85ab03cc45cdbdbfd2825952e0f8eba18827f2e463812809f2276e2a`).

## Config to add (`/etc/miniguard/miniguard.toml`)

```toml
[ai_forensics]
engine_dir = "/home/YOUR_USER/.local/share/mg-llm-core"   # absolute path; ~ is not expanded
operator_uid = 1000                                       # your uid: `id -u`
```

Without this block nothing is pulled (violations are still itemised); `forensics origin ai` says
"no engine folder configured".

## Deploy (owner)

1. Review and merge `ai-forensics` into `main` (or deploy from the worktree), then your usual
   build → seal → deploy (`seal.sh`), checking the sha256 of the release binaries against the table
   above before and after.
2. Add the config block above, then restart the daemon.
3. Point the desktop app (or the engine service) at engine `51f784ff` so events are written.

## Verify (live)

- `sudo trust_terminal` → `forensics origin ai today` (events appear within ~5 s of a reply that
  was hedged, withheld, declined, explored …).
- `forensics ai.violations today` — itemised violations, if any.
- `forensics ai dump` explains; `forensics ai dump DUMP` archives and prints each part's sha256
  (checkable with `sha256sum`) and the content sha256; the audit chain gets one "ai-log dump" entry.
- `forensics ai trace <id>` on a day the engine has already packed: proves the helper starts, drops
  to nobody and is allowed to exec under the real unit (only checkable live).
- The daemon log has `ai-forensics:` lines only when something changes (refusals, recovery).

## Tests in this branch

- `cargo test -p miniguard --lib ai_forensics` — 31; `cargo test -p miniguard --test ai_trace_helper`
  — 6 (runs the real binary as the helper: packed parts, a gap between parts, a part-count flood, a
  decompression bomb, garbage, a helper that can't start); `brain::tests::violations_reach_the_itemised_store`;
  `cargo test -p trust_terminal kernel_cmd` — 30.
- Full suites, owner's own (ungoverned) terminal at cbd704e: miniguard lib 1688 passed / 0 failed;
  trust_terminal 246 passed / 2 failed (the two `native_git` tests, which need `.git` to be a
  folder — it is a file in any git worktree). At e51bdca, inside a governed session: miniguard lib
  1684 passed / 6 failed and trust_terminal 245 / 3, all governance-only (TCP bind, nested
  Landlock, `/proc/self/comm`, seccomp child socket, D-Bus) plus the two `native_git`.
- Engine: 1250 passed / 0 failed.

## Known limits (accepted, from the reviews)

- A dump undone in the narrow window after its audit entry but before its commit marker leaves an
  audit entry naming parts that no longer exist (events are back in the live log; the recovery log
  line says the undo supersedes the entry).
- `forensics ai.*` with archives (the default) unpacks every dump part while holding the store lock:
  time grows with the number of dumps (memory stays bounded).
- Trace lookup parses the one matching line fully in the daemon (a hostile 1 MiB line costs a short
  ~16–32 MiB spike).
- Several large concurrent answers to clients that don't read could add up toward the 256 MiB cap
  (the socket needs the root token).
- Offsets track (inode, bytes): a deleted-and-recreated engine file that reuses the inode and is
  longer than the old offset resumes mid-file.
