# Proposal

## Why

Settings does nothing when chosen from the application launcher, and Wi-Fi and Bluetooth cannot be chosen there at all. 29 desktop entries run `gnome-control-center`, but 28 are `NoDisplay=true` and only "Settings" is listed. The program refuses to start: `Running gnome-control-center is only supported under GNOME and Unity, exiting`, exit 1. The person who picks "Settings" sees the launcher close and nothing open, with no message; typing "Wi-Fi" shows an empty list.

Connecting to a network is the first thing anybody does on a new machine. This is the one program on the system with that refusal, found by searching every binary under `/usr/bin` and `/usr/libexec` for the message, so the problem is narrow even though the symptom is not.

## What Changes

**This change ships implementation.**

- Makes the settings panel start from the launcher, by running it with a desktop identity it accepts. That identity is given to the one program's own environment and to nothing else: the session, the launcher and every other program keep naming Gridline.
- Adds three launcher entries owned by Gridline, Wi-Fi, Bluetooth and Network, shown in Gridline sessions only, because the archive's own entries for them are hidden from the launcher. Each opens its panel through the same program.
- Tells the person when the panel could not be started, instead of leaving them to wonder whether the launcher or the machine is broken.
- Records which panels of the settings application do something under this session and which write a setting nothing here reads. Measuring that is a task, and what it finds is recorded rather than hidden.

**Out of scope.**

- Changing the session's identity. `Gridline:GNOME` would pass the check and was measured to, and it would also change what every program and every portal sees. The project asked for that identity to be Gridline's, and this is not a reason to take it back.
- Making the inert panels work. If the measurement finds panels whose settings nothing applies, applying them is a change of its own.
- Declaring the settings application as a dependency. It arrives with the Ubuntu desktop, and this change assumes it the way the session assumes a display manager.

## Capabilities

### New Capabilities

None.

### Modified Capabilities

- `desktop-session`: one requirement **added**, that a person can reach the machine's settings from the launcher. The capability already owns what a session presents and how a person reaches it.

## Impact

- `packages/gridline/src/session/gridline-menu`: the launcher role, which gains the one place the identity workaround is made visible to launched programs.
- `packages/gridline/src/applications/`: three desktop entries, Gridline-named, shown only where Gridline is the session.
- `packages/gridline/src/session/`: a small Gridline-owned program standing in for the settings application's name, in a directory of Gridline's own.
- `docs/user-experience.md` and `docs/component-ownership.md`: neither mentions settings at all.
- Verification runs on the LXD VM with the session unlocked, because it drives the real launcher and reads which window appears.

## Prior measurement

Four facts settled before writing, each changing what gets built.

**The check reads one variable and matches any element of it.** Tried with the display unset so that a passing check produces a different message than a refusing one: `Gridline:GNOME`, `GNOME:Gridline` and `gnome` all pass; `Gridline:Hyprland`, `X-Generic` and an empty value are refused; `XDG_SESSION_DESKTOP=GNOME` with Gridline's identity is refused. Unexpectedly `Unity` is refused too, so the message promises more than the program does.

**So the workaround can append rather than replace.** The panel accepts the session's own list with `GNOME` added, which keeps Gridline first.

**Hiding the broken entries was considered and rejected.** The launcher can filter entries by the `OnlyShowIn` key, and all five sampled panel entries say `GNOME`, so filtering would remove the 29 broken entries. It would also remove the panel, which is the thing the person asked for, and any other application that carries the key and works fine.

**The only way to be sure the whole path works is to drive it.** The workaround was tried by hand and opened `org.gnome.Settings`. Whether it opens from the launcher, as the person would reach it, is the first task.

**Only one of the 29 entries is visible (found while applying, task 1.1).** 28 carry `NoDisplay=true`, so the launcher never lists them: the person can reach "Settings" and nothing named Wi-Fi, Bluetooth or Network. The first draft of this proposal and of the delta spec assumed those could be chosen, from the entry count alone and without opening the launcher. Decided: this change adds visible entries for the three, owned by Gridline (design.md).
