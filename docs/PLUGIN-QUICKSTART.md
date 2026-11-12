# Your first MiniGuardian plugin (about 15 minutes)

A MiniGuardian plugin is **your own program, in any language**, that runs
inside MiniGuardian's open plugin shell. You don't sign anything and you don't
touch MiniGuardian's code. You need no `sudo`.

What you get for free:

- Your program runs **contained**: it has no network, it can't read your
  files, and it isn't running as you.
- To do anything (read a file, write one, fetch a URL, run a command), it
  **asks MiniGuardian**. MiniGuardian checks the request against what the
  machine's owner approved when they installed your plugin.
- Reading is automatic inside the approved folders. Writing, running commands
  and network access show the owner a pop-up each time.
- Every run is logged: who called it, and every action with its outcome.

## 1. Create it (1 minute)

```
mg-plugin new hello --lang python      # or: --lang node, rust, apodeixis
cd hello
```

This creates a working plugin. It also creates `~/mg-plugin-data/hello/hello.txt`,
the one file the starter is allowed to read.

## 2. Run it (2 minutes)

```
mg-plugin test
```

1. MiniGuardian copies your folder and fingerprints it.
2. The supply-chain sentinel checks its dependencies.
3. A pop-up asks the machine owner to approve it. On your own computer,
   that's you. It shows the plugin's name, fingerprint, every permission it
   asks for, and the dependency verdict.
4. Once you approve, the plugin runs. You see its log lines, each action it
   asked for and what happened, and its answer.

Change the code and run `mg-plugin test` again. A changed plugin has a new
fingerprint, so it needs a new approval: nobody's code runs without the
owner seeing it.

## 3. How a plugin talks

One JSON object per line on stdin/stdout. Every message has `"mgp": 1`.

| You receive | You send |
|---|---|
| `{"type":"request","id":"r1","input":…,"caller":{"kind":"human"\|"agent","name":…}}` | `{"type":"log","level":"info","msg":"…"}` (any time) |
| `{"type":"action_result","id":"a1","ok":true,"output":"…","exit_code":0}` or `{… "ok":false,"denied":"why"}` | `{"type":"action","id":"a1","tool":"read_file","args":{"path":"~/…"}}` |
| | `{"type":"answer","id":"r1","output":…}` — the end of the run |

Unknown fields are ignored, so newer MiniGuardian versions won't break your plugin.

## 4. Ask for what you need: `mg-plugin.toml`

```toml
name = "hello"
version = "0.1.0"
author = "Your Name"
api = 1
run = ["python3", "main.py"]   # python3, node, or a file you list below
files = ["main.py"]            # everything that ships with the plugin

[[capability]]                 # one block per tool
tool = "read_file"
paths = ["~/mg-plugin-data/hello"]

[[capability]]
tool = "write_file"
paths = ["~/mg-plugin-data/hello/out"]

[[capability]]
tool = "network_fetch"
hosts = ["api.example.com"]

[[capability]]
tool = "run_command"
commands = ["git"]             # program names only
paths = ["~/projects/hello"]   # folders it may work in (pass "cwd")
```

| Tools you can declare | Scope |
|---|---|
| `read_file list_files file_metadata grep search_code tree git_status` | `paths` (read) |
| `write_file append_file edit_file create_dir copy_file move_file delete_file` | `paths` (write) |
| `network_fetch` | `hosts` |
| `run_command` | `commands` + `paths`; no shell operators (`; & \| $ \` > <`) |

Always pass full paths: `~/…` means the installer's home. A tool's default
folder isn't something the owner approved, so leaving out a path is denied.

## Languages

- **Any language** can be a plugin. `run[0]` is either a file in your plugin
  folder (a native binary) or the name of a runtime program on the machine,
  such as `python3`, `node`, `ruby` or `java`. `python3`, `node` and
  `apodeixis` are approved everywhere. For any other runtime, the owner's
  install pop-up shows one extra line ("needs runtime ruby — /usr/bin/ruby"),
  and after that one approval it works for every plugin.
- **Mixed-language plugins** list the extra runtimes they call:
  `runtimes = ["node"]` next to `run = ["python3", "main.py"]`.
- **Apodeixis** (`mg-plugin new hello --lang apodeixis`): deterministic,
  step-budgeted and information-flow checked before it runs. Numeric input
  fields arrive as `signal("name")`, and `emit_action(tool, target)` asks for
  one governed action.

- **Python**: runs on the system `python3`. To use packages, put their wheels
  in `.mg-wheelhouse/` with a hashed `requirements.txt` or `uv.lock`, and list
  all three in `files`. The sentinel blocks any package it can't verify.
- **Node**: runs on the system `node`. To use packages, run `npm ci`, then list
  `node_modules`, `package.json` and `package-lock.json` in `files`.
- **Rust**: `mg-plugin test` builds it with `cargo build --release` and copies
  the binary to `bin/<name>`. The starter has no dependencies. If you add
  crates, list `Cargo.toml` and `Cargo.lock` in `files`.
- **Anything else**: ship a native binary in `bin/` and set `run = ["bin/name"]`.

## Commands

```
mg-plugin new <name> --lang rust|python|node
mg-plugin test [DIR] [--input '{"any":"json"}']
mg-plugin install [DIR]          # a normal (non-dev) install
mg-plugin run <name> [JSON]
mg-plugin list
mg-plugin logs <name> [-n 20]
mg-plugin remove <name>
```

In the TRust terminal, the same commands work as `plugin new …` and so on.
AI agents connected through MiniGuardian's MCP broker see each installed
plugin as a tool named `plugin_<name>`.

## When it fails

`mg-plugin` shows where your plugin's own code broke, and any program can be
explained with `mg-why -- <command>` (see `docs/EXPLAIN-A-FAILURE.md`).

## Network

- **`network_fetch`** fetches one URL from a host listed in its `hosts`.
- **Commands** (`run_command`: `npm install`, `pip`, `cargo build`, your
  program) reach the internet only through MiniGuardian's egress proxy.
  `HTTPS_PROXY` and friends are set for them, and most tools follow those
  automatically.
- **Hosts:** the `hosts` you list are approved at install. Any other host
  pauses your command while the owner gets a pop-up. If they answer later,
  just run it again.
- **Programs that ignore the proxy variables have no network at all.** The
  command's error output then says so.
- **HTTPS only.** Private and local addresses are never reachable through
  the proxy.
- **Limits:** large uploads and downloads, and bursts of connections, ask
  the owner before continuing (`[egress]` in `miniguard.toml`).
- **Time:** commands that use the proxy may run up to 15 minutes. Every
  connection appears in `mg-plugin run` output.

## Confidential data

If your plugin reads something confidential (a folder the owner marked in
`[open_plugins] confidential`, a `.env`, an SSH or cloud key, or content that
looks like a secret) and then tries to send anything off the machine, the
owner sees a pop-up naming what was read and where it would go. Results
carry `"label": "secret"` from then on, so your plugin can tell.

## When something is denied

`mg-plugin` prints a hint under every denial. The common ones:

| Message starts with | What to do |
|---|---|
| `tool not approved` | Add a `[[capability]]` for that tool and run `mg-plugin test` again. |
| `outside approved scope` | Use a path inside your listed folders, a listed host, or a listed command. |
| `supply sentinel: BLOCK` | Vendor your dependencies (see Languages above). |
| `installed files changed since approval` | Run `mg-plugin install` again to approve the current version. |
| `approval timed out` | Nobody answered the pop-up within 30 minutes. Run it again. |
| `time budget exceeded` | Your plugin used more than 30 s of its own time (approval waits don't count). |
| `… is owned by uid …` | Install only folders you own. |

## What you can't do, by design

- Reach the network, your files or other programs directly. Everything goes
  through actions.
- Do anything the owner didn't approve. A new permission means a new install
  approval.
- Approve your own installs or actions from a script or an AI agent. Only a
  human at the console can approve.
