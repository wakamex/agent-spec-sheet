# What clanker-analytics would gain from the Agent Spec Sheet

Question: would [clanker-analytics](https://github.com/wakamex/clanker-analytics), which reads every harness's session logs to count tokens and cost, benefit from the harness specification library proposed in this folder's README?

Answer: yes, more than from the credential facts the README starts with. clanker-analytics reads no credentials, but it encodes where seven sources keep their session history and which fields carry token counts. Its parsers turn a missing field into zero, so a harness that moves its logs or renames a field makes usage disappear from the reports without an error. Two such gaps existed on this host on 2026-10-20, both since fixed: 31 archived Codex sessions (475 MB) were never read, and Claude Code transcripts under a second `CLAUDE_CONFIG_DIR` were never scanned. A specification of history locations, plus a daily check of the fields each consumer reads, would have reported both.

How this was judged: by listing what `src/clanker_analytics/main.py` hard-codes about each harness, and by checking those paths against this host's files on 2026-10-20.

## What clanker-analytics encodes about each harness

| Harness | Where it reads history | What it assumes about the format |
| --- | --- | --- |
| Claude Code | `~/.claude/projects/<launch dir with non-alphanumerics as ->/<session>.jsonl`, subagents as `subagents/agent-*.jsonl` | `message.usage.input_tokens`, `cache_creation_input_tokens`, `cache_read_input_tokens`, `cache_creation.ephemeral_1h_input_tokens`; `isSidechain` marks subagents; workflow journals sit beside subagent transcripts and are skipped |
| Codex | `~/.codex/sessions/**/rollout-*.jsonl` | `token_count` events with cumulative `info.total_token_usage` and per-request `info.last_token_usage`; `rate_limits.primary` and `secondary` with `window_minutes` and `plan_type`; replayed history in forked sessions |
| Gemini CLI | `~/.gemini/tmp/<hash>/chats/*.json` | per-message `tokens` and a `timestamp` whose type has changed once |
| Antigravity (agy) | `~/.gemini/antigravity-cli/brain/*/.system_generated/logs/transcript_full.jsonl` | the model is not in the transcript; it comes from `conversations/<session>.db` and `cache/conversation_metadata.json` |
| Zcode | `~/.zcode/cli/db/db.sqlite` and its write-ahead log | the `model_usage` table; usage-limit messages give reset times in UTC+8 |
| AOP | `~/.local/state/aop/results/*.json` | `aop-token-usage-v3` with `state_root` |
| All of the above in WSL | `//wsl$/<distro>/home/<user>/...` | the same layouts under each distribution's home |

AOP ([agent-orchestration-process](https://github.com/wakamex/agent-orchestration-process)) is the only source here that documents its format and versions it.

## Two gaps on this host on 2026-10-20, both since fixed

Both come from a location fact clanker-analytics does not have, not from a parser bug:

- Codex moves archived conversations to `~/.codex/archived_sessions/`. clanker-analytics scans only `~/.codex/sessions/`, so none of the 31 archived logs here, 475 MB in all, has ever been read. A log archived after a scan keeps its cached rows; one archived before any scan is missing from every report.
- Claude Code keeps transcripts in `$CLAUDE_CONFIG_DIR/projects/` when that variable is set. clanker-analytics scans only `~/.claude/projects/`, so a second account's transcripts are never read, and anyone who works mainly in a second account loses most of their usage. clanker-usage already records that account's quota, so pace compares a quota that counts this usage with local usage that does not.

The specification's home-directory entry, "the default path, the variable that moves it, and other variables that select an account", is exactly the fact the second gap needs. The first needs history locations listed as a set, with archived and live sessions both included.

## Format drift that reads as zero

The parsers use `coalesce(field, 0)` throughout, because a request legitimately has no cache write or no reasoning tokens. The same expression hides a renamed field: if Claude Code renamed `cache_read_input_tokens`, every report would show cache reads at 0% and a higher cost, with no warning. clanker-analytics has absorbed format details one at a time, each with its own fix and test:

- Gemini CLI's `timestamp` comes in more than one type, which broke date slicing until it was cast first.
- Codex forked sessions replay the parent's history, which counted the parent's tokens twice until fork replay windows were detected.
- Antigravity does not name the model in its transcripts, so models come from its conversation databases.
- Claude Code reports one-hour cache writes separately, under `cache_creation.ephemeral_1h_input_tokens`, and they cost more than five-minute writes, so a missed rename there misprices every main conversation.

The README's daily check compares file structure with the specification. For clanker-analytics the useful version is narrower: each consumer declares the fields it reads, such as `message.usage.cache_read_input_tokens` in Claude Code transcripts, and the check reports when today's logs stop containing a declared field. That turns a silent zero into a report on the day the harness changes.

## What clanker-analytics would use

- History locations per harness and version: default path, the variable that moves it, other account homes, and every directory that holds sessions, including archives.
- The fields each format carries, by harness version, for the daily check. clanker-analytics would keep its own DuckDB parsers; the library cannot parse for it, since the parsers are SQL over its own cache.
- Identity files only for labels: clanker-analytics reads `oauthAccount` from `.claude.json` for the organization and email of the Claude login. That location belongs in the specification; the file's credentials do not.

What it would not use: credential lookup order, update commands, and AOP's copy rules.

## Costs for clanker-analytics

- It is a public package on PyPI, so the library would have to be published and versioned too, and a wrong path in it would hide usage for every user at once. Today a wrong path hides it only in clanker-analytics, which is no better, but it is clanker-analytics' own release to fix.
- Finding every account home means scanning more directories. Claude Code has no list of its homes, so the library would need one from configuration or from clanker-usage's cached accounts in `~/.cache/clanker-usage/claude/`, which already names each account.

## Recommendation

The two gaps needed no library and are fixed in clanker-analytics as of 2026-10-21: it scans `~/.codex/archived_sessions/`, which added 20 sessions and 1.26 billion tokens to its reports here, and the `projects/` directory of `CLAUDE_CONFIG_DIR` and of every config directory listed in clanker-usage's configuration. If the library is built, include session history locations from the start rather than later as the README suggests, and add consumer-declared fields to the daily check. clanker-analytics would adopt the library for paths as soon as it exists, and register its field list with the check.
