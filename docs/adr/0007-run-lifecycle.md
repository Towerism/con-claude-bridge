# ADR-0007: Run lifecycle, tool-result mailboxes and error mapping

**Status:** Accepted, 2026-10-16. Added after the architecture review for issue
#1. That review found that abandoned runs could block new turns, that some run
behaviour was undefined, and that Con's retries could propose approved tools a
second time. [`docs/REVIEW-NOTE.md`](../REVIEW-NOTE.md) lists the findings.
Amended for issue #5: the error row for a bridge that can't start runs at
all, Claude Code's heartbeat events, Con's full list of retry trigger words,
and the error chunk.

## Context

A run is one `claude` process serving one Con turn
([ADR-0002](0002-claude-code-as-a-step-in-cons-agent-loop.md)). Between Con's
requests no HTTP connection is open, and Con doesn't tell the bridge when a turn
ends early. A turn ends early when:

- the user presses stop. Con stops reading the stream (`provider.rs`,
  around line 2943), or the approval wait returns "Tool approval timed out or
  cancelled" (`hook.rs:207`);
- rig, the library behind Con's agent loop, runs out of model calls for the
  turn;
- Con quits.

Con runs a turn's tool calls one at a time and sends all their results in one
request. Claude Code may make its Model Context Protocol (MCP) tool calls after
Con's results have already arrived, for example when a user hook slows it down.

Con retries a whole turn, up to 3 more times, when an error message contains
any of `429`, `500`, `502`, `503`, `rate_limit`, `Rate limit`,
`Internal Server Error`, `Bad Gateway`, `Service Unavailable`,
`overloaded`, `timed out`, `connection reset` or `Connection reset`
(`is_transient_error` in `crates/con-core/src/harness.rs`). They match as
plain substrings anywhere in the text, which includes the status code, the
response body and, for connection errors, the URL. A retry starts again
from the turn's first request, so Con would propose a tool call the user had
already approved a second time.

## Decision

### States

Each run is in exactly one state:

| State | Meaning | Leaves when |
|---|---|---|
| `streaming` | An HTTP response from Con is open, and Claude's output goes to it. | Claude's assistant message has `tool_use` blocks → `waiting`. Claude finishes → `done`. Con disconnects → `dead`. |
| `waiting` | Tool calls went to Con, and no response is open. | A request brings results for this run → `streaming`. The idle timer fires → `dead`. Claude emits anything other than a `tool_progress` heartbeat, or an MCP call ends on Claude's side → `desynced`. Claude Code prints a heartbeat every 30 seconds while an MCP call is open, so heartbeats don't count. |
| `desynced` | Claude moved on without Con's results, after an MCP timeout or a crash. | At once → `dead`, with its IDs marked stale. |
| `done` / `dead` | The bridge has killed the process group and removed the run from its registry. | — |

### Mailboxes

The bridge closes Con's response when Claude's complete assistant message
arrives in `stream-json`, not when the first MCP call arrives. At that point it
opens one mailbox per `tool_use` ID.

A request with results puts every `tool` message after the last assistant
`tool_calls` message into those mailboxes. A result may arrive before or after
its MCP call; whichever comes second completes the pair.

When a request's IDs belong to a `waiting` run, the request goes to that run
and never counts as a miss. When they match no run, or a stale one, the bridge
replays the turn ([ADR-0003](0003-con-history-is-the-source-of-truth.md)).

### Limits and timers

- Only `streaming` runs count toward the limit of 2 agent runs and 2 quick
  runs. A `waiting` run doesn't count, and an agent request never queues behind
  one.
- When a third run would start, the bridge kills the oldest `waiting` run to
  make room. If every run is `streaming`, the request gets HTTP 409 (see
  Errors).
- A `waiting` run dies after 6 minutes idle, which is Con's 5-minute approval
  timeout with a minute to spare.
- There is no per-conversation limit. Con sends no conversation ID, and the
  bridge doesn't invent one (ADR-0003).

### Errors

| Situation | Response | Why |
|---|---|---|
| Claude Code's own API is rate-limited or overloaded before any output | 429 or 503 | Nothing has run yet, so Con's retry is safe. |
| The run is lost or desynced mid-turn, or the agent limit is reached | HTTP 409, with a message that contains none of Con's retry trigger words | A retry would propose already-approved tools again. |
| The bridge can't start runs at all: `claude` isn't found, or the self-test or an MCP call showed that Claude Code no longer sends `_meta["claudecode/toolUseId"]`. This includes Con's next step for a run killed on that discovery. | 503, with a message that says which, and points to the journal | Nothing runs, and every retry is refused the same way, so Con's retries are harmless. |
| Bad request: schema, auth or `Host` | 400, 401 or 403 | Retrying can't help. |

Any error response kills the run it belongs to.

Once a stream has started, an error goes to Con as one Server-Sent Events
(SSE) data line. Its JSON has an `error` object and no `choices` key. Then
come `data: [DONE]` and a normal close:

```text
data: {"error": {"message": "The Claude Code run ended unexpectedly. See the bridge's journal.", "type": "bridge_error"}}

data: [DONE]
```

Con shows the JSON as it is, so the same wording rules apply as for a status
code and body. A message that must not cause a retry contains none of the
trigger words above, and none of the phrases Con rewrites: `MaxTurnError`,
`max turn`, `tool use` and `tool_use`. When such a message quotes Claude
Code's own error text, the bridge rewrites any of those words in the quote.

## Consequences

- Pressing stop twice during approvals no longer blocks new turns.
- An abandoned run lives at most 6 minutes.
- The tests must cover every state change in the table, results that arrive
  before their MCP call, several results in one request, a cancel during
  approval followed by a new turn, and each error response checked against
  Con's retry words.
