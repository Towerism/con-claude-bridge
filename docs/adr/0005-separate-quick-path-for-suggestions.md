# ADR-0005: A separate, cheap path for suggestions and tab labels

**Status:** Accepted, 2026-10-16. Amended for issue #5: the quick model is a
setting, `quick_model`, with `haiku` as its default, and inline-suggestion
replies are cleaned up.

## Context

Con's inline shell suggestions and its tab labels use the "suggestion model",
which falls back to the main provider when no suggestion provider is set. Both
are one-shot calls with no tools. Suggestions ask for `max_tokens = 100`
(`AgentProvider::complete` in `provider.rs`) and tab labels for 2048
(`tab_summary.rs`), while Con's agent requests always carry tools.

These calls fire often: suggestions on pauses in typing, and labels on tab
changes. Running a full Claude Code agent for each would be slow and would
use up the Claude plan's limits.

## Decision

The bridge serves two model names, both listed in `/v1/models`:

| Model name | Profile | What runs |
|---|---|---|
| `claude-code` | `agent` | The full Claude Code run from [ADR-0002](0002-claude-code-as-a-step-in-cons-agent-loop.md) and [ADR-0003](0003-con-history-is-the-source-of-truth.md). |
| `claude-quick` | `quick` | `claude -p --model <quick_model> --tools "" --no-session-persistence --setting-sources ""`, one shot with no session, and Con's system prompt passed on as the system prompt. |

`quick_model` is `haiku` unless the user sets it
([`configuration.md`](../configuration.md#bridge-settings)).

The bridge sends a request to `quick` when it names `claude-quick` or carries no
`tools`. The second rule catches suggestions and tab labels even if the user
never sets `con.agent.suggestion_model.model`. The setup guide still sets it:

```ini
con.agent.suggestion_model.model = claude-quick
```

The guide also shows how to turn suggestions off,
`con.agent.suggestion_model.enabled = false`, for anyone who wants no
background traffic.

A quick run ends as soon as Con disconnects. Con cancels a stale suggestion
that way, by dropping the request, and the bridge kills the run at once, so old
runs don't hold the slots.
[ADR-0007](0007-run-lifecycle.md) sets the limits. Past them, a quick request
gets HTTP 429, which Con treats as "no suggestion".

The quick path loads no settings and no `CLAUDE.md`. `--bare` would do the
same job, but it refuses OAuth logins, which is how Claude Code is usually
signed in.

Con's source, read on 2026-10-16, decides how each reply is treated. Tab-label
replies pass through unchanged, because Con takes the first JSON object with a
`label` and an `icon` from the reply and ignores fences or prose around it.
Inline-suggestion replies get cleaned, because Con shows them as typed ghost
text. The bridge spots them by Con's system prompt for suggestions, "You are a
shell command completion assistant", and before replying:

- unwraps a reply that is a single line inside one pair of backticks or a
  code fence;
- turns any other reply with more than one line into an empty reply, as Con
  itself would by dropping it;
- turns `empty` or `(empty)` into an empty reply, which Con reads as no
  suggestion.

It never removes quotes or spaces. Con's prompt asks the model to preserve
them, and a suffix such as `"fix typo"` needs its quotes. If Con changes that
system prompt, replies stop being cleaned and reach Con as they would from
any other provider.

## Consequences

- Suggestions stay fast and cheap. They still go through the Claude Code login,
  so they count against the plan.
- The "no tools" rule relies on how Con shapes its requests today. If Con
  changes that, the explicit `claude-quick` setting keeps working.
