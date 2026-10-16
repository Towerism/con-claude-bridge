# ADR-0003: Con's history is the source of truth; live runs are matched by tool-call ID

**Status:** Proposed, 2026-10-16. Replaces the draft "map Con conversations to
Claude Code sessions by hashing the history".

## Context

Con sends two kinds of requests:

- **The first step of a turn:** after you send a message, Con sends its history
  (its last 20 messages, with system and tool messages then removed, so only
  user and assistant text remains,
  see `Conversation::to_rig_history` in `crates/con-agent/src/conversation.rs`)
  plus your new message.
- **Later steps in the same turn:** the same messages, plus the assistant's tool
  calls and Con's results, linked by `tool_call_id`.

The first draft hashed message text to find a Claude Code session to resume. A
small change in how Con resends text would silently lose context.

## Decision

- **Later steps match by ID only.** A request whose last message is a `tool`
  result is routed by its `tool_call_id` to the waiting run (ADR-0002,
  ADR-0007). IDs are
  values the bridge itself issued, so a match is exact and a miss is certain.
- **First steps always start a new run.** The prompt holds Con's history as a
  transcript, then the new message. The bridge doesn't try to resume an earlier
  Claude Code session. Claude sees what Con has, as any Con provider would.
- **A miss is never a guess.** If a tool result's ID matches no live run (after
  a restart or a 15-minute idle kill), the bridge starts a new run from the
  whole request. That includes the tool calls and results in this turn, written
  into the transcript. Claude continues from there.
- **Sessions aren't persisted.** Runs use `--no-session-persistence`, so the
  bridge keeps no session files and has no map to prune.

## Options considered

| Option | Why not chosen |
|---|---|
| Hash history text and resume the matching session (the first draft). | Fragile, as described above. |
| Hide a conversation ID in the reply text (an invisible marker). | Con stores, shows and may trim that text. It is another kind of guessing. |
| Write Claude Code session files from Con's history, then `--resume`. | Depends on Claude Code's private file format. |

## Consequences

- No state outlives a turn, apart from the live run in memory.
- Within a turn, Claude keeps its full context, including its thinking.
  Between turns, it gets Con's 20-message text history, the same as other
  providers.
- Each new turn starts a `claude` process and resends the history. Prompt
  caching may not help much, because Con's system prompt changes every turn
  with fresh terminal context. That is measured, not assumed.
