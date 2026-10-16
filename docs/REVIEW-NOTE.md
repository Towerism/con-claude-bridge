# Review note: architecture and ADRs

**Pass:** one adversarial review (Saboteur, New Hire, Security Auditor), run by a
fresh-context reviewer on 2026-10-16. It checked claims against Con commit
`de2d7c7`, rig-core 0.40.0 and `claude --help` (2.1.299).
**Verdict:** BLOCK (1 critical, 6 warnings, 6 notes). All fixed in the docs, as
listed below. Per the one-pass rule, the review was not re-run.

| ID | Severity | Finding | Handling |
|---|---|---|---|
| C1 | CRITICAL | Con sends nothing when a turn ends early. Abandoned runs would fill the 2 agent slots and new turns would queue for up to 15 minutes. The "per session" limit can't be built, because Con sends no session ID. | Fixed. New ADR-0007: run states, only `streaming` runs count toward limits, never queue behind a `waiting` run, evict the oldest waiting run, 6-minute idle timer, no per-session limit. |
| W1 | WARNING | Routing several results per step was underspecified. A result could arrive before Claude's MCP call. | Fixed. ADR-0007 mailboxes: close the response on the complete assistant message, register every ID, route all `tool` messages. |
| W2 | WARNING | Claude emitting output while no Con response is open was undefined. | Fixed. ADR-0007 `desynced` state: kill the run, mark its IDs stale, replay. |
| W3 | WARNING | `max_tokens <= 64` never matches. Suggestions use 100, tab labels 2048. | Fixed. ADR-0005 routes requests with no `tools` to the quick path, and kills quick runs on disconnect. Facts row corrected. |
| W4 | WARNING | Agent runs would load the user's hooks, plugins and global `CLAUDE.md`, and no working directory was set. | Fixed. ADR-0002 points 7 and 8: `--setting-sources ""` plus a bridge settings file, no global `CLAUDE.md`, project `CLAUDE.md` kept, working directory from Con's terminal context. Needs user confirmation. |
| W5 | WARNING | Con retries a whole turn on certain error words, which would re-propose approved tools. | Fixed. ADR-0007 error table: 409 for mid-turn failures, with no retry words. |
| W6 | WARNING | The token would show in `/proc/<pid>/cmdline`, and one token unlocked everything. | Fixed. ADR-0006: MCP config in a `0600` temp file, a per-run MCP token, and `con.conf` set to `0600`. |
| N1 | NOTE | `hashlib` left over from the hashing design. | Fixed (ADR-0004). |
| N2 | NOTE | The "last 20 messages" wording was slightly wrong. | Fixed (architecture, ADR-0003). |
| N3 | NOTE | The prompt-caching claim is doubtful, because the system prompt changes every turn. | Fixed: the claim is replaced with "measured, not assumed" (ADR-0003). |
| N4 | NOTE | Con has an ACP client for handoff history. | Fixed: "no ACP agent or provider support" (ADR-0001). |
| N5 | NOTE | Denial text varies. Thinking was hidden. | Fixed: results are forwarded unchanged, and thinking goes out as `reasoning_content` (ADR-0002). |
| N6 | NOTE | Partial-message flag, tool-name prefix and tool-search deferral weren't stated. | Fixed: ADR-0002 point 1, open question 3. |
