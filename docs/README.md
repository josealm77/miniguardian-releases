# MiniGuardian documentation

Checked against the source tree on 2026-09-26. Every file path, binary name,
command-line flag and socket command mentioned in the documents in this
directory was verified to exist; counts (tools, commands, languages, engines)
were re-measured from the code. Behaviour that is stubbed or not yet verified in
this beta is labelled where it is described.

## Start here
- [overview.md](overview.md) — what MiniGuardian is, what is implemented, what is partial.
- [ROADMAP_COHOST.md](ROADMAP_COHOST.md) — the plan for enforcement beneath the OS (cohost mode), with phases, test criteria and team needs.
- [MINIGUARDIAN_ONE_PAGE.md](MINIGUARDIAN_ONE_PAGE.md) — one-page summary.
- The repository [README](../README.md) (install, build, test) and
  AGENTS.md (full operator and developer notes, design history).

## Using it
| Document | What it covers |
|---|---|
| [TUI_USER_GUIDE.md](TUI_USER_GUIDE.md) | `miniguard-status`: every page, panel, key and flag, including Tool Approvals |
| [MINIGUARDIAN_ERROR_GUIDE.md](MINIGUARDIAN_ERROR_GUIDE.md) | Troubleshooting |
| [SECURE_UPDATES_GUIDE.md](SECURE_UPDATES_GUIDE.md) | Signed over-the-air updates |
| [TRUST_COMMAND_MANUAL.md](TRUST_COMMAND_MANUAL.md) | TRust terminal commands; each row carries its status in this build |

## AI containment and governance
| Document | What it covers |
|---|---|
| [AI_CONTAINMENT_MANUAL.md](AI_CONTAINMENT_MANUAL.md) | How governed AI sessions are contained |
| [GOVERNED_TOOLS_REFERENCE.md](GOVERNED_TOOLS_REFERENCE.md) | The 29 governed tools, the human approval flow, the MCP bridge |
| [PLUGIN-GUIDE.md](PLUGIN-GUIDE.md) | Signed, sandboxed plugins |

## Security and internals
| Document | What it covers |
|---|---|
| [SECURITY_FEATURES_CATALOG.md](SECURITY_FEATURES_CATALOG.md) | Every security feature, with its module |
| [APODEIXIS_LANGUAGE_GUIDE.md](APODEIXIS_LANGUAGE_GUIDE.md) | The proof-checked mission language |
| [ASTRAXIS_ROBOTICS_GUIDE.md](ASTRAXIS_ROBOTICS_GUIDE.md) | Robotics movement and mission engine |
| [JAYCE_1000HZ_TICK_WHITEPAPER.md](JAYCE_1000HZ_TICK_WHITEPAPER.md) | The deterministic 1 kHz tick |
| [papers/](papers/) | Whitepapers (historical, kept as written) |

## Other folders
- design/ — design specifications and risk gates (some describe work
  not yet built; each says so).
- reports/ — dated snapshots: red-team assessment, containment
  validator results, governance report, and the 2026-09-19 claims audit.
- archive/ — superseded, SDK-era, marketing and draft documents.
  Kept for history; **not maintained and not accurate for the current build.**

## Legal
- [../LICENSE](../LICENSE) — Jayce Automata Research Source License v1.0 (proprietary).
- [DEVBETA_PARTNER_LICENSE.md](DEVBETA_PARTNER_LICENSE.md) — dev-beta partner terms.
