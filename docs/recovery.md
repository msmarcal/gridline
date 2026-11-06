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

Installing Gridline adds its own session entry, and the compositor it depends on
adds one or two of its own. On the supported baseline the display manager ends up
offering **Gridline**, **Hyprland** and **Ubuntu**. A fourth entry exists,
`hyprland-uwsm.desktop`, and is withheld because its `TryExec` names a binary the
baseline does not install.

**Gridline hides none of them, and will not.** A session entry belongs to the
package that ships it, and the only directory with precedence over
`/usr/share/wayland-sessions` is `/usr/local/share/wayland-sessions`, which
Debian Policy §9.1.2 reserves for the local administrator and forbids any package
from writing to. Hiding another package's entry would also contradict the rule
that Gridline is additive: a compositor you have installed stays launchable.

**An administrator can, and here is the measured way.** The display manager
resolves session entries by filename across its search path and the first match
wins, so a file of the same name in the higher-precedence directory replaces the
one the package shipped. Combined with `Hidden=true`, the entry disappears from
the selector entirely:

```
sudo mkdir -p /usr/local/share/wayland-sessions
sudo tee /usr/local/share/wayland-sessions/hyprland.desktop >/dev/null <<'EOF'
[Desktop Entry]
Type=Application
Name=Hyprland
Hidden=true
EOF
sudo systemctl restart gdm3
```

Undo it by deleting that file and restarting the display manager again. Nothing
the `hyprland` package owns is touched, `dpkg --verify hyprland` stays clean, and
an upgrade of that package does not fight the override.

This works for any session entry, not only the compositor's: the filename is what
matches, so the same recipe hides any entry whose file you name. **Do not hide the
stock Ubuntu session.** It is the fallback every recovery on this page depends on,
and a machine whose only remaining entry is Gridline has no route back if Gridline
stops starting.

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
