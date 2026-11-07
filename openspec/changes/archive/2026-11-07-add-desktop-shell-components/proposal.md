# Proposal

## Why

The session works and shows nothing. A user logs into Gridline and gets a bare compositor: no bar, so the clock, the battery and the workspace they are on are invisible; no notification daemon, so anything an application tries to tell them is discarded silently; no background, so the desktop is whatever the compositor paints when nothing is there. `add-gridline-configuration-defaults` made the session operable from the keyboard, which is half of the project's thesis. The other half is that it should look like a finished desktop rather than a compositor someone has not configured yet.

## What Changes

**This change ships implementation.** It selects and configures the components that draw the desktop continuously, and gives each one a way to start.

- Selects and declares a status bar, a notification daemon and a wallpaper daemon, all from the archive.
- Configures each enough to be usable on first login, without the user editing anything.
- Starts each through the session target, extending the `Upholds=` mechanism that already holds the idle daemon: these degrade the desktop when absent and do not make it unsafe, so none of them blocks startup.
- Records what the rule that rejected two earlier components actually tests. The wallpaper daemon selected here links the same configuration library they were rejected for, and the difference is that its configuration path can be named rather than searched for.
- Corrects `docs/user-experience.md`, which records the notification daemon as absent from the archive when it is present under a different package name.

**Out of scope.** The application launcher, deliberately: `docs/user-experience.md` calls the command menu built over it the element worth the most to the experience, and choosing a launcher now would constrain that menu before anyone has designed it. The two belong in one change. Also out: the capture tools, which are keybindings over three small packages and carry no styling or lifecycle question; opinionated terminal, editor and monitor defaults; and theming, which the same document already separates into Gridline-authored integration rather than archive-package configuration.

**The boundary with theming needs stating rather than assuming.** A bar with no styling is not a finished desktop, so this change ships some. What it ships is a default appearance for the components it delivers, not a mechanism that restyles components together, which is what the theme is and what a later change owns.

## Capabilities

### New Capabilities

None.

### Modified Capabilities

- `desktop-session`: gains one requirement, that the session presents a desktop rather than a bare compositor. The capability already requires a session that reaches a usable desktop and, since the last change, one that is operable from the keyboard without configuration. Neither says anything about the user being able to see the state of their own machine, or about a notification reaching them rather than being dropped.
- `desktop-session`: one requirement is modified for its prose and not its behaviour. `confine-gridline-configuration` left "a newly installed session is usable without configuration" stating, in the requirement itself, the outcome that one of its own scenarios already states word for word, which put the requirement text over the length the validator accepts. The duplicated clause is dropped and the scenario keeps it. Nothing is required here that was not required before, and no task implements it.

`platform-integration` is **not** modified. Its scenarios about network, audio, battery and bluetooth already require those to work; a bar makes them visible, which is presentation rather than a new integration.

## Impact

- `packages/gridline/src/adapters/<compositor>/`: the compositor-side configuration that launches and places these components.
- `packages/gridline/src/config/`: their own configuration and styling, which is compositor-neutral and belongs outside the adapter.
- `packages/gridline/debian/control`: three dependencies on `gridline-session`, all `universe`.
- `packages/gridline/debian/gridline-config.install`: the configuration and styling each component reads.
- `packages/gridline/src/session/gridline-session.target`: the components held by `Upholds=`, plus a Gridline-authored unit for the one the archive ships without one.
- `docs/user-experience.md`: the corrected availability of the notification daemon, and which elements of the target table are now delivered.
- `docs/component-ownership.md` and `docs/package-matrix.md`: three selections and their rows.
- Verification runs on the LXD VM, from the greeter, where what the user sees is the measurement that matters and several of these cannot be checked any other way.
