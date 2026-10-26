# Tasks

Scope note: this change ships implementation. It creates `packages/gridline/debian/` and verifies it by building and installing on the development host, which is Ubuntu 26.04.1 LTS resolute on amd64 and therefore the declared baseline. The packages own directories and carry no content. No maintainer script is written, no dependency entry is named, and `gridline-desktop` is not built.

Build command is `dpkg-buildpackage`; `debuild`, `lintian` and `sbuild` are not installed and no task may depend on them.

## 1. Source package metadata

- [x] 1.1 Create `packages/gridline/debian/source/format` containing `3.0 (native)` and verify `dpkg-source --print-format packages/gridline` reports that format
- [x] 1.2 Create `packages/gridline/debian/changelog` with version `0.0.1`, distribution `resolute`, urgency `low`, and a body stating that the packages are structural and carry no content, and verify `dpkg-parsechangelog -l packages/gridline/debian/changelog` reports version `0.0.1` without a parse warning
- [x] 1.3 Create `packages/gridline/debian/control` declaring source `gridline` and the three binary packages `gridline-session`, `gridline-config` and `gridline-tools`, each with a description saying what it will own and that it ships no content yet, and verify every binary package name begins with `gridline-` and that no `gridline-desktop` stanza is present
- [x] 1.4 Set `Homepage` in `debian/control` to the canonical project URL recorded in `openspec/config.yaml`, and verify the value matches that file character for character
- [x] 1.5 Create `packages/gridline/debian/copyright` in machine-readable format 1.0 with one `Files: *` stanza, `License: MIT`, and the MIT text reproduced inline, and verify the text matches `LICENSE` in the repository root and that no stanza references `/usr/share/common-licenses/`

## 2. Build rules and path ownership

- [x] 2.1 Create `packages/gridline/debian/rules` as an executable debhelper sequence with no overrides beyond what the gate in task 2.5 requires, and verify `dpkg-buildpackage -us -uc -b` run from `packages/gridline` completes and produces exactly three `.deb` files
- [x] 2.2 Create `packages/gridline/debian/gridline-session.dirs` declaring `usr/libexec/gridline` and nothing else, and verify `dpkg-deb -c` on the built package lists that directory and no file beyond the changelog and copyright Debian policy requires
- [x] 2.3 Create `packages/gridline/debian/gridline-config.dirs` declaring `etc/xdg/gridline` and `etc/xdg/gridline/generated`, and verify `dpkg-deb -c` lists both directories, no file beyond the policy-required documentation, and nothing declared at the top level of `etc/xdg`
- [x] 2.4 Create `packages/gridline/debian/gridline-tools.dirs` declaring `usr/share/gridline` and nothing else, and verify `dpkg-deb -c` lists that directory and no file beyond the policy-required documentation
- [x] 2.5 Add the ownership gate as a script in the source package, invoked from `debian/rules` at the last point where every staging tree is complete, which is before `dh_builddeb` and not after `dh_install`, failing the build when two binary packages would ship the same file or declare the same directory, and verify it fails on a duplicated file installed by `.install`, fails on a duplicated path created by a helper that runs after `dh_install`, fails on a duplicated declared directory, and reports a file count that matches what the built packages actually contain
- [x] 2.6 Verify no install manifest declares a path an archive package already owns, by querying `dpkg -S` for each declared path and confirming each one is Gridline-named

## 3. Install verification on the baseline

- [x] 3.1 Install the three built packages together and verify the installation completes with no file conflict reported by `dpkg`
- [x] 3.2 Verify `dpkg -L` attributes each claimed directory to the package the capability assigns it to: `usr/libexec/gridline/` to `gridline-session`, both `etc/xdg/gridline/` paths to `gridline-config`, and `usr/share/gridline/` to `gridline-tools`
- [x] 3.3 Verify that no Gridline install manifest declares `/usr/share/wayland-sessions`, `/usr/share/xdg-desktop-portal`, `/etc/xdg` or `/usr/libexec`, and record that `dpkg -S` on those paths does list a Gridline package, because dpkg records the shared parents of any nested path and every package installing under them does the same
- [x] 3.4 Verify `gridline-tools` installs on its own, without the other two, and that no Gridline package depends on another at this stage

## 4. Removal, upgrade and downgrade on the baseline

- [x] 4.1 Remove the three packages and verify no Gridline-named directory remains under `/etc/xdg`, `/usr/libexec` or `/usr/share`, that the shared parents still exist, and that querying their owners returns the same archive packages as before Gridline was installed
- [x] 4.2 Build `0.0.2` from a changelog entry, upgrade from `0.0.1`, and verify the upgrade completes and `dpkg -l` reports the new version for all three
- [x] 4.3 Downgrade to `0.0.1` and verify the downgrade completes and leaves no file from `0.0.2` behind, then remove the packages to return the host to its prior state
- [x] 4.4 Verify the host is back to its prior state by confirming no `gridline-*` package appears in `dpkg -l` and no Gridline-named directory remains at the claimed paths

## 5. Documentation and sequence

- [x] 5.1 Record the build command and the three resulting packages in `packages/gridline/README.md`, and verify the documented command is the one task 2.1 ran rather than `debuild`
- [x] 5.2 Record the declared and deliberately undeclared paths in `docs/packaging.md`, with the reason shared directories are not declared and the note that dpkg still records them as shared parents, and verify the declared paths match the `.dirs` manifests exactly
- [x] 5.3 Verify every requirement added in `specs/package-delivery/spec.md` appears in this change's attribution table in `proposal.md` with exactly one owning change, and that no requirement is listed twice or omitted
- [x] 5.4 Verify `gridline-desktop` is absent from the build output and still recorded as reserved in `docs/package-matrix.md` and `packages/gridline/README.md`

## Workflow follow-up

- Run `/opsx:verify` once the tree builds and the verification tasks have run.
- Archive this change, then propose `add-gdm-wayland-session`, which ships the first real file into the directories created here.
