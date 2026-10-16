# ADR-0002: Claude Code acts as one step of Con's agent loop, using only Con's tools

**Status:** Accepted, 2026-10-16. Replaces the first draft, which ran
`claude -p` once per request and let Claude Code use its own tools. Amended
for issue #5: the `agent_model` setting, what happens when the self-test
can't finish, and the answers to the questions this ADR left open, tested on
2026-10-16.

## Context

Con's harness has to stay intact. Every action Claude takes should appear as a
Con tool call, wait for Con's approval prompt (when
`con.agent.auto_approve_tools = false`) and run in Con's panes. Zed gets this
for Claude Code through the Agent Client Protocol (ACP), which Con doesn't
support ([ADR-0001](0001-openai-chat-completions-as-the-con-interface.md)).

Con's agent loop is built on the rig library and makes up to
`con.agent.max_turns` model calls per turn. Each request carries Con's tool
definitions. When a reply contains `tool_calls`, Con asks for approval, runs
them, and sends the next request with each result tagged by its
`tool_call_id`. A denial comes back as a result with the text
`User denied tool execution` (`crates/con-agent/src/hook.rs`). Con waits up to
5 minutes for an approval.

## Decision

1. Claude Code gets only Con's tools. For each agent request, the bridge starts
   `claude -p` with every built-in tool turned off (`--tools ""`) and one
   Model Context Protocol (MCP) server, which is how Claude Code takes tools
   from outside. The bridge serves that server itself at
   `http://127.0.0.1:<port>/mcp/<run-id>`. It lists exactly the tools in Con's
   request, with the same names, descriptions and JSON schemas.
   `--allowedTools mcp__con` lets Claude Code call them without asking, since
   Con does the asking. The bridge strips the `mcp__con__` prefix from tool
   names on the way to Con and adds it back on the way in.
2. A tool call ends the HTTP response but not the run. When Claude's message
   ends, at its `message_stop` event in `stream-json`, the bridge sends Con
   every `tool_use` block of that message as `tool_calls`. Claude Code prints
   one `assistant` event per content block, so the bridge collects the blocks
   from every `assistant` event with the message's ID. It sends them using
   Claude's `tool_use` ID as the call ID, with `finish_reason = "tool_calls"`.
   Then it closes the response. Claude Code's MCP calls carry the same ID in
   `_meta["claudecode/toolUseId"]`. Those calls stay open and the `claude`
   process waits. [ADR-0007](0007-run-lifecycle.md) defines the run's states and
   how the bridge pairs results with calls.
3. Con's next request carries the result. It contains a `tool` message with
   `tool_call_id = <toolUseId>`. The bridge finds the waiting call by that ID
   and returns Con's result, unchanged, as the MCP result. A denial goes
   through the same way, whether it says `User denied tool execution` or gives
   the user's own reason, and Claude reads it like any other result. Claude's
   next output streams on the new HTTP response.
4. One step can hold several tool calls. Con runs them one at a time and sends
   all the results in its next request. Claude Code may make its MCP calls
   before or after those results arrive. ADR-0007's mailboxes pair each result
   with its call by ID either way.
5. The bridge keeps the waiting call alive. In a spike on 2026-10-16, Claude
   Code 2.1.299 aborted an MCP call after 300 seconds with no response or
   progress, and Con's approval can take that long. So the bridge answers each
   MCP call as a Server-Sent Events (SSE) stream and sends a
   `notifications/progress` message every 30 seconds, using the call's
   `progressToken`. It sends the stream's headers at once, because Claude
   Code gives up after 60 seconds without them when no per-server timeout is
   set. It also sets a per-server
   `"timeout": 900000` (15 minutes) in the `--mcp-config` it passes. Tests on
   2026-10-16 showed that progress messages alone, and the timeout alone, each
   kept a call alive for 330 seconds.
6. Text and thinking stream as they arrive. With `--include-partial-messages`,
   Claude's text deltas become `content` deltas, and its thinking deltas become
   `reasoning_content` deltas, which Con shows as thinking.
7. Which Claude Code configuration loads. Agent runs use `--setting-sources ""`
   and a settings file the bridge owns, so the user's hooks, plugins and
   permission rules don't load. They would otherwise act on Con's tool calls
   where Con can't see them. The user's choice of model doesn't load either, so
   the bridge's `agent_model` setting picks it, and Claude Code's default
   applies when that isn't set
   ([`configuration.md`](../configuration.md#bridge-settings)). The bridge also
   aims to leave out the global `~/.claude/CLAUDE.md`. The user wrote it for
   their own Claude Code sessions, with rules such as an issue pipeline and
   skills, and it would steer Con's agent. A `CLAUDE.md` in the project's
   working directory does apply.

   A test on 2026-10-16 showed that `--setting-sources ""` loads no
   `CLAUDE.md` at all, the project's included. So the bridge adds the
   project's files itself. It resolves the run's working directory to its
   real path, then reads `CLAUDE.md`, `.claude/CLAUDE.md` and
   `CLAUDE.local.md` from it and from each directory above it:

   - Inside the home directory, it stops below the home directory itself, and
     it never reads anything in `~/.claude`, so the global file never loads.
   - Outside the home directory, it stops below `/`.
   - It reads only regular files, never symlinks, devices or pipes, and at
     most 64 KiB per file and 256 KiB in all. Anything larger is cut, with a
     note in the journal.
   - It doesn't expand `@path` imports in those files.

   It adds the files, outermost first, to the appended system prompt, which
   reaches `claude` as a file through `--append-system-prompt-file`. The file
   names and their order follow Claude Code's documented behaviour; the test
   on 2026-10-16 covered only `CLAUDE.md`. These files come from whatever
   project the pane is in, which may be a freshly cloned repository, so
   their text is untrusted ([ADR-0006](0006-localhost-service-with-token.md)).

   The same appended prompt starts with a short note from the bridge, because
   Con's prompt and tool results name tools without the `mcp__con__` prefix:

   > Con's instructions and tool results name its tools without a prefix,
   > such as `terminal_exec`. Here each Con tool is `mcp__con__<name>`, so
   > `terminal_exec` means `mcp__con__terminal_exec`, and a `suggested_tool`
   > in a result means the same. Names such as `exec_visible_shell` or
   > `query_tmux` are pane capabilities that `list_panes` reports. A name
   > that also appears in your tool list, such as `probe_shell_context`, is a
   > tool as well.
   > Use only Con's tools: Con runs every action.
8. The run starts in the focused pane's directory, which the bridge reads from
   the `<terminal_context>` block of Con's system prompt. If that isn't an
   existing directory, the run starts in `$HOME`.

## Options considered

| Option | Why not chosen |
|---|---|
| Claude Code runs its own tools and the bridge returns only text (the first draft). | Con never sees the actions, so there are no tool cards and no approval in Con. This breaks Con's harness. |
| Claude Code runs its own tools and sends its permission prompts to Con (`--permission-prompt-tool`). | This is how Zed works, but Con can only show and approve its own tools. Claude's Bash or Edit calls would have no place in Con's interface. |
| A fake `codex app-server` behind `CON_CODEX_BIN`. | Same result, through Con's experimental Codex protocol, which is larger and less stable (ADR-0001). |

## Consequences

- Con shows every action, asks for approval and runs it in its panes, as it
  does for any provider.
- From Claude Code, Con gets the login and the model. The project's `CLAUDE.md`
  still applies, because the bridge adds it. Claude Code's built-in tools,
  skills, hooks and plugins don't run. Con's own skills still apply.
- A `claude` process stays alive between HTTP requests for the length of one
  Con turn. Con doesn't tell the bridge when a turn ends early, so ADR-0007's
  6-minute idle timer kills abandoned runs. If the run is gone when results
  arrive, for example after the bridge restarts, the replay in
  [ADR-0003](0003-con-history-is-the-source-of-truth.md) takes over.
- Claude Code's tool search stays off, because `--tools ""` removes the tool
  it needs, so all of Con's tools are visible from the start. The bridge also
  sets `ENABLE_TOOL_SEARCH=false`, in case a later Claude Code release changes
  that.
- Each tool step depends on the `_meta["claudecode/toolUseId"]` field. The
  bridge runs a self-test for it at start-up. If the field is missing, it
  refuses agent requests instead of guessing which call a result belongs to.
  The self-test may not finish, for example when the user service starts
  before the network is up. The bridge then checks the field on every MCP
  call instead, and starts refusing at the first call without it.
  [The architecture document](../architecture.md#start-up-and-shutdown)
  gives the details.
