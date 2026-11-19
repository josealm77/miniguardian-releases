# MiniGuardian documentation

Checked against the source tree on 2026-09-26, 2026-11-13 (beta-31) and
2026-11-19 (beta-37). File paths, binary names, command-line flags and socket
commands were checked against the code; counts (tools, commands, languages,
engines) were re-measured from it. Behaviour that is stubbed or not yet
verified in this beta is labelled where it is described.

## Start here
- [overview.md](overview.md) — what MiniGuardian is, what is implemented, what is partial.
- [ROADMAP_COHOST.md](ROADMAP_COHOST.md) — the plan for enforcement beneath the OS (cohost mode), with phases, test criteria and team needs.
- [MINIGUARDIAN_ONE_PAGE.md](MINIGUARDIAN_ONE_PAGE.md) — one-page summary.
- [OPEN-PLUGIN-DEVELOPER-MANUAL.md](OPEN-PLUGIN-DEVELOPER-MANUAL.md) — for developers: what MiniGuardian and the Jayce kernel are, building open plugins in any language, TUI/TRust commands, approvals, and an honest security summary.
- The repository [README](../README.md): install, verify, uninstall.

## Using it
| Document | What it covers |
|---|---|
| [TUI_USER_GUIDE.md](TUI_USER_GUIDE.md) | `miniguard-status`: every page, panel, key and flag, including Tool Approvals |
| [MINIGUARDIAN_ERROR_GUIDE.md](MINIGUARDIAN_ERROR_GUIDE.md) | Troubleshooting |
| [SECURE_UPDATES_GUIDE.md](SECURE_UPDATES_GUIDE.md) | Signed over-the-air updates |
| [TRUST_COMMAND_MANUAL.md](TRUST_COMMAND_MANUAL.md) | Operator commands (`mg-explain`, `mg-op`, `mg-fleet`, `mg-pentest`, …) and TRust terminal commands; each row carries its status in this build |
| [EXPLAIN-A-FAILURE.md](EXPLAIN-A-FAILURE.md) | `mg-why`: where your program broke and why, in any language |
| [WRITE-A-LANGUAGE-FILE.md](WRITE-A-LANGUAGE-FILE.md) | Teach `mg-why` a new language |
| [HOST-SUPPLY-SCAN.md](HOST-SUPPLY-SCAN.md) | `mg-supply host-scan`: known-vulnerable and tampered software |

## AI containment and governance
| Document | What it covers |
|---|---|
| [AI_CONTAINMENT_MANUAL.md](AI_CONTAINMENT_MANUAL.md) | How governed AI sessions are contained |
| [GOVERNED_TOOLS_REFERENCE.md](GOVERNED_TOOLS_REFERENCE.md) | The 52 governed tools, the human approval flow, the MCP bridge |
| [LEAK-GUARD.md](LEAK-GUARD.md) | What a governed AI session may publish (`git push`, gists, uploads) |
| [PLUGIN-GUIDE.md](PLUGIN-GUIDE.md) | Signed, sandboxed plugins |
| [PLUGIN-QUICKSTART.md](PLUGIN-QUICKSTART.md) | Your first open plugin, in any language |

## Security and internals
| Document | What it covers |
|---|---|
| [SECURITY_FEATURES_CATALOG.md](SECURITY_FEATURES_CATALOG.md) | Every security feature, with its module |
| [APODEIXIS_LANGUAGE_GUIDE.md](APODEIXIS_LANGUAGE_GUIDE.md) | The proof-checked mission language |
| [ASTRAXIS_ROBOTICS_GUIDE.md](ASTRAXIS_ROBOTICS_GUIDE.md) | Robotics movement and mission engine |
| [JAYCE_1000HZ_TICK_WHITEPAPER.md](JAYCE_1000HZ_TICK_WHITEPAPER.md) | The deterministic 1 kHz tick |
| [papers/](papers/) | Whitepapers (historical, kept as written) |

Design specifications, dated reports (red-team assessment, containment
validator results, the 2026-09-19 claims audit) and archived documents are
kept internally and are not published; reviewers can ask for them.

## Legal
- [../LICENSE](../LICENSE) — Jayce Automata Research Source License v1.1 (proprietary; official packages may ship as a dependency, plugins are open for development).
- [DEVBETA_PARTNER_LICENSE.md](DEVBETA_PARTNER_LICENSE.md) — dev-beta partner terms.
