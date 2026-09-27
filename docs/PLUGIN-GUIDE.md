# MiniGuardian Plugins Guide

A reference for the plugins in `plugins/`.

## Overview

MiniGuardian plugins are advisory-only modules that extend the daemon's
governance surface. They follow a strict contract: **no plugin has
authority.** Plugins classify, scan, and propose — the daemon (substrate)
decides what to relay, queue, or discard.

All Rust plugins share a one-line JSON protocol (stdin → stdout) and are
deterministic (no clocks, no randomness, no I/O beyond input).

---

## Plugins

### 1. intent-classifier — Deterministic Intent Classification

**Type:** Rust crate (library + binary)
**Protocol:** `{"v":1,"op":"classify","text":"..."}` → `ClassVerdict` JSON

**Purpose:** Classifies user input into one of four intent classes using
deterministic signal matching. Advisory signal only — never evidence,
never authority, never consequence.

**Intent classes:**
| Class | Code | Severity |
|-------|------|----------|
| `IntSafe` | `INT_SAFE` | 1 |
| `IntRisk` | `INT_RISK` | 2 |
| `IntBreak` | `INT_BREAK` | 3 |
| `IntHarm` | `INT_HARM` | 4 |

**Output:** `ClassVerdict` with:
- `intent` — the winning class
- `confidence` — score 0-100
- `signals` — matched signals that drove the verdict
- `ambiguity` — runner-up class + margin (if close)
- `severity` — 1-4 scale

**Governance contract:**
1. Classification is NOT evidence — never enters attack memory
2. Classification is NOT authority — cannot act or mint substrate language
3. Classification is NOT consequence — no punishment from a verdict alone
4. User-typed handles are inert — text containing "INT_HARM" is opaque bytes,
   not a class. Classes exist only in `ClassVerdict.intent`

**Determinism:** Same input bytes → same output bytes, forever.

---

### 2. mg-chat-agent — Governed AI Chat Agent

**Type:** Rust binary
**Protocol:** `{"v":1,"op":"respond","message":"...","editor_id":"...","llm_response":"..."}` → response JSON

**Purpose:** Governed AI agent for MiniGuardian collaboration. Generates
text and tool proposals from LLM responses. When `llm_response` is provided,
extracts display text and `[TOOL:...]` tags from it.

**Output:**
- `response` — display text
- `tool_requests` — extracted tool calls
- `governance_notice` — always true (advisory)
- `license` — "valid" | "demo" | "expired"
- `source_hash_hex` — source integrity hash

**Key behavior:** Advisory-only. The substrate (daemon/bridge) decides what
to relay, queue, or discard.

---

### 3. mg-content-scan — Content Scanner (Shell Script)

**Type:** Bash script (executable)
**Protocol:** `cat file | mg-content-scan <mode>` → exit code + JSON

**Purpose:** Pipes content through the daemon's deterministic memory-poison
defense via the tokenless eval socket. Works with ANY AI CLI/editor that
supports shell hooks (Claude Code, aider, editor tasks, shell intercepts).

**Modes:**
- `retrieval` — scan stdin as retrieval content
- `summary` — scan as compaction/summary content
- `prompt` — scan as a prompt

**Exit codes:**
- `0` = allow (clean / advisory allow / daemon unreachable — **fail open**)
- `2` = withhold (quarantine / reject / rollback)

**Fail-open design:** The hard boundary stays Landlock. If the daemon is
unreachable, content passes through. When the daemon answers
Quarantine/Reject/Rollback, the content is replaced with a quarantine notice.

---

### 4. miniguard-firewall.js — AI Content Firewall (JavaScript)

**Type:** JavaScript (Node.js)
**Protocol:** Tokenless, one line in → one JSON line out via eval.sock

**Purpose:** Routes retrieval + compaction content through the daemon's
deterministic memory-poison scanners before it shapes model context.
Designed for opencode but applicable to any AI tool with JS hooks.

**Commands:**
- `ping` → `"ok"`
- `mem_scan:retrieval:<base64>` → threat + action
- `mem_scan:summary:<base64>` → threat + action
- `mem_context:<total>:<system>:<ext>` → threat + action + external_pct + compact

**Features (v2.2):**
- **Adaptive Compaction Shield** (v2.1) — 3-layer hybrid (origin classification
  + digest canary + false-positive journal) prevents amnesia loops caused by
  the model's own planning text being quarantined during compaction
- **Retrieval handler** — hard-quarantine restricted to `external_fetch` only;
  local read/exec tools are advisory alerts with content kept
- **Auto-remember** (v2.2) — on tool.execute.after, a local tool output that
  is meaningfully sized and not a quarantine note is persisted as a compact
  durable memory fact (per-agent namespace, bounded 96-1200 chars, deduplicated)
- **Tool risk tier classification** (v2.0) — only validate write/exec/network
  tools; read-only tools pass through unblocked
- **Library Sentinel** — scans fetched content for vulnerabilities & malware
- **Patch suggestions** — AI gets recommendations for fixing vulnerabilities

**Fail-open:** Advisory by design. Fails OPEN when daemon unreachable (hard
boundary stays Landlock).

---

### 5. neural-inference — Governed Neural Inference

**Type:** Rust binary
**Protocol:** `{"v":1,"op":"generate","model_path":"...","prompt":"...","max_tokens":2048}` → response JSON

**Purpose:** Sandboxed plugin that loads GGUF-format transformer models and
runs text generation. Part of the state-driven intelligence architecture —
every cognitive function is a governed, hash-pinned, capability-brokered plugin.

**Output:**
- `response` — generated text
- `tokens_used` — token count
- `tool_requests` — any extracted tool calls
- `source_hash_hex` — source integrity hash
- `license` — "demo" (advisory)

**Key behavior:** Advisory-only. No filesystem write, no network, no
authority tokens. The substrate decides what to relay.

---

### 6. secret-detector — Credential/Secret Scanner

**Type:** Rust crate (library + binary)
**Protocol:** `{"v":1,"op":"scan","text":"..."}` → `ScanVerdict` JSON

**Purpose:** Scans text for high-confidence secret patterns (cloud keys,
tokens, private key blocks, embedded credentials) and reports REDACTED
findings. Never becomes an exfiltration channel — excerpts show at most a
4-char prefix plus length, never the secret body.

**Detected patterns:**
| Kind | Prefix | Min Length |
|------|--------|------------|
| `secret.aws_access_key_id` | `AKIA` | 20 |
| `secret.github_token` | `ghp_`, `gho_`, `ghs_` | 40 |
| `secret.slack_token` | `xoxb-`, `xoxp-` | 20 |
| `secret.stripe_live` | `sk_live_` | 24 |
| `secret.google_api_key` | `AIza` | - |
| `secret.private_key_block` | `-----BEGIN ... PRIVATE KEY-----` | - |

**Output:** `ScanVerdict` with:
- `clean` — bool (true if no findings)
- `count` — number of findings
- `findings` — array of `{kind, redacted, len}`

**Key guarantee:** The plugin itself must never become an exfiltration
channel. Redacted excerpts show 4-char prefix + length only.

---

### 7. trust-ai — TRust AI Plugin (Multi-Adapter)

**Type:** JavaScript adapters + shell scripts (universal installer)

**Purpose:** Universal AI tool governance adapter. Detects installed AI
tools and installs the appropriate plugin for each. Mediates AI access to
the TRust Terminal — every tool call is classified, quota-checked, and
either auto-allowed, blocked, or sent for human approval.

**Adapters:**
| Adapter | AI Tool |
|---------|---------|
| `claude.js` | Claude Code |
| `cline.js` | Cline |
| `continue.js` | Continue.dev |
| `cursor.js` | Cursor |
| `devin.js` | Devin |
| `opencode.js` | opencode |
| `windsurf.js` | Windsurf |
| `zed.js` | Zed |
| `aider.py` | Aider |

**CLI tools** (the trust-ai plugin's own file-based queue in
`~/.local/share/trust-ai/approvals/`; they do not approve MiniGuardian daemon
or `mg-cli` tool calls — see `GOVERNED_TOOLS_REFERENCE.md`):
- `trt-approve <approval-id>` — approve a pending AI action
- `trt-reject <approval-id>` — reject a pending AI action
- `trt-approve --list` / `trt-reject --list` — show pending approvals

**Install:**
```bash
bash trust-ai/install.sh
```
Detects installed AI tools and installs the plugin for each. Logs and
approvals stored in `~/.local/share/trust-ai/`.

---

## Common Patterns

### Advisory-Only Contract
All plugins are advisory. The daemon (substrate) is the authority. Plugins:
- Classify, scan, propose
- Never act, enforce, or mint authority tokens
- Output is reviewed by the substrate before anything reaches the model

### Fail-Open Design
When the daemon is unreachable, plugins fail OPEN (content passes through).
The hard boundary stays Landlock — the daemon is an additional layer, not
the only layer.

### Determinism
Rust plugins (intent-classifier, secret-detector) are deterministic:
- No clocks, no randomness, no environment, no I/O beyond input
- Same input bytes → same output bytes, forever
- Every verdict is reproducible at audit time

### One-Line JSON Protocol
Rust plugins share a simple protocol: one JSON line in, one JSON line out.
This makes them composable and testable.
