# Agent Spec Sheet (ASS)

Question: would a shared description of each coding-agent harness help the projects that read harness files on this host? Here a harness is a coding-agent CLI such as Claude Code, Codex, Antigravity (agy), Gemini CLI, Cursor's agent, Devin, DeepSeek Harness (dsh), Grok, Hermes, OpenCode or Zcode. Its specification is the set of facts other programs need about it: where it keeps its home directory and which variable moves it, where its credentials and session history live and in what format, how it names models, how to read its version, and how it is updated.

Answer: two things, in this order of value. First, a daily check against the installed harnesses that runs each consumer's own probes: most harness changes that broke these projects from 2026-10-14 to 2026-10-23 were changes in behavior, which no description of files prevents, and a check would have reported most of them on the day. Second, a small data file of home directories and session history locations, which consumers test their own constants against. Wakterm reads the live sessions of seven harnesses, which meets this README's trigger for the data file, and the same home-directory fact has now been missing from three consumers. A runtime library stays deferred.

How this was judged: by counting which harness facts each consumer hard-codes, and by listing the harness changes those consumers absorbed from 2026-10-14 to 2026-10-23. The per-consumer findings for three of them are in [aop.md](aop.md), [clanker-analytics.md](clanker-analytics.md) and [model-prices.md](model-prices.md); Wakterm's are below.

## The consumers

- [Wakterm](https://github.com/wakamex/wakterm), the terminal multiplexer these agents run in, reads each harness's live session state every few hundred milliseconds to bind a pane to its session, tell busy from idle, and emit turn events. It also knows each harness's restore command and process names.
- [clanker-usage](https://github.com/wakamex/clanker-usage) reads credentials and quota.
- [sch](https://github.com/wakamex/search-clanker-history) reads session history.
- AOP ([agent-orchestration-process](https://github.com/wakamex/agent-orchestration-process)) seeds private harness homes for sandboxed runs and decides what each run may see.
- [clanker-analytics](https://github.com/wakamex/clanker-analytics) reads transcripts to count tokens and cost.
- [model-prices](https://github.com/wakamex/model-prices) resolves the model names harnesses report into prices.

[Panetone](https://github.com/wakamex/panetone), the message router between agents, reads no harness files; it consumes Wakterm's turn events, so a harness change that breaks Wakterm's observer breaks Panetone's delivery and replies with it. usage-burn and the website read only clanker-usage's output.

## Wakterm reads more harness files than any other consumer

`mux/src/agent.rs` and `mux/src/agent_event.rs` encode, per harness:

| Harness | What Wakterm reads |
| --- | --- |
| Claude Code | transcripts under `~/.claude/projects/` |
| Codex | rollouts under `~/.codex/sessions/` |
| Antigravity (agy) | `brain/<conversation>/.system_generated/logs/transcript.jsonl` and `transcript_full.jsonl` under `~/.gemini/antigravity-cli/` |
| Gemini CLI | session files under `~/.gemini/` |
| OpenCode and Zcode | OpenCode's SQLite schema, in each harness's own database |
| Devin | `~/.local/share/devin/cli/sessions.db` (tables `sessions` and `message_nodes`), and `session_locks/<name>.lock` to bind a pane to the session its process holds |

AOP's Devin knowledge covers headless runs and their trajectory export, which a live pane observer cannot use, so Wakterm shares only the path with it.

Each root has a Wakterm override (`WAKTERM_AGENT_CLAUDE_DIR`, `WAKTERM_AGENT_CODEX_DIR`, `WAKTERM_AGENT_AGY_DIR`, `WAKTERM_AGENT_GEMINI_DIR`). Without one, Wakterm reads `CLAUDE_CONFIG_DIR` and `CODEX_HOME` from each harness process's own environment, so a pane running a second account is found under that account's home.

## The same facts are written down several times

| Fact | Projects that encode it |
| --- | --- |
| Claude Code's config directory, `CLAUDE_CONFIG_DIR` or `~/.claude` | Wakterm, clanker-usage, sch, clanker-analytics, AOP |
| Codex's home, `CODEX_HOME` or `~/.codex`, and its session directories | Wakterm, clanker-usage, sch, clanker-analytics, AOP |
| Antigravity's directory `~/.gemini/antigravity-cli`, its transcripts and its token file | Wakterm, clanker-usage, clanker-analytics, AOP |
| Zcode's database `~/.zcode/cli/db/db.sqlite` | Wakterm, clanker-analytics |
| DeepSeek Harness's home, `DSH_HOME` or `~/.dsh`, and its credential store | clanker-usage, AOP |
| Devin's `credentials.toml` and session store | clanker-usage, AOP, Wakterm |
| Claude Code's `.credentials.json` | clanker-usage, AOP |
| Which lab each harness's model names belong to | model-prices, clanker-analytics, AOP |
| Gemini CLI's home, `GEMINI_CLI_HOME` | Wakterm, clanker-usage, clanker-analytics |

Most rows are one line of code each: a variable and a default path. The copies differ where one consumer learned something the others did not, and the first row shows the cost: `CLAUDE_CONFIG_DIR` was missing from AOP, clanker-analytics and Wakterm, each found separately.

## Harness changes from 2026-10-14 to 2026-10-23

| Change | Who it broke | Would a shared description have prevented it? |
| --- | --- | --- |
| dsh 0.2 rewrote its credential file into a `version: 1` layout, moved settings into profile patches, ranked the stored key above `DEEPSEEK_API_KEY`, and began uploading session logs by default | clanker-usage 0.4.0 failed; AOP missed keys kept only in the store and allowed the upload until v0.3.1, which targets dsh 0.2.1-rc.2 | No; it had to be noticed first. A shared copy would have spread the fix once it was noticed. |
| Hermes changed its installer and launcher | Every Hermes run through AOP failed for eleven days | No |
| Hermes 0.22 numbered session messages with integers | AOP discarded some resumed answers | No |
| Grok 1.0.60 added tools that reach MCP servers outside the `--tools` allowlist | A `--no-web` AOP run offered them | No; this is behavior, not files |
| agy stopped recording an end for a turn it stops on an on-screen error, such as quota or a safety filter | Wakterm showed an idle agy pane as busy | No; behavior |
| agy queues input submitted during a turn and sends it joined to the next input in one `USER_REQUEST` record | Wakterm never saw the queued message's own hash, so it could not confirm delivery | No; behavior |
| Codex's `plugin/list` response grew past 16 MiB (7,534 plugins, 17.7 MB) | A proxy size limit in front of Codex | No |
| Gemini CLI moved its OAuth client into its bundle and encrypted logins in `~/.gemini/gemini-credentials.json` | clanker-usage | No |
| Cursor's CLI moved its token to `~/.config/cursor/auth.json` | clanker-usage | No |
| Antigravity's quota service hid a model group from older agy versions; Anthropic's token endpoint moved and rejected urllib's User-Agent | clanker-usage | No |

## The daily check runs each consumer's own probes

A daily timer on this host reads each installed harness's version and, when one has changed since the last run, runs the probes the consumers already have or can add cheaply:

- clanker-usage's `check`, which reads each harness's credentials and quota against real files.
- AOP's native harness tests (`AOP_TEST_DSH_BIN`, `AOP_TEST_GROK_BIN`), which start the installed harness against a local server and check the tools a `--no-web` run offers, with no model quota. Hermes, agy and Cursor need such a test; the Hermes outage would have been caught by any test that starts a run.
- Wakterm's observer against real sessions. It already has one such test, `WAKTERM_TEST_ZCODE_DB`, which projects turn events from the real Zcode database. The agy rows above need a probe that drives a real turn to an error and to a queued input, and checks that Wakterm reports idle and delivery.
- A declared-field check for clanker-analytics: each parser lists the fields it reads, such as `message.usage.cache_read_input_tokens` in Claude Code transcripts, and the check reports when the last day's logs stop containing one. Its parsers turn a missing field into zero, so without this a renamed field shows up only as a lower number in a report.
- An unresolved-name check for model-prices: the model names seen in the last day's logs, reported when model-prices cannot resolve one or resolves it to a reseller.

Each probe stays in the consumer that owns it, so a consumer's release changes its probe and no other project. The runner reports failures the way model-prices' `tools/watch_models_dev.py` does: once per failure, with an id derived from the event, after a second failure when the only errors are timeouts, and with the exact failing lines.

The check compares fields that consumers read, not the full structure of every file. Harnesses add fields often, and a comparison of every key would report changes no consumer cares about.

## The data file covers homes and session history

Wakterm meets the trigger this README set for a shared description: a consumer that needs the facts of three or more harnesses, and it reads seven. The `CLAUDE_CONFIG_DIR` gap in three consumers shows what the copies cost, and sch showed a second cost: it recorded Antigravity as unreadable because it looked only at the editor's protobuf folder, while clanker-analytics had read agy's JSON Lines transcripts for two months. So the first shared artifact is [harnesses.toml](harnesses.toml), holding per harness:

- its home, written as `${VAR:-default}` so the variable that moves it and the default are one value, such as `${CLAUDE_CONFIG_DIR:-~/.claude}`;
- every file or database that holds sessions, relative to the home, including archives such as Codex's `archived_sessions/`;
- a one-line description of the format, for a person or agent deciding whether a harness can be read;
- which tools on this host read it.

The file replaces the hand-written history table in the agents' shared instructions, so there is one list of where history lives and who reads it.

Writing the file found copies that disagreed with it: Wakterm ignored `GEMINI_CLI_HOME`, and `XDG_DATA_HOME` for OpenCode, which sch honors, and clanker-analytics read only the older JSON Gemini CLI chats, so it counted no Gemini CLI usage from 2026-03-29 until it began reading the JSON Lines chats too.

Each consumer keeps its own constants and resolvers, vendors a pinned copy of the file, and has a test that its resolvers give the file's answers, including with the moving variable set, and that the harnesses it reads match its entries in `readers`. Picking up a change means updating the vendored copy, so no consumer's release waits on another's. A data file also suits a Rust consumer and Python consumers equally, where a library would have to be written for one language. Add a version range to an entry when a harness's layout differs by version, as DeepSeek Harness's credentials did between 0.1 and 0.2.

Publish the file at a stable URL, such as `waka.mx/ass`, as the copy that people, agents and vendoring consumers read. No consumer fetches it at run time: that would make offline use and every public package depend on the website, and one bad edit would break every consumer at once instead of failing one consumer's tests. The daily check compares each consumer's vendored copy with the published one and reports a consumer that has fallen behind.

What stays out of the file: credentials and their lookup order, the fields each transcript carries, model naming, update procedures, and AOP's copy rules and `--no-web` policy. Each of those has one or two consumers today, or is a security boundary that belongs to its consumer.

## A runtime library stays deferred

A library that consumers import at run time would add a published package that several public packages depend on, in two languages, with a release order of library before consumers. The data file gives the drift protection without that. Make a fact a runtime dependency only when consumers cannot reasonably keep their own copy, for example a list of every Claude account home, which no single variable gives and which clanker-usage, clanker-analytics, AOP and Wakterm would each need. Grow the data file instead when one of these happens:

- A harness change that the check reported is fixed in one consumer and later breaks another consumer that kept the old copy.
- A second consumer needs a fact the file leaves out, such as credential locations beyond clanker-usage and AOP, or transcript fields beyond clanker-analytics.

## Local fixes

- clanker-analytics reads `~/.codex/archived_sessions/` and every Claude Code home's transcripts. The fix is committed but not yet pushed.
- AOP seeds Claude runs from `CLAUDE_CONFIG_DIR` as of v0.4.0.
- Wakterm finds Claude Code, Codex, Gemini CLI and OpenCode sessions under the home each pane's process uses, reading `CLAUDE_CONFIG_DIR`, `CODEX_HOME`, `GEMINI_CLI_HOME` and `XDG_DATA_HOME` from that process's environment. The fix is committed but not yet pushed.

## A running service is out of scope

Every consumer runs on the same host and needs a lookup, not a network call. A daemon would add a process to keep alive and nothing a file cannot do.
