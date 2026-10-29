# Proposal

## Why

Screen capture works in a Gridline session today, and it works by accident. The portal frontend picks a backend by trying each name in `XDG_CURRENT_DESKTOP` against each backend's `UseIn` list. Gridline's identity is `Gridline:Hyprland`, no backend mentions `Gridline`, and `hyprland.portal` lists `UseIn=wlroots;Hyprland;sway;Wayfire;river`. So the match happens on the second name, through a file the compositor's package owns and can change without telling anyone.

`docs/component-ownership.md` already calls this out as a working path by accident and names `gridline-portals.conf` as what makes it deliberate. This change delivers that file.

Measuring the baseline to write it turned up something worse, and it is not about screen capture. **`gtk.portal` declares `UseIn=gnome`.** Neither element of `Gridline:Hyprland` matches that, so under Gridline's identity the GTK backend is not selected by name for any interface, including the file chooser. Screen capture at least matches something; the file chooser matches nothing and falls to whatever the frontend does last. A session where opening a file from a sandboxed application depends on undefined fallback is not a session that satisfies the portal requirements `platform-integration` already carries.

Twelve interfaces on the baseline are declared by more than one installed backend, among them `ScreenCast`, `Screenshot` and `GlobalShortcuts`, each claimed by both `gnome.portal` and `hyprland.portal`. The requirement that the session resolve a documented, deterministic choice has real work behind it rather than a formality.

## What Changes

**This change ships implementation.** It delivers a configuration file, dependency declarations, and the documentation of the mapping, and it implements requirements `platform-integration` already carries rather than adding new ones.

- Ships `gridline-portals.conf` from `gridline-config`, installed as `/usr/share/xdg-desktop-portal/gridline-portals.conf`, lower-cased because the frontend case-folds each desktop name before building the filename, and in the portal directory because the frontend appends `xdg-desktop-portal/` to each base directory it searches.
- Names a backend for every portal interface a Gridline session relies on, so that no interface is left to the frontend's fallback and none of the twelve contested interfaces resolves by start order.
- Declares `xdg-desktop-portal`, `xdg-desktop-portal-gtk` and `xwayland` where the packages require them. All three are installed on the baseline today only because `ubuntu-session` and `libmutter` pull them in, which is the informal reliance `platform-integration` forbids.
- Records the mapping and its reasons in `docs/component-ownership.md`, replacing the passage that currently describes the accidental path as the state of affairs.
- Verifies the result in a session: the frontend resolves each required interface to the named backend, and it does so by matching `Gridline` rather than by matching the compositor's name.

**Out of scope.** The diagnostic that reports a missing or incomplete portal configuration belongs to `add-session-health-check`, which owns the command surface; one scenario of the existing requirement therefore stays unsatisfiable until that change lands, and this change records that rather than pretending otherwise. Snap confinement behaviour is verified, not configured: Gridline ships no Snap interface connections. No portal backend is written, and no compositor behaviour changes.

## Capabilities

### New Capabilities

None.

### Modified Capabilities

- `platform-integration`: gains one requirement, that every portal interface the session relies on resolves to a backend Gridline names, including interfaces no installed backend claims under Gridline's identity.

Most of this change implements requirements that already exist: portals with the required backends, a Gridline-owned configuration file driving selection, X11 applications through XWayland, and Snap applications running correctly. Four requirements, no amendment needed.

The one gap is narrow and was found by measuring rather than by reading. The existing deterministic-selection scenario covers **contention**: more than one installed backend implements the same interface, so the session must resolve a documented choice. The baseline also produces the opposite case, which no scenario covers. `gtk.portal` declares `UseIn=gnome`, and neither element of `Gridline:Hyprland` matches it, so the file chooser has **no** backend claiming it rather than several. Contention and absence are different conditions and only one of them is specified.

## Impact

- `packages/gridline/src/config/gridline-portals.conf`: new, the interface-to-backend mapping.
- `packages/gridline/debian/gridline-config.install`: new, to ship it.
- `packages/gridline/debian/control`: dependency fields on `gridline-config` and `gridline-session`.
- `docs/component-ownership.md`: the portal section, which currently documents the accidental path as the current state.
- Verification runs on the LXD VM where the session already starts.
