# ADR-0008: Ship the bridge as an Arch User Repository (AUR) release package

**Status:** Proposed, 2026-10-16

## Context

The bridge runs on Arch Linux as a systemd user service
([ADR-0006](0006-localhost-service-with-token.md)). Without a package, installing
it means copying files by hand, and nothing tracks them for updates or removal.
Arch users expect `pacman` to own installed files, and the AUR is where
community packages live. An AUR package is a git repo holding a `PKGBUILD` (the
build script `makepkg` runs) and a generated `.SRCINFO` file.

Facts checked on 2026-10-16:

| Fact | Evidence |
|---|---|
| No AUR package is named `con-claude-bridge`. | AUR RPC v5 `info` and `search` |
| Claude Code is packaged in the AUR as `claude-code` (2.1.301-1). On this machine `/usr/bin/claude` comes from it (2.1.299-1). Anthropic's own installer puts `claude` in `~/.local/bin` instead. | AUR RPC; `pacman -Qo /usr/bin/claude` |
| Releases are tagged by semantic-release as `vX.Y.Z`. | Global git conventions for this repo |
| The repo has no license file, and GitHub reports no license. | `gh repo view --json licenseInfo` |
| The runtime needs only Python 3.12 or later and its standard library. | [ADR-0004](0004-python-standard-library-runtime.md) |
| `python -m installer` puts modules under a folder named for the Python minor version, `/usr/lib/python3.14/site-packages` here. The AUR doesn't rebuild packages when Arch moves to a new Python. | `sysconfig.get_path('purelib')` |
| pacman reloads systemd user units when files under `usr/lib/systemd/user/` change, but restarts nothing. | `/usr/share/libalpm/hooks/30-systemd-daemon-reload-user.hook` |
| `~/.config/systemd/user/` takes priority over `/usr/lib/systemd/user/`. | `systemd-analyze --user unit-paths` |

## Decision

1. One release package, `con-claude-bridge`. It is built from the GitHub
   tarball of a release tag,
   `https://github.com/Towerism/con-claude-bridge/archive/refs/tags/v$pkgver.tar.gz`,
   with its `sha256sums` pinned. `pkgver` is the tag without the `v`. Only
   stable releases get packaged, because pre-release tags contain `-`, which
   `pkgver` doesn't allow. There is no `-git` package.
2. The PKGBUILD lives in this repo at `packaging/aur/PKGBUILD`, next to
   `con-claude-bridge.install` and a `LICENSE` file. This repo is the source of
   truth. The AUR repo (`ssh://aur@aur.archlinux.org/con-claude-bridge.git`)
   only receives copies plus a `.SRCINFO` made with `makepkg --printsrcinfo`.
   The packaging files are licensed 0BSD, as the AUR guidelines encourage,
   whatever license the project itself uses.
3. The package's contents:
   - `arch=('any')`, because the package is pure Python.
   - `depends=('python>=3.12' 'python<3.15')`. The upper bound is the next
     Python minor version after the one the package was built with, written by
     hand. When Arch moves to a new Python, pacman then refuses the upgrade or
     names the conflict, instead of the service failing with
     `ModuleNotFoundError`. The bridge needs nothing else at run time.
   - `makedepends=('python-build' 'python-installer' 'python-setuptools')`.
     The project builds with setuptools from `pyproject.toml`.
   - `checkdepends=('python-pytest')`. `check()` runs the test suite, which
     uses a fake `claude` and works offline (ADR-0004).
   - `optdepends=('claude-code: the claude CLI, if not installed another way')`.
     `claude-code` isn't a hard dependency. It is an AUR package itself, and
     users of Anthropic's installer already have `claude` in `~/.local/bin`.
     ADR-0006 says how the bridge finds `claude`.
   - `build()` runs `python -m build --wheel --no-isolation`. `package()` runs
     `python -m installer --destdir="$pkgdir" dist/*.whl`, following the
     [Arch Python package guidelines](https://wiki.archlinux.org/title/Python_package_guidelines).
     This installs the `/usr/bin/con-claude-bridge` entry point. Run with no
     subcommand, it serves. `con-claude-bridge setup` creates the token if it
     is missing and prints the `con.conf` lines to paste (ADR-0006).
   - The systemd user unit goes to
     `/usr/lib/systemd/user/con-claude-bridge.service`, with
     `ExecStart=/usr/bin/con-claude-bridge` and ADR-0006's hardening.
   - The project's license file goes to `/usr/share/licenses/con-claude-bridge/`.
4. The package never enables, starts or stops the service, because Arch
   packages leave that to the user. The `.install` file prints what to run:

   | Hook | Prints |
   |---|---|
   | `post_install` | `con-claude-bridge setup`, then `systemctl --user enable --now con-claude-bridge` |
   | `post_upgrade` | `systemctl --user restart con-claude-bridge`, so the old process stops using replaced files |
   | `pre_remove` | `systemctl --user disable --now con-claude-bridge`, and switching Con's provider back (ADR-0006) |

   The token under `$XDG_CONFIG_HOME/con-claude-bridge/` belongs to the user,
   not the package. Removal leaves it in place on purpose. Settings such as
   `BRIDGE_CLAUDE_BIN` go in a drop-in made with
   `systemctl --user edit con-claude-bridge`, never in a copied unit, because a
   copy in `~/.config/systemd/user/` hides the packaged unit from then on.
5. The owner publishes by hand at first. For each stable release:
   1. Bump `pkgver`, reset `pkgrel=1` and update `sha256sums` with
      `updpkgsums` (from `pacman-contrib`).
   2. Build in a clean chroot with `extra-x86_64-build` (from `devtools`) and
      lint with `namcap`.
   3. Regenerate `.SRCINFO`. Commit the PKGBUILD change here through a PR with
      a `build:` commit, which doesn't trigger a release. Then copy `PKGBUILD`,
      `.SRCINFO`, the `.install` file and `LICENSE` into the AUR repo and push.

   Packaging-only fixes bump `pkgrel` instead, and so does a rebuild for a new
   Python minor version, which also raises the upper bound in `depends`.

   The checksum needs the tag, so the PKGBUILD inside each release's tarball is
   one version behind. That is expected. If GitHub's tarball checksum for an
   existing tag ever changes, compare its contents with the tag's commit before
   running `updpkgsums`, and never move a tag.

   Automating this in CI needs an AUR SSH key stored as a repo secret. That is
   a separate decision, made once releases are routine.

## Preconditions before the first AUR push

- The project needs a license. The repo has none, so others have no right to use
  or redistribute the code. The owner picks one and adds a `LICENSE` file, and
  `license=()` uses its SPDX (Software Package Data Exchange) license
  identifier.
- semantic-release must have tagged a first stable release, with
  `pyproject.toml`, the entry point and the `setup` subcommand in place.

## Options considered

| Option | Why not chosen |
|---|---|
| A `-git` package built from `main` | Nothing is pinned, and each build can pick up unreleased work. The tags already exist, so a release package costs nothing extra. |
| Both a release and a `-git` package | Two PKGBUILDs to maintain for little benefit. Can be added later with `provides`/`conflicts`. |
| PKGBUILD kept only in the AUR repo | Changes to packaging would be split from the code they track, and couldn't be reviewed in PRs here. |
| `depends=('claude-code')` | Forces the AUR copy of Claude Code on users of Anthropic's installer, giving them two `claude` binaries. |
| `depends=('python>=3.12')` with no upper bound | A new Python minor version would break the installed package with no warning from pacman. |
| Keep the hand-installed unit in `~/.config/systemd/user` | Nothing tracks the files, and updates are manual. |

## Consequences

- This changes where [ADR-0006](0006-localhost-service-with-token.md) puts the
  unit when the bridge is installed from the package:
  `/usr/lib/systemd/user/` instead of `~/.config/systemd/user/`. The unit's
  name, hardening and behaviour stay the same. Remove any hand-installed copy
  in `~/.config/systemd/user/` when switching to the package, because it takes
  priority.
- Each release needs a short manual publishing step until CI does it.
- Each Python minor version upgrade on Arch needs a `pkgrel` bump and a rebuild.
  Until then, pacman reports the `python<3.N` conflict.
- `claude-code` updates come from wherever the user installed it, not from this
  package.
