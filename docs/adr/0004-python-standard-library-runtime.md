# ADR-0004: Python 3 with the standard library only at runtime

**Status:** Proposed, 2026-10-16

## Context

The bridge is a small local service: one HTTP server, child processes, JSON and
Server-Sent Events (SSE) streaming. It runs under systemd on this Arch Linux machine (Python 3.14.8). There is
no `uv`, and system pip is externally managed.

## Decision

- Runtime: Python 3.12 or later, standard library only. That means
  `http.server.ThreadingHTTPServer`, `subprocess`, `json`, `secrets` and
  `tomllib`. One thread per request is enough, because Con sends a few requests
  at a time.
- Development: `pytest` and `ruff` in a project venv (`.venv`), and `mypy
  --strict` for type checking. The project is described in `pyproject.toml`.
- Tests replace the real `claude` with a fake executable that prints recorded
  `stream-json` events, so tests run offline and cost nothing.

## Options considered

| Option | Why not chosen |
|---|---|
| FastAPI or aiohttp | Async and nicer routing, but adds dependencies the service needs at run time, and a venv the systemd unit depends on. Not needed for this load. |
| Rust (to match Con) | A single binary, but more code for the same job, and slower to change. |
| Node.js | Same trade-offs as Python, with no advantage here. |

## Consequences

- No dependencies to install or update for the service itself.
- Revisit if the bridge needs many concurrent streams or WebSockets.
