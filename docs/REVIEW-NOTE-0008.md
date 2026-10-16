# Review note: ADR-0008, the AUR release package

**Pass:** one adversarial review, run by a fresh-context reviewer on
2026-10-16 for issue #3. It checked claims against the Arch Python package
guidelines, the AUR submission guidelines, GitHub's archive stability notice,
`systemd-analyze --user unit-paths`, `systemctl --user show-environment` and
the Arch package and file listings.
**Verdict:** CONCERNS (0 critical, 4 warnings, 5 notes). All fixed, as listed
below. Per the one-pass rule, the review was not re-run.

| ID | Severity | Finding | Handling |
|---|---|---|---|
| W1 | WARNING | Modules install under a Python-version folder, and the AUR doesn't rebuild packages, so a Python minor upgrade breaks the service. | Fixed. `depends` gains an upper bound (`python<3.15`), and each Python minor upgrade means a `pkgrel` bump and rebuild. Added to Consequences. |
| W2 | WARNING | Upgrade and removal weren't defined. The old process keeps running after an upgrade, and removal leaves an enabled symlink. | Fixed. A hook table: `post_upgrade` prints the restart command, `pre_remove` prints the disable command and Con rollback. The token is left in place on purpose. |
| W3 | WARNING | The `claude` lookup was new runtime behaviour in a packaging ADR. Its order preferred a possibly stale `/usr/bin/claude`, it ignored `PATH`, and `/healthz` could leak the home directory. | Fixed. Moved to ADR-0006: `BRIDGE_CLAUDE_BIN`, then `~/.local/bin/claude`, then `PATH`. Logged at start-up. `/healthz` says only `claude not found`. The `config.py` row in architecture.md links to it. Settings go in a `systemctl --user edit` drop-in. |
| W4 | WARNING | The AUR guidelines ask for a license on the packaging files themselves. | Fixed. `packaging/aur/LICENSE` (0BSD) is copied to the AUR repo. |
| N1 | NOTE | `python-wheel` isn't needed with current setuptools. | Fixed: removed from `makedepends`. |
| N2 | NOTE | GitHub tarball checksums can change. | Fixed: compare contents with the tag's commit before `updpkgsums`, and never move a tag. |
| N3 | NOTE | The entry point's subcommands were unclear. | Fixed: no subcommand serves, and `setup` creates the token if it is missing. `post_install` lists `setup` first. |
| N4 | NOTE | "One user and one machine" weakens the AUR usefulness case. | Fixed: reworded to "for little benefit". |
| N5 | NOTE | The in-repo PKGBUILD is one release behind, and the bump goes through a PR. | Fixed: stated as expected, with a `build:` commit that doesn't trigger a release. |
