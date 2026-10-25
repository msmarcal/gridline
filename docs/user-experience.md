# User experience reference

Gridline's user experience is modelled on Omarchy, which demonstrated that a
tiling Wayland desktop can be immediately usable instead of a configuration
project. This document records which parts of that experience Gridline adopts,
which it cannot, and why, so that a later change does not reopen a settled
question or promise a feature the baseline cannot deliver.

## The moving target

Omarchy 4.0 "Quattro" replaced its entire desktop with a single Quickshell
process that provides the bar, launcher, menu, notifications, on-screen
displays, system panels, lock screen and polkit agent together. That release
dropped Waybar, Walker, Mako, SwayOSD, swaybg, hyprlock, hypridle and
polkit-gnome, and moved its compositor configuration to Lua.

The consequence for Gridline is specific and slightly ironic: the stack Omarchy
discarded is the stack Ubuntu packages, and the stack Omarchy adopted is not in
the archive at all.

| Quattro component | Ubuntu 26.04 LTS baseline |
| --- | --- |
| quickshell | absent from the archive |
| Hyprland with Lua configuration | archive has 0.53.3; Lua arrived upstream in 0.55 |
| walker, mako, satty, mise, impala | absent from the archive |
| waybar, fuzzel, hyprlock, hypridle, hyprpaper, hyprpolkitagent | present in universe |
| foot, btop, neovim, grim, slurp, wf-recorder, swayosd, swaybg | present in universe |

The Lua gap was verified against the packaged binary rather than inferred from
the version number: the archive's Hyprland executable exports no Lua symbols and
links no Lua library, so a Lua compositor configuration does not load on the
baseline. Upstream has also stated that the legacy configuration language will
be dropped one to two releases after 0.55, which means the configuration format
Gridline targets is the one upstream is retiring. The compositor adapter exists
for exactly this class of change.

## What Gridline targets

Gridline targets the Omarchy 3.x experience built entirely from archive
packages. This is a deliberate choice of a reachable target over a closer
imitation, and it follows from the project's existing non-goals: Gridline does
not pursue visual parity with another project, and no bespoke shell is planned.

Pursuing Quattro would require packaging and maintaining Quickshell and a
newer Hyprland outside the archive. That contradicts the preference for archive
packages, widens the security-update gap that `security-and-manageability.md`
already declares, and puts the project's maintenance burden above its delivery
capacity.

## Experience inventory

Every element below is classified by what it demands of Gridline. The
classification is the input to scope decisions in later changes, not a feature
list.

### Configuration of archive packages

These need no new software and no code. They are compositor and component
settings, and they are where most of the experience lives.

| Element | Mechanism |
| --- | --- |
| Tiling, workspaces, focus and swap, resize, fullscreen, grouping, scratchpad | compositor settings |
| A Super-centric keymap covering navigation, windows and launching | compositor key bindings |
| Unified copy and paste that behaves the same in the terminal and elsewhere | key bindings plus terminal configuration |
| Screenshot of region or window, screen recording, colour picker | archive capture tools bound to keys |
| Idle detection and screen lock | the idle daemon and lock screen, retained from the 3.x stack |
| Wallpaper, including per-theme backgrounds | the archive wallpaper daemon |
| A status bar with workspaces, clock, tray, battery and media | the archive bar |
| An application launcher with fuzzy matching | the archive launcher |
| A polkit authentication agent inside the session | the archive agent |
| Opinionated terminal, editor and system monitor defaults | component configuration |

### Gridline-authored integration

These are the parts of the experience that are not a setting and not a new
desktop component. They are glue over archive tools, owned by Gridline, and each
belongs to a downstream change rather than to the foundation.

| Element | Approach |
| --- | --- |
| A theme that restyles bar, terminal, editor, lock screen and wallpaper together | a Gridline mechanism applying a named theme across components |
| A command menu over both applications and system actions | built on the archive launcher, driven by Gridline-owned entries |
| System panels for audio, bluetooth, network, display and power | menu entries wrapping the CLI tools already in the baseline |
| A single Gridline command as the entry point for session operations | the `gridline` command, already required by `package-delivery` |
| Editing configuration and restarting only what the edit affects | a Gridline mechanism over user services |
| Reporting when the session is not in the state it should be | `gridline doctor`, already required |

The command menu is the element worth the most to the experience. Omarchy's menu
is the single surface through which most of the desktop is reached, and
reproducing it over the archive launcher is the clearest usability difference
between Gridline and a bare tiling compositor.

### Not reachable on this baseline

Not defects, and not a backlog. Each would require software the archive does not
carry.

- The unified Quickshell desktop, and everything that is a property of it: the
  plugin system and live-reloading colour palette.
- Lua compositor configuration, until the baseline carries a Hyprland that
  supports it.
- Omarchy's own applications.

### Deliberately excluded

These are rejected on policy, not availability, and they are the clearest place
where Gridline's intent diverges from Omarchy's.

- **Driving system updates.** Omarchy routes package upgrades through its own
  tool. On Ubuntu, patching is apt's responsibility. Gridline does not interpose
  itself between the user and their package manager.
- **Shortcuts to proprietary services.** Omarchy binds keys to specific
  commercial mail, music, messaging and password products. Gridline binds keys
  only to software it delivers. The corresponding key combinations are left
  unbound and documented as available, so a user or an administrator can bind
  them without colliding with a Gridline default.
- **Installing software on demand.** Omarchy can fetch coding agents when first
  invoked. Gridline delivers everything through packages; post-install download
  is prohibited by the delivery rules, because a machine that fetches software
  after install can no longer be returned to its prior state by removing
  packages.

## Consequences for later changes

- Key bindings ship for delivered software only, and the unbound combinations
  are documented rather than silently free.
- The theme mechanism and the command menu are Gridline-owned capabilities, so
  they are subject to the policy layer: an administrator can set and lock them.
- **Four of the elements above are mechanisms, and no change in the recorded
  sequence delivers them.** The command menu, the theme mechanism, the system
  panels and the targeted restart appear in no specification and in no row of
  [package-matrix.md](package-matrix.md). The two that do have owners, the
  `gridline` command and `gridline doctor`, are already required by
  `package-delivery`.

  The gap is an attribution failure rather than a deferral, and it is not closed
  by assigning them to `add-gridline-configuration-defaults`. That change owns
  configuration content, and [architecture.md](architecture.md) states that the
  configuration layer ships no executables. A menu that reads entries and invokes
  actions is a program. By the layer boundaries these four belong to the tools
  layer, shipped by `gridline-tools`, alongside the command surface it already
  owns.

  Recording the gap here is not the same as closing it. A change has to own
  them, and naming it belongs to whoever next revises the sequence, since the
  tools-layer change in it today is scoped to diagnostics rather than to the
  experience.
- Nothing in this document introduces a component outside the archive. If a
  later change needs one, it has to argue against the preference for archive
  packages and account for the security-update gap.
