# Proposal

## Why

Ubuntu offers no supported tiling desktop: users who want one today assemble a compositor, portals, audio, and networking by hand, outside the archive, with no upgrade story and no way back if it breaks. Gridline makes a Wayland tiling session a first-class, packaged, *optional* choice on Ubuntu 26.04 LTS, installable and removable like any other desktop, with the stock GNOME session always intact behind it.

This change establishes the foundation every later change builds on. The repository is currently empty, so the durable commitments have to be settled before any package exists: what Gridline is delivered as, which package owns what, what the session must integrate with, and how the project avoids welding itself to one compositor. Settling these as specs now is cheaper than discovering them as contradictions across four packages later.

## What Changes

- Declare Gridline's delivery model: **Debian packages only**. Every capability a user gets must come from a package in the archive, never from an image, an install script, or a post-install fetch.
- Fix the package boundaries and what each one owns:
  - `gridline-session`: GDM session entry, session launcher, core integration glue.
  - `gridline-config`: defaults under `/etc/xdg`, keybindings, themes.
  - `gridline-tools`: `gridline doctor`, theme and diagnostic commands.
  - `gridline-desktop`: metapackage, declared now as a reserved boundary, shipped later.
- Commit to the fallback guarantee: installing Gridline leaves the regular Ubuntu GNOME session installed, selectable, and supported. Gridline is additive and removable.
- Commit to the mandatory integration surface: GDM, NetworkManager, PipeWire and WirePlumber, BlueZ, XDG Desktop Portals (a compatible Hyprland backend plus the GTK backend), Snap applications, AppArmor, and Ubuntu security updates.
- Establish compositor isolation: Hyprland is the initial compositor, but compositor-specific logic stays behind an internal boundary, and the desktop must not depend on undocumented Hyprland internals.
- Establish the image rule: a future community Ubuntu Remix or ISO consumes these same packages and must never be the only place a core capability exists.

No breaking changes: nothing ships yet.

## Capabilities

### New Capabilities

- `desktop-session`: Gridline's existence as a selectable desktop session - GDM registration, launch and teardown, and coexistence with the stock GNOME session as a supported fallback.
- `package-delivery`: The delivery contract - that Gridline reaches users only as Debian packages, which packages exist, what each owns, and the rule that images consume packages rather than carry functionality.
- `platform-integration`: The integration surface the session must satisfy to be a usable Ubuntu desktop - networking, audio, Bluetooth, portals, Snap applications, and the security model.
- `compositor-abstraction`: The boundary that keeps compositor-specific logic isolated, so the compositor is a replaceable component rather than a dependency of the whole desktop.

### Modified Capabilities

None. `openspec/specs/` is empty; this change creates the project's first specs.

## Impact

- **Repository layout**: establishes `packages/<source-package>/` as the structure the existing `.gitignore` already anticipates, with each source package owning its own `debian/` directory.
- **Code**: none changed. No implementation exists yet; this change writes specs only.
- **Downstream changes**: per-package behavioral specs (config defaults and keybindings, `gridline doctor` checks, theme commands) depend on these boundaries and follow as separate changes.
- **Deliberately out of scope here**: Hyprland configuration, install scripts, declared APT dependencies, Omarchy visual parity, a custom Quickshell shell, per-device hardware automation, an application catalogue, and AI features. Several of these will become their own capabilities later; naming them as excluded keeps this change's specs about structure rather than content.
