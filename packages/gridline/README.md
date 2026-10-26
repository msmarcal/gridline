# gridline source package

This is the source package that produces every Gridline binary package.

## Repository layout

```
packages/
  gridline/              <- this source package
    debian/              <- packaging; debian/ never appears at the repo root
      source/            <- tracked
      patches/           <- tracked, unused by a native package
    src/
      adapters/          <- the compositor adapter layer, the isolation boundary
        hyprland/        <- the only adapter in this phase
docs/                    <- project documentation
openspec/                <- specifications and changes
```

Every Gridline source package lives under `packages/<name>/` and owns its own `debian/` directory. Nothing is packaged from the repository root, so `debian/` at the top level would be a mistake rather than an alternative.

## One source package, several binary packages

`packages/gridline/` produces `gridline-session`, `gridline-config` and `gridline-tools`, and will later produce `gridline-desktop`.

**Why one source package.** The binaries are developed, versioned and released together. One source package gives them one version and one changelog, and it turns a file-ownership conflict into a build failure rather than an install-time surprise, which is what makes the "no file is claimed twice" requirement in `openspec/specs/package-delivery/spec.md` cheap to verify.

**Why not one per binary.** Four source packages would buy independent release cadence the project does not need, at the cost of a four-way version matrix and cross-package dependency pinning on every change.

**Room for more.** The `packages/<name>/` layout exists so that additional *source* packages can be introduced later, not so that each binary gets its own. A vendored compositor or a shell would be a reasonable second source package. Adding one requires no restructuring.

The trade-off accepted here is that a fix to `gridline-tools` ships a new version of all of them. If that becomes a problem, splitting out a source package is the escape hatch, and the layout is already shaped for it.

## Source format

`3.0 (native)`: the upstream and the packaging are the same tree, so there is no upstream tarball to patch.

`debian/patches/` is still tracked, and stays useful if a later source package vendors third-party code and adopts `3.0 (quilt)`. Doing so needs no change to `.gitignore`.

## The `gridline-` prefix is a convention the tooling relies on

**Every binary package produced from this repository is named `gridline-<something>`.**

This is not a naming preference. The repository's `.gitignore` ignores build staging trees with:

```
packages/*/debian/gridline-*/
```

One glob covers every current and future binary package, which is why there is no per-package ignore line to forget when a package is added. It works because `debian/source/` and `debian/patches/` do not match the prefix, so both stay tracked without needing un-ignore rules.

A binary package named without the prefix would have its staging tree show up as untracked files after every build. If a future package genuinely cannot carry the prefix, the `.gitignore` glob has to be revisited in the same change.

## Build

```
dpkg-buildpackage -us -uc -b
```

Run from `packages/gridline/`. It produces three binary packages in the parent directory:

| Package | Declares |
|---|---|
| `gridline-session_<version>_all.deb` | `/usr/libexec/gridline/` |
| `gridline-config_<version>_all.deb` | `/etc/xdg/gridline/`, `/etc/xdg/gridline/generated/` |
| `gridline-tools_<version>_all.deb` | `/usr/share/gridline/` |

`debuild` is not used: it is not installed on the supported baseline, and `dpkg-buildpackage` is what the packaging is verified against.

The build runs `tools/check-path-ownership` as `execute_before_dh_builddeb`, which is the last point where every staging tree is complete. It fails the build when two binary packages would ship the same file or declare the same directory, so an ownership collision is a build failure rather than a file conflict discovered at install time. Intermediate parents such as `/etc/xdg` are shared by construction and are not collisions.

The anchor matters. `dh_install` is not the last helper to populate a staging tree, so a gate placed there misses everything `dh_installdocs`, `dh_link`, `dh_compress` and `dh_fixperms` stage afterwards. On a clean build the gate reports six files and four declared directories; if it ever reports fewer files than the packages actually contain, it is looking at an incomplete tree.

Build-time ignore patterns are documented above; staging artifacts are ignored by explicit path, so a clean tree after a build is the expected state.

## `gridline-desktop` is a reserved boundary

**The name `gridline-desktop` is reserved. It has no implementation in the `establish-gridline-vision` change, and that change produces no `gridline-desktop` package.**

This matches two places and contradicts neither:

- `openspec/specs/package-delivery/spec.md` specifies it as a reserved metapackage boundary, with a scenario asserting that no `gridline-desktop` package is produced by this change, and a separate scenario covering what it must contain when a later change does deliver it.
- The deferred requirements table in `openspec/changes/establish-gridline-vision/proposal.md` lists it as deliberately having no downstream change assigned, because the reservation is satisfied by the reserving change itself. The metapackage ships once the component set it would depend on is stable, which no scheduled change reaches.

When it is delivered, it carries no executable, configuration or data files beyond its own packaging metadata. Its entire purpose is to depend on the other packages.

## Related documents

- [docs/package-matrix.md](../../docs/package-matrix.md) for what each binary package owns
- [docs/compositor-adapter.md](../../docs/compositor-adapter.md) for the contract that governs `src/adapters/`
- [docs/architecture.md](../../docs/architecture.md) for the layers and their boundaries
