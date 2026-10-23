# What AOP would gain from the Agent Spec Sheet

Question: would [AOP (agent-orchestration-process)](https://github.com/wakamex/agent-orchestration-process), which runs coding-agent harnesses inside private homes and a sandbox, benefit from the harness specification library proposed in this folder's README?

Answer: yes, mostly from the daily check rather than from the paths. AOP hard-codes more harness facts than any other consumer, and in the week to 2026-10-20 six harness changes broke AOP, weakened what it enforces, or left it unable to report something, without any error when they happened. One left every Hermes run failing for eleven days. Another let a `--no-web` Grok run offer the model tools that reach MCP servers. A daily check that starts each installed harness would have reported the three worst on the day they happened. The paths themselves AOP would take from the library gladly, but the rules that decide what a run may see must stay in AOP.

How this was judged: by listing what `src/agent_orchestration_process/runner.py` encodes about each harness, and by the harness changes AOP had to absorb from 2026-10-14 to 2026-10-20, each with the AOP commit that fixed it.

## What AOP encodes about each harness

AOP gives every run a private copy of the harness's home, seeded from the user's real home, and must know which files to copy, which to leave behind, and which single credential to project.

| Harness | Source home AOP reads | What else AOP encodes |
| --- | --- | --- |
| Claude Code | `$HOME/.claude` and `$HOME/.claude.json` | which config files a sealed run may receive; the flags that keep out MCP servers and synced skills |
| Codex | `CODEX_HOME` or `~/.codex` | seeded `rules` and `skills`; runtime files left out (`history.jsonl`, `session_index.jsonl`, `version.json`, `usage-limits.json` and others); the app-server protocol |
| Antigravity (agy) | `~/.gemini/antigravity-cli` | seeded OAuth files and nested config; minimum version 1.1.16 |
| Grok | `GROK_HOME` or `~/.grok` | 21 runtime names left out; the `--no-web` tool allowlist |
| Hermes | `HERMES_HOME`, `~/.hermes`, or its active profile | seed directories, runtime files, the launcher's `--print-runtime-command` interface, the dependency store under `~/.hermes/installs` and tools under `~/.hermes/tools` |
| DeepSeek Harness (dsh) | `DSH_HOME` or `~/.dsh` | the `version: 1` credential layout, the settings sections per provider route, plugin ids to disable, the targeted version, 0.2.1-rc.2 since v0.3.1, stated in `docs/harnesses.md` and verified by the native tests rather than checked in code |
| Cursor, Devin, OpenCode, Zcode | each harness's config and data directories | runtime directories left out, seeded auth files, Zcode's provider config |

That is eight seeding functions and a dozen allowlists and denylists in one 7,000-line module, each a copy of a harness fact.

## Harness changes that broke AOP this week

Each was found by hand, not by AOP:

- Hermes changed its installer on 2026-10-08: a new launcher with inline code, dependencies in a store under `~/.hermes/installs`. AOP recognized only the old launcher, so every Hermes run failed before starting until 2026-10-19 (`6a451da`). Nothing reported it, because nothing ran Hermes through AOP in between.
- Hermes 0.22 numbers session messages with integers. AOP accepted only string ids and discarded a resumed answer that repeated the previous one (`06bb917`).
- dsh 0.2 changed four facts at once: it calls DeepSeek's Messages API, its headless profile reads the key only from the environment, its credential file had already moved to a `version: 1` layout in 0.1.5-rc.3 (so AOP had been missing a key kept only in the store), and it uploads each session's log with every request by default (`ba0fb4f`).
- Grok 1.0.60 added `search_tool` and `use_tool`, which find and call connected MCP servers' tools, outside the `--tools` allowlist AOP passes under `--no-web`. A `--no-web` run offered them until 2026-10-19 (`8d72d6e`).
- Claude Code reports refusals through `model_refusal_*` events and `stop_reason: refusal`. AOP could not tell a refusal from an outage until it read them (`e96fef9`).
- clanker-usage moved Codex quota data out of `~/.codex/usage-limits.json`. AOP keeps that name on its leave-out list, because old installs still write it there.

All six surfaced while someone was doing something else: endchat asked for refusals, the usage workspace asked about the Codex file, and the Hermes, dsh and Grok problems turned up while Mihai was updating harnesses and asking whether AOP still worked with them.

## The `CLAUDE_CONFIG_DIR` gap is fixed in v0.4.0

AOP v0.3.1 seeds Claude from `$HOME/.claude` and ignores `CLAUDE_CONFIG_DIR`, so a user who selects a second Claude account with that variable gets a run seeded from the first account's home. AOP v0.4.0 fixes this: it seeds the run from `CLAUDE_CONFIG_DIR`, including that account's `.claude.json`, removes the variable from the run so Claude uses the private copy, and keeps it for `--native-state` runs. A sealed run with `CLAUDE_CONFIG_DIR` set received the selected account's login, matched by fingerprint. This is the same home-directory fact that clanker-analytics misses.

## The check matters more than the paths for AOP

Two of the six changes were not about paths at all: Grok's new tools and dsh's new default upload weakened what AOP enforces, and no file-structure comparison would catch them. AOP already has the right probes. Its opt-in native tests run the installed harness against a local server or before any model request, so they need no quota:

- `AOP_TEST_DSH_BIN` runs dsh against a local Messages API server and checks the tools a `--no-web` run offers.
- `AOP_TEST_GROK_BIN` checks the tools a `--no-web` Grok run announces.

The daily check should run these for every installed harness, alongside the README's structure comparison, and report a failure the same day. A harness without such a test yet (Hermes, agy, Cursor) is the next thing to add, and the Hermes outage shows why: a run that fails before starting is caught by any test that starts one.

## What AOP would use

- Home directories and the variables that move them or select an account, by harness version. AOP would read these instead of its own constants.
- Credential file locations and formats, such as dsh's `version: 1` layout with `refs`. AOP still reads the one credential it projects itself.
- Launch interfaces, such as Hermes's `--print-runtime-command`, and version ranges per AOP release, so AOP can refuse a version it was not tested with instead of failing inside the sandbox.
- Update procedures for this host: the Codex fork through its repository's `just install-local`, dsh through npm at one version, `devin update` in a terminal.

## What stays in AOP

- The copy rules. Which files a private home receives is AOP's security boundary, so it stays an allowlist in AOP's code, tested with AOP's releases. The library can say where a harness keeps its history; only AOP decides that a sealed run never sees it.
- The `--no-web` policy: tool allowlists, plugin ids to disable, network endpoints per harness. These are enforcement, and a wrong entry in a shared library would silently widen what a model can reach.
- Reading credentials, as the README already says.

## Costs for AOP

- AOP is on PyPI, so the library must be published and versioned, and released before AOP adopts a change.
- A wrong path in the library would make AOP seed a run from the wrong place. AOP must keep failing closed: a missing or unexpected home is an error, as it is today.

## Recommendation

Build the daily check first, and have it run AOP's native tests as well as the structure comparison. With a Hermes test added, that would have reported the Hermes launcher change, the dsh 0.2 change and Grok's new tools on the day they happened, and a resume step in the Hermes test would have caught its integer ids too. The refusal events and the Codex file were new needs rather than breakage, and no check would report them. The `CLAUDE_CONFIG_DIR` fix needed no library. When the library exists, AOP would adopt its home and credential locations and its version ranges, and keep its copy rules and `--no-web` policy in its own code.
