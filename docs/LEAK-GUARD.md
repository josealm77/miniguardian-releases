# Leak guard

AI tools have pushed people's keys, tokens and personal details to GitHub.
The leak guard stops a governed AI session from doing that.

## What it does

Before an AI session publishes anything, MiniGuardian reads what would
leave the machine. This happens before you are even asked. Publishing means:

- `git push` (any form, including `git -C <dir> push` and `sh -c "git push"`);
- `gh gist create <files>`;
- `gh release create <tag> <files>` and `gh release upload <tag> <files>`;
- `gh repo create … --push`.

For a push it reads every commit that is on a local branch or tag but not
yet on any remote. For a brand-new repository's first push, that is all
of it. It reads them as the session's user, with every repository setting
that could run a program switched off (hooks, fsmonitor, diff drivers,
pager).

| Found in what would be published | What happens |
|---|---|
| A credential: a private key, an AWS / GitHub / GitLab / Slack / `sk-` token, a password inside a URL, or `PASSWORD=…`-style values in a config or `.env` file | **Refused.** The AI gets the list (secrets masked) and how to fix it. You are not asked. |
| A secrets file by its name: `.env`, `id_rsa`, `*.pem`, `*.key`, `credentials`, `.netrc`, `.ssh/…` | **Refused**, as above. |
| A secret-looking assignment in code (`token = …`) | Your pop-up says **POSSIBLE SECRET IN THIS PUSH** with the lines. |
| Your personal information: always your home path, plus your own list | Your pop-up says **PERSONAL INFORMATION IN THIS PUSH** with the lines. |
| Something it could not read fully (over 32 MB or 2000 commits, a repository outside your home, `--git-dir`) | Your pop-up says **COULD NOT CHECK THIS PUSH FULLY**. |
| Nothing | The ordinary push pop-up. |

It is deterministic: patterns and your list, never a model's opinion.

## Your personal list

Add the things that should never leave in an AI's commit to
`/etc/miniguard/miniguard.toml`:

```toml
[leak_guard]
enabled = true
block_credentials = true
personal = ["Your Name", "you@example.com", "555-0100", "12 Example Street"]
```

Entries shorter than 4 characters are ignored. Matching ignores case.
`block_credentials = false` turns refusals into pop-ups (not recommended).

## When a push is refused

1. Remove the secret from the history, not just from the latest commit:
   `git rebase -i` for a few commits, `git filter-repo` for more.
2. **Rotate the credential anyway.** Assume anything that was in a commit
   may have been seen.
3. A line that is a deliberate example (documentation, a test fixture) can
   carry the marker `mg-leak-guard: allow`. AWS's documented `…EXAMPLE` keys
   are already ignored.

## Limits

- It covers sessions with `[session_egress]` on. Without it, an AI
  session's commands have no network at all, so nothing can be published.
- A script can hide a push (for example, a program that calls the GitHub
  API itself). The real control is a repository-scoped token you can
  revoke. The leak guard catches the ordinary ways AI agents publish.
- It reads text. Binary files are listed as not checked.
- Humans pushing outside a governed session are not affected.
