# MiniGuardian — Governed Tool System Reference

*How AI agents use tools under MiniGuardian governance, and the complete list of available commands.*

---

## What Is the Governed Tool System?

> This page covers the **29 tools an AI can call** through MiniGuardian's
> approval gate. For **which AI programs** MiniGuardian recognises and governs
> (Claude Code, opencode, Cursor, Aider, and 60+ more), see the
> [AI Containment Manual §14.12](AI_CONTAINMENT_MANUAL.md#1412-recognised-ai-programs).

The **Embedded Tool System** (`ToolRegistry` in `trust_terminal/src/terminal/smart_terminal/tool_system.rs`) is MiniGuardian's native, in-process alternative to external MCP servers. When an AI agent runs under MiniGuardian governance (via `mg-cli`), **this is the tool system it uses**.

Every tool invocation passes through five security layers before any side effect occurs:

1. **Argument validation** — type checking, path traversal detection (`../etc/passwd` blocked before any file touch)
2. **Apodeixis mission gate** — a proof-checked mission reads the invocation's arguments as injected signals and vetoes unsafe inputs (fail-closed)
3. **Capability check** — ReadOnly tools auto-execute; Write/Exec/Delete/System/Network tools queue for human approval
4. **Human approval gate** — a human approves or rejects out of band (desktop Review pop-up, the TUI's Tool Approvals page, or the auth socket), always behind sudo; the agent cannot bypass this
5. **Audit logging** — every invocation recorded with effects, exit code, and approval ID (bounded to 2,000 entries, FIFO eviction)

### Why embedded (not external MCP)?

- **No attack surface**: subprocess eval is attack surface (PATH hijack, binary swap, runaway processes). The embedded tool system runs in-process, deterministic, fail-closed.
- **No network dependency**: everything is native Rust, no external servers to compromise or spoof.
- **Human-gated**: write/exec/delete/system/network tools return `202 Pending` until a human signs off.
- **Apodeixis-guarded**: every tool can carry an Apodeixis constraint mission; proof failure vetoes the tool before any side effect.
- **Audit trail**: every invocation lands in the audit log with effects and approval ID.

---

## How AI Agents Use Tools

### From the TRust Terminal (in-process)

When running inside the Smart Terminal, tools are invoked directly:

```
tool: write_file path=src/main.rs content="fn main() { ..."
```

The terminal routes through `ToolRegistry::invoke()`, which applies all five security layers and returns:
- `SUCCESS` + output (read-only tools)
- `PENDING` + approval_id (write/exec/delete/system/network tools)
- `ERROR` (validation failure, path traversal, Apodeixis veto)

### From the CLI (`mg-trust-tool`)

The `mg-trust-tool` binary connects to the daemon's auth socket and routes tool invocations through the governed `ToolRegistry`:

```bash
# Read-only tools (execute immediately, still audited)
mg-trust-tool read_file /path/to/file
mg-trust-tool list_files /path/to/dir
mg-trust-tool grep "pattern" /path/to/dir
mg-trust-tool search_code "fn main" /path/to/src
mg-trust-tool tree /path/to/dir
mg-trust-tool glob "*.rs" /path/to/dir
mg-trust-tool file_metadata /path/to/file
mg-trust-tool git_status /path/to/repo

# Write tools (queue for human approval)
mg-trust-tool write_file /path/to/file "content to write"
mg-trust-tool edit_file /path/to/file "old text" "new text"
mg-trust-tool append_file /path/to/file "content to append"
mg-trust-tool apply_patch /path/to/file "old" "new"
mg-trust-tool create_dir /path/to/new/dir
mg-trust-tool copy_file /source/path /dest/path
mg-trust-tool move_file /old/path /new/path
mg-trust-tool delete_file /path/to/file

# Execute tools (queue for human approval)
mg-trust-tool run_command "cargo build --release"
mg-trust-tool setenv MY_VAR "value"

# Network tools (queue for human approval)
mg-trust-tool network_fetch "https://example.com/api"
```

### From MCP Clients (Msty Studio, Antigravity, Claude, Gemini, etc.)

The `mg-mcp-bridge` binary fronts the daemon's `ToolRegistry` to any Model Context Protocol client over STDIO:

```
MCP client ──STDIO(MCP JSON-RPC)──> mg-mcp-bridge ──AF_UNIX(token)──> daemon tool:invoke ──> ToolRegistry
```

- `tools/list` advertises all 29 tools with JSON schemas
- `tools/call` maps to `tool:invoke <name> <json>` on the auth socket
- Write/exec/delete/network calls return MCP `remotePending: true` with the approval ID
- Approval stays human-owned and out-of-band — no MCP client can auto-approve its own mutations

### From Governed Sessions (`mg-cli`)

When launched via `mg-cli`, the child AI tool inherits the governed profile. The tool system is available through the daemon socket:

```bash
sudo mg-cli --curated-dev --deterministic --share-data --workspace ~ /path/to/ai-tool
```

---

## Human Approval Flow

When a write/exec/delete/system/network tool is invoked, it queues instead of
running. The caller gets exit code `202` and a message like:

```
[TOOL GATE] 'write_file' requires human approval
ID: tool-0
Preview: ...
```

A human then decides, out of band — never the agent:

- **Desktop pop-up.** The logged-in user gets a notification; clicking it (or
  its **Review** button, where the desktop shows buttons) opens a terminal
  running `sudo -k miniguard-status --approvals` — one screen for daemon calls,
  unregistered tools and every `mg-cli` session's queue, password asked every
  time. The notification itself approves nothing, so a forged click cannot
  approve. On a notification server without click support the notification
  text says which command to run. Pop-ups are coalesced (one per review
  target at most every 2 minutes).
- **TUI.** `sudo miniguard-status`, press `t` (Tool Approvals): `↑`/`↓` select,
  `Enter` approve, `x` reject. Governed tool calls are listed as `call`.
- **Auth socket** (token first line):
  ```bash
  sudo sh -c '{ printf "%s\n" "$(cat /run/jayce-operator/miniguard.sock.token)"; echo "tool:pending"; } | socat - UNIX-CONNECT:/run/jayce-operator/miniguard.sock'
  sudo sh -c '{ printf "%s\n" "$(cat /run/jayce-operator/miniguard.sock.token)"; echo "tool:approve tool-0"; } | socat - UNIX-CONNECT:/run/jayce-operator/miniguard.sock'
  sudo sh -c '{ printf "%s\n" "$(cat /run/jayce-operator/miniguard.sock.token)"; echo "tool:reject tool-0"; } | socat - UNIX-CONNECT:/run/jayce-operator/miniguard.sock'
  ```

Tool calls queued inside an `mg-cli` session (the hook bridge) live in that
session's own registry: decide them with `sudo mg-cli session review`, which
the session's Review pop-up opens.

`trt-approve` / `trt-reject` (`plugins/trust-ai/bin`) manage the trust-ai
plugin's own file-based queue in `~/.local/share/trust-ai/approvals/`. They do
**not** approve daemon or `mg-cli` tool calls.

## Auto-approve (operator rules, earned trust, timed window)

Off by default. Configured by the operator in `/etc/miniguard/miniguard.toml`
(root-owned and immutable under the seal, so a governed agent cannot add rules
for itself); decided deterministically in `miniguard/src/auto_approve.rs`; every
auto-approval is logged (`[auto_approve] AUTO-APPROVED ...`), goes through the
same registry approve path and audit log as a human approval, and is reported to
the caller as `"auto_approved": "<basis>"`. Applies to calls queued in the
**daemon** (`mg-trust-tool`, `mg-mcp-bridge`); session items and unregistered
tools always wait for a human.

```toml
[auto_approve]
enabled = true

# 1. Rules: exact tool + every path the call touches under path_prefix.
[[auto_approve.rules]]
tool = "write_file"
path_prefix = "/home/you/projects/myapp"
# agent = "some-agent-id"        # optional: only this requesting agent

# 2. Earned trust: agents whose conditioning record earned
#    operator_prompt_threshold <= 1 get in-place edits approved under these
#    paths. Any fresh block steps the threshold back to 2 and ends it.
earned_trust = true
earned_trust_paths = ["/home/you/projects"]

# 3. Timed window: the operator opens it (TUI approvals page `w`, or
#    `autoapprove on 20` on the auth socket); capped here and at 120 min;
#    closed by `autoapprove off` or a daemon restart.
max_window_minutes = 30
```

| Mechanism | Tools it can approve |
|---|---|
| Rule | the one tool named in the rule |
| Earned trust | `write_file`, `edit_file`, `append_file`, `create_dir`, `apply_patch`, `doc_*` |
| Timed window | the earned set plus `copy_file`, `ingest_file`, `ingest_document` |

**Never auto-approved, whatever the config:** `run_command`, `network_fetch`,
`ingest_link`, `setenv`; relative paths or paths containing `..`; and any path
whose literal *or symlink-resolved* form is sensitive — shell rc files, `.ssh`,
`.gnupg`, `.git/hooks`, `.claude`, `.config`, `.local/share`, `.local/bin`,
`settings.json`, `hooks.json`, `.mcp.json`, `.gitconfig`, `/etc`, `/usr`,
`/root`, `/run`, MiniGuardian's own state, and similar. Deletes and moves are
only ever approved by an explicit rule.

## Complete Tool List (29 Tools)

> **Ported from the Jayce SDK.** The TRust Smart Terminal was vendored from the SDK's
> `trust_core` into the `trust_terminal` crate, and the governed-tool layer was
> extended here (covert-channel hard-deny, taint window, fixed `PATH` for
> `run_command`). The SDK registers four more tool families that this build does
> **not** register: action/editing (`action_tools`), browser (`browser_tools`),
> research/ingest pipeline (`research_tools`) and web development (`web_tools`), about
> 5,200 lines in total (tool names include `governed_fetch`, `governed_scrape`,
> `governed_run`, `browser_action_click`, `browser_governed_js_eval`). No document or
> changelog records why they were left out. The 29 tools below are what this build offers.

### Read-Only Tools (10) — Auto-approve, still audited

| Tool | Description | Parameters |
|------|-------------|------------|
| `read_file` | Read a file's contents | `path` (required) |
| `list_files` | List files in a directory | `path` (optional, default `.`) |
| `search_code` | Search for a pattern in source files | `pattern` (required), `path` (optional) |
| `grep` | Regex search across files | `pattern` (required), `path` (optional), `case_insensitive` (optional), `include` (optional glob filter) |
| `file_metadata` | Return metadata (size, type, permissions, modified) | `path` (required) |
| `tree` | Recursively list a directory tree | `path` (optional), `max_depth` (optional, default 4) |
| `glob` | Expand a glob pattern into matching paths | `pattern` (required) |
| `list_env` | List environment variable names | *(no parameters)* |
| `getenv` | Read the value of an environment variable | `name` (required) |
| `git_status` | Read-only git status for the current repo | `path` (optional) |

### Write Tools (8) — Require human approval

| Tool | Description | Parameters |
|------|-------------|------------|
| `write_file` | Write content to a file | `path` (required), `content` (required) |
| `edit_file` | Replace text in a file | `path` (required), `old_string` (required), `new_string` (required) |
| `append_file` | Append content to an existing file | `path` (required), `content` (required) |
| `apply_patch` | Apply a unified-diff patch to a file | `path` (required), `old_string` (required), `new_string` (required), `count` (optional, 0=all) |
| `create_dir` | Create a directory (and parents) | `path` (required), `recursive` (optional, default false) |
| `copy_file` | Copy a file or directory to a destination | `source` (required), `destination` (required), `recursive` (optional) |
| `move_file` | Move or rename a file or directory | `source` (required), `destination` (required) |
| `delete_file` | Delete a file | `path` (required) |

### Execute Tools (2) — Require human approval

| Tool | Description | Parameters |
|------|-------------|------------|
| `run_command` | Run a shell command | `command` (required), `cwd` (optional) |
| `setenv` | Set an environment variable; empty value unsets it | `name` (required), `value` (optional) |

### Network Tools (1) — Require human approval

| Tool | Description | Parameters |
|------|-------------|------------|
| `network_fetch` | Fetch a URL over HTTP(S), bounded response | `url` (required), `max_bytes` (optional, default 8192, hard cap 1 MiB) |

### Governed Document Tools (5) — Apodeixis mission-gated

These tools carry sealed Apodeixis missions that proof-check arguments before execution. Write tools additionally require human approval.

| Tool | Description | Parameters | Approval |
|------|-------------|------------|----------|
| `doc_read_lines` | Read a numbered line range from a text document | `path` (required), `start_line` (optional, default 1), `end_line` (optional, 0=to end) | Auto |
| `doc_insert_line` | Insert content after a given line number | `path` (required), `line` (required), `content` (required) | Required |
| `doc_replace_range` | Replace a range of lines (start..end inclusive) | `path` (required), `start_line` (required), `end_line` (required), `content` (required) | Required |
| `doc_regex_replace` | Replace all regex matches in a document | `path` (required), `pattern` (required), `new` (required), `count` (optional, 0=all) | Required |
| `doc_format_edit` | Edit a heading-delimited section (markdown/list aware) | `path` (required), `section` (required), `content` (required) | Required |

---

### Semantic-Memory Ingest Tools (3) — Apodeixis mission-gated, require human approval

These store external content in the daemon's semantic memory graph, so each
carries a sealed `ingest_mission` and always queues for approval.

| Tool | Description | Parameters | Capabilities |
|------|-------------|------------|--------------|
| `ingest_file` | Read a local file and store its contents in the semantic memory graph | `path` (required), `tag` (optional) | ReadOnly + WriteFile |
| `ingest_document` | Read a document file and store its contents in the semantic memory graph | `path` (required), `tag` (optional, default `document`) | ReadOnly + WriteFile |
| `ingest_link` | Fetch a URL and store its contents in the semantic memory graph | `url` (required), `tag` (optional), `max_bytes` (optional, default 100 KB) | NetworkAccess + WriteFile |

---

## Apodeixis Mission Gating (Doc Tools)

The five document tools are protected by embedded, proof-checked Apodeixis missions. These missions read the invocation's arguments as injected `signal()` inputs and vetoes (fail-closed) on:

- **Path traversal**: any `..` or `/../` or absolute path starting with `/` in path-like arguments
- **Oversized payloads**: content > 1 MiB, patterns > 16 KB, paths > 4,096 bytes
- **Invalid ranges**: `start_line > end_line`, `start_line < 1`, `end_line < start_line`
- **Invalid counts**: negative counts in regex replace

The missions are sealed in the binary at compile time. The `args_to_signals()` function projects the tool's arguments into numeric signals (path-traversal flags, byte lengths, line positions, counts) and the mission cross-checks the invariants — proof of any disagreeing signal vetoes the invocation before any side effect.

```rust
// Example: doc_write_mission vetoes path traversal + oversized content
fn main() {
    let p = signal("path_has_traversal");
    let c = signal("content_len");
    if p > 0.0 || c > 1000000.0 || signal("path_len") > 4096.0 {
        emit_action("block_doc_insert_line", "unsafe_input")
    } else {
        emit_action("doc_insert_line", "docpath")
    }
}
```

---

## Security Layers (in order)

```
Agent invokes tool
    │
    ▼
┌─────────────────────────────┐
│ 1. Argument Validation      │  Type checking, path traversal, bounds
└─────────────┬───────────────┘
              │ pass
              ▼
┌─────────────────────────────┐
│ 2. Apodeixis Mission Gate   │  Proof-checked mission; fail-closed veto
└─────────────┬───────────────┘
              │ pass
              ▼
┌─────────────────────────────┐
│ 3. Capability Check         │  ReadOnly = auto-execute
│                             │  Write/Exec/Delete/System/Network = queue
└─────────────┬───────────────┘
              │
              ├─ ReadOnly ──────> Execute + Audit
              │
              ▼
┌─────────────────────────────┐
│ 4. Human Approval Gate      │  Review pop-up / TUI / socket (sudo)
└─────────────┬───────────────┘
              │ approved
              ▼
┌─────────────────────────────┐
│ 5. Execute + Audit Log      │  Recorded with effects, exit code, approval ID
└─────────────────────────────┘
```

---

## Capability Classification

| Capability | Auto-execute? | Examples |
|-----------|--------------|----------|
| `ReadOnly` | Yes (audited) | read_file, list_files, grep, tree, glob, file_metadata, list_env, getenv, git_status, doc_read_lines |
| `WriteFile` | No (approval required) | write_file, edit_file, append_file, apply_patch, create_dir, copy_file, move_file, doc_insert_line, doc_replace_range, doc_regex_replace, doc_format_edit |
| `ExecuteCommand` | No (approval required) | run_command |
| `DeleteFile` | No (approval required, double-confirm) | delete_file |
| `NetworkAccess` | No (approval required) | network_fetch |
| `SystemModify` | No (approval required) | setenv |

---

## Audit Trail

Every tool invocation is recorded in the audit log (bounded to 2,000 entries with FIFO eviction):

```
ToolResult {
    success: bool,
    stdout: String,
    stderr: String,
    exit_code: i32,
    audit_id: String,      // unique invocation ID
    approval_id: Option<String>,  // set if human-approved
    effects: Vec<String>,  // e.g. ["write:src/main.rs:bytes=1024"]
}
```

The audit log is accessible via the daemon's management socket and the TUI's Tool Governance panel (`g` key in `miniguard-status`).

---

## Adding Custom Tools

Register a `ToolDefinition` with a Rust closure handler and optional Apodeixis constraint:

```rust
registry.register(ToolDefinition {
    name: "my_tool".into(),
    description: "Does something useful".into(),
    parameters: vec![
        ToolParameter {
            name: "input".into(),
            description: "The input value".into(),
            param_type: ParamType::String,
            required: true,
            default: None,
        },
    ],
    capabilities: vec![ToolCapability::ReadOnly],  // or WriteFile, etc.
    requires_approval: false,  // true for write/exec/delete/system/network
    apodeixis_constraint: None,  // or Some(mission_string)
    handler: Arc::new(|args| {
        let input = args.get("input").ok_or("Missing input")?;
        Ok(ToolResult {
            success: true,
            stdout: format!("Processed: {}", input),
            stderr: String::new(),
            exit_code: 0,
            audit_id: format!("my-tool-{}", now_secs()),
            approval_id: None,
            effects: vec![format!("my_tool:{}", input)],
        })
    }),
});
```

Keep handlers deterministic (no I/O outside args, no randomness, no unbounded loops). See `tool_system.rs` and the doc tool implementations for examples.

---

## MCP Bridge (`mg-mcp-bridge`)

For any MCP-capable client (Msty Studio, Antigravity, Claude, Gemini, etc.), the MCP bridge fronts the daemon's `ToolRegistry` over STDIO:

```bash
# Build and install
cargo build --release --bin mg-mcp-bridge
sudo cp target/x86_64-unknown-linux-musl/release/mg-mcp-bridge /usr/local/bin/

# Register with the client as a STDIO tool pointing at the binary
# (e.g. Msty's Toolbox, or `agy mcp add`)
```

Architecture: `MCP client ──STDIO(MCP JSON-RPC)──> mg-mcp-bridge ──AF_UNIX(token)──> daemon`

- Advertises all 29 tools with JSON schemas
- Write/exec/delete/network calls return MCP `remotePending: true`
- Approval stays human-owned — no MCP client can auto-approve its own mutations
- No third-party code bundled; uses the open MCP protocol as a plain client

---

## Files

- `trust_terminal/src/terminal/smart_terminal/tool_system.rs` — ToolRegistry core, all 29 tools, missions, audit
- `miniguard/src/bin/mg_trust_tool.rs` — CLI access to the tool system via daemon socket
- `miniguard/src/bin/mg_mcp_bridge.rs` — MCP bridge for any MCP-capable client
- `miniguard/src/approval_prompt.rs` + `scripts/mg-approval-prompt.sh` — desktop Review pop-up
- `miniguard-status` (`--approvals`, page `t`) — approval screen for daemon tool calls and unregistered tools
