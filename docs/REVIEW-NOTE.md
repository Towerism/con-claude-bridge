# Review note: architecture and ADRs 0001–0007

This records the one adversarial review of the architecture document and ADRs
0001–0007, done for issue #1, and how each finding was handled. Use it to see
why those documents say what they do.

A fresh-context reviewer ran the pass on 2026-10-16, reading as three hostile
personas: a saboteur, a new hire and a security auditor. It checked the claims
against Con commit `de2d7c7`, rig-core 0.40.0 and `claude --help` from Claude
Code 2.1.299. The reviewer's verdict was to block the change, on 1 critical
finding, 6 warnings and 6 notes. The documents were revised to fix all of
them. Under the project's one-pass rule, the review was not run again.

| Finding | Severity | What was wrong | How it was handled |
|---|---|---|---|
| Abandoned runs block new turns | Critical | Con sends nothing when a turn ends early. Abandoned runs would fill both agent slots, and new turns would queue for up to 15 minutes. The planned per-session limit couldn't be built, because Con sends no session ID. | New [ADR-0007](adr/0007-run-lifecycle.md): run states, a limit that counts only `streaming` runs, no queueing behind a `waiting` run, eviction of the oldest waiting run, a 6-minute idle timer, and no per-session limit. |
| Several results per step | Warning | Routing several results in one step was underspecified, and a result could arrive before Claude's MCP call. | ADR-0007's mailboxes: close the response on Claude's complete assistant message, register every ID, and route every `tool` message. |
| Claude acting with no response open | Warning | Nothing said what happens when Claude produces output while no Con response is open. | ADR-0007's `desynced` state: kill the run, mark its IDs stale, and replay. |
| Quick-path rule never matched | Warning | The rule `max_tokens <= 64` never matches, because suggestions use 100 and tab labels 2048. | [ADR-0005](adr/0005-separate-quick-path-for-suggestions.md) now sends requests with no `tools` to the quick path and kills quick runs on disconnect. The facts table was corrected. |
| User configuration leaking into runs | Warning | Agent runs would load the user's hooks, plugins and global `CLAUDE.md`, and nothing set the working directory. | [ADR-0002](adr/0002-claude-code-as-a-step-in-cons-agent-loop.md) points 7 and 8: `--setting-sources ""` with a bridge settings file, no global `CLAUDE.md`, the project's `CLAUDE.md` kept, and the working directory taken from Con's terminal context. This needs the user's confirmation. |
| Con's retries repeat approved tools | Warning | Con retries a whole turn when an error contains certain words, which would propose already-approved tools again. | ADR-0007's error table: HTTP 409 for failures mid-turn, worded to avoid every retry word. |
| Token exposure | Warning | The token would show in `/proc/<pid>/cmdline`, and one token unlocked everything. | [ADR-0006](adr/0006-localhost-service-with-token.md): the MCP config goes in a `0600` temp file, each run gets its own MCP token, and `con.conf` is set to `0600`. |
| Leftover `hashlib` | Note | `hashlib` was left over from the hashing design. | Removed from [ADR-0004](adr/0004-python-standard-library-runtime.md). |
| History wording | Note | The "last 20 messages" wording was slightly wrong. | Corrected in the architecture document and [ADR-0003](adr/0003-con-history-is-the-source-of-truth.md). |
| Prompt-caching claim | Note | The claim that prompt caching would help is doubtful, because the system prompt changes every turn. | ADR-0003 now says the first implementation issue measures it. |
| Con's ACP support | Note | Con does have an ACP client, used to read history during handoff. | [ADR-0001](adr/0001-openai-chat-completions-as-the-con-interface.md) now says Con can't prompt an ACP agent. |
| Denials and thinking | Note | Denial text varies, and Claude's thinking was hidden. | ADR-0002 forwards results unchanged and sends thinking as `reasoning_content`. |
| Unstated details | Note | The partial-message flag, the tool-name prefix and tool-search hiding weren't stated. | ADR-0002 point 1, and open question 3 in the architecture document. |
