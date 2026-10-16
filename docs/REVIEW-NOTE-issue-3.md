# Review notes: issue #3

Issue #3 added [ADR-0008](adr/0008-aur-release-package.md), which ships the
bridge as an Arch User Repository (AUR) package. The user then widened it to a
plain-language revision of every document in the repo. Each change got one
adversarial review, and this file records both reviews and how each finding was
handled. Under the project's one-pass rule, neither review was run again.

## ADR-0008

A fresh-context reviewer ran the pass on 2026-10-16. It checked the ADR against
the Arch Python package guidelines, the AUR submission guidelines, GitHub's
notice on the stability of source archives, `systemd-analyze --user unit-paths`,
`systemctl --user show-environment`, and Arch's package and file listings. The
verdict was "concerns": no critical findings, 4 warnings and 5 notes. The ADR
was revised to fix all of them.

| Finding | Severity | What was wrong | How it was handled |
|---|---|---|---|
| Python upgrades break the package | Warning | Modules install under a folder named for the Python minor version, and the AUR doesn't rebuild packages, so moving Arch to a new Python would stop the service from starting. | `depends` gains an upper bound, `python<3.15`. Each Python minor upgrade now means a `pkgrel` bump and a rebuild, as Consequences states. |
| Upgrade and removal undefined | Warning | After an upgrade the old process would keep running against replaced files, and removal would leave the enabled-unit symlink behind. | A table of `.install` hooks: `post_upgrade` prints the restart command, and `pre_remove` prints the disable command and how to switch Con back. The token stays in place on purpose. |
| Finding `claude` | Warning | The lookup was new runtime behaviour hidden in a packaging ADR. It preferred a possibly stale `/usr/bin/claude`, ignored `PATH`, and could leak the home directory through `/healthz`. | Moved to [ADR-0006](adr/0006-localhost-service-with-token.md): `BRIDGE_CLAUDE_BIN`, then `~/.local/bin/claude`, then `PATH`, logged at start-up. `/healthz` says only `claude not found`. The `config.py` row in the architecture document links to it, and settings go in a `systemctl --user edit` drop-in. |
| License for the packaging files | Warning | The AUR guidelines ask for a license on the packaging files themselves. | `packaging/aur/LICENSE` (0BSD) is copied to the AUR repo with the rest. |
| Unneeded `python-wheel` | Note | Current setuptools builds wheels without it. | Removed from `makedepends`. |
| Tarball checksums can change | Note | GitHub's checksums for tag tarballs can change. | Before running `updpkgsums`, compare the contents with the tag's commit, and never move a tag. |
| Entry point subcommands unclear | Note | The ADR didn't say what the bare command does or when the token is made. | With no subcommand it serves, and `setup` creates the token if it is missing. `post_install` lists `setup` first. |
| Wording that weakens the AUR case | Note | "For one user and one machine" argued against the package being useful to others. | Reworded to "for little benefit". |
| `PKGBUILD` one release behind | Note | The checksum needs the tag, so the in-repo `PKGBUILD` lags a release, and the bump has to go through a PR. | Stated as expected. The bump uses a `build:` commit, which doesn't trigger a release. |

## Plain-language revision of all documents

A second fresh-context reviewer compared every document with its last committed
version on 2026-10-16, looking for any change in meaning beyond the intended
ones. Those were the 6-minute idle timer in ADR-0003, finding IDs removed from
ADR-0007 and the review notes, ADR-0008's new layout, and the longer README. It
confirmed those landed and every link resolves. The verdict was "concerns": no
critical findings, 4 warnings and 11 notes, all about wording that drifted from
the original. All were fixed.

| Finding | Severity | What was wrong | How it was handled |
|---|---|---|---|
| Configuration claim too strong | Warning | ADR-0002 point 7 opened with "the user's own Claude Code configuration doesn't load", though the project `CLAUDE.md` does load and global `CLAUDE.md` exclusion is untested. | The lead is "Which Claude Code configuration loads" again. The global file is something the bridge "aims to leave out". |
| Python version reads as Arch's | Warning | ADR-0004 turned a local observation into a claim about Arch's Python release. | It now says the development machine had Python 3.14.8 as of 2026-10-16. The added reason about pip was dropped. |
| Quick-run rule narrowed | Warning | ADR-0005 lost the rule that any quick run ends when Con disconnects. | The rule leads the paragraph again. |
| pacman behaviour misdescribed | Warning | ADR-0008 said pacman "holds back the upgrade". In fact it refuses the whole transaction. | Both places now say pacman reports the conflict and refuses the upgrade. |
| Detail added to `setup` | Note | ADR-0001 said `setup` fills in the real token. | Now says it prints the lines for the user to paste. |
| ACP approval overstated | Note | ADR-0001 said Zed asks before each tool runs. | Now says Zed shows the tool calls and their approval prompts. |
| "Nobody has checked" | Note | ADR-0001, ADR-0002 and the architecture document turned "this project hasn't checked" into a claim about everyone. | Each now says this project hasn't checked or tested it. |
| Caching commitment unlisted | Note | ADR-0003 promised the first implementation issue would measure prompt caching, but the open questions didn't list it. | Added as open question 6, and ADR-0003 links to it. |
| Frequency overstated | Note | ADR-0005 said "every pause" and "every tab change". | "Every" removed. |
| OAuth claim too broad | Note | ADR-0005 said the bridge's login is always OAuth. | Now says OAuth is how Claude Code is usually signed in. |
| Retry mechanism narrowed | Note | The architecture document said only error codes control Con's retries, but the error wording matters too. | Now says error codes and wording. |
| Persistence claim too broad | Note | "Nothing else persists, because…" ignored the journal's metadata logs. | Now says no session files persist, and only the journal keeps run metadata. |
| ADR-0007 status incomplete | Note | The status summary left out Con's retries re-proposing approved tools. | Added. |
| Conventions source vague | Note | ADR-0008 cited "the project's git conventions", which no file in the repo holds. | Now names the owner's conventions: Conventional Commits, with tags made only by semantic-release. |
| Hypothetical read as observed | Note | This note described upgrade and removal problems in the past tense, and gave GitHub's checksum changes a cause no source states. | Changed to "would", and the checksum row now only says the checksums can change. |
