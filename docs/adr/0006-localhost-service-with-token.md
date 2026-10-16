# ADR-0006: A localhost-only systemd user service with a bearer token

**Status:** Proposed, 2026-10-16

## Context

Anything that can reach the bridge can make Claude Code run tools as the user.
That includes other local processes, and web pages in a browser, which can send
requests to `127.0.0.1`. A plain `text/plain` POST request skips the browser's cross-origin (CORS) preflight check, and
DNS rebinding lets a remote page pose as `localhost`.

## Decision

- Bind to `127.0.0.1` only. The port defaults to `8765` (`BRIDGE_PORT`).
- Require `Authorization: Bearer <token>` on every route except `/healthz`. The
  token is 32 random bytes from `secrets.token_urlsafe`, created on first start
  at `$XDG_CONFIG_HOME/con-claude-bridge/token` with mode `0600`. Con sends it
  through `con.agent.providers.openai-compatible.api_key`.
- Reject requests whose `Host` isn't `127.0.0.1:<port>` or `localhost:<port>`.
  This blocks DNS rebinding.
- The MCP (Model Context Protocol) endpoint that `claude` calls (`/mcp/<run-id>`, ADR-0002) needs a
  separate token per run, valid only for that run's path. The bridge writes the
  `--mcp-config` (URL and token header) to a mode-`0600` temp file, passes its
  path, and deletes it when the run ends. Tokens never go on a command line,
  where other local users could read them in `/proc/<pid>/cmdline`. `<run-id>`
  is a random UUID, and a request for an unknown run gets 404.
- Reject `POST` bodies whose `Content-Type` isn't `application/json`, and send no
  CORS headers. This blocks simple cross-site requests from browsers.
- Run as a systemd **user** unit (`~/.config/systemd/user/con-claude-bridge.service`,
  `WantedBy=default.target`), so it runs as the user with their Claude Code
  login. Hardening: `NoNewPrivileges=yes`, `PrivateTmp=yes`. No filesystem
  sandboxing, because Claude Code needs the user's projects.
- Logs go to the journal. Prompts and replies aren't logged by default, only
  metadata: profile, run ID, state changes, duration and exit status.

## Options considered

| Option | Why not chosen |
|---|---|
| Unix socket | Safest, but Con's HTTP client only takes an `http(s)` `base_url`. |
| No token (rely on localhost) | Exposed to browser attacks and to any local process. |
| Run on demand from Con | Con can't start a process for HTTP providers. |

## Consequences

- Rotating the token means updating `con.conf` too. The setup command prints the
  config lines to paste, and sets `~/.config/con/con.conf` to mode `0600` (it
  already is on this machine).
- Removing the bridge means disabling the unit and switching Con's provider back.
