# ADR-0008: Ship the bridge as an Arch User Repository (AUR) release package

**Status:** Accepted, 2026-10-16. Amended for issue #5: semantic-release isn't
set up yet.

## Context

The bridge runs on Arch Linux as a systemd user service
([ADR-0006](0006-localhost-service-with-token.md)). Installing it by hand means
copying files that nothing tracks for updates or removal. Arch users expect
`pacman` to own installed files, and the AUR is where community packages live.
An AUR package is a git repo holding a `PKGBUILD`, the build script that
`makepkg` runs, and a `.SRCINFO` file generated from it.

Facts checked on 2026-10-16:

| Fact | Evidence |
|---|---|
| No AUR package is named `con-claude-bridge`. | AUR RPC v5 `info` and `search` |
| Claude Code is in the AUR as `claude-code` (2.1.301-1). On the development machine, `/usr/bin/claude` comes from that package (2.1.299-1). Anthropic's own installer puts `claude` in `~/.local/bin` instead. | AUR RPC; `pacman -Qo /usr/bin/claude` |
| Releases will be tagged by semantic-release, as `vX.Y.Z`. As of 2026-10-16 it isn't set up and no tags exist. | The owner's conventions: Conventional Commits, with only semantic-release creating tags; the repo's tag list |
| The repo has no license file, and GitHub reports no license. | `gh repo view --json licenseInfo` |
| The runtime needs only Python 3.12 or later and its standard library. | [ADR-0004](0004-python-standard-library-runtime.md) |
| `python -m installer` puts modules in a folder named for the Python minor version, such as `/usr/lib/python3.14/site-packages`. The AUR doesn't rebuild packages when Arch moves to a new Python. | `sysconfig.get_path('purelib')` |
| pacman reloads systemd user units when files under `usr/lib/systemd/user/` change, but it restarts nothing. | `/usr/share/libalpm/hooks/30-systemd-daemon-reload-user.hook` |
| A unit in `~/.config/systemd/user/` takes priority over one in `/usr/lib/systemd/user/`. | `systemd-analyze --user unit-paths` |

## Decision

There is one package, `con-claude-bridge`, built from the GitHub tarball of a
release tag:
`https://github.com/Towerism/con-claude-bridge/archive/refs/tags/v$pkgver.tar.gz`.
Its `sha256sums` are pinned, and `pkgver` is the tag without the `v`. Only
stable releases get packaged, because pre-release tags contain `-`, which
`pkgver` doesn't allow. There is no `-git` package.

This repo is the source of truth for packaging. `packaging/aur/` holds the
`PKGBUILD`, `con-claude-bridge.install` and a `LICENSE` file. The AUR repo,
`ssh://aur@aur.archlinux.org/con-claude-bridge.git`, gets copies of them plus a
`.SRCINFO` made with `makepkg --printsrcinfo`. The packaging files are under
the 0BSD license, which the AUR guidelines encourage, whatever license the
project itself picks.

### What the package contains

- `arch=('any')`, because the code is pure Python.
- `depends=('python>=3.12' 'python<3.15')`. The upper bound is one minor
  version past the Python the package was built with, and the packager edits it
  by hand. When Arch moves to a newer Python, pacman reports the conflict and
  refuses the upgrade, instead of the service failing later with
  `ModuleNotFoundError`. The bridge
  needs nothing else at run time.
- `makedepends=('python-build' 'python-installer' 'python-setuptools')`. The
  project builds with setuptools from `pyproject.toml`.
- `checkdepends=('python-pytest')`. `check()` runs the test suite, which uses a
  fake `claude` and works offline (ADR-0004).
- `optdepends=('claude-code: the claude CLI, if not installed another way')`.
  `claude-code` is optional because it is itself an AUR package, and anyone who
  used Anthropic's installer already has `claude` in `~/.local/bin`. ADR-0006
  says how the bridge finds it.
- `build()` runs `python -m build --wheel --no-isolation`, and `package()` runs
  `python -m installer --destdir="$pkgdir" dist/*.whl`, as the
  [Arch Python package guidelines](https://wiki.archlinux.org/title/Python_package_guidelines)
  describe. That installs the `/usr/bin/con-claude-bridge` entry point. With no
  subcommand it serves. `con-claude-bridge setup` creates the token if it is
  missing and prints the `con.conf` lines to paste (ADR-0006).
- The systemd user unit at `/usr/lib/systemd/user/con-claude-bridge.service`,
  with `ExecStart=/usr/bin/con-claude-bridge` and ADR-0006's hardening.
- The project's license file in `/usr/share/licenses/con-claude-bridge/`.

### What the package leaves to the user

Arch packages don't enable, start or stop services, and this one doesn't
either. Its `.install` file prints the commands to run:

| Hook | Prints |
|---|---|
| `post_install` | `con-claude-bridge setup`, then `systemctl --user enable --now con-claude-bridge` |
| `post_upgrade` | `systemctl --user restart con-claude-bridge`, so the old process stops running against replaced files |
| `pre_remove` | `systemctl --user disable --now con-claude-bridge`, and how to switch Con's provider back (ADR-0006) |

The token in `$XDG_CONFIG_HOME/con-claude-bridge/` belongs to the user, not the
package, so removing the package leaves it in place. Settings such as
`BRIDGE_CLAUDE_BIN` go in a drop-in made with
`systemctl --user edit con-claude-bridge`. A full copy of the unit in
`~/.config/systemd/user/` would hide the packaged unit, and its updates, from
then on.

### How a release reaches the AUR

The owner publishes by hand at first. For each stable release:

1. Bump `pkgver`, reset `pkgrel=1`, and update `sha256sums` with `updpkgsums`
   (from `pacman-contrib`).
2. Build in a clean chroot with `extra-x86_64-build` (from `devtools`), and
   lint the result with `namcap`.
3. Regenerate `.SRCINFO`. Commit the change here through a PR, using a `build:`
   commit so semantic-release doesn't cut a new release for it.
4. Copy `PKGBUILD`, `.SRCINFO`, the `.install` file and `LICENSE` into the AUR
   repo and push.

A packaging-only fix bumps `pkgrel` instead. So does a rebuild for a new Python
minor version, which also raises the bound in `depends`.

The checksum can only be computed once the tag exists, so the `PKGBUILD` inside
each release tarball is one version behind. That's harmless. If GitHub's
tarball for an existing tag ever gets a new checksum, compare its contents with
the tag's commit before running `updpkgsums`, and never move a tag.

Publishing from CI would need an AUR SSH key stored as a repo secret. That is
a separate decision for when releases are routine.

## Preconditions before the first AUR push

- The project needs a license, because without one nobody else has the right
  to use or redistribute the code. The owner picks one and adds a `LICENSE`
  file, and the `PKGBUILD`'s `license=()` uses its SPDX (Software Package Data
  Exchange) identifier.
- semantic-release must have tagged a first stable release that includes
  `pyproject.toml`, the entry point and the `setup` subcommand.

## Options considered

| Option | Why not chosen |
|---|---|
| A `-git` package built from `main` | Nothing is pinned, and a build can pick up unreleased work. Release tags exist anyway, so a release package costs nothing extra. |
| Both a release and a `-git` package | Two `PKGBUILD`s to maintain for little benefit. One can be added later with `provides` and `conflicts`. |
| Keep the `PKGBUILD` only in the AUR repo | Packaging changes would live apart from the code they track, and couldn't be reviewed in PRs here. |
| `depends=('claude-code')` | Forces the AUR copy of Claude Code on people who used Anthropic's installer, leaving them with two `claude` binaries. |
| `depends=('python>=3.12')` with no upper bound | A new Python minor version would break the installed package, and pacman wouldn't warn. |
| Keep a hand-installed unit in `~/.config/systemd/user/` | Nothing tracks the files, and every update is manual. |

## Consequences

- When the bridge comes from the package, the unit lives in
  `/usr/lib/systemd/user/` instead of the `~/.config/systemd/user/` path in
  ADR-0006. Its name, hardening and behaviour don't change. Delete any
  hand-installed copy when switching to the package, because that copy would
  take priority.
- Each release needs a short manual publishing step until CI takes it over.
- Each Python minor upgrade on Arch needs a `pkgrel` bump and a rebuild. Until
  then, pacman reports the `python<3.N` conflict and refuses the upgrade.
- Claude Code updates come from wherever the user installed it, not from this
  package.
