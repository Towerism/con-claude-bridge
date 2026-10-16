# Review note: finishing the architecture documents

This note records the one adversarial review of issue #5's documents and how
each finding was handled. All 21 findings are fixed in commit `a8b5185`.

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
| Reset at shutdown triggers retry | Warning | At shutdown a reset connection could reach Con as `connection reset`, a retry trigger, so approved tools would be proposed again. | Fixed. Streams end with an error chunk free of trigger words, then `[DONE]`, then a normal close. Con's handling of that chunk is open question 7. |
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
