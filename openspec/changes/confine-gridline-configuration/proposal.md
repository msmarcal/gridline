# Proposal

## Why

`gridline-config` declares `/etc/xdg/hypr/hyprland.conf`. `package-delivery` requires that a Gridline package declare ownership only of paths whose name identifies them as Gridline's. Neither `hypr` nor `hyprland.conf` does, so a requirement the project wrote is broken by code the project shipped, verified and archived.

Nothing caught it, and the reasons are worth stating because they are not oversight. The ownership gate tests whether two Gridline packages claim the same path, not whether a claimed path is Gridline-named. The isolation gate scans `src/` and `tools/`, and the manifest line that declares this path lives in `debian/`, which that gate excludes. The requirement has a scenario asking that a declared path belonging to an archive package be reported before release, and this path belongs to no other package, so even that would not have fired.

The same file is also the mechanism behind an open finding recorded in `docs/compositor-adapter.md`: because it sits where the compositor searches, every session of that compositor reads Gridline's defaults, not only Gridline's. One file, two problems, one fix.

## What Changes

**This change ships implementation.**

- Moves the compositor's shipped defaults to a path named for Gridline, and has the adapter name that path explicitly when it starts the compositor rather than relying on the compositor's search finding a file Gridline placed in its directory.
- Stops `gridline-config` declaring any path that is not Gridline-named, which removes `/etc/xdg/hypr/` from the set of directories Gridline creates and owns.
- Adds a build gate that fails when an install manifest declares a path whose name does not identify it as Gridline's. The requirement has been in force since `establish-gridline-vision` with nothing enforcing it.
- Keeps the customization story the previous change established, in whatever form the new arrangement allows, since "a user can decline a shipped default" is a scenario that already exists and is not weakened here.
- Names the terminal a Gridline session opens, through the desktop-keyed preference file the terminal launcher reads, and declares that terminal. Today Gridline's dependency changes which terminal the launcher picks for the whole machine, which is the opposite of additive, and it does so by winning an ordering rule nobody chose.
- Corrects `docs/package-matrix.md`, which records the terminal as declared because the baseline guarantees none. The baseline recommends one, and apt installs recommends by default, so the recorded reason is false.

**Out of scope.** The bar, the notification daemon and the wallpaper daemon, which `add-desktop-shell-components` owns and which waits on the pattern this change settles. The `start-hyprland` question and the logout crash, both recorded in `docs/compositor-adapter.md` and neither touched by moving a configuration file.

## Capabilities

### Modified Capabilities

- `desktop-session`: one requirement is modified. The scenario that a user's own configuration does not silently discard the defaults was written against a mechanism this change replaces, and the behaviour it asks for has to be restated against the new one rather than quietly dropped.

### New Capabilities

None.

**`package-delivery` is deliberately not modified.** Its requirement is correct as written and was correct when it was written; what was missing is enforcement, and a gate is implementation rather than a change of behaviour. Modifying a requirement because the code broke it would be the wrong direction.

## Impact

- `packages/gridline/debian/gridline-config.install`: the compositor defaults move to a Gridline-named path, and the loader placed in the compositor's own directory goes away.
- `packages/gridline/src/adapters/hyprland/adapter`: the `start` verb names the configuration path.
- `packages/gridline/src/adapters/hyprland/hyprland.conf`: removed, being the loader that only existed to be found by the compositor's search.
- `packages/gridline/tools/`: a gate for Gridline-named declared paths.
- `docs/configuration.md`: the section describing what Gridline does about first-file-wins, which describes the mechanism this change replaces.
- `docs/compositor-adapter.md`: the open finding about Gridline's defaults reaching other sessions, which this closes.
- `packages/gridline/debian/control`: the declared terminal changes, and the reason it is declared changes with it.
- `packages/gridline/src/config/`: the terminal preference file, Gridline-named and keyed on Gridline's desktop name.
- `docs/package-matrix.md`: the owned-path list, and the terminal row whose stated reason is false.
- Verification runs on the LXD VM, where a session other than Gridline's is the measurement for the finding this closes.
