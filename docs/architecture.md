# con-claude-bridge architecture

**Status:** Draft, revised 2026-10-16. Once reviewed, this document and the ADRs
in [`docs/adr/`](adr/) are the source of truth. A change of direction needs a
new or updated ADR.

This document explains how the bridge sends the Con terminal's AI panel through
Claude Code while keeping Con's harness in charge. Read it before changing the
bridge's protocol, run handling, process model or deployment.

## What the bridge does

con-claude-bridge is a small HTTP server on `127.0.0.1`. Con's built-in agent
sees it as an "OpenAI-compatible" model provider. Behind it, Claude Code
(`claude -p`, signed in with the login already on this machine) does the
thinking.

Con stays in charge of everything else. Claude Code gets exactly Con's tools
and none of its own. When Claude wants to run a command or edit a file, the
request shows up in Con as a normal tool call. Con asks for approval, runs it
in its panes and shows the result, as it would for any provider. The bridge
hands Con's tools to Claude Code as an MCP (Model Context Protocol) server, the
standard way Claude Code takes tools from outside. See
[ADR-0002](adr/0002-claude-code-as-a-step-in-cons-agent-loop.md).

## How one turn works

```mermaid
sequenceDiagram
    participant Con as Con agent loop
    participant B as con-claude-bridge
    participant CC as claude -p (built-in tools off)
    Con->>B: POST /chat/completions (history, new message, Con's tools)
    B->>CC: start run; MCP server /mcp/<run-id> lists Con's tools
    CC-->>B: text deltas
    B-->>Con: streamed content
    CC->>B: MCP tools/call terminal_exec (toolUseId=toolu_1)
    B-->>Con: tool_calls [id=toolu_1], finish_reason=tool_calls
    Note over CC,B: MCP call held open, progress every 30 s
    Con->>Con: show tool card, ask approval, run in pane
    Con->>B: POST /chat/completions (+ tool result for toolu_1)
    B->>CC: MCP result for toolu_1
    CC-->>B: more text, maybe more tool calls
    B-->>Con: streamed content, finish_reason=stop
    B->>CC: run ends
```

The diagram shows one turn with one tool call. Each tool step is a separate
HTTP request from Con. The same `claude` process stays alive across them,
waiting inside its MCP call. The bridge links each request to its process by
the tool call ID, which the bridge issued itself. See
[ADR-0003](adr/0003-con-history-is-the-source-of-truth.md).

## Facts the design depends on

From Con's source ([nowledge-co/con-terminal](https://github.com/nowledge-co/con-terminal),
commit `de2d7c7`, 2026-10-16; installed version `0.1.0-beta.135`):

| Fact | Where |
|---|---|
| The `openai-compatible` provider sends streaming `POST {base_url}/chat/completions` requests (OpenAI Chat Completions). | `crates/con-agent/src/provider.rs`, `build_openai_compatible_client_for` |
| A localhost `base_url` works without a key. A key set in config is still sent as `Authorization: Bearer`. | same function, `should_skip_default_env_api_key` |
| Con's agent loop sends its tools, runs returned `tool_calls` one at a time after approval, and sends all results in the next request by `tool_call_id`. A denial comes back as the result text `User denied tool execution` (or the user's reason). The approval prompt times out after 5 minutes. | `crates/con-agent/src/hook.rs`; rig-core 0.40.0 |
| Pressing stop only makes Con stop reading the stream. Con sends the bridge nothing when a turn ends early. | `consume_stream` in `provider.rs`; `hook.rs:207` |
| Con retries a whole turn, up to 3 more times, when the error text contains `429`, `500`, `502`, `503`, `timed out` or `connection reset`. | `is_transient_error` in `crates/con-core/src/harness.rs` |
| Across turns, Con sends its last 20 messages with system and tool messages removed, so only user and assistant text remains. | `Conversation::to_rig_history` in `crates/con-agent/src/conversation.rs` |
| Inline suggestions (`max_tokens = 100`) and tab labels (2048) are one-shot calls with no tools. They use the main provider unless a suggestion provider is set. Agent requests always carry tools. | `AgentProvider::complete` in `provider.rs`; `tab_summary.rs` |

From a spike against Claude Code 2.1.299 on 2026-10-16:

| Fact | Evidence |
|---|---|
| `claude -p --tools "" --strict-mcp-config --mcp-config <http server>` sees only that server's tools and calls them without built-in tools. | init event listed only `mcp__con__terminal_exec` |
| Each MCP `tools/call` carries `_meta["claudecode/toolUseId"]`, matching the `tool_use` block's ID. | server log |
| Claude Code aborts an MCP call after 300 s with no response or progress. Progress messages or a per-server `timeout` setting extend that. | tool result text: "sent no response or progress for 300s; aborting" |

## Components

The bridge is one Python process, split into modules with one job each.
Dependencies point downward only. The Claude runner knows nothing about HTTP or
Con.

| Module | Job |
|---|---|
| `server.py` | HTTP routes: `/v1/models`, `/v1/chat/completions`, `/mcp/<run-id>`, `/healthz`. Token, `Host` and content-type checks. Streaming responses. |
| `openai_protocol.py` | Parses Chat Completions requests and builds chunks, including `tool_calls`. No I/O. |
| `mcp_protocol.py` | The small part of MCP the bridge serves: `initialize`, `tools/list`, `tools/call`, progress messages. Converts Con's tool definitions to MCP tools. No I/O. |
| `runs.py` | The live-run registry and state machine ([ADR-0007](adr/0007-run-lifecycle.md)). Each run has its `claude` process, a state, one result mailbox per issued tool call ID, and an idle timer. |
| `routing.py` | Picks `agent` or `quick` ([ADR-0005](adr/0005-separate-quick-path-for-suggestions.md)), and for `agent`, a new run or an existing one. |
| `prompting.py` | Writes Con's history, and when replaying, this turn's tool calls and results, as the prompt for a new run. Passes Con's system prompt with `--append-system-prompt`, and reads the pane's working directory from its `<terminal_context>` block. |
| `claude_runner.py` | Starts `claude -p` in its own process group, reads `stream-json` events, and kills the group on cancel. |
| `config.py` | Settings from environment variables and an optional `config.toml`. |

## Request handling

1. `server.py` checks the token, `Host` and content type
   ([ADR-0006](adr/0006-localhost-service-with-token.md)).
2. `routing.py` sends requests with no `tools`, and requests for
   `claude-quick`, to the quick path: one cheap call, no tools, no run kept.
3. If the request ends with `tool` results whose IDs belong to a run in the
   `waiting` state, the bridge puts every one of them into that run's
   mailboxes and streams what Claude does next. ADR-0007 covers results that
   arrive before Claude's matching MCP call.
4. Otherwise it starts a new run. The prompt is Con's history plus anything
   from this turn, and the MCP server lists Con's tools from this request.
5. The response ends with `finish_reason = "tool_calls"` when Claude's
   assistant message has tool calls, or `"stop"` when its turn ends. On
   `"stop"` the process exits. Errors are mapped so that Con retries only when
   nothing has run yet (ADR-0007).

## Where state lives

| State | Location | Lifetime |
|---|---|---|
| Live runs | Bridge memory | One Con turn. Killed at turn end, when Con disconnects mid-stream, or after 6 minutes waiting for Con. |
| Bearer token | `$XDG_CONFIG_HOME/con-claude-bridge/token` (mode `0600`), and in `con.conf` | Until regenerated. |

Nothing else persists. Runs use `--no-session-persistence`.

## Decisions

- [ADR-0001](adr/0001-openai-chat-completions-as-the-con-interface.md): use Con's
  `openai-compatible` provider with Chat Completions. The Agent Client Protocol (ACP), which Zed uses, is the
  long-term path.
- [ADR-0002](adr/0002-claude-code-as-a-step-in-cons-agent-loop.md): Claude Code
  acts as one step of Con's agent loop, using only Con's tools.
- [ADR-0003](adr/0003-con-history-is-the-source-of-truth.md): Con's history is
  the source of truth; live runs are matched by tool call ID.
- [ADR-0004](adr/0004-python-standard-library-runtime.md): Python 3, standard
  library only at runtime.
- [ADR-0005](adr/0005-separate-quick-path-for-suggestions.md): a cheap path for
  suggestions and tab labels.
- [ADR-0006](adr/0006-localhost-service-with-token.md): a localhost-only systemd
  user service with a bearer token.
- [ADR-0007](adr/0007-run-lifecycle.md): run states, result mailboxes, limits
  and error mapping.

## Open questions

The first implementation issue answers these by recording real requests from
Con, then updates this document.

1. **How Con's system prompt reads to Claude.** Con's prompt names tools like
   `terminal_exec`. Claude sees them as `mcp__con__terminal_exec`. A short
   appended note may be needed to connect the two.
2. **Whether progress messages keep a call alive.** The spike showed the 300 s
   limit and the error text's advice. Sending progress hasn't been tested yet.
3. **Whether Claude Code defers MCP tools behind tool search** when Con
   registers about 30 tools. If it does, set `ENABLE_TOOL_SEARCH=false`.
4. **Whether `--setting-sources ""` also stops `~/.claude/CLAUDE.md` loading,
   and whether a project `CLAUDE.md` still loads** (ADR-0002, point 7).
5. **Whether tab-label requests on the quick path return bare JSON**
   (ADR-0005).
