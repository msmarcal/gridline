# Component ownership

Every component a Gridline session depends on, who owns it, and what happens when it is missing or broken.

Two rules govern this table. First, **where Ubuntu already provides a component, Gridline does not reimplement it**; Gridline owns the integration, not the component. Second, **every component has a stated fallback**, because a desktop whose failure modes are undefined is a desktop that fails silently.

## Reading the table

- **Upstream** is the project that develops the component.
- **Ubuntu source** is the archive source package expected to provide it. Confirming the exact source package, archive component and version on the Ubuntu 26.04 LTS baseline is owned by `define-supported-ubuntu-integration`; entries marked "confirm" are the ones where availability or version is not safe to assume.
- **Gridline ownership** names what Gridline itself owns. "Integration only" means Gridline owns how the session reaches the component and nothing about the component.
- **Fallback** is the behavior when the component is absent or fails.

## Session and compositor

| Component | Upstream | Ubuntu source | Gridline ownership | Fallback |
|---|---|---|---|---|
| GDM | GNOME | `gdm3` | Integration only: Gridline ships a session entry GDM reads | Not applicable. GDM is the baseline display manager; without it there is no login screen for any session, Gridline's or GNOME's |
| Hyprland | Hyprland | `hyprland` (confirm) | Integration only, through the compositor adapter | Session does not start. The launcher reports the missing compositor and returns the user to GDM, where GNOME remains selectable |
| systemd-logind | systemd | `systemd` | Integration only: seat, session and power event handling | Not applicable. Part of the baseline init system |
| Compositor adapter | Gridline | `gridline-session` | **Owned by Gridline** | Session does not start. A missing or failing adapter is a Gridline bug, reported with the compositor named |

## Desktop integration

| Component | Upstream | Ubuntu source | Gridline ownership | Fallback |
|---|---|---|---|---|
| NetworkManager | GNOME | `network-manager` | Integration only: the session surfaces state and answers secret prompts | Session starts without network management. The condition is reported by the diagnostic commands; the user can reach a network from the GNOME session |
| PipeWire | PipeWire | `pipewire` | Integration only | Session starts without audio. Reported as a capability gap, not a startup failure |
| WirePlumber | PipeWire | `wireplumber` | Integration only | Session starts, audio routing is unmanaged. Reported as a capability gap |
| BlueZ | BlueZ | `bluez` | Integration only | Session starts without Bluetooth. Reported as a capability gap |
| UPower | freedesktop | `upower` | Integration only: battery and power source observability | Session starts, battery state is not observable. Reported as a capability gap; relevant only on machines with a battery |
| XWayland | X.Org | `xwayland` (confirm) | Integration only | Session starts, X11-only applications do not run. Reported as a capability gap, since native Wayland applications are unaffected |
| snapd | Canonical | `snapd` | Integration only: Gridline sets the desktop identity confined Snaps resolve against | Session starts, Snap applications may lose theme or portal access. Reported as a capability gap |

## Portals

The compositor's own backend is the exception on this page: it is the only portal component whose absence blocks startup, because screen capture on Wayland cannot be served by any other backend. The frontend and the GTK backend degrade in the ordinary way.

| Component | Upstream | Ubuntu source | Gridline ownership | Fallback |
|---|---|---|---|---|
| xdg-desktop-portal | freedesktop | `xdg-desktop-portal` | Integration only; Gridline owns the interface-to-backend mapping | Session starts, portal-mediated features fail. Reported as a capability gap. Sandboxed applications are affected most |
| xdg-desktop-portal-hyprland | Hyprland | `xdg-desktop-portal-hyprland` (confirm) | Integration only | **Session does not start.** Without it the adapter cannot report the `screencast` capability, and `screencast` is required: see [compositor-adapter.md](compositor-adapter.md). The GTK backend cannot substitute, because it does not implement screen capture on Wayland |
| xdg-desktop-portal-gtk | GNOME | `xdg-desktop-portal-gtk` | Integration only | File chooser and the interfaces it backs become unavailable. Reported as a capability gap |
| Backend mapping | Gridline | `gridline-config` | **Owned by Gridline** | Backend selection falls back to whatever the portal frontend resolves on its own, which is the non-deterministic behavior the mapping exists to prevent. Treated as a configuration bug |

## Fail-safe components

These two are different in kind. Their absence **blocks session startup** rather than degrading it, because a session that runs without them has a security consequence rather than a missing feature. See [architecture.md](architecture.md) for the required-component rule.

| Component | Upstream | Ubuntu source | Gridline ownership | Fallback |
|---|---|---|---|---|
| Screen locker | Not yet chosen; selection owned by `add-session-lifecycle-integration` | Confirm once chosen | Integration only, once chosen | **Session does not start.** A desktop that cannot lock must not run, because the failure surfaces as an unlocked unattended machine. The user returns to GDM with the cause named |
| polkit authentication agent | polkit, plus the chosen agent; selection owned by `add-session-lifecycle-integration` | `polkit-1`, plus the agent's own source (confirm once chosen) | Integration only, once chosen | **Session does not start.** Without an agent, privileged actions fail with no prompt and no explanation. Reported with the missing agent named |

## Security and updates

| Component | Upstream | Ubuntu source | Gridline ownership | Fallback |
|---|---|---|---|---|
| AppArmor | AppArmor | `apparmor` | Gridline may own enforce-mode profiles for its own executables, and owns the rule that it never weakens an existing profile | Not applicable as a degradation path. Gridline never disables a profile to make itself work; if a Gridline executable cannot run under confinement, the profile is fixed, not removed |
| APT and the Ubuntu update path | Debian, Canonical | `apt`, `unattended-upgrades` | Integration only: Gridline ships no pin, hold or preference file | Not applicable. Gridline packages update like any other; there is no Gridline-specific update mechanism to fall back from |

## Components Gridline owns outright

Four entries above are Gridline's own, and the rest are integration only:

- the compositor adapter and the session launcher, in `gridline-session`
- the portal interface-to-backend mapping and the shipped defaults, in `gridline-config`
- the diagnostic command surface, in `gridline-tools`
- the reserved `gridline-desktop` metapackage boundary, with no implementation

Everything else on this page is Ubuntu's, consumed as the archive provides it.
