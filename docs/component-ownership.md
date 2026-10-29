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

Gridline ships `gridline-portals.conf` and the frontend reads it, which is what decides backend selection in a Gridline session. Before that file existed the frontend fell back to matching each element of `XDG_CURRENT_DESKTOP` against each backend's own `UseIn` list, and what it produced is worth recording because it explains the shape of the file.

Two mechanical details decide whether the file is read at all, and both are verified against `xdg-desktop-portal 1.21.1+ds-1ubuntu3.1`:

- **The filename is lower-cased.** The frontend case-folds each desktop name with `g_ascii_strdown` before building the filename, so the identity `Gridline` yields `gridline-portals.conf`. A file named `Gridline-portals.conf` is never found.
- **The search directory is a portal directory, not a generic config root.** The frontend appends `xdg-desktop-portal/` to each base directory it searches, so the vendor default belongs at `/usr/share/xdg-desktop-portal/gridline-portals.conf` and an administrator override at `/etc/xdg-desktop-portal/gridline-portals.conf`.

**What the file decides**, measured on the baseline:

| Interface | Backend | Why |
|---|---|---|
| `org.freedesktop.impl.portal.ScreenCast` | the compositor's | `specs/platform-integration` relies on it through screen sharing and screen capture, and the GTK backend does not implement screen capture on Wayland, so a compositor backend is the only implementation rather than the preferred one |
| `org.freedesktop.impl.portal.FileChooser` | `gtk` | `specs/platform-integration` relies on it through the file chooser scenario. `gtk.portal` declares `UseIn=gnome`, which neither element of `Gridline:Hyprland` matches, so before this file existed **no backend claimed this interface at all** |
| every other interface | the compositor's, then `gtk` | A default rather than an enumeration, so no request reaches the frontend's own matching. The frontend passes over a backend that implements nothing for the interface it is resolving, which is how one file serves every compositor Gridline supports |

**The GNOME backend is named nowhere.** It implements eighteen interfaces, including all three the compositor backend serves, and it is installed because GNOME is installed and must remain installed as the fallback desktop. Naming it would make a Gridline session's behaviour depend on whether the user still has GNOME, which inverts the guarantee that removing Gridline leaves a working desktop rather than the other way round. An interface only GNOME's backend implements therefore has no entry, and that absence is deliberate: relying on such an interface is what would need declaring first.

**An interface the session relies on with no entry is a Gridline fault.** Not a case the frontend may improvise on, and not acceptable because something happens to resolve. Until the diagnostic commands exist, nothing reports it at runtime; `add-session-health-check` owns detecting it, and one scenario of the existing portal requirement stays unsatisfiable until that change lands.

**The mapping is assembled at build time rather than written by hand.** Each adapter declares the portal backend serving its compositor, inside the isolation boundary, and a template outside it names no compositor. Supporting a second compositor stays what `compositor-abstraction` requires it to be: adding an adapter, not editing a configuration file that happens to know a compositor's name.

**What the fallback produced before the file existed**, kept because it explains why the file is not optional: `hyprland.portal` lists `UseIn=wlroots;Hyprland;sway;Wayfire;river` without `Gridline`, so screen capture matched on the second element of the identity, through a file the compositor's package owns and can change. That was a working screen-capture path by accident, which is worse than a clean failure, and the file chooser had no match at all.

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
| Screen locker | swaywm | `swaylock` 1.8.4-1 | **universe** | Integration only: the session requires it to be present before starting | **Session does not start.** A desktop that cannot lock must not run, because the failure surfaces as an unlocked unattended machine. The user returns to GDM with the component named |
| polkit authentication agent | polkit, plus Hyprland for the agent | `policykit-1` 127-2ubuntu1.1, binary `polkitd` (main); `hyprpolkitagent` 0.1.3-1 | main / **universe** | Integration only: the session requires the agent to be present before starting | **Session does not start.** Without an agent, privileged actions fail with no prompt and no explanation. The user returns to GDM with the component named |

Both candidates exist on the baseline and neither selection is final; `add-session-lifecycle-integration` still owns the choice. Two facts from the archive constrain it:

**The selected locker authenticates out of the box.** `swaylock` ships `/etc/pam.d/swaylock` containing `auth include login`, so no PAM work is needed to make locking function. That stack is broader than a locker strictly needs, GDM uses a narrower one, which is a security review item rather than a defect. The rejected candidate ships an identical stack, so the observation is about the convention and not about either package.

**The archive units cannot refuse startup.** `hyprpolkitagent.service` and `xdg-desktop-portal-hyprland.service` are systemd user units with `PartOf=graphical-session.target`, `ConditionEnvironment=WAYLAND_DISPLAY` and `Restart=on-failure`. A failing unit restarts quietly and the session comes up regardless. The blocking rule in [architecture.md](architecture.md) therefore needs a Gridline-owned startup check to be enforceable at all; systemd alone will not enforce it.

### How the lifecycle components were chosen

Every candidate for a screen locker and a polkit authentication agent on this baseline is in `universe`, so the support gap applies whichever is chosen and the choice rests on other evidence. Read from the archive packages without installing them:

| Package | Hyprland libraries | Pulls a desktop environment | Idle or lock protocol |
|---|---|---|---|
| `swaylock` | none | no | `ext_session_lock_manager_v1` |
| `hyprlock` | `libhyprgraphics4`, `libhyprlang2`, `libhyprutils10` | no | `ext_session_lock_manager_v1` |
| `swayidle` | none | no | `ext_idle_notifier_v1` |
| `hypridle` | `libhyprlang2`, `libhyprutils10` | no | `ext_idle_notifier_v1` |
| `hyprpolkitagent` | none | no | n/a |
| `lxqt-policykit` | none | **yes**, `lxqt-session` | n/a |
| `mate-polkit` | none | **yes**, MATE components | n/a |

**The locker and the idle daemon were chosen for what they do not link.** `libhyprlang` is the compositor's own configuration language, and [security-and-manageability.md](security-and-manageability.md) records the consequence: its `findConfig` returns the first file it finds and stops, with no merge and no lock, which is why the manageability capability had to exist. Putting that model in the fail-safe path would take the obstacle the project already identified and build the lock on top of it. `swaylock` and `swayidle` link none of those libraries and speak the same protocols.

**The polkit agent was chosen for the same reason the locker was not.** `hyprpolkitagent` is the only candidate that pulls no desktop environment, and it carries no Hyprland library despite its name. What makes a component compositor-specific is what it links and requires, not what it is called, and the same test that rejected `hyprlock` accepts this one.

**One package description is out of date and the binary settles it.** `swayidle`'s description says it is compatible with the KDE idle protocol. Its binary references `ext_idle_notifier_v1`, which is what the compositor advertises and what the adapter reports, so the description describes an older version than the one the archive ships.

## The five blocking components, and who declares each

| Blocking component | Package | Declared by |
|---|---|---|
| Hyprland | `hyprland` | `gridline-session`, by `add-hyprland-session-baseline` |
| Compositor portal backend | `xdg-desktop-portal-hyprland` | `gridline-session`, by `add-hyprland-session-baseline` |
| Screen locker | `swaylock` | `gridline-session`, by `add-session-lifecycle-integration` |
| polkit authentication agent | `hyprpolkitagent` | `gridline-session`, by `add-session-lifecycle-integration` |
| Compositor adapter | none, it is Gridline's own | ships in `gridline-session` |

Nothing on this list is deferred any longer. The section this replaces existed to keep two situations apart, a dependency missing because nobody got to it and one missing because the decision it encodes had not been made, and it is kept in the archived artifacts of the changes that needed the distinction.

**Declaring a blocking component is not the same as enforcing it.** The archive's user units carry `PartOf=graphical-session.target`, `ConditionEnvironment=WAYLAND_DISPLAY` and `Restart=on-failure`, so a failing one restarts quietly and the session comes up regardless. Systemd will not refuse a session for a missing lock component. The session launcher checks for these components itself before starting anything, which is what makes the classification on this page mean something at runtime.

The degrading components carry no declaration, which is a different situation again: a session missing one of them starts, reports the gap, and remains usable.

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
