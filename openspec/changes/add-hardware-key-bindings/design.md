# Design

## Context

See proposal.md for why. What the measurements settled, because all three mechanisms were decided by asking the machine rather than by choosing a tool.

**The compositor's plain `bind` is the wrong one for these keys.** This compositor distinguishes bind flavours by suffix letters: `e` makes a binding repeat while held, `l` makes it act while the session is locked. The shipped configuration uses plain `bind` forty times and `bindm` twice, and has never needed either letter, because every binding it has is a deliberate chord a person presses once while using the desktop. A volume key is neither: it is held, and it is pressed at the lock screen.

**Volume is already reachable.** `wpctl` is in `/usr/bin` from `wireplumber`, which arrives through `pipewire-audio` and `ubuntu-desktop-minimal` and is active in the session. Nothing needs declaring and nothing needs starting.

**Brightness was a permissions problem until logind was asked.** `brightnessctl` from the archive ships a single binary, no udev rule, no setuid bit, and - measured by reading the binary - no reference to logind at all. It writes `/sys/class/backlight` directly, which the session user cannot do, being in `ubuntu adm cdrom sudo dip lxd` and not `video`. Fixing that would mean adding a user to a group or shipping a udev rule, both of which are changes to system state outside the paths Gridline owns, and the project forbids itself both.

`org.freedesktop.login1.Session` exposes `SetBrightness` with signature `ssu`, introspected on the running session. That is the whole answer: logind holds the privilege and hands it to the active session's owner, which is exactly the arrangement the screen lock already uses for `SetLockedHint`.

**The VM has no backlight.** `/sys/class/backlight` is empty, so brightness cannot be exercised here at all. That is a limit of the verification machine and is recorded rather than worked around.

## Goals / Non-Goals

**Goals:**

- The keys a laptop prints on itself do what they say, with no configuration.
- No new privilege, no group change, no file outside Gridline's paths.
- The keymap stays free of tool names, as it already is for the terminal and the lock.

**Non-Goals:**

- An on-screen indicator. That is a daemon and belongs to a change that can give it a unit, a lifecycle and a line in the diagnostic.
- Keyboard backlight, which differs too much between machines to bind without one in front of us.
- Making the audio or brightness *state* visible anywhere. The bar has a battery and a network module and no volume module; adding one is a separate question about the bar.

## Decisions

### One program, one role per key

`SUPER D` and `SUPER SHIFT D` already name `gridline-menu` with a role argument, and the desktop shell components name `gridline-shell` the same way. The hardware keys follow it: one program, a role per action, named in the keymap.

The alternative - writing `wpctl set-volume @DEFAULT_AUDIO_SINK@ 5%+` into `defaults.conf` - is fewer files and breaks the rule that file states about itself. It would also put three different tools' command lines into the one file this project most expects a user to copy and edit, so a user who changed audio stacks would be editing keybindings to do it.

### Brightness asks logind, and that is not a convenience

Reading the current and maximum brightness needs no privilege: `/sys/class/backlight/*/brightness` and `max_brightness` are world-readable. Writing needs one, and logind has it.

So the program reads the two values, computes the next one, and calls `SetBrightness` with the subsystem, the device name and the new value. Nothing is installed, nothing joins a group, and the privilege stays where it was.

**This also decides what happens on a machine with no backlight**, which is every machine this project can currently test on: there is no device to name, the program says so once and exits without error, and the key does nothing rather than failing loudly at someone who pressed it by accident.

### Mute restores what was there

`wpctl set-mute @DEFAULT_AUDIO_SINK@ toggle` is the whole implementation, and the reason to name it as a decision is the alternative somebody always writes: setting the volume to zero and back to a remembered number. That version loses the user's level the first time anything else changes it, and the requirement says the restored level is theirs rather than one the session chose.

### Media keys address the interface, not the player

`playerctl` speaks MPRIS, which is the interface every player on a Linux desktop already advertises, and the session's bus already has one registered. This is the single dependency the change adds. The alternative - calling the MPRIS methods over `busctl` directly - avoids the package but requires the program to find the player's bus name itself, which is the part `playerctl` exists to do and the part that would need maintaining.

### Working while locked is a decision, not a detail

The `l` flavour makes these keys act at the lock screen. That is deliberate and worth defending: silencing a machine that has started making noise, or dimming a screen, is not a privileged action and needs no authentication. The lock exists to stop someone using the session, and neither key gives access to anything in it.

The keys that are **not** given `l` are the media ones. Pausing is harmless; skipping tracks on a locked machine is someone else deciding what a stranger's music does, and the argument for it is weaker than the argument for volume.

## Risks / Trade-offs

- **Brightness cannot be verified on any machine this project has.** → It is implemented against the interface logind documents and introspection confirms, and recorded as unexercised with the reason. The first person with a laptop closes it in one keypress.
- **`SetBrightness` requires the calling session to be the active one.** → Which it is, when somebody is pressing a key on that machine's keyboard. A key pressed into an inactive session is a case that does not arise.
- **`l` bindings act at the lock screen, which is a surface an attacker can reach.** → Measured against what they can do: change a volume and a backlight. Neither reveals anything, neither unlocks anything, and both are reachable by the physical buttons on most machines regardless of what software does.

## Open Questions

- Whether the bar should show the volume, now that something changes it. That is a question about the bar and its modules rather than about the keys.
- Whether `XF86AudioMicMute` has a sensible target when the session has several input devices. `wpctl` has a default source, and whether the default is the one a user means on a machine with a webcam and a headset is not something this VM can answer.
