# Component ownership

Every component a Gridline session depends on, who owns it, and what happens when it is missing or broken.

Two rules govern this table. First, **where Ubuntu already provides a component, Gridline does not reimplement it**; Gridline owns the integration, not the component. Second, **every component has a stated fallback**, because a desktop whose failure modes are undefined is a desktop that fails silently.

**This page is authoritative for component facts.** Where it and [package-matrix.md](package-matrix.md) carry the same version or archive component, this page wins. That page keeps the same measurement as a packaging-side view, and states the same rule, so a reader who finds the two disagreeing knows which one to correct.

## Reading the table

- **Upstream** is the project that develops the component.
- **Ubuntu source** is the archive source package that provides it, with the version published for the Ubuntu 26.04 LTS (`resolute`) baseline.
- **Component** is the archive component the package sits in. Universe carries **no Canonical security-update commitment**. Main does. Every Hypr-ecosystem package Gridline depends on is in `universe`, which is recorded here because it is a support fact rather than packaging trivia, and every row below that names `universe` is a row carrying that gap.
- **Gridline ownership** names what Gridline itself owns. "Integration only" means Gridline owns how the session reaches the component and nothing about the component.
- **Fallback** is the behavior when the component is absent or fails. Every Ubuntu-provided component falls into exactly one of three categories, and the cell says which:
  - **Session does not start.** The component's absence blocks startup. Five rows on this page carry it.
  - **Session starts**, followed by the capability that is lost. The component's absence degrades the session. Nine rows carry it, and each names what degrades rather than saying the session degrades generally.
  - **Not applicable.** A deliberate category, not a missing entry: it marks a component that cannot be absent without the baseline itself being broken, so there is no Gridline-side degradation path to describe. Each such row states why.

  Gridline's own artifacts, the adapter, the session entry, the launcher, `gridline-portals.conf` and the backend mapping, are not Ubuntu-provided and are not classified this way. Their cells describe the failure they produce, which for several of them is neither a blocked start nor a degraded capability: a missing session entry means the session is never offered, which is not the same as failing.

## Session and compositor

| Component | Upstream | Ubuntu source | Component | Gridline ownership | Fallback |
|---|---|---|---|---|---|
| GDM | GNOME | `gdm3` 50.1-0ubuntu0.1 | main | Integration only: Gridline ships a session entry GDM reads | Not applicable. GDM is the baseline display manager; without it there is no login screen for any session, Gridline's or GNOME's |
| Hyprland | Hyprland | `hyprland` 0.53.3+ds-4 | **universe** | Integration only, through the compositor adapter | **Session does not start.** The launcher reports the missing compositor and returns the user to GDM, where GNOME remains selectable |
| Stock Ubuntu session | GNOME | `ubuntu-session` 50.1-0ubuntu0.1 | main | Integration only: Gridline never modifies it, never removes it and never makes it the non-default | Not applicable, and for a reason no other row has: it is the fallback every other row's failure path returns to. Without it there is no supported desktop to fall back to, so a Gridline failure would have nowhere to land |
| systemd-logind | systemd | `systemd` 259.9-0ubuntu0.1 | main | Integration only: seat, session and power event handling | Not applicable. Seat, session and power event handling come from the baseline init system; without it the machine has no user sessions at all, Gridline's or GNOME's |
| Compositor's own session entry | Hyprland | `hyprland` 0.53.3+ds-4 | **universe** | None. Gridline neither owns nor modifies it. It arrives with the compositor dependency and is listed here because Gridline introduced it | Not applicable as a Gridline failure path, and that is the point of the row. Selecting it starts the compositor without the Gridline launcher, so no capability check runs, no failure record is written under Gridline's syslog identifier, and the user has not started Gridline. On screen the two are identical |
| Compositor adapter | Gridline | `gridline-session` | n/a | **Owned by Gridline** | **Session does not start.** A missing or failing adapter is a Gridline bug, reported with the compositor named |
| Session entry | Gridline | `gridline-session` | n/a | **Owned by Gridline.** A Gridline-named drop-in in `/usr/share/wayland-sessions/`, which `ubuntu-session` owns. Gridline declares the file, never the directory | The session is not offered at the login screen. The stock Ubuntu entry is unaffected and still selectable, so a user is never left without a desktop to choose |
| Wayland registry query | freedesktop | `wayland-utils` 1.3.0-1 | **universe** | Integration only: the compositor adapter uses it to ask the running compositor which protocols it advertises, since the compositor offers no command that lists them | **Session does not start.** Without it the adapter cannot confirm any protocol-backed capability, so every required one reads as unreported and the launch ends naming the first of them |
| Session launcher | Gridline | `gridline-session` | n/a | **Owned by Gridline** | The session entry names it in `TryExec`, so if the launcher is absent the entry is withheld rather than offered and failing. The stock session remains selectable |

## Desktop integration

| Component | Upstream | Ubuntu source | Component | Gridline ownership | Fallback |
|---|---|---|---|---|---|
| NetworkManager | GNOME | `network-manager` 1.54.3-2ubuntu3.1 | main | Integration only: the session surfaces state and answers secret prompts | Session starts without network management. The condition is reported by the diagnostic commands; the user can reach a network from the GNOME session |
| PipeWire | PipeWire | `pipewire` 1.6.2-1ubuntu1.2 | main | Integration only | Session starts without audio. Reported as a capability gap, not a startup failure |
| WirePlumber | PipeWire | `wireplumber` 0.5.13-1ubuntu1.1 | main | Integration only | Session starts, audio routing is unmanaged. Reported as a capability gap |
| BlueZ | BlueZ | `bluez` 5.85-4ubuntu0.4 | main | Integration only | Session starts without Bluetooth. Reported as a capability gap |
| UPower | freedesktop | `upower` 1.91.1-1 | main | Integration only: battery and power source observability | Session starts, battery state is not observable. Reported as a capability gap; relevant only on machines with a battery |
| XWayland | X.Org | `xwayland` 2:24.1.10-1 | main | Integration only | Session starts, X11-only applications do not run. Reported as a capability gap, since native Wayland applications are unaffected |
| snapd | Canonical | `snapd` 2.77.1+ubuntu26.04.2 | main | Integration only: Gridline sets the desktop identity confined Snaps resolve against | Session starts, Snap applications may lose theme or portal access. Reported as a capability gap |

## Portals

The compositor's own backend is the exception on this page: it is the only portal component whose absence blocks startup, because screen capture on Wayland cannot be served by any other backend. The frontend and the GTK backend degrade in the ordinary way.

There is a second, less obvious failure here, and it is a Gridline problem rather than a missing component. `xdg-desktop-portal` selects backends by looking for `<desktop>-portals.conf`, taking the desktop name from the first entry in `XDG_CURRENT_DESKTOP`. Gridline's identity is `Gridline:Hyprland`, so the frontend looks for `gridline-portals.conf`. The `hyprland` package ships `/usr/share/xdg-desktop-portal/hyprland-portals.conf`, which is never consulted under that identity, and `hyprland.portal` lists `UseIn=wlroots;Hyprland;sway;Wayfire;river;` without `Gridline` - so the deprecated `UseIn` fallback does not match either. **Gridline must ship its own `gridline-portals.conf`**; without it, backend selection falls to a last-resort fallback and screen capture fails while the session appears healthy. The file is a Gridline-named drop-in, not a modification of a file another package owns.

Two mechanical details decide whether that file is ever read, and both are verified against `xdg-desktop-portal 1.21.1+ds-1ubuntu3.1`:

- **The filename is lower-cased.** The frontend case-folds each desktop name with `g_ascii_strdown` before building the filename, so the identity `Gridline` yields `gridline-portals.conf`. A file named `Gridline-portals.conf` is never found.
- **The search directory is a portal directory, not a generic config root.** The frontend appends `xdg-desktop-portal/` to each base directory it searches, so the vendor default belongs at `/usr/share/xdg-desktop-portal/gridline-portals.conf` and an administrator override at `/etc/xdg-desktop-portal/gridline-portals.conf`. A file placed at the root of a Gridline-owned directory, or in `/etc/xdg` without the `xdg-desktop-portal/` component, is not on any search path.

The frontend also tries **every** name in `XDG_CURRENT_DESKTOP` in order before falling back, so with `Gridline:Hyprland` a missing `gridline-portals.conf` means the `hyprland-portals.conf` shipped by the compositor package is consulted next. That is a working screen-capture path by accident, which is worse than a clean failure: it hides the missing file until someone changes the identity or the compositor package drops its default.

| Component | Upstream | Ubuntu source | Component | Gridline ownership | Fallback |
|---|---|---|---|---|---|
| xdg-desktop-portal | freedesktop | `xdg-desktop-portal` 1.21.1+ds-1ubuntu3.1 | main | Integration only; Gridline owns the interface-to-backend mapping | Session starts, portal-mediated features fail. Reported as a capability gap. Sandboxed applications are affected most |
| xdg-desktop-portal-hyprland | Hyprland | `xdg-desktop-portal-hyprland` 1.3.11-1build2 | **universe** | Integration only | **Session does not start.** Without it the adapter cannot report the `screencast` capability, and `screencast` is required: see [compositor-adapter.md](compositor-adapter.md). The GTK backend cannot substitute, because it does not implement screen capture on Wayland |
| xdg-desktop-portal-gtk | GNOME | `xdg-desktop-portal-gtk` 1.15.3-2ubuntu1 | main | Integration only | File chooser and the interfaces it backs become unavailable. Reported as a capability gap |
| `gridline-portals.conf` | Gridline | `gridline-config` | n/a | **Owned by Gridline** | Backend selection falls to the portal frontend's last-resort fallback. Screen capture fails while the session reports itself healthy, which is the silent failure the file exists to prevent. Treated as a configuration bug |
| Backend mapping | Gridline | `gridline-config` | n/a | **Owned by Gridline** | Backend selection falls back to whatever the portal frontend resolves on its own, which is the non-deterministic behavior the mapping exists to prevent. Treated as a configuration bug |

## Fail-safe components

These two are different in kind. Their absence **blocks session startup** rather than degrading it, because a session that runs without them has a security consequence rather than a missing feature. See [architecture.md](architecture.md) for the required-component rule.

| Component | Upstream | Ubuntu source | Component | Gridline ownership | Fallback |
|---|---|---|---|---|---|
| Screen locker | Hyprland (candidate: `hyprlock`); selection owned by `add-session-lifecycle-integration` | `hyprlock` 0.9.2-1build4 | **universe** | Integration only, once chosen | **Session does not start.** A desktop that cannot lock must not run, because the failure surfaces as an unlocked unattended machine. The user returns to GDM with the cause named |
| polkit authentication agent | polkit, plus the chosen agent (candidate: `hyprpolkitagent`); selection owned by `add-session-lifecycle-integration` | `policykit-1` 127-2ubuntu1.1, binary `polkitd` (main); `hyprpolkitagent` 0.1.3-1 | main / **universe** | Integration only, once chosen | **Session does not start.** Without an agent, privileged actions fail with no prompt and no explanation. Reported with the missing agent named |

Both candidates exist on the baseline and neither selection is final; `add-session-lifecycle-integration` still owns the choice. Two facts from the archive constrain it:

**`hyprlock` authenticates out of the box.** It ships `/etc/pam.d/hyprlock` containing `auth include login`, so no PAM work is needed to make locking function. That stack is broader than a locker strictly needs - GDM uses a narrower one - which is a security review item for the change that adopts it, not a defect.

**The archive units cannot refuse startup.** `hyprpolkitagent.service`, `hypridle.service` and `xdg-desktop-portal-hyprland.service` are systemd user units with `PartOf=graphical-session.target`, `ConditionEnvironment=WAYLAND_DISPLAY` and `Restart=on-failure`. A failing unit restarts quietly and the session comes up regardless. The blocking rule in [architecture.md](architecture.md) therefore needs a Gridline-owned startup check to be enforceable at all; systemd alone will not enforce it.

## Dependencies not yet declared

Five components block session startup, and no Gridline package declares any of them today. All three binary packages carry `Depends: ${misc:Depends}` and nothing else. That is the correct state rather than an oversight, and this section exists so the two cannot be confused: a dependency that is missing because nobody got to it looks exactly like one that is missing because the decision it encodes has not been made.

| Blocking component | Why it cannot be declared yet | Change that owns the declaration |
|---|---|---|
| Hyprland | The adapter that drives a compositor does not exist, so declaring one would install software the launcher cannot use | `add-hyprland-session-baseline` |
| `xdg-desktop-portal-hyprland` | Follows the compositor: the backend is only required once the adapter reports the `screencast` capability | `add-hyprland-session-baseline` |
| Screen locker | The component is not selected. `hyprlock` is a candidate, not a choice | `add-session-lifecycle-integration` |
| polkit authentication agent | The agent is not selected. `hyprpolkitagent` is a candidate, not a choice | `add-session-lifecycle-integration` |
| Compositor adapter | Not deferred and not third-party. It is Gridline's own and ships in `gridline-session` | n/a |

The degrading components carry no declaration either, for a different reason: the session cannot run at all yet, so a recommendation would describe a desktop that does not start. The change that makes the session start is the one that can state what it recommends.

## Session entries the dependency brought

Depending on the compositor installs session entries the compositor's own package owns. On the baseline, `hyprland` ships `hyprland.desktop` and `hyprland-uwsm.desktop` into `/usr/share/wayland-sessions/`.

One is withheld: `hyprland-uwsm.desktop` names `uwsm` in `TryExec`, which the baseline does not install, so the display manager does not offer it. That is the same mechanism Gridline relies on for its own entry, working here for another package.

The other is offered. Selecting it reaches the compositor directly, with no capability check, no failure record under Gridline's syslog identifier, and none of the guarantees the Gridline session entry carries. A user who picks it has not started Gridline.

**Gridline does not own either file and must not modify or remove them.** Archive packages own their own files, and a package that deletes another's entry would break that package on its own terms. What Gridline owns is having introduced them, by declaring the dependency that installs them.

**The offered entry has a row above, decided after the session was observed starting.** The login screen offers three sessions, Gridline, Hyprland and Ubuntu, and nothing on screen distinguishes the first two: both run the same compositor with its own defaults, down to the same startup banner. During verification the compositor's entry was selected twice by someone who knew both existed and was checking for exactly this. A reader who does not know cannot do better, which is why the row exists: not to claim ownership Gridline does not have, but to record a component the project put in front of its users and the consequence of choosing it.

## Security and updates

| Component | Upstream | Ubuntu source | Component | Gridline ownership | Fallback |
|---|---|---|---|---|---|
| AppArmor | AppArmor | `apparmor` 5.0.2-0ubuntu1~26.04.1 | main | Gridline may own enforce-mode profiles for its own executables, and owns the rule that it never weakens an existing profile | Not applicable as a degradation path. Gridline never disables a profile to make itself work; if a Gridline executable cannot run under confinement, the profile is fixed, not removed |
| APT and the Ubuntu update path | Debian, Canonical | `apt` 3.2.0, `unattended-upgrades` 2.12ubuntu9 | main | Integration only: Gridline ships no pin, hold or preference file | Not applicable. Gridline packages update like any other; there is no Gridline-specific update mechanism to fall back from |

## Components Gridline owns outright

Four entries above are Gridline's own, and the rest are integration only:

- the compositor adapter and the session launcher, in `gridline-session`
- the portal interface-to-backend mapping, `gridline-portals.conf`, and the shipped defaults, in `gridline-config`
- the diagnostic command surface, in `gridline-tools`
- the reserved `gridline-desktop` metapackage boundary, with no implementation

Everything else on this page is Ubuntu's, consumed as the archive provides it.
