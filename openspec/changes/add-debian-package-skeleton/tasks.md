# Tasks

Scope note: this change ships implementation. It creates `packages/gridline/debian/` and verifies it by building and installing on the development host, which is Ubuntu 26.04.1 LTS resolute on amd64 and therefore the declared baseline. The packages own directories and carry no content. No maintainer script is written, no dependency entry is named, and `gridline-desktop` is not built.

Build command is `dpkg-buildpackage`; `debuild`, `lintian` and `sbuild` are not installed and no task may depend on them.

## 1. Source package metadata

- [ ] 1.1 Create `packages/gridline/debian/source/format` containing `3.0 (native)` and verify `dpkg-source --print-format packages/gridline` reports that format
- [ ] 1.2 Create `packages/gridline/debian/changelog` with version `0.0.1`, distribution `resolute`, urgency `low`, and a body stating that the packages are structural and carry no content, and verify `dpkg-parsechangelog -l packages/gridline/debian/changelog` reports version `0.0.1` without a parse warning
- [ ] 1.3 Create `packages/gridline/debian/control` declaring source `gridline` and the three binary packages `gridline-session`, `gridline-config` and `gridline-tools`, each with a description saying what it will own and that it ships no content yet, and verify every binary package name begins with `gridline-` and that no `gridline-desktop` stanza is present
- [ ] 1.4 Set `Homepage` in `debian/control` to the canonical project URL recorded in `openspec/config.yaml`, and verify the value matches that file character for character
- [ ] 1.5 Create `packages/gridline/debian/copyright` in machine-readable format 1.0 with one `Files: *` stanza, `License: MIT`, and the MIT text reproduced inline, and verify the text matches `LICENSE` in the repository root and that no stanza references `/usr/share/common-licenses/`

## 2. Build rules and path ownership

- [ ] 2.1 Create `packages/gridline/debian/rules` as an executable debhelper sequence with no overrides beyond what the gate in task 2.5 requires, and verify `dpkg-buildpackage -us -uc -b` run from `packages/gridline` completes and produces exactly three `.deb` files
- [ ] 2.2 Create `packages/gridline/debian/gridline-session.install` claiming `usr/libexec/gridline/` and nothing else, and verify `dpkg-deb -c` on the built package lists that directory and no file
- [ ] 2.3 Create `packages/gridline/debian/gridline-config.install` claiming `etc/xdg/gridline/` and `etc/xdg/gridline/generated/`, and verify `dpkg-deb -c` lists both directories, no file, and nothing at the top level of `etc/xdg`
- [ ] 2.4 Create `packages/gridline/debian/gridline-tools.install` claiming `usr/share/gridline/` and nothing else, and verify `dpkg-deb -c` lists that directory and no file
- [ ] 2.5 Add the ownership gate as a script in the source package, invoked from `debian/rules` after `dh_install`, failing the build when two binary packages would ship the same path, and verify it fails on a deliberately duplicated directory and passes once the duplicate is removed
- [ ] 2.6 Verify no built package claims a directory an archive package already owns, by querying `dpkg -S` for each parent of the claimed paths and confirming the claimed paths themselves are Gridline-named

## 3. Install verification on the baseline

- [ ] 3.1 Install the three built packages together and verify the installation completes with no file conflict reported by `dpkg`
- [ ] 3.2 Verify `dpkg -L` attributes each claimed directory to the package the capability assigns it to: `usr/libexec/gridline/` to `gridline-session`, both `etc/xdg/gridline/` paths to `gridline-config`, and `usr/share/gridline/` to `gridline-tools`
- [ ] 3.3 Verify `dpkg -S` on `/usr/share/wayland-sessions`, `/usr/share/xdg-desktop-portal`, `/etc/xdg` and `/usr/libexec` lists no Gridline package, which is the scenario that would pass silently if the claim were wrong
- [ ] 3.4 Verify `gridline-tools` installs on its own, without the other two, and that no Gridline package depends on another at this stage

## 4. Removal, upgrade and downgrade on the baseline

- [ ] 4.1 Remove the three packages and verify no Gridline-owned directory remains under `/etc/xdg`, `/usr/libexec` or `/usr/share`, and that the shared parents still exist and are still owned by their archive packages
- [ ] 4.2 Build `0.0.2` from a changelog entry, upgrade from `0.0.1`, and verify the upgrade completes and `dpkg -l` reports the new version for all three
- [ ] 4.3 Downgrade to `0.0.1` and verify the downgrade completes and leaves no file from `0.0.2` behind, then remove the packages to return the host to its prior state
- [ ] 4.4 Verify the host is back to its prior state by confirming no `gridline-*` package appears in `dpkg -l` and no Gridline-named directory remains at the claimed paths

## 5. Documentation and sequence

- [ ] 5.1 Record the build command and the three resulting packages in `packages/gridline/README.md`, and verify the documented command is the one task 2.1 ran rather than `debuild`
- [ ] 5.2 Record the claimed and deliberately unclaimed paths in `docs/packaging.md`, with the reason shared directories are not claimed, and verify the claimed paths match the `.install` manifests exactly
- [ ] 5.3 Verify every requirement added in `specs/package-delivery/spec.md` appears in this change's attribution table in `proposal.md` with exactly one owning change, and that no requirement is listed twice or omitted
- [ ] 5.4 Verify `gridline-desktop` is absent from the build output and still recorded as reserved in `docs/package-matrix.md` and `packages/gridline/README.md`

## Workflow follow-up

- Run `/opsx:verify` once the tree builds and the verification tasks have run.
- Archive this change, then propose `add-gdm-wayland-session`, which ships the first real file into the directories created here.
