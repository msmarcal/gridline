# Design

## Context

See proposal.md for motivation. Four facts measured on the baseline shape the approach.

**The frontend matches by name, and no backend names Gridline.** `xdg-desktop-portal` 1.21.1 tries each entry of `XDG_CURRENT_DESKTOP` in order against each backend's `UseIn` list. Gridline's identity is `Gridline:Hyprland`. The three installed backends declare:

| Backend | `UseIn` | Interfaces it claims |
|---|---|---|
| `hyprland.portal` | `wlroots;Hyprland;sway;Wayfire;river` | Screenshot, ScreenCast, GlobalShortcuts |
| `gtk.portal` | `gnome` | FileChooser, AppChooser, Print, Notification, Inhibit, Access, Account, Email, DynamicLauncher, Lockdown, Settings |
| `gnome.portal` | `gnome` | eighteen, including ScreenCast, Screenshot and GlobalShortcuts |

So screen capture resolves through `Hyprland`, the second element, and the file chooser resolves through nothing at all.

**Twelve interfaces are claimed by more than one installed backend**, including the three the compositor backend claims, each also claimed by `gnome.portal`.

**The filename and location are decided by the frontend, not by preference.** It case-folds each desktop name before building the filename and appends `xdg-desktop-portal/` to each base directory it searches, so the file is `/usr/share/xdg-desktop-portal/gridline-portals.conf` and nothing else resolves.

**Three archive packages are present only through someone else's dependency.** `xdg-desktop-portal`, `xdg-desktop-portal-gtk` and `xwayland` are installed because `ubuntu-session` and `libmutter` require them. `platform-integration` forbids relying on a component without declaring it.

## Goals / Non-Goals

**Goals:**

- Every relied-on interface served by a backend Gridline named, matched through `Gridline` rather than through the compositor's name.
- The portal packages declared rather than inherited.
- The mapping and its reasons recorded where the component matrix already discusses portals.

**Non-Goals:**

- The diagnostic that reports a missing or incomplete mapping. `add-session-health-check` owns the command surface, and one scenario of the existing requirement stays unsatisfiable until it lands.
- Writing or patching a portal backend.
- Configuring Snap interface connections. Snap behaviour is verified, not configured.

## Decisions

### The mapping names a backend for every relied-on interface, not only the contested ones

A first reading suggests naming only the interfaces two backends fight over. Measurement says otherwise: the file chooser has the opposite problem, no backend claims it under this identity, and an interface nothing claims is as unresolved as one two backends claim.

So the file names a backend for each interface the session relies on. The compositor backend takes the interfaces that need the compositor: `ScreenCast`, `Screenshot` and `GlobalShortcuts`. The GTK backend takes the toolkit interfaces it implements, with the file chooser first among them. An interface with no entry is a gap this change is responsible for, not a case the frontend may improvise on.

### `gnome.portal` is named nowhere

It claims eighteen interfaces, including all three the compositor backend serves, and it is installed because GNOME is installed and must stay. Naming it for anything would make a Gridline session's behaviour depend on whether the user still has GNOME, which is the opposite of what the fallback guarantee promises. Where GNOME's backend is the only implementation of an interface the session does not rely on, the interface simply has no entry, because relying on it is what would need declaring.

### Resolution must stop depending on the compositor's name

The test that distinguishes a real fix from an apparent one is whether removing `Hyprland` from the identity would change the outcome. Today it would: the only match is on that element. After this change the match is on `Gridline`, and the compositor's own `UseIn` list becomes irrelevant to Gridline sessions. That property is what makes the result survive a change to a file the compositor's package owns.

### The three archive packages are declared where they are required

`xdg-desktop-portal` and `xdg-desktop-portal-gtk` are required by `gridline-config`, because the mapping it ships names backends those packages provide and a mapping naming an absent backend is worse than no mapping. `xwayland` is required by `gridline-session`, because X11 applications are a session capability rather than a configuration one.

The alternative was declaring all three on `gridline-session`, which is simpler and wrong: `gridline-config` installed without the backends it names produces a configuration file that points at nothing, and the package that ships a claim should be the package that requires what the claim depends on.

### The mapping is shipped configuration, not a conffile

`docs/configuration.md` fixes the test: a file an administrator writes or is expected to edit is a conffile; a file Gridline produces is not. The mapping is Gridline's own decision about which backend serves which interface, and an administrator who overrides it does so through the frontend's own administrator path at `/etc/xdg-desktop-portal/`, which exists for exactly that and which Gridline does not own.

## Risks / Trade-offs

- **The compositor package could stop claiming an interface the mapping names.** → The mapping names the backend by its D-Bus name rather than relying on the backend's own matching, so the entry survives a change to that package's `UseIn`. What it does not survive is the backend being removed, which is a dependency problem and is why the packages are declared.
- **An interface relied on later and not added to the mapping fails the new requirement silently.** → Until `add-session-health-check` exists, nothing reports it at runtime. The mitigation here is a build-time check only if one is cheap; otherwise the obligation is recorded and the diagnostic change owns detection.
- **Naming the GTK backend for the file chooser ties Gridline to a package whose `UseIn` says `gnome`.** → The entry names the backend directly, so the mismatch between its `UseIn` and Gridline's identity stops mattering. That mismatch is exactly why the entry is needed.
- **Snap confinement is verified rather than configured, so a failure there has no owner in this change.** → If a confined Snap cannot reach a portal, the finding is recorded with the change that owns the affected component rather than absorbed here.

## Migration Plan

Installing this version adds a configuration file and three dependency declarations. A session already running is unaffected until it restarts, because the frontend reads the mapping when it starts. Removing the Gridline packages removes the mapping, and backend resolution returns to the frontend's own matching, which is the behaviour this change replaces rather than something it breaks.
