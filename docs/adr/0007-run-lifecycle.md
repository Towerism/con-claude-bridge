# ADR-0007: Run lifecycle, tool-result mailboxes and error mapping

**Status:** Proposed, 2026-10-16. Added after the adversarial review (findings
C1, W1, W2 and W5).

## Context

A run is one `claude` process serving one Con turn (ADR-0002). Between Con's
requests, no HTTP connection is open, and Con doesn't tell the bridge when a
turn ends early. That happens when:

- the user presses stop (Con stops reading the stream, `provider.rs` around
  line 2943, or the approval wait returns "Tool approval timed out or
  cancelled", `hook.rs:207`);
- rig's model-call budget runs out;
- Con quits.

Con also runs a turn's tool calls one at a time and sends all their results in
one request. Claude Code may make its MCP (Model Context Protocol) tool calls later than Con's results arrive,
for example when a user hook slows it down.

Con retries a whole turn, up to 3 more times, when an error message contains
`429`, `500`, `502`, `503`, `timed out` or `connection reset`
(`is_transient_error` in `crates/con-core/src/harness.rs`). A retry starts again
from the turn's first request, so a tool call that was already approved would be
proposed again.

## Decision

### States

Each run is in exactly one state:

| State | Meaning | Leaves when |
|---|---|---|
| `streaming` | An HTTP response from Con is open. Claude's output goes to it. | Claude's assistant message has `tool_use` blocks → `waiting`. Claude finishes → `done`. Con disconnects → `dead`. |
| `waiting` | Tool calls went to Con. No response is open. | A request brings results for this run → `streaming`. The idle timer fires → `dead`. Claude emits anything, or an MCP call ends on Claude's side → `desynced`. |
| `desynced` | Claude moved on without Con's results (an MCP timeout or a crash). | Immediately → `dead`, and its IDs are marked stale. |
| `done` / `dead` | The process group is killed and removed from the registry. | — |

### Mailboxes

- The bridge closes Con's response when the **complete assistant message**
  arrives in `stream-json`, not when the first MCP call arrives. At that moment
  it registers one mailbox per `tool_use` ID.
- A request with results routes **every** `tool` message after the last
  assistant `tool_calls` message into those mailboxes. Results may arrive before
  or after the matching MCP call. Whichever comes second completes the pair.
- If a request's IDs belong to a run in `waiting`, it goes to that run. It is
  never treated as a miss. If they match no run, or a stale one, it goes to
  replay (ADR-0003).

### Limits and timers

- The limit counts only `streaming` runs: at most 2 agent runs and 2 quick
  runs. A run in `waiting` doesn't count. An agent request never waits behind a
  run that is `waiting`.
- When a third run would start, the oldest `waiting` run is killed to make room.
  If every run is `streaming`, the request gets HTTP 409 (below).
- The idle timer for `waiting` is 6 minutes. That is Con's 5-minute approval
  timeout plus a 1-minute margin.
- There is no per-conversation limit. Con sends no conversation ID, and the
  bridge doesn't invent one (ADR-0003).

### Errors

| Situation | Response | Why |
|---|---|---|
| Claude Code's own API is rate-limited or overloaded before any output | 429 or 503 | Con's retry is correct here: nothing has run yet. |
| Run lost or desynced mid-turn, or the agent limit is reached | HTTP 409 with a message that avoids every retry trigger word | A retry would propose already-approved tools again. |
| Bad request (schema, auth, `Host`) | 400, 401 or 403 | Not retryable. |

Any error response kills the run it belongs to.

## Consequences

- Pressing stop twice during approvals no longer blocks new turns.
- An abandoned run lives at most 6 minutes.
- Tests must cover every state change in the table, results arriving before
  their MCP call, several results in one request, cancel during approval
  followed by a new turn, and the error mapping against Con's retry words.
