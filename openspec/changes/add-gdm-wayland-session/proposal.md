# Proposal: add-gdm-wayland-session

## Why

Installing Gridline today gives you three packages that create four empty directories. Nothing appears at the login screen, so none of the seven `desktop-session` requirements can be exercised: they describe a session that does not yet exist.

This change makes the session appear and makes it fail correctly. That second half is the point. The fallback guarantee is the constraint the whole project rests on, and it is the one thing no amount of specification can establish: either selecting Gridline returns you to a working login screen with the cause recorded, or it does not. Until a session entry exists, that promise has never been tested.

The compositor is deliberately not part of it. `add-hyprland-session-baseline` delivers the adapter, and Hyprland is not even installed on the baseline. What this change can deliver, and verify, is the path from selecting the session to being handed back control with a named reason.

## What Changes

**This change ships implementation.** It creates the session entry and the launcher, and verifies both on the supported baseline.

- Registers a Wayland session entry from `gridline-session`, installed into `/usr/share/wayland-sessions/` as a Gridline-named file. The directory belongs to `ubuntu-session` and stays its own.
- Sets the desktop identity through `DesktopNames` in that entry, which is the mechanism GDM uses to populate `XDG_CURRENT_DESKTOP`.
- Adds the session launcher in `/usr/libexec/gridline/`, which checks the components the session requires, and when one is missing writes an attributable record to the journal naming it and exits so GDM regains the seat.
- Adds `TryExec` pointing at the launcher, so the entry is offered only when the launcher is actually present.
- Verifies on the baseline: the entry appears in GDM's session list, GNOME stays selectable and preselected, selecting Gridline returns to GDM with the missing compositor named in the journal, and removing the packages takes the entry with it.

Teardown is specified but only partially exercisable: a session that never reaches a usable desktop has no logout to perform. What this change verifies is that the launcher releases the seat on the failure path; the logout path becomes testable when a compositor runs.

## Capabilities

### New Capabilities

None.

### Modified Capabilities

- `desktop-session`: gains one requirement, that a session entry is offered only when the launcher it names is present. Inspecting the baseline showed the mechanism exists and is already relied on: `ubuntu.desktop` carries `TryExec=/usr/bin/gnome-session`, and GDM hides an entry whose `TryExec` cannot be resolved. Without it, a partially configured install would offer a session that cannot start, which contradicts the promise that a failure is visible and attributable rather than mysterious. No existing requirement is modified or removed.

## Impact

- **Affected specs:** `desktop-session`, by addition only. Six of its seven existing requirements become exercisable for the first time.
- **Code:** the session entry and the launcher, both in `gridline-session`. First executable Gridline code that a user can reach.
- **Breaking changes:** none. The entry is additive and GNOME remains preselected.
- **System effects during implementation:** verification installs the packages and selects the Gridline session at the login screen, which means logging out of the current desktop. The failure path is the thing being tested, so the expected outcome is a return to GDM.
- **Unblocks:** `add-hyprland-session-baseline`, which replaces the missing-compositor failure with a real adapter.

## Out of scope

- Starting a compositor. The adapter contract and the Hyprland adapter are `add-hyprland-session-baseline`. This change's launcher reports the compositor as missing, which is the specified behavior when a required capability is unavailable.
- Portals, audio, networking, Bluetooth, locking and the polkit agent. Those are `add-desktop-portal-integration` and `add-session-lifecycle-integration`.
- Shipped defaults, keybindings and themes, which are `add-gridline-configuration-defaults`.
- The diagnostic command surface. `gridline doctor` is `add-session-health-check`; this change writes the journal record that command will later read.
- Declaring a dependency on a compositor package. The launcher reports a missing compositor rather than requiring one, so nothing here needs `hyprland` installed.

## Downstream changes

| Requirement | Owned by | Why there |
|---|---|---|
| A session entry is offered only when its launcher is present | `add-gdm-wayland-session` | `TryExec` lives in the session entry this change writes |
| Session is selectable in GDM | `add-gdm-wayland-session` | The entry is what GDM reads |
| Session is opt-in, never the default | `add-gdm-wayland-session` | Adding an entry must not change what GDM preselects |
| Session launch reaches a usable desktop or fails visibly | `add-gdm-wayland-session` | The launcher decides which of the two happens and records why |
| A failed Gridline session is recoverable without Gridline | `add-gdm-wayland-session` | Returning the seat to GDM is the launcher's job; the guarantee holds when nothing Gridline runs |
| Session teardown releases the seat | `add-gdm-wayland-session` | Verified here on the failure path, and again by `add-ci-vm-smoke-tests` once a compositor runs |
| Session identifies itself to the desktop stack | `add-gdm-wayland-session` | `DesktopNames` in the entry is what sets the identity |
| GNOME session remains installed and supported | `define-debian-package-architecture` | Already archived: install and removal behavior is packaging, not session logic |

This change runs before `define-supported-ubuntu-integration`, inverting the order the foundation recorded. The inversion is safe because this change declares no dependency entry: the launcher reports a missing compositor rather than requiring one, so it needs nothing from the component matrix that change produces. It is worth doing because this is the first change where the session appears at the login screen and the fallback path can be exercised, and because leaving the recorded sequence disagreeing with the executed one is the divergence this project has paid for repeatedly.

`add-session-health-check` owns no requirement here. It reads the journal record this change writes, which is why the record has to be attributable before that change exists.
