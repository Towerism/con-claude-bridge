# con-claude-bridge configuration reference

Look things up here: every setting the bridge reads, the Con settings that
point at it, and every file an install puts on the machine. For why the bridge
works this way, read [`architecture.md`](architecture.md) and the ADRs it
links.

The bridge doesn't exist yet. This page describes the settings the
architecture commits to, as of 2026-10-16.

## Bridge settings

Each setting can come from an environment variable or from
`$XDG_CONFIG_HOME/con-claude-bridge/config.toml`, which is
`~/.config/con-claude-bridge/config.toml` when `XDG_CONFIG_HOME` isn't set.
The environment variable wins. The bridge reads settings once, at start-up.

| `config.toml` key | Environment variable | Default | What it does |
|---|---|---|---|
| `port` | `BRIDGE_PORT` | `8765` | The port on `127.0.0.1`. The bridge never binds any other address ([ADR-0006](adr/0006-localhost-service-with-token.md)). It refuses to start, logging why and exiting with status 1, if the port's digits contain `429`, `500`, `502` or `503`, because Con's connection errors quote the URL and Con would take those digits as a reason to retry ([ADR-0007](adr/0007-run-lifecycle.md)). |
| `claude_bin` | `BRIDGE_CLAUDE_BIN` | Not set | An absolute path to `claude`. When it is set, the bridge uses only that path. If nothing executable is there, `claude` counts as not found, with no fallback. When it isn't set, the bridge tries `~/.local/bin/claude` and then `claude` on `PATH` (ADR-0006). |
| `agent_model` | `BRIDGE_AGENT_MODEL` | Not set | The `--model` for agent runs, such as `opus` or `sonnet`. When it isn't set, agent runs use Claude Code's default model. The user's own Claude Code settings don't load in agent runs, so this is the supported way to pick one ([ADR-0002](adr/0002-claude-code-as-a-step-in-cons-agent-loop.md), point 7). |
| `quick_model` | `BRIDGE_QUICK_MODEL` | `haiku` | The `--model` for quick runs: suggestions, tab labels and the self-test ([ADR-0005](adr/0005-separate-quick-path-for-suggestions.md)). |
| `log_content` | `BRIDGE_LOG_CONTENT` | `false` | When `true`, the journal also gets prompts, replies and tool results. For debugging only: it makes the journal a copy of every conversation. |

In the environment, `log_content` takes `true` or `false`.

The bridge refuses to start if `config.toml` has an unknown key, a value of
the wrong type, or text that isn't valid TOML (the file's format). It logs the
problem and exits with status 1, so a typo can't go unnoticed.

An example `config.toml`:

```toml
agent_model = "opus"
quick_model = "haiku"
```

For the systemd service, put environment variables in a drop-in rather than
editing the unit. `systemctl --user edit con-claude-bridge` opens one:

```ini
[Service]
Environment=BRIDGE_CLAUDE_BIN=%h/bin/claude
Environment=BRIDGE_PORT=8766
```

`%h` is systemd's shorthand for the home directory. Then restart the service
with `systemctl --user restart con-claude-bridge`.
After changing the port, update `base_url` in `con.conf` as well.

### Values that aren't settings

These follow from how Con and Claude Code behave, so they are fixed in the
code. [ADR-0007](adr/0007-run-lifecycle.md) explains each.

| Value | Why it is fixed |
|---|---|
| 2 streaming agent runs and 2 quick runs at a time | Con sends only a few requests at once. |
| 6 minutes before a `waiting` run is killed | Con's approval prompt times out after 5 minutes, plus a minute to spare. |
| A progress message every 30 seconds during an MCP call | Claude Code aborts an MCP call after 300 seconds without one. |

## Con settings

Con reads `~/.config/con/con.conf`. `con-claude-bridge setup` prints these
lines with the real token filled in, and sets the file to mode `0600`,
because it then holds the token:

```ini
con.agent.provider = openai-compatible
con.agent.providers.openai-compatible.base_url = "http://127.0.0.1:8765/v1"
con.agent.providers.openai-compatible.model = claude-code
con.agent.providers.openai-compatible.api_key = "<token from the bridge>"
con.agent.suggestion_model.model = claude-quick
```

The bridge serves two model names:

| Model name | Use |
|---|---|
| `claude-code` | Con's agent, with Con's tools ([ADR-0002](adr/0002-claude-code-as-a-step-in-cons-agent-loop.md)). |
| `claude-quick` | Inline suggestions and tab labels, with no tools ([ADR-0005](adr/0005-separate-quick-path-for-suggestions.md)). |

To stop all background traffic, turn suggestions off with
`con.agent.suggestion_model.enabled = false`.

To stop using the bridge, set `con.agent.provider` back to the provider you
used before, then run `systemctl --user disable --now con-claude-bridge`.

## Files on the machine

From the Arch User Repository (AUR) package
([ADR-0008](adr/0008-aur-release-package.md)), which pacman owns:

| Path | What it is |
|---|---|
| `/usr/bin/con-claude-bridge` | The entry point. With no subcommand it serves. `setup` creates the token and prints the Con settings. |
| `/usr/lib/python3.N/site-packages/con_claude_bridge/` | The Python package, where `N` is the Python minor version it was built for. |
| `/usr/lib/systemd/user/con-claude-bridge.service` | The systemd user unit. |
| `/usr/share/licenses/con-claude-bridge/` | The project's license. |

From a checkout, for development:

| Path | What it is |
|---|---|
| `.venv/bin/con-claude-bridge` | The entry point, from `pip install -e .`. |
| `~/.config/systemd/user/con-claude-bridge.service` | Optional. A copy of `packaging/systemd/con-claude-bridge.service` with `ExecStart` pointing at the venv. It hides the packaged unit, so delete it before installing the package. |

Belonging to the user, whichever way the bridge was installed. Removing the
package leaves these in place:

| Path | Mode | What it is |
|---|---|---|
| `~/.config/con-claude-bridge/token` | `0600` | The bearer token, created on first start or by `setup`. To change it, delete it, run `setup` again, paste the new lines into `con.conf` and run `systemctl --user restart con-claude-bridge`, because the bridge reads the token only at start-up. |
| `~/.config/con-claude-bridge/config.toml` | Any | Optional settings, as above. |
| `~/.config/systemd/user/con-claude-bridge.service.d/override.conf` | Any | Optional drop-in from `systemctl --user edit`. |
| `~/.config/con/con.conf` | `0600` | Con's own settings, holding the token. |

While the bridge runs, each live run also has two files with mode `0600` in
the unit's private `/tmp`: its `--mcp-config` and its system prompt. The
bridge deletes them when the run ends.
