# con-claude-bridge implementation plan

This page gives the order in which the bridge gets built, written on
2026-10-16 for issue #5. Use it to pick the next piece of work. Each step
below becomes one GitHub issue when work on it starts, and goes through the
usual spec, review and test-first process. None of them is filed yet.

[`architecture.md`](architecture.md) is the design these steps build. When a
step finds that the design is wrong, it updates that document or an ADR
first.

## The order

Each step leaves something that works and can be tested on its own. A step
starts only when every step it depends on has merged.

| Step | Delivers | Depends on |
|---|---|---|
| 1. Record real traffic | Fixtures and answers to the open questions | Nothing |
| 2. Project skeleton | A service that starts, checks tokens and answers `/healthz` | 1 |
| 3. Quick path | Working suggestions and tab labels | 2 |
| 4. Agent path | A full Con turn with tool calls and approvals | 3 |
| 5. Run lifecycle | Limits, timers, replay, errors and the self-test | 4 |
| 6. Service and first release | A systemd unit and a semantic-release tag | 5, and a license |
| 7. AUR package | `con-claude-bridge` in the Arch User Repository (AUR) | 6 |
| 8. Setup guide | A README a new user can follow | 7 |

## 1. Record real traffic

A spike, so its code is thrown away and only its findings merge. A throwaway
server records what Con 0.1.0-beta.135 sends. It also records Claude Code's
`stream-json` output and its Model Context Protocol (MCP) tool calls. The
turns to record have no tools, one tool call, two tool calls in one reply, a
denial and a stop.

It merges:

- the recordings, as fixtures in `tests/fixtures/`, with the token removed;
- answers to the six [open questions](architecture.md#open-questions),
  written into `architecture.md` and any ADR they change.

## 2. Project skeleton

`pyproject.toml`, the `src/con_claude_bridge/` package, and the parts that
don't start `claude`:

- `cli.py`, `config.py`, `auth.py` and `log.py`;
- `errors.py`, with the statuses used so far;
- the `claude` lookup in `claude_runner.py`;
- `server.py`, with `/healthz` and `/v1/models`.

`con-claude-bridge setup` works. Chat requests get 501 until step 3. A CI
workflow runs `pytest`, `ruff` and `mypy --strict` on every PR.

Done when an integration test starts the server on a free port and the token,
`Host` and content-type checks from
[ADR-0006](adr/0006-localhost-service-with-token.md) each reject a bad
request.

## 3. Quick path

`claude_runner.py`, the `quick` profile in `routing.py` and `runs.py`, and the
first version of the fake `claude`. A request with no `tools`, or for
`claude-quick`, gets a streamed answer
([ADR-0005](adr/0005-separate-quick-path-for-suggestions.md)).

Done when tests show the quick limit returning 429, and a dropped connection
killing its run within a second.

## 4. Agent path

`mcp_protocol.py`, `prompting.py`, the `/mcp/<run-id>` route, result
mailboxes and the `agent` profile
([ADR-0002](adr/0002-claude-code-as-a-step-in-cons-agent-loop.md),
[ADR-0003](adr/0003-con-history-is-the-source-of-truth.md)). The fake
`claude` gains MCP calls.

Done when an integration test runs a turn with two tool calls in one reply,
one of whose results arrives before its MCP call. One real turn through Con
must also work by hand.

## 5. Run lifecycle

The rest of [ADR-0007](adr/0007-run-lifecycle.md) and of the start-up and
shutdown design:

- the idle reaper and eviction of the oldest `waiting` run;
- the `desynced` state and replay;
- the rest of `errors.py`, with the retry-word test;
- progress messages during MCP calls;
- the clean stream ending at shutdown;
- `selftest.py` and the per-call `toolUseId` check.

Done when every state change in ADR-0007's table and every row of the failure
table in [`architecture.md`](architecture.md#what-happens-when-things-fail)
has a test.

## 6. Service and first release

`packaging/systemd/con-claude-bridge.service`, semantic-release
configuration, and the first stable `v1.0.0` tag.

This step needs a decision from the owner first: the project's license.
Without one, nobody else may use or redistribute the code, and
[ADR-0008](adr/0008-aur-release-package.md) can't publish it.

## 7. AUR package

`packaging/aur/` as [ADR-0008](adr/0008-aur-release-package.md) describes,
built in a clean chroot, checked with `namcap` and pushed to the AUR by hand.

## 8. Setup guide

The README gains a how-to that takes a new user from installing the package
to a first turn in Con, using the settings in
[`configuration.md`](configuration.md). Its commands must have been run on a
clean machine.
