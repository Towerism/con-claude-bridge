# ADR-0003: Con's history is the source of truth, and the bridge finds live runs by tool call ID

**Status:** Proposed, 2026-10-16. Replaces the first draft, which hashed the
history to map Con conversations to Claude Code sessions.

## Context

Con sends the bridge two kinds of request. The first step of a turn carries
Con's history and the user's new message. The history is Con's last 20
messages with system and tool messages then removed, so only user and
assistant text remains (`Conversation::to_rig_history` in
`crates/con-agent/src/conversation.rs`). Each later step in the same turn
carries the same messages plus the assistant's tool calls and Con's results,
linked by `tool_call_id`.

The first draft hashed message text to find a Claude Code session to resume.
If Con resent the text with even a small change, the hash would miss and the
run would lose its context without any error.

## Decision

- A later step goes to its run by ID and nothing else. When a request's last
  message is a `tool` result, the bridge sends it to the waiting run that
  issued that `tool_call_id`
  ([ADR-0002](0002-claude-code-as-a-step-in-cons-agent-loop.md),
  [ADR-0007](0007-run-lifecycle.md)). The bridge issued those IDs itself, so a
  match is exact and a miss is certain.
- The first step of a turn always starts a new run. Its prompt is Con's
  history written as a transcript, followed by the new message. The bridge
  doesn't resume an earlier Claude Code session, so Claude sees what Con has,
  the same as any other Con provider.
- When a tool result's ID matches no live run, the bridge replays instead of
  guessing. This happens after a restart, or after ADR-0007's 6-minute idle
  timer has killed the run. The bridge starts a new run from the whole request,
  with this turn's tool calls and results written into the transcript, and
  Claude continues from there.
- Runs use `--no-session-persistence`. The bridge keeps no session files, so
  there is no map to prune.

## Options considered

| Option | Why not chosen |
|---|---|
| Hash the history text and resume the matching session (the first draft). | Breaks without warning when Con resends text slightly differently. |
| Hide a conversation ID in the reply text as an invisible marker. | Con stores, shows and may trim that text, so the marker can vanish. It is guessing by another route. |
| Write Claude Code session files from Con's history, then `--resume`. | Depends on Claude Code's private file format. |

## Consequences

- Nothing outlives a turn except the live run in memory.
- Within a turn, Claude keeps its full context, thinking included, but between
  turns it gets only Con's 20-message text history, like any other provider.
- Every new turn starts a `claude` process and resends the history. Prompt
  caching may not cut that cost by much, because Con's system prompt carries
  fresh terminal context and so changes on every turn. The first
  implementation issue measures it, as open question 6 in
  [the architecture document](../architecture.md#open-questions) says.
