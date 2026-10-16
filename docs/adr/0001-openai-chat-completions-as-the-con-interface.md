# ADR-0001: Use Con's openai-compatible provider with Chat Completions

**Status:** Accepted, 2026-10-16

## Context

Con 0.1.0-beta.135 has no provider for Claude Code. It reaches models in two
ways:

- HTTP providers: Anthropic, OpenAI, an `openai-compatible` provider that takes
  a custom `base_url`, and others.
- `codex-local`: Con starts `codex app-server --listen stdio://` and talks to it
  over Codex's JSON-RPC (remote procedure call) protocol. Con finds the binary
  on `PATH`, or at the path in `CON_CODEX_BIN`.

## Decision

The bridge serves the OpenAI Chat Completions API: `/v1/chat/completions`,
streamed as Server-Sent Events (SSE), and `/v1/models`. Con's config points at
it:

```ini
con.agent.provider = openai-compatible
con.agent.providers.openai-compatible.base_url = "http://127.0.0.1:8765/v1"
con.agent.providers.openai-compatible.model = claude-code
con.agent.providers.openai-compatible.api_key = "<token from the bridge>"
```

`con-claude-bridge setup` prints these lines for the user to paste
([ADR-0006](0006-localhost-service-with-token.md)).

## Options considered

| Option | For | Against |
|---|---|---|
| Chat Completions through `openai-compatible` (chosen) | A stable, widely documented protocol. Con's code path already handles local hosts. Only two routes to serve. | The bridge has to translate between Con's tool calls and Claude Code ([ADR-0002](0002-claude-code-as-a-step-in-cons-agent-loop.md)). |
| Anthropic Messages through Con's `anthropic` provider with a custom `base_url` | Close to Claude's own message format. | A bigger protocol to copy: content blocks, thinking and beta headers. This project hasn't checked whether that provider accepts a local `base_url` with no key. Once Claude Code makes the model call, it offers nothing Chat Completions lacks. |
| A fake `codex app-server`, started through `CON_CODEX_BIN` | Con passes its tools as "dynamic tools", so Claude Code could drive Con's panes. | It copies an experimental, undocumented Codex protocol (`dynamicTools`, `thread/inject_items`) that changes between releases. Much more to build and keep working. |

## Consequences

- Con treats the bridge like any OpenAI-compatible host, and `/v1/models` lists
  the bridge's model names.
- Con's own tools reach Claude Code through the bridge (ADR-0002), so Con's tool
  cards and approval prompts keep working.
- The Agent Client Protocol (ACP) is the long-term path. Zed uses it so the
  editor can show Claude Code's own tool calls and its approval prompts for
  them.
  Con 0.1.0-beta.135 can't prompt an ACP agent. It only reads conversation
  history over ACP when handing a chat to another agent. On 2026-10-16 Con's
  issue tracker had no request for ACP agent support. If Con adds it, switch to
  an ACP adapter for Claude Code and retire this bridge.
