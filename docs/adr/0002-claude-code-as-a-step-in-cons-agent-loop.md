# ADR-0002: Claude Code acts as one step of Con's agent loop, using only Con's tools

**Status:** Proposed, 2026-10-16. Replaces the draft "run `claude -p` once per
request and let Claude Code use its own tools".

## Context

The bridge must keep Con's harness intact. Every action Claude takes should
show up as a Con tool call, go through Con's approval prompt (when
`con.agent.auto_approve_tools = false`), and run in Con's panes. Zed does the
same thing for Claude Code through the Agent Client Protocol (ACP), which Con
doesn't support (see ADR-0001).

Con's agent loop (rig, up to `con.agent.max_turns` model calls) sends its tool
definitions in each request. It runs any `tool_calls` it gets back, after
approval, and sends the next request with the results attached by
`tool_call_id`. A denied approval comes back as a tool result with the text
`User denied tool execution` (`crates/con-agent/src/hook.rs`). Con waits up to
5 minutes for an approval.

## Decision

1. **Claude Code gets only Con's tools.** For each `agent` request, the bridge
   starts `claude -p` with all built-in tools turned off (`--tools ""`) and one
   MCP (Model Context Protocol) server, which is how Claude Code gets tools
   from outside. The bridge serves it at `http://127.0.0.1:<port>/mcp/<run-id>`. The server lists exactly the tools in
   Con's request, with the same names, descriptions and JSON schemas.
   `--allowedTools mcp__con` lets Claude Code call them without its own
   prompt, because Con does the asking. Tool names go to Con without the
   `mcp__con__` prefix and come back with it added.
2. **A tool call ends the HTTP response but not the run.** When Claude's
   complete assistant message arrives in `stream-json`, the bridge sends every
   `tool_use` block in it to Con as `tool_calls`, using Claude's `tool_use` ID
   as the call ID, with `finish_reason = "tool_calls"`. Then it closes the
   response. Claude Code's MCP calls carry the same ID in
   `_meta["claudecode/toolUseId"]`. They stay open, and the `claude` process
   waits. ADR-0007 defines the run's states and how results are matched.
3. **Con's next request carries the result.** Con sends a `tool` message with
   `tool_call_id = <toolUseId>`. The bridge finds the waiting call by that ID
   and returns Con's result as the MCP result, unchanged. That includes Con's
   denial text (`User denied tool execution`, or the user's own reason), which
   Claude reads like any other result. Claude then continues, and its output
   streams on the new HTTP response.
4. **Several tool calls in one step.** Con runs them one at a time and sends
   all the results in its next request. Claude Code may make its MCP calls
   before or after those results arrive. ADR-0007's mailboxes pair each result
   with its call by ID either way.
5. **The waiting call is kept alive.** Claude Code aborts an MCP call after 300
   seconds with no response or progress (measured on 2.1.299 in a spike on
   2026-10-16). Con's approval can take that long. The bridge answers each MCP
   call as a Server-Sent Events stream and sends a `notifications/progress`
   message every 30 seconds, using the call's `progressToken`. It also sets a
   per-server `timeout` in the `--mcp-config` it passes.
6. **Text and thinking stream normally.** With `--include-partial-messages`,
   Claude's text deltas become `content` deltas and its thinking deltas become
   `reasoning_content` deltas, which Con shows as thinking.
7. **Which Claude Code configuration loads.** Agent runs use
   `--setting-sources ""` and a settings file owned by the bridge. So your
   Claude Code hooks, plugins and permission rules don't run inside Con's
   turns: they would act on Con's tool calls outside Con's view. Your global
   `~/.claude/CLAUDE.md` doesn't apply either, because it is written for Claude
   Code sessions (issue pipeline, skills) and would steer Con's agent. A
   project's `CLAUDE.md` in the working directory does apply. Whether
   `--setting-sources` also controls `CLAUDE.md` loading is checked in the
   first issue. If it doesn't, the bridge uses `--system-prompt-file` and adds
   the project file itself.
8. **Working directory.** The run starts in the focused pane's directory, read
   from the `<terminal_context>` block of Con's system prompt, if that is an
   existing directory. Otherwise it starts in `$HOME`.

## Options considered

| Option | Why not chosen |
|---|---|
| Claude Code runs its own tools; the bridge returns only text (the first draft). | Con never sees the actions. There's no approval in Con and no tool cards. This breaks Con's harness. |
| Claude Code runs its own tools, and its permission prompts go to Con (`--permission-prompt-tool`). | This is Zed's model, but Con can only show and approve its own tools. Claude's Bash or Edit calls would have no Con UI. |
| A fake `codex app-server` behind `CON_CODEX_BIN`. | Gives the same result through Con's experimental Codex protocol, which is larger and less stable (ADR-0001). |

## Consequences

- Con shows every action, asks for approval, and runs it in its panes, as it
  does for any provider.
- Claude Code adds its login, model and the project's `CLAUDE.md`. Its
  built-in tools, skills, hooks and plugins don't run. Con's own skills system
  still applies.
- A live `claude` process waits between HTTP requests during one Con turn. Con
  doesn't tell the bridge when a turn ends early, so ADR-0007's idle timer
  (6 minutes) reaps abandoned runs. If the run is gone when results arrive, for
  example after a bridge restart, ADR-0003's replay takes over.
- Claude Code may defer MCP tools behind its tool search when there are many.
  Con registers about 30 tools, so the first issue checks this, and sets
  `ENABLE_TOOL_SEARCH=false` if needed.
- Each tool step depends on the MCP `_meta["claudecode/toolUseId"]` field. The
  bridge checks for it at startup with a self-test and refuses `agent` requests
  if it is missing, instead of guessing.
