# Design

## Context

See proposal.md for the measurements. What shapes the design is that the refusal is in a program Gridline does not own, keyed on a variable Gridline sets on purpose, and that several of the obvious fixes each cost something a different part of the project cares about.

**The identity is Gridline's by decision.** `desktop-session` requires the session to identify itself, and portal backend selection reads exactly that variable: a window in which it is wrong is a window in which backend resolution is wrong. So the fix cannot be "name GNOME in the session".

**The launcher already has a role that is the right place.** `gridline-menu apps` starts the launcher for Gridline sessions only and declines to act anywhere else, and it is the one place every launcher-started program passes through. It already carries a Gridline-owned decision about the environment, namely whether to hand the launcher Gridline's configuration.

**The notification path already exists.** The command menu tells a person when an action did not happen, through `notify-send`, which `gridline-session` declares for exactly that reason.

## Goals / Non-Goals

**Goals:**

- Choosing a settings entry opens it.
- The workaround is visible to one program and to nothing else.
- A failure is said, not silent.

**Non-Goals:**

- Changing the session's identity.
- Making every panel of the settings application do something under this session.
- Fixing the same refusal for a program that does not yet have it.

## Decisions

### A stand-in with the application's name, ahead on the launcher's PATH only

A small Gridline-owned program named `gnome-control-center`, in a directory of Gridline's own, runs the real application with `GNOME` appended to its identity. `gridline-menu apps` puts that directory ahead on `PATH` for the launcher it starts, so every desktop entry whose `Exec` names the application by name reaches the stand-in. That is 29 entries today, 28 of them hidden from the launcher, and it covers the ones GNOME adds later, which is why this was chosen over listing them. It also covers any entry Gridline adds for itself (see the first open question).

**Appended, not replaced.** The check was measured to accept the session's own list with `GNOME` added, so the process sees `Gridline:Hyprland:GNOME`. Gridline stays first, and the one program that needed a different answer is the only thing that gets one.

**Alternatives, each rejected on a measurement or a rule:**

- *Name GNOME in the session's identity.* Passes the check, and changes what every program and every portal sees. The project asked for the identity to be Gridline's.
- *Give the launcher a command prefix.* fuzzel supports one, and it applies to every program started from it, so the whole point of confining the change is lost.
- *Shadow the desktop entries by putting a directory of Gridline's first in the data path.* Works, and needs a copy of each of the 29 entries and a new one every time GNOME adds a panel.
- *Hide the broken entries.* Removes the panel the person asked for, and any other application whose entry carries the same key and works.
- *Patch the application or its entries.* They belong to another package, and the project does not edit those.

### The stand-in finds the real program by absolute path

It must not look the application up by name, which would find itself. The absolute path is the Debian one, and if it is not there the stand-in says so rather than looping.

### Failure is reported only when the program fails to start

The real application is long-running, so its exit status is not a verdict on the launch. A non-zero exit within a few seconds of starting is a failure to start. The same exit hours later is a crash, and a notification about it at that point would be noise about something the person already saw.

So the stand-in runs the application and tells the person only if it exits non-zero inside that window. The window is a constant at the top of the program, with the reason beside it.

### Only the launcher gets the directory

Programs started from the compositor's own key bindings, and from a terminal, do not pass through `gridline-menu apps`, so typing `gnome-control-center` in a terminal still fails. That is a limit rather than an oversight: putting the directory on the whole session's `PATH` would widen the change from "what the launcher starts" to "everything", for a command a person is unlikely to type.

## Risks / Trade-offs

- **The settings application, once it believes it is under GNOME, offers panels that write settings nothing here reads.** → Likely for Mouse and Touchpad, Keyboard, Display and Appearance, whose effect normally arrives through `gnome-settings-daemon` or the shell, neither of which runs. This is a hypothesis and the first task is to measure it panel by panel. It decides whether the requirement is enough or whether a person is being handed controls that do nothing.
- **The stand-in's name has to match the application's.** → If Ubuntu renames the binary, the stand-in becomes dead code and the entries break the way they do today. A check that the shipped name still matches a package's file is cheap and is a task.
- **`PATH` ahead of the system's is a place something could be shadowed.** → The directory holds exactly one file, named for the one program with the refusal, and nothing else is ever to go in it.

## Open Questions

- **Whether Gridline ships its own visible entries for Wi-Fi, Bluetooth and Network.** Measured: only "Settings" is listed, and typing Wi-Fi finds nothing. Fixing the stand-in alone makes "Settings" open, and the person then navigates inside it. Gridline-owned entries (`OnlyShowIn=Gridline;`, `Exec=gnome-control-center wifi`, resolved through the stand-in) would make the three directly reachable, at the cost of three files and a wider requirement. Undecided; the delta's first scenario names all four and is not yet true of the design.
- Which panels do nothing under this session. Measured in task 1.4 (see tasks.md); the follow-up is to apply those settings or say plainly that they are not applied.
- Whether to extend the `PATH` to the whole session once a second refusing program turns up. One does not exist today.
