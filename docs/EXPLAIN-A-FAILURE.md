# Explain a failure with `mg-why`

`mg-why` tells you **where your code broke, why, and which processes were
involved**, for a program in any language. It works on any machine. With
MiniGuardian installed, it also tells you what MiniGuardian did to your
program. It doesn't need `sudo`, but as root it can see more (below).

```
mg-why -- python3 app.py          # run it; if it fails, explain
mg-why last                       # explain the last failure again
mg-why pid 4242                   # a recorded or still-running process
mg-why --json -- npm test         # the same, as JSON (for AI agents and tools)
```

When the program succeeds, `mg-why` stays silent and exits with the
program's exit code. When it fails, you get something like this:

```
✗ python3 app.py: KeyError
  where:  /home/me/app/main.py:30 in run [python]
  error:  KeyError: 'id'
  chain:  python3.12 (4242) [python] → exit 1
  next:   A dictionary was asked for a key it does not contain.
```

**System failures.** When the failure comes from Linux itself rather than
your code (a driver that won't load, a full disk, a busy package manager, a
missing systemd unit, a port in use, DNS), fixed system rules name the cause
and the next step. Some reasons only the kernel knows: lockdown, AppArmor,
seccomp, out-of-memory kills. Run the command as `sudo mg-why -- <command>`
and `mg-why` also reads the kernel log from while it ran, matches it against
the same rules, and shows those lines. Without root it says when the kernel
log might hold the answer. A `sudo` run is saved in your own history, so
`mg-why last` and the TRust terminal's `suggest` find it.

```
$ sudo mg-why -- modprobe uvcvideo
✗ modprobe uvcvideo: Kernel lockdown refused to load a driver (modprobe)
  next:   Lockdown (switched on by MiniGuardian's seal) only loads drivers …
  kernel log while it ran:
    Lockdown: modprobe: unsigned module loading is restricted; see man kernel_lockdown.7
```

Add your own system rules in `~/.config/miniguard/system/*.toml` (format at
the top of `why_core/system/system.toml`; yours are checked first).

**Mixed languages.** Every process in the chain is read in its own language,
so a Python program that calls a Rust helper shows both: the Rust panic
inside the helper, and the Python line that called it.

**What MiniGuardian did.** If MiniGuardian froze, refused or blocked
something, the report says so first, with how to resume. When a dependency
was blocked, it names the exact branch of the supply-chain policy mission
that decided, and the values it saw.

**Apodeixis programs** are explained by Apodeixis itself: the stage (type,
proof, information flow, step budget), the line, and the reason.

**Privacy.**
- Secrets (tokens, passwords, keys) are masked everywhere.
- When an AI agent reads the report (`--json`, or any program in a governed
  AI session), it never receives the program's raw output or kernel log
  lines. It gets the
  facts (file, line, error type, exit code, chain) and the masked messages.

**Other tools that use it.**
- In the TRust terminal, type `why -- <command>` or `why`.
- In this repo, run `bash scripts/mg_check.sh run -- <command>`.
- A failed `mg-plugin run` shows where the plugin's own code broke.

**Unsupported languages.** If a language isn't built in, `mg-why` still gives
you the process chain, the exit code or signal, and any `file:line` it can
spot. To teach it the language, see
[WRITE-A-LANGUAGE-FILE.md](WRITE-A-LANGUAGE-FILE.md).
