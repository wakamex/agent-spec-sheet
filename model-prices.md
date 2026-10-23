# What model-prices would gain from the Agent Spec Sheet, and what it would give

Question: would [model-prices](https://github.com/wakamex/model-prices), which prices the model names that harnesses report, benefit from the harness specification library proposed in this folder's README?

Answer: model-prices would use very little of it, and mostly contribute to it. It reads no harness files, credentials or session histories, so the README's starting facts mean nothing to it. But it holds harness facts of a different kind: which lab each harness's model names belong to, and how each harness decorates a model name before reporting it. Those facts are copied in three projects and have broken pricing twice this month. They belong in the specification as harness facts, while turning a name into a price stays in model-prices. model-prices also has a working version of the README's daily check, which the check could start from.

How this was judged: by listing what `src/model_prices/__init__.py` and `data/aliases.toml` encode about harnesses, by comparing that with what AOP ([agent-orchestration-process](https://github.com/wakamex/agent-orchestration-process)) and [clanker-analytics](https://github.com/wakamex/clanker-analytics) encode when they call model-prices, and by the naming failures model-prices fixed from 2026-10-13 to 2026-10-20.

## What model-prices encodes about harnesses

model-prices resolves a name from a log, such as `gemini-4-argon-low` or `claude-opus-5-5[1m]`, to a priced model. Part of that is knowledge of models, which is its job. Part is knowledge of harnesses:

| Harness fact | Where model-prices encodes it | Example |
| --- | --- | --- |
| Which lab a harness's model names belong to | `[providers]` in `data/aliases.toml`, for 9 harnesses | `agy = "google"`, `dsh = "deepseek"`, `zcode = "zai"` |
| Effort levels appended to the name | `_EFFORT_SUFFIX` | Antigravity's `gemini-4-argon-low`, `-medium`, `-high` |
| Context-window markers | `_CONTEXT_MARKER` | Claude Code's `[1m]` |
| Names an endpoint reports for another model | `_GROK_BUILD_ALIAS` | Grok Build's subscription endpoint reports `grok-4.6` as `grok-4.6-build` |
| Display names in place of ids | name normalization | `Gemini 3.5 Flash (High)` |
| A routing prefix before the model | `provider/model` splitting | opencode's `opencode/deepseek-v4.1-flash` |

## The same facts are copied in its consumers

The lab each harness's names belong to is written down three times:

| Project | Where | Harnesses |
| --- | --- | --- |
| model-prices | `data/aliases.toml`, `[providers]` | agy, claude, codex, dsh, gemini, grok, kimi, qwen, zcode |
| clanker-analytics | `TOOL_PROVIDERS` in `pricing.py` | Agy, Claude Code, Codex, DeepSeek Harness, Gemini, Grok, Zcode |
| AOP | `_repriced` in `migration.py` | agy, claude, dsh, grok |

They differ in one place. DeepSeek Harness can route a run to Anthropic, Google, OpenAI or xAI models. AOP asks its DeepSeek Harness adapter which provider a run used, and leaves a run through a provider the adapter does not recognize unpriced. model-prices and clanker-analytics map `dsh` and `DeepSeek Harness` to `deepseek`. That does not misprice a routed run, since the hint only sets which lab model-prices searches first, and a name such as `claude-opus-5-5` still resolves to its own lab, as no DeepSeek id is shared with those four labs. What differs is policy: whether a run through an unrecognized provider gets the model lab's price or none.

## Naming failures this month

Neither failure had anything to do with prices themselves. Both came from how a harness reports names:

- On 2026-10-13, AOP's `aop models --agent agy` listed `gemini-4-argon-low`, `-medium` and `-high` with no price on the day Google released Gemini 4 Argon. Here the suffix handling already worked, and only the model was new. It was found because AOP happened to list models, not by any check.
- On 2026-10-18, opencode's `opencode/deepseek-v4.1-flash` began resolving to SenseNova's free resale of the model instead of DeepSeek's paid one. The cause was models.dev renaming DeepSeek's lab name to the dated `deepseek-v4.1-flash-1017`. model-prices' tests caught it, because that exact harness name is a test case. A harness name without a test case would have been priced wrongly with no error.

The second shows what the specification could add: the names each harness actually reports, by version, as test cases for whoever resolves them.

## What the library could hold for model-prices' consumers

Per harness and version:

- The lab whose models it runs by default, and where a run records a different one, such as DeepSeek Harness's inference provider. That replaces the three provider maps above.
- How it decorates model names: effort suffixes, context markers, routing prefixes and display names, with examples taken from real logs.
- Where a run's model name is recorded. clanker-analytics found that Antigravity's transcripts leave it out, so the model comes from its conversation database.

What it should not hold: what a name costs, aliases between model ids, or which lab's price applies to a resold model. Those depend on price data that changes daily, which model-prices already publishes with dated history. A copy in the library would go stale within a day.

## What model-prices would contribute

- The harness facts above, which it holds now and would move into the specification. model-prices would keep its own provider hints, since it is a public package whose callers may not use the library, but would test them against the specification so the copies cannot drift.
- A check of unresolved names. The README's daily check compares file structure. A narrower check would take the model names seen in the last day's logs, which clanker-analytics already reads, and report any name that model-prices cannot resolve or resolves to a resale. That would have reported the Argon names when they first appeared in a log.
- A tested pattern for the check itself. `tools/watch_models_dev.py` in model-prices has run every minute since 2026-10-13 and has reported failures to an agent session since 2026-10-14. Its practices carry over directly:
  - it checks for change cheaply, with `git ls-remote`, before reading anything larger;
  - it reports each failure once, with a [Panetone](https://github.com/wakamex/panetone) message id derived from the event, so a retry never delivers twice;
  - it waits for a second failure when the only errors are timeouts or server errors;
  - its report carries the exact failing lines, so a fix needs no other lookup.

  It sent six reports by 2026-10-20. Three were real breakage, each fixed the same day: a harness name resolving to a reseller, Perplexity removing Sonar's prices, and a launch discount ending. The other three cleared without a change: one bad page response, and two runs during one provider's site outage. The second outage report came because the outage outlasted two runs.

## Costs for model-prices

- None at run time, if model-prices keeps its own hints and only tests against the specification. A dependency would make model-prices' data stale for callers of older library versions, as its snapshot is designed to avoid.
- The unresolved-name check needs log access, which model-prices does not have. It would run beside clanker-analytics or in the library's check, and call model-prices.

## Recommendation

Include each harness's model naming in the specification from the start: the default lab, the field that records a different one, the name decorations, and example names by version. Have model-prices, clanker-analytics and AOP test their provider maps against it. Add the unresolved-name check when the daily check is built, starting from model-prices' watcher. model-prices would be a contributor and a test target here, rarely a consumer.
