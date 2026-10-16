# con-claude-bridge architecture

**Status:** Accepted, 2026-10-16. Completed for issue #5 with goals, module
ownership, threads, start-up, deployment, failure modes, observability and
testing. This document and the decision records (ADRs) in
[`docs/adr/`](adr/) are the source of truth, and a change of direction needs a
new or updated ADR.

Read this before changing the bridge's protocol, run handling, process model or
deployment. It explains how the bridge routes the Con terminal's AI panel
through Claude Code while Con's harness stays in charge. Two companion pages
cover the rest: [`configuration.md`](configuration.md) lists every setting and
file path, and [`implementation-plan.md`](implementation-plan.md) gives the
order in which the bridge gets built. The [glossary](#glossary) at the end
defines turn, step, run and the other terms used here.

## What the bridge does

con-claude-bridge is a small HTTP server on `127.0.0.1`. Con's built-in agent
sees it as an "OpenAI-compatible" model provider. Behind it, Claude Code does
the thinking, run as `claude -p` with the login already on the machine.

Everything else stays with Con. Claude Code gets Con's tools and none of its
own. When Claude wants to run a command or edit a file, the request appears in
Con as a normal tool call. Con asks for approval, runs it in a pane and shows
the result, as it would for any provider. The bridge hands Con's tools to
Claude Code as a Model Context Protocol (MCP) server, which is the standard way
Claude Code takes tools from outside. See
[ADR-0002](adr/0002-claude-code-as-a-step-in-cons-agent-loop.md).

## Goals and non-goals

The bridge succeeds when all of these hold:

- Every action Claude takes is a Con tool call, shown in Con's tool card and
  run only after Con's approval prompt, exactly as with any other provider.
- Nothing but the user's own Con can drive it: no other user's process, no
  web page and no other machine
  ([ADR-0006](adr/0006-localhost-service-with-token.md)). A process running
  as the same user can read the token, as it can read everything else the
  user owns.
- It installs with `pacman` and needs nothing at run time beyond Python and
  `claude` ([ADR-0004](adr/0004-python-standard-library-runtime.md),
  [ADR-0008](adr/0008-aur-release-package.md)).
- An abandoned turn costs at most one idle `claude` process for 6 minutes
  ([ADR-0007](adr/0007-run-lifecycle.md)), and background suggestions use a
  cheap model ([ADR-0005](adr/0005-separate-quick-path-for-suggestions.md)).

These are out of scope:

- Claude Code's built-in tools, skills, hooks and plugins. Con's tools replace
  them.
- Carrying Claude's context from one Con turn to the next. Between turns
  Claude sees only Con's history
  ([ADR-0003](adr/0003-con-history-is-the-source-of-truth.md)).
- Serving more than one user, or any client other than Con.
- Platforms other than Linux with a systemd user session. Arch Linux is the
  tested one.
- Replacing an Agent Client Protocol (ACP) integration. If Con gains ACP agent
  support, the bridge retires
  ([ADR-0001](adr/0001-openai-chat-completions-as-the-con-interface.md)).

## Who talks to whom

The diagram shows each party the bridge deals with and which ones it trusts.
The dashed inner box is the trust boundary: only Con, holding the bearer
token, and the bridge's own `claude` runs, holding a per-run token, get past
it.

```mermaid
flowchart LR
    subgraph user["User's machine, user's account"]
        Con["Con terminal<br/>(agent loop, tools, approval)"]
        subgraph trusted["Trust boundary: token required"]
            Bridge["con-claude-bridge<br/>127.0.0.1:8765"]
            Claude["claude -p runs<br/>(no built-in tools)"]
        end
        Other["Other local processes<br/>and browser pages"]
        Files["Project files"]
    end
    API["Anthropic API"]

    Con -- "Chat Completions + bearer token" --> Bridge
    Bridge -- "starts, reads stream-json, kills" --> Claude
    Claude -- "MCP over HTTP + per-run token" --> Bridge
    Claude -- "model calls with the user's login" --> API
    Con -- "runs approved tools in panes" --> Files
    Bridge -. "reads the project's CLAUDE.md files" .-> Files
    Other -. "rejected: no token, wrong Host<br/>or content type" .-> Bridge
    style trusted stroke-dasharray: 5 5
```

Claude Code never reads or writes the project itself. Every change to the
files goes through a Con tool call that the user approved, and the bridge
reads only the project's `CLAUDE.md` files, as ADR-0002 point 7 describes.

## How one turn works

The diagram follows one turn in which Claude makes a single tool call.

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

Each tool step is a separate HTTP request from Con, but the same `claude`
process stays alive across them, waiting inside its MCP call. The bridge finds
the right process for each request by the tool call ID, which it issued
itself. See [ADR-0003](adr/0003-con-history-is-the-source-of-truth.md).

## Facts the design depends on

From Con's source ([nowledge-co/con-terminal](https://github.com/nowledge-co/con-terminal),
commit `de2d7c7`, 2026-10-16), with version `0.1.0-beta.135` installed:

| Fact | Where |
|---|---|
| The `openai-compatible` provider sends streaming `POST {base_url}/chat/completions` requests in the OpenAI Chat Completions format. | `crates/con-agent/src/provider.rs`, `build_openai_compatible_client_for` |
| A localhost `base_url` works without a key. A key set in the config is still sent as `Authorization: Bearer`. | Same function, `should_skip_default_env_api_key` |
| Con's agent loop sends its tools, runs returned `tool_calls` one at a time after approval, and sends all the results in the next request by `tool_call_id`. A denial comes back as the result text `User denied tool execution`, or the user's reason. The approval prompt times out after 5 minutes. | `crates/con-agent/src/hook.rs`; rig-core 0.40.0 |
| Pressing stop only makes Con stop reading the stream. Con sends the bridge nothing when a turn ends early. | `consume_stream` in `provider.rs`; `hook.rs:207` |
| Con retries a whole turn, up to 3 more times, when the error text contains any of `429`, `500`, `502`, `503`, `rate_limit`, `Rate limit`, `Internal Server Error`, `Bad Gateway`, `Service Unavailable`, `overloaded`, `timed out`, `connection reset` or `Connection reset`. These are plain substrings, so `15003` matches too. | `is_transient_error`, `crates/con-core/src/harness.rs:1510-1520` |
| The error text Con checks is built from the whole response. A non-200 status before streaming gives `Invalid status code 409 Conflict with message: <body>`, so it holds the status number, its reason and the raw body. An SSE data line whose JSON has an `error` object and no `choices` key ends the stream with an error that quotes the raw JSON. | rig-core 0.40.0, `http_client/mod.rs:20-21, 257-258`, `providers/internal/openai_chat_completions_compatible.rs:23-42` |
| A connection error quotes the request URL, port included: `error sending request for url (http://127.0.0.1:<port>/v1/chat/completions)`. | rig-core 0.40.0, `http_client/mod.rs:256` |
| Con also rewrites some error text. `MaxTurnError` or `max turn` turns the error into a successful "reached the turn limit" reply, and `tool use` or `tool_use` becomes "This model does not support tool use". | `consume_stream`, `provider.rs:3045-3057` |
| A stream that closes cleanly counts as a complete, successful reply, with or without `finish_reason` or `data: [DONE]`. Con waits for the body to end before it finishes the turn. | rig-core 0.40.0, `sse.rs:298`; `provider.rs:3029-3034` |
| Across turns, Con sends its last 20 messages with system and tool messages removed, so only user and assistant text remains. | `Conversation::to_rig_history` in `crates/con-agent/src/conversation.rs` |
| Inline suggestions (`max_tokens = 100`) and tab labels (2048) are one-shot calls with no tools. They use the main provider unless a suggestion provider is set. Agent requests always carry tools. | `AgentProvider::complete` in `provider.rs`; `tab_summary.rs` |
| Con's agent system prompt names its tools without a prefix about 150 times, for example "→ terminal_exec", and tool results carry `suggested_tool` with a bare name. It also uses pane capability labels, such as `exec_visible_shell`, that look like tool names. Con's own Codex integration appends a note about which tools to use. | `context.rs:2653-2790`, `playbooks.rs`, `tools.rs:2035-2083`, `codex_local.rs:29-32` |
| Con parses tab labels leniently: it takes the first `{...}` in the reply that has a `label` and an `icon`, past code fences or prose, and falls back to its own label when that fails. Inline suggestions are not cleaned: a reply with a newline is dropped, and backticks, quotes or a literal `empty` show as ghost text. | `parse_summary_json`, `tab_summary.rs:625-669`; `suggestions.rs:260-266` |

From experiments against Claude Code 2.1.299 on 2026-10-16, with `--model haiku`
and a test MCP server written for the purpose:

| Fact | Evidence |
|---|---|
| `claude -p --tools "" --strict-mcp-config --mcp-config <http server>` sees only that server's tools and calls them with no built-in tools. | The init event listed only `mcp__con__terminal_exec`. |
| Each MCP `tools/call` carries `_meta["claudecode/toolUseId"]`, matching the `tool_use` block's ID, and a `progressToken`. | Server log: `"_meta":{"claudecode/toolUseId":"toolu_016V…","progressToken":2}` |
| Without a per-server `timeout`, an MCP call fails after 60 s if the server hasn't sent response headers, and after 300 s with no response or progress. A `notifications/progress` message every 30 s kept a call alive for 330 s, and so did a per-server `"timeout": 900000` on its own. | Failures at 60 s (`TimeoutError`) and 300 s ("sent no response or progress for 300s; aborting"); both fixes returned the result at 330 s. |
| While an MCP call is open, `stream-json` carries a `tool_progress` heartbeat event every 30 s, whatever the server sends. | `{"type":"tool_progress",…,"heartbeat":true}` in every long run |
| Claude Code prints one `assistant` event per content block, all with the same `message.id`, and each comes before that block's `content_block_stop`. The message ends with `message_delta`, carrying `stop_reason`, then `message_stop`, then a `rate_limit_event`. Only after those do the heartbeats start. | Event order in the 330 s progress run: `assistant` (tool_use), `content_block_stop`, `message_delta`, `message_stop`, `rate_limit_event`, 10 × `tool_progress` |
| With `--tools ""`, tool search is off, so all 30 test tools loaded in full and Claude called one directly. `ENABLE_TOOL_SEARCH=false` changed nothing. | Debug log: "Tool search disabled: ToolSearchTool is not available" |
| `--setting-sources ""` loads no `CLAUDE.md` at all, neither `~/.claude/CLAUDE.md` nor the project's. The default sources load both. | A canary in the project's `CLAUDE.md` and a heading in the global one, asked about with each setting |
| Changing any part of the appended system prompt re-caches all of it: about 55% of a turn's input came from cache. With a fixed appended prompt and the changing context in the user message, 95% did. | `cache_read_input_tokens` against `cache_creation_input_tokens` in the `result` events |

## Modules

The bridge is one Python package, `con_claude_bridge`, split into modules that
each do one job. Every behaviour named in the ADRs has exactly one owner here.

| Module | Owns |
|---|---|
| `cli.py` | The `con-claude-bridge` entry point. With no subcommand it serves: it runs the start-up sequence below, using `claude_runner.py` to find `claude`, and then the HTTP server. `con-claude-bridge setup` creates the token if it is missing, prints the `con.conf` lines to paste and sets `~/.config/con/con.conf` to mode `0600` ([ADR-0006](adr/0006-localhost-service-with-token.md)). `__main__.py` calls it, so `python -m con_claude_bridge` works too. |
| `server.py` | HTTP routes: `/v1/models`, `/v1/chat/completions`, `/mcp/<run-id>` and `/healthz`. Streams Server-Sent Events (SSE) responses to Con and to `claude`, using HTTP/1.1 chunked encoding and `Connection: close`. The final empty chunk marks a complete reply, and Con finishes the turn when the socket closes. If the bridge dies mid-stream, the missing final chunk makes Con report an error, without a retry trigger word, instead of taking the cut-off reply as complete. Sends an MCP call's response headers at once, before Claude Code's 60-second header timeout. While an MCP `tools/call` waits for its result, the handler sends a progress message every 30 seconds ([ADR-0002](adr/0002-claude-code-as-a-step-in-cons-agent-loop.md), point 5). Detects a client that has gone away. |
| `auth.py` | Creates and reads the bearer token file, compares tokens in constant time with `hmac.compare_digest`, checks the `Host` header and content type, and issues each run's MCP token (ADR-0006). |
| `openai_protocol.py` | Parses Chat Completions requests and builds response chunks, including `tool_calls` and `reasoning_content`. No I/O. |
| `mcp_protocol.py` | The small part of MCP the bridge serves: `initialize`, `tools/list`, `tools/call` and `notifications/progress`. Converts Con's tool definitions to MCP tools, and adds or strips the `mcp__con__` prefix. No I/O. |
| `routing.py` | Decides what a request is: a `quick` request, results for a `waiting` run, a replay, or a new agent run ([ADR-0005](adr/0005-separate-quick-path-for-suggestions.md), [ADR-0003](adr/0003-con-history-is-the-source-of-truth.md)). Marks inline-suggestion requests, whose replies get cleaned up. Takes the parsed request and a lookup from the registry, and returns the decision. No I/O. |
| `runs.py` | The registry of live runs and their state machine ([ADR-0007](adr/0007-run-lifecycle.md)): states, result mailboxes, the agent and quick limits, eviction of the oldest `waiting` run, and the idle reaper. It checks every MCP call for `_meta["claudecode/toolUseId"]`, and holds the self-test status that `/healthz` reports and that decides whether agent requests are refused. Each run owns its `claude` process, its MCP token and its temp files. |
| `prompting.py` | Builds the two texts a run gets ([ADR-0002](adr/0002-claude-code-as-a-step-in-cons-agent-loop.md), point 7). The appended system prompt holds the bridge's tool-name note, Con's system prompt without its `<terminal_context>` block, and the project's `CLAUDE.md` files. The prompt starts with that `<terminal_context>` block, then Con's history as a transcript, adding this turn's tool calls and results when replaying. Reads the pane's working directory from the same block. Gets the `CLAUDE.md` text from `claude_runner.py` by way of `runs.py`. No I/O. |
| `claude_runner.py` | Finds the `claude` binary (ADR-0006) before each run, builds its command line, and starts it in its own process group. Writes the prompt to its standard input, and passes the system prompt and the `--mcp-config` as mode `0600` temp files, never as arguments: an argument is limited to 128 KiB, and any local user can read it in `/proc/<pid>/cmdline`. Parses `stream-json` events from its output, drains its standard error, and kills the group on cancel. Finds and reads the project's `CLAUDE.md` files, with ADR-0002's limits. Knows nothing about HTTP or Con. |
| `selftest.py` | Runs the start-up self-test for `_meta["claudecode/toolUseId"]` ([ADR-0002](adr/0002-claude-code-as-a-step-in-cons-agent-loop.md)) and records its result in the registry. [Start-up and shutdown](#start-up-and-shutdown) describes it. |
| `errors.py` | Maps each failure to the HTTP status and message in ADR-0007's error table, builds the error chunk that ends a stream, and removes Con's retry trigger words and rewrite phrases from any message that must not cause a retry, including Claude Code error text quoted in it. No I/O. |
| `config.py` | Reads settings from environment variables and `config.toml`. Environment variables win. [`configuration.md`](configuration.md) lists them. |
| `log.py` | Sets up logging to standard error, which systemd sends to the journal, and keeps prompt and reply text out of the log unless `log_content` is on ([Observability](#observability)). |

Dependencies point one way only, from the top of the diagram down. The pure
modules marked "No I/O" can be tested without a network or a process, and
`claude_runner.py` can't import anything about HTTP or Con.

```mermaid
flowchart TD
    cli --> server
    cli --> selftest
    cli --> auth
    cli --> log
    cli --> claude_runner
    selftest --> runs
    server --> routing
    server --> runs
    server --> errors
    server --> auth
    server --> openai_protocol
    server --> mcp_protocol
    routing --> openai_protocol
    runs --> claude_runner
    runs --> prompting
    runs --> auth
    claude_runner --> config
    auth --> config
    log --> config
```

`config.py` is a leaf that any module may import. The diagram shows only the
imports that matter for layering.

### Repository layout

```text
pyproject.toml                 project metadata, setuptools build, tool settings
src/con_claude_bridge/         the modules above, __init__.py, __main__.py
tests/unit/                    pure modules and the run state machine
tests/integration/             a real server on a free port, fake claude
tests/fake_claude/fake_claude  the fake claude executable (see Testing)
tests/fixtures/                recorded Con requests and Claude Code stream-json
packaging/systemd/con-claude-bridge.service
packaging/aur/                 PKGBUILD, .install file, LICENSE (ADR-0008)
docs/                          this document, the ADRs and the companion pages
```

## How the bridge handles a request

1. `server.py` checks the token, the `Host` header and the content type
   (ADR-0006).
2. `routing.py` sends requests with no `tools`, and requests for
   `claude-quick`, to the quick path: one cheap call with no tools, and no run
   kept afterwards.
3. If the request ends with `tool` results whose IDs belong to a run in the
   `waiting` state, the bridge puts every one of them in that run's mailboxes
   and streams what Claude does next. ADR-0007 covers results that arrive
   before Claude's matching MCP call.
4. Otherwise it starts a new run. The prompt is Con's terminal context, then
   Con's history as a transcript with anything from this turn, then the new
   message. The MCP server lists the tools from this request.
5. The response ends with `finish_reason = "tool_calls"` when Claude's
   assistant message has tool calls, or `"stop"` when its turn ends, and on
   `"stop"` the process exits. The bridge chooses error codes and wording so
   that Con retries only when nothing has run yet (ADR-0007).

## Threads and locking

The bridge uses `http.server.ThreadingHTTPServer` (ADR-0004), so each HTTP
request gets its own thread. That covers Con's requests and every MCP request
from `claude`. A few more threads run alongside:

| Thread | Count | Job |
|---|---|---|
| Request handler | One per open HTTP request | Serves the request. For Con's chat request it streams the run's events until the response ends. For an MCP `tools/call` it waits on the run's mailbox, waking every 30 seconds to send progress. |
| Run readers | Two per live run | One reads `claude`'s standard output line by line, parses each `stream-json` event and hands it to the run. The other drains standard error into a 64 KiB buffer, so `claude` never blocks on a full pipe. |
| Idle reaper | One | Sleeps until the earliest `waiting` run's 6-minute deadline, or until a new deadline is set, and kills the runs that are due. |
| Self-test | One, at start-up | Runs the self-test without holding up the server. |

One lock in `runs.py` guards the registry and every run's state. State
changes, opening mailboxes and counting runs against the limits all happen
while holding it, so two requests can't both take the last free slot.

A tool call's mailbox opens at whichever comes first: the run reader parsing
Claude's assistant message, or Claude's MCP call for that ID. The two arrive
by different routes, standard output and HTTP, so either can be first. The
later one finds the mailbox already open and uses it. Nothing holds
the lock while writing to a socket or waiting on a process. Each run passes
events to the open Con response through its own queue, and each mailbox is a
one-slot queue that its MCP handler thread waits on.

A handler streaming to Con notices that Con has gone away when a write fails,
or when a write blocks for 60 seconds because Con stopped reading. While it
waits for the next event, it also checks the socket for end-of-file every
second. Without the check, a stale suggestion would keep its quick
run until Claude next produced output.

## Start-up and shutdown

When the service starts, `cli.py`:

1. Reads the settings (`config.py`). If `config.toml` isn't valid, it logs
   why and exits with status 1.
2. Finds `claude` (ADR-0006) and logs its path and `claude --version`. If it
   finds nothing, it logs the paths it checked and keeps serving, so
   `/healthz` can report the problem. The bridge looks `claude` up again
   before each run and logs when the path changes, so installing or moving
   `claude` needs no restart.
3. Reads the bearer token, creating it if it is missing.
4. Binds `127.0.0.1` on the configured port. If the port is taken, it logs the
   port and exits with status 1, and systemd shows the unit as failed.
5. Starts the idle reaper and the HTTP server.
6. Starts the self-test in the background.

The self-test is a run of its own kind, `selftest`, outside ADR-0007's states
and limits. It uses the quick model, and its MCP server lists a single tool,
`bridge_selftest`, which Claude is asked to call. It costs one small call to
the quick model each time the service starts. It has three outcomes:

- **Passed:** the MCP call's `_meta["claudecode/toolUseId"]` matches the
  `tool_use` ID in `stream-json`.
- **Failed:** the MCP call arrived without the field, or with an ID that
  doesn't match. Nothing else counts as failed.
- **Couldn't finish:** anything else, such as no MCP call within 60 seconds,
  Claude not calling the tool, or `claude` exiting with an error.

| Self-test result | What the bridge does |
|---|---|
| Passed | Serves agent requests. |
| Failed | Refuses agent requests with 503 until restarted, as ADR-0002 requires, and logs why. Quick requests still work. |
| Couldn't finish, for example because Claude Code isn't logged in or the network isn't up yet (a user service can start before it is) | Serves agent requests, and checks the field on every MCP call instead. The first call that has the field marks the self-test passed. The first call without it marks it failed, kills that run at once and marks its IDs stale. From then on every agent request gets 503, including Con's next step for those IDs. Con may retry, but each retry is refused before anything runs, so no tool is proposed twice. The next agent request also reruns the self-test once, in the background. |
| Not run, because `claude` wasn't found | Every chat request gets 503 (ADR-0007). The self-test runs once `claude` is found. |

On `SIGTERM`, which is how systemd stops the unit, the bridge stops accepting
requests and ends each open stream to Con cleanly. It sends an error chunk
saying the bridge is stopping, with none of Con's retry trigger words, then
`data: [DONE]`, then closes the socket normally, never with a reset. Con
shows the chunk's JSON as the turn's error and doesn't retry, because the text
holds no trigger word. Ending the stream without the chunk would be worse:
Con would take the cut-off reply as complete. Then the bridge kills every
run's process group, deletes the runs' temp files and exits with status 0.

## Where state lives

| State | Location | Lifetime |
|---|---|---|
| Live runs | Bridge memory | One Con turn. Killed when the turn ends, when Con disconnects mid-stream, or after 6 minutes waiting for Con. |
| Bearer token | `$XDG_CONFIG_HOME/con-claude-bridge/token` (mode `0600`), and in `con.conf` | Until regenerated. |
| Settings | Environment variables, a `systemctl --user edit` drop-in, and `$XDG_CONFIG_HOME/con-claude-bridge/config.toml` | Until the user changes them. Read at start-up. |
| Run MCP config and system prompt | Mode `0600` temp files per run, in the unit's private `/tmp` | One run. |
| Logs | The systemd journal | As the journal is configured. |

`$XDG_CONFIG_HOME` falls back to `~/.config` when it isn't set, which is usual
for a user service. No Claude Code session files persist, because runs use
`--no-session-persistence`. Only the journal keeps run metadata (ADR-0006).

## Deployment

The bridge runs as the systemd user unit `con-claude-bridge.service`, started
at login through `WantedBy=default.target`, as the user and with the user's
Claude Code login (ADR-0006). There are two ways to install it:

- **The Arch User Repository (AUR) package**
  ([ADR-0008](adr/0008-aur-release-package.md)), for users. pacman installs the
  entry point at `/usr/bin/con-claude-bridge`, the package under
  `/usr/lib/python3.N/site-packages/` and the unit at
  `/usr/lib/systemd/user/con-claude-bridge.service`. The user then runs
  `con-claude-bridge setup` and
  `systemctl --user enable --now con-claude-bridge`.
- **From a checkout**, for development. `pip install -e .` in the project's
  `.venv` (ADR-0004) puts the entry point at `.venv/bin/con-claude-bridge`.
  It runs in the foreground, or as a copy of
  `packaging/systemd/con-claude-bridge.service` in `~/.config/systemd/user/`
  with `ExecStart` pointed at the venv. Delete that copy before installing the
  package, because it takes priority over the packaged unit.

Either way the unit has `NoNewPrivileges=yes` and `PrivateTmp=yes`, and
settings go in a drop-in made with `systemctl --user edit con-claude-bridge`.
[`configuration.md`](configuration.md) lists every file each install puts on
the machine.

## What happens when things fail

| Failure | What the user sees | What the bridge does |
|---|---|---|
| `config.toml` isn't valid, or the port contains `429`, `500`, `502` or `503` | The unit shows as failed | Logs the problem and exits with status 1. |
| `claude` isn't found | Con shows an error saying `claude` wasn't found. `/healthz` shows `"claude": "not_found"`. | Answers each chat request with 503 and runs no self-test. Nothing has run, so Con's retries are safe. The journal lists the paths checked. |
| Claude Code isn't logged in, or its API is rate-limited or down, before any output | Con retries, then shows Claude Code's error | 429 or 503 with Claude Code's error text: the `result` event's text, or the last line of standard error if `claude` exits without one (ADR-0007). |
| The user presses stop while Claude is streaming | Nothing more | Con disconnects, so the run is `dead` and killed at once (ADR-0007). |
| The user presses stop during approval, or the approval times out | Nothing more | The `waiting` run stays until a later turn needs its slot or its 6-minute idle timer kills it (ADR-0007). |
| The bridge restarts mid-turn | The next step continues, with a fresh `claude` process | The result's ID matches no run, so the bridge replays the turn from Con's request (ADR-0003). |
| `claude` crashes, or an MCP call times out on Claude Code's side | Con shows an error for the turn and doesn't retry | The run becomes `desynced` and then `dead`. Its IDs go stale. The next request for them gets 409 (ADR-0007). |
| A third agent turn starts while two are streaming | Con shows a "busy" error and doesn't retry | 409, with no retry trigger words (ADR-0007). |
| Claude Code stops sending `toolUseId` | Agent requests fail with a message pointing to the journal. Suggestions still work. | 503 for every agent request until restarted (see [Start-up and shutdown](#start-up-and-shutdown)). |
| The bridge is stopped or restarted mid-turn | Con shows a "bridge stopping" error for that turn | Ends each stream cleanly, as above. The next step replays (ADR-0003). |
| The port is taken | The unit shows as failed | Logs the port and exits with status 1. |

Every row that kills a run also kills its process group and deletes its temp
files.

## Observability

Logs go to standard error, which systemd puts in the journal. Read them with:

```sh
journalctl --user -u con-claude-bridge -f
```

At start-up the bridge logs the `claude` path and version, the port and the
self-test result. For each run it logs one line per state change. The line
holds the run ID, the profile (`agent` or `quick`), the new state and the
time since the run started, and the last line adds `claude`'s exit status.
Prompts, replies and tool results stay out of the log, and so does `claude`'s
standard error, which can quote them. The `log_content` setting adds them, for
debugging only, because the journal is then a copy of the conversation
([`configuration.md`](configuration.md)).

`GET /healthz` needs no token, so it shows only what is safe for any local
process to see:

```json
{"status": "ok", "claude": "found", "self_test": "passed"}
```

| Field | Values |
|---|---|
| `status` | `ok` when `claude` is found and the self-test hasn't failed, otherwise `degraded`. |
| `claude` | `found` or `not_found`. |
| `self_test` | `pending` (not run, not finished, or couldn't finish), `passed` or `failed`. The per-call check in [Start-up and shutdown](#start-up-and-shutdown) can also move it from `pending` to `passed` or `failed`. |

It never shows paths, run counts or versions. Those are in the journal.

The bridge keeps no metrics. If cost or latency needs tracking later, the
per-run log lines already carry durations.

## Testing

Tests run offline and cost nothing, because a fake `claude` stands in for the
real one (ADR-0004). The tools are `pytest`, `ruff` and `mypy --strict`, run
from the project venv:

```sh
.venv/bin/pytest
.venv/bin/ruff check .
.venv/bin/mypy --strict src tests
```

These commands can't run yet, because no code exists.

| Layer | What it covers |
|---|---|
| Unit tests | The pure modules (`openai_protocol`, `mcp_protocol`, `routing`, `prompting`, `errors`) against recorded fixtures, and every state change in ADR-0007's table, driven by a fake clock and no real processes. |
| Integration tests | A real server on a free port, with `BRIDGE_CLAUDE_BIN` pointing at `tests/fake_claude/fake_claude`. The fake reads a scenario file. It prints the scenario's `stream-json` events, reads its `--mcp-config` and makes the scenario's MCP calls back to the bridge, so a whole turn runs over real HTTP. |
| Error wording | A test that every error message meant not to cause a retry is free of Con's retry trigger words and rewrite phrases, even when it quotes Claude Code's error text, and that every status code is the one ADR-0007 gives. |
| Manual check | Before each release, one real turn through Con with an approval, a denial and a stop. Not automated, because it needs Con, a display and a Claude login. |

The integration scenarios include everything ADR-0007's consequences list,
and a restart that forces a replay:

- results that arrive before their MCP call;
- an MCP call that arrives before the assistant message that announces it;
- several results in one request;
- one assistant message with two `tool_use` blocks, and one with thinking
  before its `tool_use`;
- a cancel during approval, followed by a new turn;
- the idle timer.

They also cover an MCP call held open past 30 seconds,
checking that progress messages go out.

The fixtures come from step 1 of
[`implementation-plan.md`](implementation-plan.md): real requests from Con
0.1.0-beta.135 and real `stream-json` output from Claude Code. When either
changes its format, recording new fixtures is the first step.

## Decisions

- [ADR-0001](adr/0001-openai-chat-completions-as-the-con-interface.md): Con
  talks to the bridge through its `openai-compatible` provider, using Chat
  Completions. The Agent Client Protocol (ACP), which Zed uses, is the
  long-term path.
- [ADR-0002](adr/0002-claude-code-as-a-step-in-cons-agent-loop.md): Claude Code
  acts as one step of Con's agent loop and uses only Con's tools.
- [ADR-0003](adr/0003-con-history-is-the-source-of-truth.md): Con's history is
  the source of truth, and the bridge finds live runs by tool call ID.
- [ADR-0004](adr/0004-python-standard-library-runtime.md): Python 3, with only
  the standard library at run time.
- [ADR-0005](adr/0005-separate-quick-path-for-suggestions.md): a cheap path for
  suggestions and tab labels.
- [ADR-0006](adr/0006-localhost-service-with-token.md): a localhost-only systemd
  user service with a bearer token, and how the bridge finds `claude`.
- [ADR-0007](adr/0007-run-lifecycle.md): run states, result mailboxes, limits
  and error mapping.
- [ADR-0008](adr/0008-aur-release-package.md): ship the bridge as an Arch User
  Repository (AUR) release package, with the `PKGBUILD` kept in this repo.

## Glossary

| Term | Meaning |
|---|---|
| Turn | Everything Con's agent does in answer to one user message, from the first model request to the final reply. |
| Step | One model request within a turn. A turn with two tool calls in separate replies has three steps. |
| Run | One `claude -p` process serving one turn, and the bridge's record of it. |
| Profile | Which kind of run: `agent`, with Con's tools, or `quick`, with none ([ADR-0005](adr/0005-separate-quick-path-for-suggestions.md)). |
| Mailbox | The slot where a tool result waits until Claude's matching MCP call takes it, or the call waits for its result ([ADR-0007](adr/0007-run-lifecycle.md)). |
| Replay | Starting a new run from a mid-turn request whose run is gone, with the turn so far written into the prompt ([ADR-0003](adr/0003-con-history-is-the-source-of-truth.md)). |
| Con | The Con terminal ([nowledge-co/con-terminal](https://github.com/nowledge-co/con-terminal)), whose built-in agent is the bridge's only client. |
| MCP | Model Context Protocol, the way Claude Code takes tools from an outside server. |
| SSE | Server-Sent Events, the streaming format of both Chat Completions and MCP over HTTP. |
