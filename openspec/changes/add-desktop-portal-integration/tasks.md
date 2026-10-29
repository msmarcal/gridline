# Tasks

## 1. The mapping

- [x] 1.1 List the portal interfaces the session relies on, deriving them from the scenarios in `openspec/specs/platform-integration/spec.md` rather than from the backends' own claims, and record the list with the scenario each interface comes from. An interface nothing in the specification relies on does not belong in the mapping
- [x] 1.2 Add the mapping as a template in `src/config/` naming no compositor, with each adapter declaring the portal backend serving its compositor inside the isolation boundary, and a build step assembling them. Entries take the backend's short name, the basename of its `.portal` file, which `portals.conf(5)` defines and which is not the backend's D-Bus name. Verify the generated file's explicit entries match the list from 1.1 in both directions, that the default leaves no interface resolving through the frontend's own matching, and that the isolation gate passes
- [x] 1.3 Verify no entry names `gnome.portal`'s backend, and record why for any interface where that leaves no entry at all: the session relies on it and something else must serve it, or it does not rely on it and the absence is correct
- [x] 1.4 Verify every backend the file names is provided by a package the baseline carries, by resolving each short name to the `.portal` file whose basename it is and that file to its owning package with `dpkg -S`

## 2. Packaging

- [x] 2.1 Add `packages/gridline/debian/gridline-config.install` placing the file at `usr/share/xdg-desktop-portal/gridline-portals.conf`, and verify with `dpkg -c` on the built package that the path is exactly that, lower-cased and inside the portal directory, since neither is a preference
- [x] 2.2 Declare `xdg-desktop-portal` and `xdg-desktop-portal-gtk` as `Depends` of `gridline-config`, and `xwayland` as `Depends` of `gridline-session`, and verify each with `dpkg-deb -f` on the built packages rather than by reading the control file
- [x] 2.3 Verify the three packages are no longer relied on informally, by confirming they were present on the baseline only through `ubuntu-session` and `libmutter` before this change and are declared by Gridline after it
- [x] 2.4 Verify the mapping is not registered as a conffile, since it is Gridline's own decision rather than a file an administrator is expected to edit, and record where an administrator overrides it instead
- [x] 2.5 Build and verify both existing gates still pass, and that the ownership gate's file count rose by exactly one

## 3. Documentation

- [x] 3.1 Replace the passage in `docs/component-ownership.md` that describes the accidental path as the current state, and verify the document no longer presents matching through the compositor's name as how resolution works
- [x] 3.2 Record the mapping in `docs/component-ownership.md` with the reason for each assignment, and verify every interface in the document matches the shipped file character for character, since the file and the document are two copies of one claim
- [x] 3.3 Record that `gnome.portal` is named nowhere and why, so a later reader does not add it to fix an interface that has no entry on purpose

## 4. Verification in a session

Execution note: this group needs the session running on the VM, started from the greeter. The frontend reads the mapping when it starts, so a session that was already running before the package was installed does not exercise it.

- [x] 4.1 With the package installed, start a Gridline session and verify the portal frontend resolves each relied-on interface to the backend the mapping names, reading the frontend's own record of what it selected rather than inferring from which services are running
- [x] 4.2 Verify resolution no longer depends on the compositor's name, by confirming the match is on `Gridline`. The check that distinguishes a real fix from an apparent one is whether the outcome would change if the compositor's own claim were removed
- [x] 4.3 Verify the file chooser resolves to the GTK backend, which before this change matched nothing under Gridline's identity, and record what the frontend did with it beforehand so the difference is on record rather than assumed
- [x] 4.4 Verify screen capture still resolves to the compositor backend, now through Gridline's entry rather than through the compositor's `UseIn`
- [x] 4.5 Verify a confined graphical Snap application runs and reaches portal services, and record which Snap was used. Used `gnome-text-editor` 48.3, strictly confined and running under the AppArmor profile `snap.gnome-text-editor.gnome-text-editor (enforce)` with `GTK_USE_PORTAL=1`. It was chosen because opening a file goes through the FileChooser interface, the one no backend claimed under Gridline's identity before this change, so the test exercises what changed rather than what already worked. The dialog was served by `xdg-desktop-portal-gtk`, the only backend with a running process, while the GNOME backend that also implements FileChooser stayed inactive
- [x] 4.6 Verify an X11-only application runs under XWayland, and record which one. Used `xeyes` from `x11-apps` 7.7+11build4, chosen because it links `libX11` and no Wayland library at all, so its rendering in a Wayland session can only be XWayland. Confirmed by contrast: the snap running beside it links `libwayland-client` as well

## 5. Coherence

- [x] 5.1 Verify the requirement added by this change is satisfied by something a reader can check, and name the file and section that satisfies each of its scenarios
- [x] 5.2 Record that one scenario of the existing portal requirement, the one requiring a missing configuration to be reported by the diagnostic commands, stays unsatisfiable until `add-session-health-check` delivers that surface, and verify the record names that change as its owner
- [x] 5.3 Verify the only file this change added under `packages/gridline/src/adapters/` is the adapter's declaration of which portal backend serves its compositor, and that the isolation gate reports zero references outside the boundary. This task was written assuming nothing in the change would be compositor-specific and that the directory would go untouched. The gate disproved that premise: naming a portal backend is compositor knowledge, and the gate rejected the first version of the mapping for holding it outside the boundary. The directory is touched, which is the correct outcome rather than a violation of it

## Workflow follow-up

- Run `/opsx:verify` once the mapping has been exercised in a running session.
- Archive the change, then propose `add-session-lifecycle-integration`, which selects the screen locker and the polkit agent, the two blocking components still recorded without an owner.
