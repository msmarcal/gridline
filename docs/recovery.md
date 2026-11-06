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

If `gridline-tools` is installed, `gridline doctor` reports capability gaps in a readable form, and it runs without a Gridline session running:

```
gridline doctor
```

This step is a convenience, not a dependency. Everything needed to diagnose a failed session is available from step 3 with stock tools, because a diagnostic command that only runs inside a working session would be useless exactly when it is needed.

### 5. If you want Gridline gone

Removing the Gridline packages returns the system to a stock Ubuntu desktop. No session entry remains, GNOME works, and nothing Gridline owned is left behind apart from configuration files you edited yourself.

You can keep using GNOME indefinitely with Gridline still installed. An unselected session costs nothing.

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

**That session is not a Gridline session.** It reads Gridline's compositor
defaults, because they sit at the path any session of that compositor reads, so
it arrives with Gridline's keybindings and without Gridline's locker, polkit
agent or portal mapping. The lock key in particular does nothing. Choose it
knowing that.

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
