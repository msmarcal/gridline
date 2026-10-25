# Design

## Context

See proposal.md - Why. `docs/packaging.md` and the 13 requirements in `openspec/specs/package-delivery/spec.md` settle the rules; this change turns them into a tree `dpkg-buildpackage` accepts.

What the environment allows, checked rather than assumed:

- **The development host is the baseline.** Ubuntu 26.04.1 LTS resolute, amd64. Install and removal behavior can be verified here instead of described.
- **`dpkg-buildpackage` and `dh` are present**, with debhelper 13.31ubuntu1.
- **`debuild`, `lintian`, `dh_make` and `sbuild` are absent.** The build command is therefore `dpkg-buildpackage`, and no task may depend on `lintian` output without first changing the host's package set, which is out of scope.
- **The candidate parent directories are already owned.** `/usr/share/wayland-sessions` belongs to `ubuntu-session`; `/usr/share/xdg-desktop-portal` to `xdg-desktop-portal`, `gnome-session-common` and `kwallet6`; `/etc/xdg` to `systemd`, `snapd` and others; `/usr/libexec` to `polkitd` and the `sssd` packages. This is the finding that shaped the decisions below.

## Goals / Non-Goals

**Goals:**

- A source package that builds three binary packages from a clean checkout with one command.
- Path ownership asserted where it is meaningful, and deliberately not asserted where it would be noise.
- A build gate that makes a duplicated path a build failure.
- Verification on the baseline covering install, removal, upgrade and downgrade.

**Non-Goals:**

- Any shipped content. The packages own directories and carry no files.
- Dependency entries. The fields exist; the entries belong to the changes that use components.
- Maintainer scripts, `lintian` cleanliness, and CI.

## Decisions

### Only Gridline-named directories are claimed

| Package | Claims | Does not claim |
|---|---|---|
| `gridline-session` | `/usr/libexec/gridline/` | `/usr/libexec`, `/usr/share/wayland-sessions` |
| `gridline-config` | `/etc/xdg/gridline/`, `/etc/xdg/gridline/generated/` | `/etc/xdg`, `/usr/share/xdg-desktop-portal` |
| `gridline-tools` | `/usr/share/gridline/` | `/usr/bin` |

*Why:* a directory in Debian is shared ownership. Several packages may own `/etc/xdg` at once, so claiming it produces no conflict, no error, and no useful assertion either: `dpkg -S /etc/xdg` would simply list Gridline alongside `systemd` and `snapd`. The claim that means something is the one only Gridline can make.

*Why this is a requirement and not only a decision:* the failure is silent. A `.install` line claiming a shared directory installs cleanly and looks correct, so nothing catches it except a rule written down. It also matters for the fallback guarantee: a Gridline package that owns `/etc/xdg` is a Gridline package whose removal can touch a directory the GNOME session depends on.

*Consequence for later changes:* the session entry lands in `/usr/share/wayland-sessions/` as a Gridline-named file, and `gridline-portals.conf` lands in `/usr/share/xdg-desktop-portal/` the same way. Files, not directories.

### The generated configuration destination is `/etc/xdg/gridline/generated/`

`gridline-config` owns both `/etc/xdg/gridline/` and `/etc/xdg/gridline/generated/`, and ships neither with content.

*Why under the Gridline config directory:* the generated output has to be on the search path the launcher prepends to `XDG_CONFIG_DIRS`, or the components never read it. A destination outside that path would need a second mechanism to be found.

*Why a separate subdirectory:* it keeps the authorship boundary visible in the filesystem. Everything directly under `/etc/xdg/gridline/` is administrator-authored and registered as a conffile; everything under `generated/` is output, is not a conffile, and goes away with the package. An administrator who has to ask which kind a file is has already been failed by the layout.

*Named here because ownership requires a name.* `define-debian-package-architecture` assigned the ownership requirement to this change precisely because `.install` manifests are where the boundary becomes real, and a manifest needs a path.

### One source package, three binary packages, `3.0 (native)`

Confirms what `docs/packaging.md` and `packages/gridline/README.md` already record. `gridline-desktop` is not built.

*Why `debian/rules` stays a plain debhelper sequence:* there is nothing to compile. `dh $@` with the directories installed from `.install` manifests is the whole build, and any override added now would be machinery with no work to do.

### The ownership gate compares built packages, not the source tree

The gate runs after the binary packages are staged and compares the file lists dpkg would install.

*Why after staging rather than by reading the `.install` files:* an `.install` manifest may use a glob, and two globs can overlap without looking like it. Comparing what was actually staged catches the overlap a textual comparison would miss.

*Where it lives:* a script in the source package, invoked from `debian/rules` after `dh_install`, so a duplicated path fails `dpkg-buildpackage` rather than waiting for someone to install both packages.

*The gate does not check the other half.* Whether a claimed directory belongs to an archive package is a different question, answered against the installed system rather than against the build, and the spec requires it to surface in build or review. The honest placement is a separate check the build can run only on a machine that has the archive package installed, which is why the requirement says "build or review" rather than naming one.

### Version `0.0.1`, and the changelog says what the package is

The first changelog entry is `0.0.1` targeting `resolute`, with `urgency=low`, and its body states that the packages are structural and carry no content.

*Why that matters:* anyone who installs these packages gets three entries in `dpkg -l` that do nothing. A changelog that describes features would be a lie discoverable by `apt changelog`, and a changelog that says nothing leaves the reader to guess.

### `debian/copyright` reproduces the MIT text

Machine-readable format 1.0, one `Files: *` stanza, `License: MIT` with the full text inline.

*Why inline:* `/usr/share/common-licenses/` carries the GPL and LGPL texts and not MIT, so a reference would point at a file that does not exist. `docs/packaging.md` already records this.

## Risks / Trade-offs

- **Empty packages invite the question of what they are for** → The changelog and the package descriptions say so plainly. The alternative, delaying packaging until there is content, is what produced the gap this change exists to close: seven changes that each assume a package and none that makes one.
- **Verification happens on one machine, which is also the author's** → A clean host would be stronger evidence, and `add-ci-vm-smoke-tests` is where that happens. What this change can prove is that the behavior works somewhere real, which is more than the specs had before.
- **Installing and removing packages on the development host** → The packages own four directories and carry no files, and removal is one of the verified behaviors. The risk is bounded by the thing being tested.
- **`/etc/xdg/gridline/generated/` puts generated output under `/etc`** → Unusual for output, but required by the search path, and it has precedent in the archive. The mitigation is the separate subdirectory and the conffile rule, so the two kinds never mix in one directory.
- **The gate catches Gridline-against-Gridline overlap only** → A claim against an archive package's directory still depends on review. Naming the limit in the design and in the requirement is the honest handling; closing it needs a check against an installed system, which belongs with the CI change.
- **A later change may need a directory this change did not create** → Adding one is cheap. The expensive mistake would be creating directories speculatively now, which is why the list covers only what the capability already assigns an owner to.

## Migration Plan

Nothing is installed anywhere, so there is nothing to migrate from. The rollback path for the verification tasks is the removal they already test: `apt remove` returns the host to its prior state, and the clean-removal requirement is what makes that claim checkable rather than hopeful.

## Open Questions

- Whether the ownership gate eventually grows a check against installed archive packages, or that stays with `add-ci-vm-smoke-tests`. Deferred: the requirement accepts either placement, and the build-time half is the part this change can deliver.
- Whether `/usr/libexec/gridline/` or a `/usr/lib/gridline/` path is the better home for session helpers. Deferred: it changes one `.install` line, and the first change that ships a helper is better placed to judge.
