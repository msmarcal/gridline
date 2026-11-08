# Recovery

What to do when a Gridline session will not start.

The short version: **select Ubuntu at the login screen.** Nothing on this page requires a working Gridline desktop, a Gridline command, or a terminal inside a Gridline session. That is the point of it.

## The recovery path

### 1. You are already back at the login screen

When a required component of the Gridline session fails to start, the session exits and returns you to GDM. You do not need to force anything, and you do not need to reboot.

If the screen appears to hang instead of returning, switching virtual terminals (`Ctrl+Alt+F1` through `F6`) reaches a text login, and GDM is usually on one of the first two.

### 2. Select the Ubuntu session

At GDM, choose your user, then use the session selector (the gear or session menu, depending on the GDM version) and pick the regular **Ubuntu** session. It starts normally.

The Ubuntu GNOME session is installed and supported at all times while Gridline is installed. Installing Gridline never removed it, never changed it, and never made it the non-default. If you had been using GNOME before installing Gridline, it is still what GDM preselects.

### 3. Find out what failed

From the GNOME session, open a terminal and read what the Gridline launcher wrote:

```
journalctl -t gridline-session
```

`gridline-session` is the **syslog identifier** the launcher writes its records under, and `-t` selects by that identifier. One command is enough because selecting by author needs no time window, no knowledge of which boot the failure happened in, and no search for the word "Gridline" in records other programs wrote. A failure records which component did not start.

Searching the journal by word instead returns whatever else mentions Gridline: other programs' records, and every line on a machine whose hostname contains the name. Selecting by identifier returns the launcher's records and nothing else.

If the failure was a missing compositor capability, the record names the capability. If it was a missing fail-safe component, such as the screen locker or the polkit authentication agent, the record names that component and the session's refusal to start without it is deliberate rather than a bug. See [architecture.md](architecture.md) for why those two block startup.

`journalctl` is part of a stock Ubuntu system. Nothing in this step needs a Gridline package to be working, or installed at all.

### 4. Optional: ask the diagnostics

If `gridline-tools` is installed, `gridline doctor` names each component the
session expects, whether it is running, and what its absence costs:

```
gridline doctor
```

```
A Gridline session is running. Components it expects:

  bar              running        as of 09:14
  locker           ready          as of 09:14
  notifications    running        as of 09:14
  polkit-agent     running        as of 09:14
  wallpaper        failed         as of 09:16  (the desktop works without it, with less)

Each line is the last thing the session recorded about that
component, at the time shown. Nothing here polls: a component is
recorded when it starts, and again when it stops or fails.

1 component(s) are not running.
```

Read the times. Every line is the last thing the session recorded, not a check
made when you ran the command: nothing here watches anything. A component that
stopped an hour ago carries an hour-old time, and that is the difference
between this report and a lie. An earlier version of this command printed no
times and had no way to learn that a component had stopped, so it reported a
dead authentication agent and a given-up screen lock as running for the rest of
the sessions they died in. Both were read here, in this document's own worked
example, and believed.

`ready` is not a hedge. The screen locker runs only while the screen is locked,
so on a working desktop it is installed and not running, which is neither
`running` nor a problem. Everything else here is expected to be running.

It reports from what the session wrote down as each component started, stopped
or failed, so it answers **without a Gridline session running**, which is the
case you are most likely to be in. The first line says which of those you are
looking at: a report from a session that has ended is history rather than
status, and it should not be read as if the machine were in that state now.
With no session recorded at all it says so in those words rather than printing
an empty list, because silence and health must not look alike.

It exits 0 when every component is accounted for, 1 when one is not, and 64 if
you mistype the command, so it is usable from a script as well as read by a
person.

**What it does not cover** is the components Gridline integrates with rather
than starts: the network, audio, Bluetooth, battery, XWayland and the portals.
Those degrade a session too and nothing reports them yet;
[component-ownership.md](component-ownership.md) says which rows are which and
why that is a different problem rather than an oversight.

This step is a convenience, not a dependency. Everything needed to diagnose a
failed session is available from step 3 with stock tools, because a diagnostic
command that only runs inside a working session would be useless exactly when
it is needed.

### 5. If you want Gridline gone

Removing the Gridline packages returns the system to a stock Ubuntu desktop. No session entry remains, GNOME works, and nothing Gridline owned is left behind apart from configuration files you edited yourself.

You can keep using GNOME indefinitely with Gridline still installed. An unselected session costs nothing.

## The screen is locked and the password does nothing

You are looking at a screen that is not Gridline's. It belongs to the
compositor, and it appears when the program that holds the lock dies: the
Wayland protocol that locks a session keeps it locked when its client goes
away, which is correct and is what stops a crash from exposing your desktop.
There is nothing to type into, because the thing that accepts passwords is the
part that died.

**Two things on that screen will send you the wrong way, and Gridline cannot
change what it says.** It names `hyprlock`; Gridline's locker is `swaylock`.
And it prints the terminal your session is on, which was wrong when this was
measured: it read `tty 0` for a session that was on `tty2`, so following it
would have you pressing a key combination for a terminal that does not exist.
The steps below do not depend on either.

### Getting back in

Switch to another virtual terminal and log in there:

```
Ctrl+Alt+F3
```

**On a virtual machine this step may not work, and the reason is not Gridline's.**
If you are watching the machine through a window on another computer, that key
combination is usually captured by the computer you are sitting at and never
reaches the guest. Use whatever second console your hypervisor provides
instead: with LXD that is `lxc console <name>` from the host, which gives a
text login inside the machine without needing the keystroke. Some viewers also
have a menu for sending key combinations through.

Start the lock again:

```
systemctl --user start gridline-lock.service
```

If it refuses because it has given up after repeated failures, clear that first:

```
systemctl --user reset-failed gridline-lock.service
systemctl --user start gridline-lock.service
```

Switch back to your session and type your password as usual. To find which
terminal it is on, ask, rather than trusting the number on that screen:

```
loginctl list-sessions
```

The graphical session is the one with a seat, and its `TTY` column gives the
number: `tty2` means `Ctrl+Alt+F2`.

### Why you should not need this

The lock is a systemd unit of its own, and a locker that dies is started again
without anyone asking. This page exists for the cases that outlive that: a
locker that cannot start at all, which stops being retried after three attempts
in a minute rather than looping forever, and anything that stops the unit
deliberately.

Both leave a record. `journalctl --user -u gridline-lock.service` names what
happened, including the case where the locker is missing from the machine
entirely.

### If none of that works

The session is still locked, which means your desktop is not exposed. You can
end it without logging back into it: from the other terminal,

```
loginctl terminate-session <id>
```

with the id from `loginctl list-sessions`. That closes the session and returns
you to the login screen, losing whatever was unsaved in it. It is the last
step rather than the first for that reason.

## Choosing what the login screen offers

On a machine with Gridline installed the login screen offers **Gridline** and
**Ubuntu**. The compositor Gridline depends on ships session entries of its own,
and `gridline-session` diverts them, so they are not offered.

**Why a session package hides another's entry.** Those entries are on the
machine because Gridline's dependency put them there: `apt-mark showmanual` does
not list the compositor. Installing one desktop would otherwise offer two, and
the one nobody chose runs a session that inherits Gridline's keybindings and none
of Gridline's components. The diversion restores what the login screen offered
before Gridline arrived. What it costs is recorded in
[security-and-manageability.md](security-and-manageability.md) and the mechanism
in [package-matrix.md](package-matrix.md).

**Removing Gridline gives them back.** The diversion is removed with the package,
on `remove` and on `purge` alike, and the compositor's files return to their own
paths byte for byte, which `dpkg --verify hyprland` confirms. A machine that
loses Gridline offers what it offered before.

### Getting the compositor's own session back without removing Gridline

The compositor stays installed and startable; only its login entry is hidden. To
have it offered again, place your own entry in the directory that takes
precedence over the package's:

```
sudo mkdir -p /usr/local/share/wayland-sessions
sudo cp /usr/share/wayland-sessions/hyprland.desktop.gridline-diverted \
        /usr/local/share/wayland-sessions/hyprland.desktop
sudo systemctl restart gdm3
```

Undo it by deleting that file and restarting the display manager again. Nothing
the `hyprland` package owns is touched and nothing Gridline did is unpicked:
`/usr/local/share` is the local administrator's, which is exactly why Gridline
cannot write there and you can.

**That session is not a Gridline session, and it no longer pretends to be.** It
arrives with the compositor's own configuration and none of Gridline's: no
keybindings, no locker, no polkit agent, no portal mapping. That is what
choosing it means, and it is now what it does.

It used to be worse. Gridline placed its defaults at the path any session of
that compositor reads, so this session inherited Gridline's keybindings while
having none of the components they assume, and the lock key did nothing in
silence. `confine-gridline-configuration` ended that by naming Gridline's
configuration path rather than occupying the compositor's. Measured afterwards:
the configuration such a session produces is byte for byte the file the
compositor generated on this machine before Gridline was installed.

### Hiding a session entry yourself

The same mechanism hides rather than restores, by giving the file `Hidden=true`
instead of copying the real one:

```
sudo tee /usr/local/share/wayland-sessions/<name>.desktop >/dev/null <<'EOF'
[Desktop Entry]
Type=Application
Name=<name>
Hidden=true
EOF
```

**Do not hide the stock Ubuntu session.** It is the fallback every recovery on
this page depends on, and a machine whose only remaining entry is Gridline has no
route back if Gridline stops starting.

Both halves of this were measured on the baseline rather than taken from the
specification. A probe file carrying a different `Name` showed the
higher-precedence file replacing the packaged one rather than appearing alongside
it, which is what establishes that the search deduplicates by filename. The same
probe carrying `Hidden=true` removed the entry from the selector, which is what
establishes that the display manager honours that key for session entries.

## Getting the desktop shell back in another session

Installing Gridline brings a status bar, a notification daemon and a wallpaper
daemon, and each of those packages enables its own user unit. That unit lives
under `graphical-session.target.wants`, which belongs to **every** graphical
session, so without intervention they would start in your GNOME session too.
`gridline-session` turns those three off at install, and enables again on
removal the ones it turned off.

Nothing of this is visible in a GNOME session, and that is not a reason to
leave it: measured there, the bar fails five times before systemd gives up
because GNOME implements no layer shell, and the notification daemon exits at
once because GNOME already holds the notification D-Bus name. Two failed units
in a session Gridline promised not to touch.

If you do want one of them in another session, enable it yourself:

```
systemctl --user enable waybar.service
```

**Gridline will not undo that.** Each unit is turned off once, recorded in
`/var/lib/gridline/disabled-user-units`, and left alone afterwards, so a later
upgrade does not revisit your choice.

Removing Gridline enables again only the units it disabled, so one you had
disabled before installing Gridline stays that way.

This is the second of two things a Gridline package does to another package.
The first is the session-entry diversion above, and the two answer the same
question from different ends: the diversion decides what the login screen
offers, and this decides what starts once a session is running.
[package-matrix.md](package-matrix.md) carries both with the reasoning.

## Why recovery does not depend on Gridline

Every step above works when no Gridline component runs:

| Step | What it needs |
|---|---|
| Returning to GDM | GDM, which Gridline never modifies |
| Selecting Ubuntu | The GNOME session, which stays installed and supported |
| Reading the failure | `journalctl`, present on a stock Ubuntu system |
| Removing Gridline | `apt`, and Gridline packages that remove cleanly |

This is a deliberate design constraint, not a happy accident. A recovery path that runs through the failed component is not a recovery path. It is why the recoverability guarantee belongs to the change that delivers the session and its fallback behavior, rather than to the change that delivers the diagnostic tooling: the guarantee has to hold when the tooling is absent.

## Correspondence with the specifications

This document is the "documented recovery procedure" that `openspec/specs/desktop-session/spec.md` refers to. Each of its fallback requirements maps to a step above:

| Requirement in `specs/desktop-session/spec.md` | Where it appears here |
|---|---|
| GNOME session remains installed and supported | Step 2, and the table above |
| Session launch reaches a usable desktop or fails visibly: the session exits, the user is returned to GDM, and the cause is written to the journal under the syslog identifier `gridline-session` | Steps 1 and 3 |
| A failed Gridline session is recoverable without Gridline: recovery to a working desktop | Steps 1 and 2 |
| A failed Gridline session is recoverable without Gridline: the failure is discoverable from the fallback session using only tools present on a stock Ubuntu system | Step 3, using `journalctl` |
| Gridline is removable, returning the system to a working stock Ubuntu desktop | Step 5 |

No step on this page contradicts those requirements, and no step adds a dependency the requirements do not allow. Step 4 is the only optional one, and it is marked optional precisely because `specs/desktop-session/spec.md` requires the discovery path to work with stock tools alone.
