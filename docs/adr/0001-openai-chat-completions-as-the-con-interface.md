# ADR-0001: Use Con's openai-compatible provider with Chat Completions

**Status:** Proposed, 2026-10-16

## Context

Con 0.1.0-beta.135 has no provider for Claude Code. It reaches models in two
ways:

- HTTP providers: Anthropic, OpenAI, an `openai-compatible` provider with a
  custom `base_url`, and others.
- `codex-local`: Con starts `codex app-server --listen stdio://` and speaks Codex's
  JSON-RPC (remote procedure call) protocol to it. It finds the binary on `PATH` or through `CON_CODEX_BIN`.

## Decision

The bridge serves the OpenAI Chat Completions API (`/v1/chat/completions` with
Server-Sent Events (SSE) streaming, and `/v1/models`). Con is set to:

```ini
con.agent.provider = openai-compatible
con.agent.providers.openai-compatible.base_url = "http://127.0.0.1:8765/v1"
con.agent.providers.openai-compatible.model = claude-code
con.agent.providers.openai-compatible.api_key = "<token from the bridge>"
```

## Options considered

| Option | For | Against |
|---|---|---|
| **Chat Completions via `openai-compatible`** (chosen) | Stable, widely documented protocol. Con's code path is made for local hosts. Small surface: two routes. | Needs the bridge to translate between Con's tool calls and Claude Code (ADR-0002). |
| Anthropic Messages via `anthropic` with a custom `base_url` | Close to Claude's own message format. | A bigger protocol to copy (content blocks, thinking, beta headers), and it isn't checked whether Con's `anthropic` provider accepts a keyless local `base_url`. No benefit over Chat Completions once Claude Code handles the model call. |
| A fake `codex app-server` reached through `CON_CODEX_BIN` | Con passes its tools as "dynamic tools", so Claude Code could drive Con's panes. | Copies an experimental, undocumented Codex protocol (`dynamicTools`, `thread/inject_items`) that changes between releases. Much larger to build and to keep working. |

## Consequences

- Con treats the bridge like any OpenAI-compatible host. Model discovery
  (`/v1/models`) lists the bridge's profiles.
- Con's own tools reach Claude Code through the bridge (ADR-0002), so Con's
  tool cards and approval prompts work.
- **Long-term path:** Zed integrates Claude Code through the Agent Client
  Protocol (ACP), where the editor shows tool calls and approval prompts for
  Claude Code's own tools. Con 0.1.0-beta.135 has no ACP agent or provider support (it reads conversation
  history over ACP for agent handoff, but can't prompt an ACP agent), and its issue
  tracker has no request for it (checked 2026-10-16). If Con adds ACP, switch to
  an ACP adapter for Claude Code and retire this bridge.
