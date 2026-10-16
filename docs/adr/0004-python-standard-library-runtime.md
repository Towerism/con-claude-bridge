# ADR-0004: Python 3 with the standard library only at runtime

**Status:** Proposed, 2026-10-16

## Context

The bridge is a small local service. It runs one HTTP server, starts child
processes, and streams JSON and Server-Sent Events (SSE). It runs under systemd
on Arch Linux. On the development machine (Python 3.14.8 as of 2026-10-16)
there is no `uv`, and system pip is externally managed.

## Decision

The service needs Python 3.12 or later and nothing outside the standard
library: `http.server.ThreadingHTTPServer`, `subprocess`, `json`, `secrets`
and `tomllib`. One thread per request is enough, because Con sends only a few
requests at a time.

Development tools live in a project venv (`.venv`): `pytest` for tests, `ruff`
for linting and `mypy --strict` for type checking. `pyproject.toml` describes
the project. Tests replace the real `claude` with a fake executable that prints
recorded `stream-json` events, so they run offline and cost nothing.

## Options considered

| Option | Why not chosen |
|---|---|
| FastAPI or aiohttp | Async and nicer routing, but the service would need dependencies at run time, and the systemd unit would depend on a venv. This load doesn't need them. |
| Rust, to match Con | One binary, but more code for the same job and slower to change. |
| Node.js | The same trade-offs as Python, with nothing gained. |

## Consequences

- The service itself has no dependencies to install or update.
- If the bridge ever needs many concurrent streams or WebSockets, revisit this.
