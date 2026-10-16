# ADR-0005: A separate, cheap path for suggestions and tab labels

**Status:** Proposed, 2026-10-16

## Context

Con's inline shell suggestions and tab labels use the "suggestion model". If no
suggestion provider is set, they go to the main provider. Both are one-shot
calls with no tools: suggestions use `max_tokens = 100`
(`AgentProvider::complete` in `provider.rs`) and tab labels use 2048
(`tab_summary.rs`). Con's agent requests always carry tools.
They fire often (suggestions on typing pauses, labels on tab changes). A full
Claude Code agent run for each one would be slow and would use up the Claude
plan's limits.

## Decision

The bridge serves two model names, listed in `/v1/models`:

| Model name | Profile | What runs |
|---|---|---|
| `claude-code` | `agent` | The full Claude Code run from ADR-0002 and ADR-0003. |
| `claude-quick` | `quick` | `claude -p --model haiku --tools "" --no-session-persistence --setting-sources ""`, one shot, no session, Con's system prompt passed through as the system prompt. |

A request is routed to `quick` if it names `claude-quick`, or if it has no
`tools`. The second rule catches suggestion and tab-label traffic even when
`con.agent.suggestion_model.model` isn't set.

The setup guide sets:

```ini
con.agent.suggestion_model.model = claude-quick
```

The guide also says how to turn suggestions off
(`con.agent.suggestion_model.enabled = false`) for anyone who wants no
background traffic.

Quick runs end when Con disconnects. Con cancels a stale suggestion by dropping
the request, and the bridge kills that run at once, so old runs don't hold the
slots. Limits are in ADR-0007. Past the limit, a quick request gets HTTP 429,
which Con treats as "no suggestion".

Tab-label requests ask for bare JSON. The quick path loads no settings and no
`CLAUDE.md`, so nothing adds text around it. The first issue checks this.
(`--bare` would do the same, but it refuses OAuth logins.)

## Consequences

- Suggestions stay fast and cheap, but they still use the Claude Code login, so
  they count against the plan.
- The "no tools" rule depends on Con's current request shapes. If Con changes
  them, the explicit `claude-quick` setting still works.
