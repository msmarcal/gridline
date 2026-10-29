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

### A default plus explicit entries, rather than an enumeration

The first draft of this design said the file names a backend for each relied-on interface and nothing else, and that it would name them by D-Bus name. Reading `portals.conf(5)` corrected both points. Entries take the backend's short name, the basename of its `.portal` file, and the format has a `default` key covering every interface not listed explicitly.

That key changes the shape of the answer. An enumeration leaves every unlisted interface resolving through the frontend's own `UseIn` matching, which is the accidental path this change exists to remove; it would remove it for two interfaces and leave it for the rest. A default removes it for all of them in one line.

So the file sets `default=hyprland;gtk`, searched in that order, and then names the two interfaces the specification relies on explicitly. The default means no request reaches the frontend's own matching. The explicit entries mean the two the specification names are decided by name rather than by the order of a list, so reordering the default cannot silently move them.

The specification relies on exactly two portal interfaces, `ScreenCast` through the screen sharing and screen capture scenarios and `FileChooser` through the file chooser scenario, derived from the scenarios rather than from what the backends offer. `Screenshot` and `GlobalShortcuts` are claimed by the compositor backend and no requirement relies on them; the default covers them without this change asserting a reliance the project has not taken.

### The mapping is assembled at build time from the adapters

The isolation gate rejected the first version of this file, and it was right to. A mapping naming `hyprland` lives outside the adapter boundary, and `compositor-abstraction` requires that supporting a second compositor be confined to selecting which adapter is active. A file someone must edit to add a compositor is not confined.

Moving the file inside the boundary was not available: `package-delivery` and three documents record `gridline-portals.conf` as `gridline-config`'s, and a mapping is configuration rather than session logic. Excluding `src/config/` from the gate was available and was rejected, because the gate would then be weaker for every change after this one and `add-ci-vm-smoke-tests` inherits it.

So the knowledge moved rather than the file. Each adapter declares the portal backend serving its compositor, in `src/adapters/<compositor>/portal-backend`, inside the boundary. A template in `src/config/` carries everything else and names no compositor. The build assembles them, `gridline-config` ships the result, and adding a compositor means adding an adapter directory.

One mapping has to serve every compositor, because the frontend reads one file per desktop identity and Gridline's identity names Gridline first whichever compositor is active. That works because the frontend searches a backend list for an implementation of the interface it is resolving and passes over a backend that offers none, which `portals.conf(5)` states and which is already why `gtk` is passed over for screen capture. Listing a compositor backend that is not installed therefore costs nothing.

### `gnome.portal` is named nowhere

It claims eighteen interfaces, including all three the compositor backend serves, and it is installed because GNOME is installed and must stay. Naming it for anything would make a Gridline session's behaviour depend on whether the user still has GNOME, which is the opposite of what the fallback guarantee promises. Where GNOME's backend is the only implementation of an interface the session does not rely on, the interface simply has no entry, because relying on it is what would need declaring.

### Resolution must stop depending on the compositor's name

The test that distinguishes a real fix from an apparent one is whether removing `Hyprland` from the identity would change the outcome. Today it would: the only match is on that element. After this change the match is on `Gridline`, and the compositor's own `UseIn` list becomes irrelevant to Gridline sessions. That property is what makes the result survive a change to a file the compositor's package owns.

### The three archive packages are declared where they are required

`xdg-desktop-portal` and `xdg-desktop-portal-gtk` are required by `gridline-config`, because the mapping it ships names backends those packages provide and a mapping naming an absent backend is worse than no mapping. The compositor's backend is already a dependency of `gridline-session`, declared by `add-hyprland-session-baseline` because the adapter cannot report `screencast` without it. `xwayland` is required by `gridline-session`, because X11 applications are a session capability rather than a configuration one.

The alternative was declaring all three on `gridline-session`, which is simpler and wrong: `gridline-config` installed without the backends it names produces a configuration file that points at nothing, and the package that ships a claim should be the package that requires what the claim depends on.

### The mapping is shipped configuration, not a conffile

`docs/configuration.md` fixes the test: a file an administrator writes or is expected to edit is a conffile; a file Gridline produces is not. The mapping is Gridline's own decision about which backend serves which interface, and an administrator who overrides it does so through the frontend's own administrator path at `/etc/xdg-desktop-portal/`, which exists for exactly that and which Gridline does not own.

## Risks / Trade-offs

- **The compositor package could stop claiming an interface the mapping names.** → The mapping names the backend directly rather than relying on that backend's own `UseIn` list, so the entry survives a change to it. What it does not survive is the backend being removed, which is a dependency problem and is why the packages are declared.
- **An interface relied on later and not added to the mapping fails the new requirement silently.** → Until `add-session-health-check` exists, nothing reports it at runtime. The mitigation here is a build-time check only if one is cheap; otherwise the obligation is recorded and the diagnostic change owns detection.
- **Naming the GTK backend for the file chooser ties Gridline to a package whose `UseIn` says `gnome`.** → The entry names the backend directly, so the mismatch between its `UseIn` and Gridline's identity stops mattering. That mismatch is exactly why the entry is needed.
- **Snap confinement is verified rather than configured, so a failure there has no owner in this change.** → If a confined Snap cannot reach a portal, the finding is recorded with the change that owns the affected component rather than absorbed here.

## Migration Plan

Installing this version adds a configuration file and three dependency declarations. A session already running is unaffected until it restarts, because the frontend reads the mapping when it starts. Removing the Gridline packages removes the mapping, and backend resolution returns to the frontend's own matching, which is the behaviour this change replaces rather than something it breaks.
