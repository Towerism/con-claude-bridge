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
  When installed from the Arch User Repository (AUR) package, the unit lives in
  `/usr/lib/systemd/user/` instead ([ADR-0008](0008-aur-release-package.md)).
- Find `claude` the same way on every install. A user service's `PATH`
  doesn't include `~/.local/bin`. So the bridge checks, in order: the
  `BRIDGE_CLAUDE_BIN` setting (an absolute path, from the environment or
  `config.toml`, with the environment winning), then `~/.local/bin/claude`
  (Anthropic's installer, which updates itself), then a lookup on `PATH`. It
  logs the chosen path and `claude --version` at start-up. If nothing is found,
  the log lists the paths it checked, and `/healthz` (which needs no token) says
  only `claude not found`, so it never reveals the home directory.
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
