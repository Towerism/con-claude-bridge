# ADR-0005: A separate, cheap path for suggestions and tab labels

**Status:** Proposed, 2026-10-16

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
| `claude-quick` | `quick` | `claude -p --model haiku --tools "" --no-session-persistence --setting-sources ""`, one shot with no session, and Con's system prompt passed on as the system prompt. |

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

Tab-label requests ask for bare JSON. The quick path loads no settings and no
`CLAUDE.md`, so nothing should wrap extra text around the JSON; the first
implementation issue checks this. `--bare` would do the same job, but it
refuses OAuth logins, which is how Claude Code is usually signed in.

## Consequences

- Suggestions stay fast and cheap. They still go through the Claude Code login,
  so they count against the plan.
- The "no tools" rule relies on how Con shapes its requests today. If Con
  changes that, the explicit `claude-quick` setting keeps working.
