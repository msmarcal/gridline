# Tasks

Scope note: this change lands the structural foundation the specs commit to. Per proposal.md, Hyprland configuration, install scripts, declared APT dependency relationships, visual parity, a custom shell, hardware automation, an application catalogue, and AI features are out of scope. Where a spec requirement is satisfied by Ubuntu's existing stack, the task is to keep it working and prove it, not to reimplement it.

## 1. Source package skeleton

- [ ] 1.1 Create `packages/gridline/` with a `debian/source/format` of `3.0 (native)` and verify `dpkg-source --print-format packages/gridline` reports `3.0 (native)`
- [ ] 1.2 Add `debian/control` declaring source `gridline` and binary packages `gridline-session`, `gridline-config`, `gridline-tools`, leaving dependency relationships minimal per the scope note, and verify `dpkg-checkbuilddeps` and `dpkg-parsechangelog` run without error
- [ ] 1.3 Add `debian/changelog`, `debian/rules` with a debhelper sequence, `debian/compat` level or equivalent `debhelper-compat` build dependency, and per-binary `.install` files, then verify `dpkg-buildpackage -us -uc -b` produces exactly three `.deb` files
- [ ] 1.4 Document the repository and package layout in `packages/gridline/README.md`, including the one-source-package decision and the room for future source packages, and verify the documented build command matches task 1.3

## 2. Package ownership guarantees

- [ ] 2.1 Add a build-time or CI check that fails when two Gridline binary packages ship the same path, and verify it fails on a deliberately duplicated file and passes once removed
- [ ] 2.2 Verify no Gridline maintainer script performs network access, by inspection of all maintainer scripts plus a build with no network reachable
- [ ] 2.3 Verify `gridline-desktop` is absent from this change's output, and record in `packages/gridline/README.md` that it is a reserved metapackage boundary with no implementation

## 3. Session registration and launch

- [ ] 3.1 Ship a Wayland session entry from `gridline-session` and verify it appears in GDM's session list on a test Ubuntu 26.04 install after package install and GDM restart
- [ ] 3.2 Verify installing `gridline-session` does not change any existing user's default session, by recording the preselected session in GDM before and after install
- [ ] 3.3 Implement the session launcher so it sets `XDG_CURRENT_DESKTOP` to `Gridline` followed by the active compositor from the adapter, and verify the value inside a running session with `printenv XDG_CURRENT_DESKTOP`
- [ ] 3.4 Implement launch failure handling so a failed required component exits the session, returns the user to GDM, and writes an attributable journal entry, and verify by forcing a required component to fail and reading `journalctl` for the Gridline identifier
- [ ] 3.5 Implement session teardown so logout terminates session processes and returns the seat to GDM, and verify no Gridline-owned processes for that user remain after logout
- [ ] 3.6 Verify the session entry is removed on package removal and that `dpkg -S` attributes the entry to `gridline-session` while installed

## 4. Compositor adapter boundary

- [ ] 4.1 Define and document the compositor adapter contract, including the capability set an adapter must report, in `packages/gridline/docs/compositor-adapter.md`, and verify the document names every capability the launcher checks in task 4.3
- [ ] 4.2 Implement the Hyprland adapter against that contract using only documented Hyprland interfaces, and verify by listing each interface used in the adapter against Hyprland's documentation in the same document
- [ ] 4.3 Implement launcher-side capability discovery that checks the adapter's reported set before declaring the session up, and verify a session with an artificially withheld required capability fails to come up and names the missing capability
- [ ] 4.4 Add a check that fails when compositor-specific references appear outside the adapter directory, and verify it flags a deliberately planted reference in `gridline-tools` and passes once removed
- [ ] 4.5 Document the adapter extraction path for a future second compositor and verify the document identifies every file that would need to change outside the adapter directory

## 5. Configuration scoping

- [ ] 5.1 Install `gridline-config` defaults into a Gridline-specific directory beneath `/etc/xdg` and verify nothing is installed at the top level of the shared `/etc/xdg` search path
- [ ] 5.2 Prepend that directory to `XDG_CONFIG_DIRS` in the Gridline session launcher only, and verify `XDG_CONFIG_DIRS` inside a GNOME session is unchanged from a system without Gridline installed
- [ ] 5.3 Register administrator-editable defaults as dpkg conffiles and verify a locally modified default survives an upgrade of `gridline-config`
- [ ] 5.4 Ship the explicit portal interface-to-backend mapping assigning compositor-dependent interfaces to the Hyprland backend and the remainder to the GTK backend, and verify the mapping resolves the same backend across repeated session starts
- [ ] 5.5 Document the configuration precedence and the session-scoped `XDG_CONFIG_DIRS` rule in `packages/gridline/docs/configuration.md`, and verify the documented precedence matches observed behavior for a user override of a shipped default

## 6. Tools and diagnostics

- [ ] 6.1 Implement the `gridline` command entry point in `gridline-tools` and verify `dpkg -S` attributes it to `gridline-tools`
- [ ] 6.2 Implement `gridline doctor` so it reads and reports the adapter capability conditions from task 4.3, and verify it names the missing capability in the withheld-capability case
- [ ] 6.3 Verify `gridline doctor` runs and reports on the system when installed without a Gridline session running and without any compositor-specific arguments
- [ ] 6.4 Document the command surface in `packages/gridline/docs/tools.md` and verify every documented invocation runs as written

## 7. Security model

- [ ] 7.1 Verify no Gridline package disables an existing AppArmor profile or sets one to complain mode, by comparing `aa-status` output before and after install
- [ ] 7.2 Ship any Gridline-authored AppArmor profile in enforce mode and verify its mode in `aa-status` after install
- [ ] 7.3 Verify Gridline packages carry no APT pin, hold, or preference file, by inspecting package contents and `apt-mark showhold` after install

## 8. Integration verification on the baseline

- [ ] 8.1 Verify the NetworkManager scenarios in `specs/platform-integration/spec.md` on a test Ubuntu 26.04 install: connection state is observable in-session, and a Wi-Fi secret prompt can be answered to completion
- [ ] 8.2 Verify the PipeWire and WirePlumber scenarios: playback and capture work on login without manual setup, and changing the default output device takes effect for subsequent playback
- [ ] 8.3 Verify the BlueZ scenario: a Bluetooth audio device and an input device pair, connect, and are usable, with the audio device appearing through PipeWire
- [ ] 8.4 Verify the portal scenarios: screen capture prompts for a selection and delivers it, and the portal file chooser returns a chosen file
- [ ] 8.5 Verify the Snap scenario: a strictly confined graphical Snap displays, accepts input, and reaches audio and portal services
- [ ] 8.6 Verify the fallback guarantee end to end: GNOME starts normally with Gridline installed, and after removing all Gridline packages no session entry remains, GNOME still starts, and no Gridline-owned files remain outside administrator-modified configuration
- [ ] 8.7 Verify a security update to Gridline or a dependency installs through APT and a subsequent Gridline session starts normally

## Workflow follow-up

- Run `/opsx:verify` once implementation is complete, to check the result against these artifacts.
- Archive the change after the project's review requirements are satisfied.
