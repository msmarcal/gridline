# Proposal

## Why

A laptop's own keys do nothing in a Gridline session. Measured against the shipped compositor configuration: volume up, volume down, mute, microphone mute, brightness up, brightness down, play, previous and next are all unbound. The file carries 42 bindings and all of them are chords on `SUPER`.

Nobody learns the volume key. It is the first key a person presses without being told, which makes this the first gap a fresh install meets. The sound itself works - pipewire, wireplumber and pipewire-pulse are active in the session - so the key is the only thing between a working audio stack and a person who can use it.

## What Changes

**This change ships implementation.**

- Binds the keys a laptop prints on itself: volume up, volume down, mute, microphone mute, brightness up, brightness down, and the media keys for play/pause, previous and next.
- Binds them to work while the screen is locked and to repeat while held. The compositor's plain `bind` does neither, and a volume key that stops working at the lock screen is a volume key half the time.
- Routes each key through a Gridline-owned program rather than naming the tool in the keymap, which is the rule the compositor configuration states about itself and keeps for the terminal and the lock. Swapping a tool is then one file, not the keymap a user is expected to copy.
- Uses what is already present where it is. Volume goes through `wpctl`, which arrives with the audio stack already running. Brightness asks logind, which exposes `SetBrightness` and needs no package, no group membership and no udev rule - the same shape as the lock, which asks logind rather than acting on the session itself. Only the media keys add a dependency.

**Out of scope.** An on-screen indicator. `swayosd` would show the volume as it changes and it is a daemon, with a unit, a lifecycle and a place in the diagnostic - the shape of a desktop shell component, which this project adds through a change of its own. The keys work without it. Whether they feel right without it is the next change's question, and this one records that the question exists.

Also out of scope: keyboard backlight, which varies too much between machines to bind without one to measure on.

## Capabilities

### New Capabilities

None.

### Modified Capabilities

- `desktop-session`: one requirement **added**, that a session responds to the hardware keys a laptop ships with. The capability already owns what a session presents and how a person reaches it; this is the part of "reaches it" that needs no instruction.

## Impact

- `packages/gridline/src/adapters/hyprland/defaults.conf`: the bindings.
- `packages/gridline/src/session/`: the program the bindings run.
- `packages/gridline/debian/control`: one dependency, for the media keys.
- `docs/user-experience.md`: the experience table has no row for hardware keys at all, which is how nine unbound keys went unlisted.
- Verification runs on the LXD VM for volume and media. **The VM has no backlight**, so brightness cannot be exercised there and is recorded as such until a machine with one is available.

## Prior measurement

Three facts settled before writing, each changing what gets built.

**Volume needs no new dependency.** `wpctl` is `/usr/bin/wpctl` from `wireplumber`, already installed and already running in the session, reached through `pipewire-audio` and `ubuntu-desktop-minimal`. The session's default sink currently reads `Volume: 0.00 [MUTED]`, which is itself a small finding: nothing in the session has ever touched the volume.

**Brightness turned out to need nothing.** The first reading was that it was a permissions problem: the archive's `brightnessctl` ships one file with no udev rule and no setuid bit, the session user is not in the `video` group, and the binary carries no reference to logind at all - so it writes `/sys/class/backlight` directly and cannot. That would have forced a group change, which is a system change outside the paths Gridline owns.

Then logind was asked instead of assumed. `org.freedesktop.login1.Session` exposes `SetBrightness`, introspected on the running session. So the brightness keys ask the same service the lock already asks, with no package, no group and no rule. The proposal above said brightness would add a dependency; measuring removed it.

**The hardware keys have never been mentioned.** Not in the experience table, not in the vision document, not in any archived change. The gap was invisible because no list had a row for it.
