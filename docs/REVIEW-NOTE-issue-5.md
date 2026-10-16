# Review note: finishing the architecture documents

This note records the three adversarial reviews of issue #5's documents and how
each finding was handled. The first covered the completed architecture, and
its 21 findings are fixed in commit `a8b5185`. The second covered the later
change that closed every open question, and its 14 findings are fixed in
commit `9c5ef08`. The third covered the full PR diff, and its 14 findings are
fixed in the commit that added its section to this note.

## First pass: the completed architecture

The pass was one adversarial review (Saboteur, New Hire, Security Auditor), run
by a fresh-context reviewer on 2026-10-16 against `origin/main...HEAD` after
commits `b2bd7b3` and `0633547`. It checked flags against `claude --help`
(Claude Code 2.1.299), `%h` and `PrivateTmp=` against systemd 262's manual
pages, and the standard-library modules against Python 3.14. Con's source wasn't
available, so claims about Con were taken from the accepted ADRs. The verdict
was BLOCK (1 critical, 11 warnings, 9 notes). Per the one-pass rule, the review
was not re-run after the fixes.

| Finding | Severity | What was wrong | How it was handled |
|---|---|---|---|
| Missing `claude` contradiction | Critical | The self-test table said a missing `claude` still serves agent requests, but the failure table and ADR-0007's new row give 503. | Fixed. "`claude` missing" left the "couldn't finish" row. A new row says no self-test runs and every chat request gets 503. |
| Fallback trips mid-turn | Warning | When the per-call fallback finds no `toolUseId` mid-turn, the docs didn't say what happens to that run or to Con's next step. | Fixed. The run is killed and its IDs marked stale. Every later agent request, including that next step, gets 503. Retries are refused the same way, so no tool is proposed twice. ADR-0007's row now covers the per-call check. |
| Self-test run undefined | Warning | The self-test run fit neither profile, had no time limit, and its outcomes weren't defined. | Fixed. It is its own kind, `selftest`, outside ADR-0007's states and limits, with a 60-second limit. "Failed" means only a missing or wrong ID, and everything else is "couldn't finish". |
| Unowned self-test state | Warning | Nothing owned the self-test status or the per-call check, and the graph lacked the `cli` to `claude_runner` edge. | Fixed. `runs.py` owns both, `selftest.py` records into it, and the edge is added. |
| MCP call before its mailbox | Warning | Claude's MCP call can arrive before the assistant event that opens its mailbox. | Fixed. Whichever arrives first opens the mailbox. Added as an integration scenario. |
| System prompt on the command line | Warning | Con's system prompt went on the command line, which is limited to 128 KiB per argument and readable in `/proc/<pid>/cmdline`. | Fixed. The prompt goes on standard input, and the system prompt and MCP config go in mode `0600` temp files. |
| Standard error never read | Warning | Nothing read `claude`'s standard error, so a full pipe could hang a run. The source of Claude Code's error text wasn't stated. | Fixed. A second reader thread per run drains it into a 64 KiB buffer, and the journal leaves it out. The error text comes from the `result` event, or the last line of standard error. |
| Reset at shutdown triggers retry | Warning | At shutdown a reset connection could reach Con as `connection reset`, a retry trigger, so approved tools would be proposed again. | Fixed. Streams end with an error chunk free of trigger words, then `[DONE]`, then a normal close. Con's handling of that chunk was later answered in commit `3f53854`: Con shows it and doesn't retry. |
| Token rotation needs a restart | Warning | Token rotation didn't say to restart the bridge, which reads the token at start-up. | Fixed in `configuration.md` and ADR-0006. |
| Silent `claude_bin` fallback | Warning | A set but missing `claude_bin` would fall back to another binary the user didn't choose. | Fixed. A set path is the only one checked, and nothing there means not found. Stated in ADR-0006 and `configuration.md`. |
| Setup guide dependency | Warning | The setup guide (step 8) needs the AUR package from step 7 but depended only on step 6. | Fixed. Step 8 depends on step 7. |
| Long lines | Warning | Some prose lines were still over 80 columns, and the issue lists that fix. | Fixed. The only lines left over 80 are table rows, headings and lines holding a single link or URL. |
| Dashed box | Note | Mermaid draws subgraphs with solid borders, yet the text said "dashed". | Fixed with `style trusted stroke-dasharray: 5 5`. |
| Same-user processes | Note | "No other local process" overstated the token's protection: same-user processes can read it. | Fixed. The goal now says no other user's process, and states the same-user limit. |
| `/healthz` values | Note | When `/healthz` says `degraded` wasn't defined, and its `claude` values were mixed. | Fixed. A field table defines each value. `claude` is `found` or `not_found`. |
| Other ways to pick a model | Note | `agent_model` isn't the only way to pick a model, because `ANTHROPIC_MODEL` also works. | Fixed: "the supported way". |
| Stop row | Note | The stop row ignored that a stop while streaming kills the run at once, and the next turn doesn't always evict a waiting run. | Fixed. Split into a streaming row and an approval row. |
| Reaper tick | Note | A 5-second reaper tick let runs live past the 6 minutes ADR-0007 promises. | Fixed. The reaper sleeps until the earliest deadline. |
| Config and `claude` changes | Note | An invalid `config.toml` and a later `claude` install weren't covered. | Fixed. Both are in the start-up steps and the failure table. The bridge looks up `claude` before each run. |
| Step 2 dependencies | Note | Step 2 needed the `claude` lookup and `errors.py`, which arrived in later steps. | Fixed. Step 2 gets both, and step 5 completes `errors.py`. |
| Reader that stops reading | Note | A client that stops reading could block a write and hold a slot forever. | Fixed. A write blocked for 60 seconds counts as a disconnect. |

## Second pass: closing the open questions

The owner asked for every open question to be closed. Commit `3f53854` did
that, using Con's source at `de2d7c7` and tests against Claude Code 2.1.299 on
2026-10-16. Because it was a new change made for a different reason, it got
its own adversarial pass. A fresh-context reviewer ran the pass on 2026-10-16
and checked the new facts against Con's source, rig-core 0.40.0 and the test
logs. The verdict was BLOCK (2 critical, 8 warnings, 4 notes). Per the
one-pass rule, the review was not re-run after the fixes.

| Finding | Severity | What was wrong | How it was handled |
|---|---|---|---|
| Waiting rule would always desync | Critical | After a `tool_use` message, Claude Code prints its closing stream events and a `rate_limit_event`, and it prints one `assistant` event per content block. Under the new rule, every run would desync at its first tool call, and a second `tool_use` block would be lost. | Fixed. A message ends at `message_stop`, its blocks are gathered by message ID, and ADR-0007 lists the events that don't count in `waiting`. A facts row records the event order, and an integration scenario covers two `tool_use` blocks. |
| Diagram contradicted ADR-0002 | Critical | The trust-boundary diagram still showed Claude Code reading `CLAUDE.md`. | Fixed. The edge now runs from the bridge. |
| `CLAUDE.md` walk underspecified | Warning | A working directory under `~/.claude` would load the global file, paths outside the home directory had no stop, and the file list wasn't tested. | Fixed. Real paths, `~/.claude` always skipped, a stop below `/` outside home, no `@path` imports, and the file list marked as untested. |
| Untrusted `CLAUDE.md` files | Warning | Any file named `CLAUDE.md` in a cloned repository went into the system prompt unchecked, including pipes or symlinks. | Fixed. Regular files only, 64 KiB per file and 256 KiB in all, and ADR-0006 now treats project instructions and tool output as untrusted, with Con's approval as the safeguard. |
| `prompting.py` gained I/O | Warning | Reading `CLAUDE.md` broke the "No I/O" claim for `prompting.py`. | Fixed. `claude_runner.py` reads the files, and `prompting.py` gets their text. |
| Caching claim too strong | Warning | The 95% figure came from a two-word message, but real history follows the changing context and so isn't cached. | Fixed. ADR-0003 says the tools and system prompt are cached and the history isn't, and that a new `CLAUDE.md` set re-caches. |
| Suggestion cleanup too aggressive | Warning | Removing quotes broke suffixes such as `"fix typo"`, and keeping the first line showed preambles that Con would drop. | Fixed. Only a single-line reply in backticks or a fence is unwrapped, other multi-line replies become empty, and quotes and spaces are kept. |
| Crash looked like success | Warning | With the body ended by closing the socket, a bridge crash would read as a complete reply. | Fixed. Streams use chunked encoding, so a missing final chunk shows as an error. |
| Trigger-word rewrite undefined | Warning | No rule said how to rewrite trigger words, and only the message was checked. | Fixed. The final bytes are checked, error responses hold no IDs or numbers, and a quote that trips the check is dropped. |
| Prompt order not updated | Warning | ADR-0003 and the request steps still put the history first. | Fixed in both. |
| 60-second header limit | Note | With a per-server timeout, the call didn't time out at 60 s. | Fixed: the limit applies only without one. |
| Port refusal undefined | Note | "Refuses a port" didn't say how. | Fixed: it logs and exits with status 1, and the failure table has the row. |
| Capability and tool overlap | Note | `probe_shell_context` is both a capability and a tool. | Fixed: the note says a name in the tool list is a tool. |
| URL in connection errors | Note | The claim had no facts row. | Fixed: a row cites rig-core's `http_client/mod.rs:256`. |

## Third pass: the full PR diff against main

Before the PR, a fresh-context reviewer ran one adversarial pass on the whole
diff against `main` on 2026-10-16, after the owner had approved the documents
as of commit `1dfc041`. The verdict was BLOCK (1 critical, 4 warnings, 9
notes). The fixes changed approved documents, so the issue went back to draft
for the owner's approval. Per the one-pass rule, the review was not re-run.

| Finding | Severity | What was wrong | How it was handled |
|---|---|---|---|
| Stale IDs had three answers | Critical | ADR-0007's mailbox section replayed stale IDs, its error table gave 409, and the self-test text gave 503. | Fixed. IDs that match no run replay. Stale IDs from a `desynced` run get 409, or 503 once Claude Code is known not to send tool call IDs. `routing.py` lists stale IDs as an outcome. |
| Retryable errors after the first step | Warning | A 429 or 503 on a later step would make Con retry the turn and propose approved tools again. | Fixed. Only a turn's first step gets 503 for a failure before output. Later steps and replays get 409, and the bridge holds back the `200` status line until Claude's first output. |
| Undefined settings file | Warning | ADR-0002 mentioned a settings file the bridge owns, with no owner or contents. | Fixed. There is no settings file: everything goes as flags. |
| Other users' `CLAUDE.md` files | Warning | The walk would read files in shared directories that another user can write. | Fixed. The walk stops at directories owned by someone else or writable by group or others, and reads only files the user or root owns that others can't write. |
| Self-test rerun ambiguous | Warning | The rerun sentence followed the failure path, so it read as a way out of a confirmed failure. | Fixed. The rerun applies while the self-test is pending, and a failure lasts until restart. |
| Stale hook example | Note | ADR-0007 said a user hook could delay an MCP call, but hooks don't load. | Fixed: auto-approval is the example now. |
| "Aims to leave out" | Note | The global `CLAUDE.md` exclusion was tested, so "aims" was stale. | Fixed: "leaves out". |
| Wrong citations | Note | ADR-0003 cited ADR-0002 point 7 for the prompt order and the tool order. | Fixed. It cites the request steps, and ADR-0002 point 1 now states the tool order. |
| Incomplete status lines | Note | Amendment lines left out several changes. | Fixed in ADR-0006, ADR-0007 and the architecture status. |
| Wrong ADR for fixed values | Note | `configuration.md` said ADR-0007 explains the 30-second progress interval. | Fixed: ADR-0002 and ADR-0007. |
| Restart row too broad | Note | Only a turn waiting for approval replays after a restart. | Fixed. |
| Long line | Note | One prose line in the facts section ran past 80 columns. | Fixed. |
| Panes in `/tmp` | Note | With `PrivateTmp=yes`, the bridge can't see a pane's directory under `/tmp`. | Fixed: ADR-0002 point 8 states the limit, and the journal logs it. |
| Dangling question reference | Note | This note pointed at open question 7, which no longer exists. | Fixed. |
