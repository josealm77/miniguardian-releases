# Teach `mg-why` a new language

A language file is a small TOML file of text patterns. It can't run
anything. Put it in `~/.config/miniguard/languages/` (just for you) or
`/usr/share/miniguard/languages/` (the whole machine) and it works
immediately, with no rebuild. A file with the same `name` as a built-in one
replaces the built-in.

Built in: Python, JavaScript/TypeScript, Rust, Go, Java, Ruby and C/C++.
Apodeixis is explained by its own engine.

## A complete example: Elixir

An Elixir crash looks like this:

```
** (ArithmeticError) bad argument in arithmetic expression
    (my_app 0.1.0) lib/my_app/calc.ex:10: MyApp.Calc.run/1
    (elixir 1.15.0) lib/kernel/cli.ex:136: anonymous fn/3 in Kernel.CLI.exec_fun/2
```

Save this as `~/.config/miniguard/languages/elixir.toml`:

```toml
# Elixir: `** (Error) message` then `    (app 0.1.0) lib/x.ex:10: Mod.fun/1`.
name = "elixir"
programs = ["elixir", "mix", "iex"]
trace_start = ['^\*\* \(']
frame = ['^\s+\([\w_]+ [\d.]+\) (?P<file>[^:\s]+\.exs?):(?P<line>\d+): (?P<func>\S+)']
message = ['^\*\* \((?P<msg>.+)$']
library_paths = ["lib/kernel/", "deps/", "lib/elixir/"]
file_suffixes = [".ex", ".exs"]

[[meaning]]
pattern = '^ArithmeticError'
says = "Arithmetic on something that is not a number (often nil)."
```

`mg-why` then reports `where: lib/my_app/calc.ex:10` (the line from
`lib/kernel/` is library code, so it's skipped), the error, and the meaning.
This exact file is tested in `why_core/tests/user_lang.rs`.

## The fields

| Field | What it is |
|---|---|
| `name` | The language name shown in reports. The same name as a built-in replaces it. |
| `programs` | Program names that run this language. They're prefix-matched, so `python` covers `python3.12`. |
| `trace_start` | Patterns for the line where an error report begins. A language is used only when one matches. |
| `frame` | Patterns for one stack frame. Named groups: `file` and `line` (required), `func` (optional). |
| `message` | Patterns with a `msg` group for the error message. If none match, the line after the trace start is used. |
| `library_paths` | Text that marks a frame as library code, not yours. "where" skips these frames. |
| `file_suffixes` | Accept frames only for these file endings (stops mixing languages up). |
| `innermost` | `"first"` (the default) or `"last"`, meaning where the crashing frame is printed. Python prints it last. |
| `[[meaning]]` | A `pattern` plus `says`: a plain-words explanation for a known error. |

## Rules

- At most 64 KiB and 64 patterns per file. Patterns use Rust's `regex`
  syntax, which runs in linear time, so a pattern can't hang the tool.
- A broken file is skipped with a warning. It never breaks `mg-why`.
- Each finding names the file it came from, so you always know which
  language file produced an explanation.
