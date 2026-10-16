# ADR-0006: A localhost-only systemd user service with a bearer token

**Status:** Accepted, 2026-10-16. Amended for issue #3: how the bridge finds
`claude`. Amended for issue #5: where settings live, `setup` creating the
token, and what `/healthz` shows.

## Context

Anything that can reach the bridge can make Claude Code run tools as the user.
That includes other local processes, and also web pages, because a browser will
send requests to `127.0.0.1`. A plain `text/plain` `POST` request skips the
browser's cross-origin (CORS) preflight check, and DNS rebinding lets a remote
page pose as `localhost`.

## Decision

- The bridge binds to `127.0.0.1` only. The port defaults to `8765` and can be
  changed with `BRIDGE_PORT`.
- Every route except `/healthz` requires `Authorization: Bearer <token>`. On
  first start, or when `con-claude-bridge setup` runs, the bridge creates the
  token, 32 random bytes from `secrets.token_urlsafe`, at `$XDG_CONFIG_HOME/con-claude-bridge/token` with
  mode `0600`. Con sends it as
  `con.agent.providers.openai-compatible.api_key`.
- The bridge rejects any request whose `Host` header isn't
  `127.0.0.1:<port>` or `localhost:<port>`, which blocks DNS rebinding.
- The Model Context Protocol (MCP) endpoint that `claude` calls,
  `/mcp/<run-id>` ([ADR-0002](0002-claude-code-as-a-step-in-cons-agent-loop.md)),
  has a separate token per run. It is valid only on that run's path. The bridge
  writes the `--mcp-config` file, with the URL and token header, to a temp
  file with mode `0600`. It passes the file's path to `claude` and deletes the
  file when the run ends. Tokens never go on a command line, since any local
  user can read those in `/proc/<pid>/cmdline`. `<run-id>` is a random UUID
  (universally unique identifier), and a request for an unknown run gets 404.
- The bridge rejects `POST` bodies whose `Content-Type` isn't
  `application/json` and sends no CORS headers. Browsers then can't make simple
  cross-site requests to it.
- It runs as a systemd user unit,
  `~/.config/systemd/user/con-claude-bridge.service` with
  `WantedBy=default.target`, so it runs as the user and with the user's Claude
  Code login. Its hardening is `NoNewPrivileges=yes` and `PrivateTmp=yes`. It
  has no filesystem sandbox, because Claude Code needs the user's projects.
  The Arch User Repository (AUR) package installs the same unit at
  `/usr/lib/systemd/user/` instead ([ADR-0008](0008-aur-release-package.md)).
- Every install finds `claude` the same way. A user service's `PATH` doesn't
  include `~/.local/bin`, so the bridge checks these in order and uses the
  first that exists:
  1. The `BRIDGE_CLAUDE_BIN` setting, an absolute path. The environment
     variable wins over `claude_bin` in
     `$XDG_CONFIG_HOME/con-claude-bridge/config.toml`, the file that holds
     every setting ([`configuration.md`](../configuration.md)).
  2. `~/.local/bin/claude`, where Anthropic's self-updating installer puts it.
  3. `claude` on `PATH`.

  At start-up the bridge logs the path it chose and the output of
  `claude --version`. If it finds nothing, the log lists the paths it checked.
  `/healthz` needs no token, so it says only `claude not found` and never shows
  the home directory. It reports only an overall status, whether `claude` was
  found and the self-test result, never paths, versions or run counts
  ([the architecture document](../architecture.md#observability)).
- Logs go to the journal. By default they hold metadata only: profile, run ID,
  state changes, duration and exit status, never prompts or replies.

## Options considered

| Option | Why not chosen |
|---|---|
| A Unix socket | The safest choice, but Con's HTTP client only accepts an `http(s)` `base_url`. |
| No token, relying on localhost | Open to browser attacks and to any local process. |
| Con starts the bridge when needed | Con can't start a process for an HTTP provider. |

## Consequences

- Changing the token means updating `con.conf` as well.
  `con-claude-bridge setup` prints the config lines to paste, and sets
  `~/.config/con/con.conf` to mode `0600`.
- To remove the bridge, disable the unit and switch Con's provider back.
